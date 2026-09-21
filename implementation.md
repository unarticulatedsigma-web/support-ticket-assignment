# Technical Design: Support Ticket Assignment

Companion to `prd.md`. That doc defines the product guarantees; this doc defines how they're built. Stack: Node.js + TypeScript, Express, PostgreSQL via Prisma, React (Vite) frontend.

## Data model

```
Company
  id            uuid pk
  name          text
  max_active_load   int      -- company-wide capacity cap, in effort-weight units (the only configurable value; reconciliation cadence is a fixed internal constant, not lead-facing)

Agent
  id            uuid pk
  company_id    uuid fk -> Company
  name          text
  timezone      text     -- IANA identifier, e.g. "Asia/Kolkata"
  last_assigned_at  timestamptz nullable   -- used for the recency tie-break

AvailabilityBlock
  id            uuid pk
  agent_id      uuid fk -> Agent
  day_of_week   int (0=Mon..6=Sun)
  start_time    time     -- local to Agent.timezone
  end_time      time     -- local to Agent.timezone

Ticket
  id            uuid pk
  company_id    uuid fk -> Company
  effort        enum('small','medium','huge')
  weight        int        -- denormalized from effort (1/2/4) for auditability
  status        enum('assignment_pending','assigned','closed')
  assigned_agent_id  uuid fk -> Agent, nullable
  created_at    timestamptz
  assigned_at   timestamptz nullable

AssignmentDecision
  id            uuid pk
  ticket_id     uuid fk -> Ticket
  created_at    timestamptz
  trigger       enum('request','event_load_drop','event_shift_start','reconciliation')
  outcome       enum('assigned','no_eligible_agent')
  assigned_agent_id  uuid nullable
  candidates    jsonb    -- [{agent_id, included: bool, reason, current_load, projected_load}]
```

`effort → weight`: `small=1, medium=2, huge=4` (constant, per `prd.md`'s stub note that this is a fixed heuristic).

An agent's **active load** is not a stored column — it's computed as `SUM(ticket.weight) WHERE ticket.assigned_agent_id = agent.id AND ticket.status = 'assigned'`. Deriving it avoids a second source of truth that could drift from actual ticket state.

## API

All endpoints scoped by `company_id`. No auth (per FAQ scope).

**`POST /companies/:companyId/tickets/:ticketId/assign`** — the core assignment API.

```
200 OK
{ "ticket_id": "...", "status": "assigned", "agent_id": "...", "decision_id": "..." }

200 OK
{ "ticket_id": "...", "status": "assignment_pending", "decision_id": "...",
  "reason": "No eligible agent: 2 unavailable, 1 at capacity" }
```
Idempotent: calling this again for an already-`assigned` ticket returns the same body without touching load or creating a new `AssignmentDecision` row. A call for a still-`assignment_pending` ticket re-runs selection against current state — that's a request-triggered retry, same as the internal event/reconciliation ones, just synchronous.

**`GET /companies/:companyId/tickets/:ticketId/assignment`** — current status + latest decision, for polling/debugging.

**`GET /companies/:companyId/tickets/pending`** — all `assignment_pending` tickets for the company, oldest-first. Backs the lead's accountability view.

**`GET /companies/:companyId/agents`** — agents with their current availability blocks and current active load (for the availability UI).

**`PUT /companies/:companyId/agents/:agentId/availability`** — replaces an agent's full set of weekly blocks in one call (simpler than per-block CRUD for this scope). Body: `{ timezone, blocks: [{day_of_week, start_time, end_time}] }`.

**`GET /companies/:companyId/coverage?tz=UTC`** — combined team coverage and detected gaps, normalized to `tz` (default UTC).

```
{ "timezone": "UTC",
  "gaps": [{ "day_of_week": 0, "start": "01:00", "end": "09:00" }] }
```

**`PATCH /companies/:companyId/tickets/:ticketId/close`** — supporting API, not a brief-requested feature in its own right. It exists to simulate ticket lifecycle changes (marking a ticket `closed`, freeing the assigned agent's load) so the capacity-based auto-retry behavior is actually demonstrable locally, since ticket creation/closure otherwise happens in an external system this trial doesn't build. This is what triggers the "on ticket close" event hook (see Assignment engine below).

```
200 OK
{ "ticket_id": "...", "status": "closed" }
```

**`GET /companies/:companyId/config`** / **`PUT /companies/:companyId/config`** — reads/updates `max_active_load`, the capacity cap the PRD calls "a company-wide configurable threshold." Without this endpoint, "configurable" would have no way to actually be configured. The reconciliation interval is deliberately **not** exposed here — it's an internal safety mechanism, not something the brief asks the lead to control, so it's a fixed constant in code instead.

```
{ "max_active_load": 10 }
```

## Assignment engine

### Core selection (used by the request path, event hooks, and reconciliation — one function, four callers)

```
function attemptAssign(companyId, ticketId, trigger):
  BEGIN TRANSACTION
    SELECT pg_advisory_xact_lock(hashtext(companyId))   -- see Concurrency below
    ticket = SELECT ... WHERE id = ticketId AND company_id = companyId FOR UPDATE
    if ticket == null:
      ROLLBACK; return 404                                    -- prevents cross-tenant access via mismatched IDs
    if ticket.status == 'assigned':
      ROLLBACK (no-op); return existing state                 -- idempotency
    if ticket.status == 'closed':
      ROLLBACK; return error                                  -- shouldn't happen; defensive

    now = current time
    agents = SELECT * FROM Agent WHERE company_id = ticket.company_id
    candidates = []
    for agent in agents:
      available = isWithinSchedule(agent, now)                -- see Availability check
      current_load = computeActiveLoad(agent.id)
      projected_load = current_load + ticket.weight
      included = available AND projected_load <= company.max_active_load
      candidates.push({agent, available, current_load, projected_load, included,
                        reason: !available ? 'unavailable' : !included ? 'at_capacity' : 'eligible'})

    eligible = candidates.filter(c => c.included)
    if eligible.empty:
      INSERT AssignmentDecision(outcome='no_eligible_agent', candidates, trigger)
      COMMIT
      return { status: 'assignment_pending' }

    winner = eligible.sort_by(current_load ASC, agent.last_assigned_at ASC NULLS FIRST, agent.id ASC)[0]
    UPDATE Ticket SET status='assigned', assigned_agent_id=winner.id, assigned_at=now
    UPDATE Agent SET last_assigned_at=now WHERE id=winner.id
    INSERT AssignmentDecision(outcome='assigned', assigned_agent_id=winner.id, candidates, trigger)
  COMMIT
  return { status: 'assigned', agent_id: winner.id }
```

### Availability check

`isWithinSchedule(agent, instantUtc)`: convert `instantUtc` into `agent.timezone` (via a timezone library — `luxon` or `date-fns-tz`, not manual offset math, so DST is handled automatically), get the resulting local day-of-week and time-of-day, and check whether it falls within any of the agent's `AvailabilityBlock` rows for that day.

### Concurrency

Two tickets can arrive for the same company at nearly the same instant. To guarantee no double-assignment and no stale-load reads (the guarantee stated in `prd.md`), `attemptAssign` takes a **Postgres advisory lock keyed by `company_id`** for the duration of the transaction — not just a row lock on the ticket. A row lock on the ticket alone would only prevent two requests for the *same* ticket from racing; it wouldn't stop two different tickets in the same company from both reading Alice's load as 2 and both assigning to her. The company-level lock serializes every assignment attempt for that company, so load reads are always fresh relative to the previous request's writes.

Trade-off: assignment for one company can't run in parallel — acceptable at trial scale (single company, low request volume); a production version at higher scale would need finer-grained locking (e.g. per-agent optimistic concurrency with retry) instead of a whole-company lock.

### Coverage gap detection

Represent a week as 672 fifteen-minute slots (`7 days × 24h × 4`). For each agent, convert each `AvailabilityBlock` into the slot indices it covers in the requested reference timezone (using the same timezone library, against the *current* real calendar week so DST offsets are correct for "now" — a schedule's mapped UTC slots can shift by an hour across a DST boundary, which is expected). OR all agents' covered slots together; any slot not covered by anyone is a gap. Merge consecutive gap slots into `{day, start, end}` ranges for the API response. Recomputed on request (`GET /coverage`) rather than maintained incrementally — cheap enough at this scale (agents × blocks is small) and avoids a second consistency problem. Fifteen minutes is arbitrary but sufficient: schedule blocks in practice are set on the hour or half-hour, so this granularity doesn't lose any real precision; a true interval-union approach would be equally valid but isn't a meaningfully simpler implementation for this scope.

### Event triggers

- **On ticket close**: the `PATCH .../tickets/:ticketId/close` handler, after committing the status change, synchronously calls `attemptAssign` for each of that company's `assignment_pending` tickets, oldest-first, trigger=`event_load_drop`. This is the only path that reduces an agent's active load, so it's the only place this hook needs to live.
- **On shift start**: a lightweight scheduler tick (every 1 minute) checks whether any agent's local time just crossed into the start of one of their blocks; if so, runs the same oldest-first pass for that company, trigger=`event_shift_start`. A precise per-agent alarm would be more exact but is unnecessary complexity for this scope — a 1-minute granularity check is an accepted simplification, worth calling out as such.
- **Reconciliation** (the safety net): every `reconciliation_interval_minutes` (default 5), for every company with at least one `assignment_pending` ticket, run the oldest-first pass regardless of whether an event fired, trigger=`reconciliation`. Implemented as an in-process interval timer for the trial; a production deployment would use a durable job scheduler so retries survive process restarts and work across multiple instances.

Both intervals (1-minute shift-check, 5-minute reconciliation) are fixed constants in code, not lead-configurable — they're internal safety mechanisms, not something the brief asks the product to expose.

## UI flow

1. **Availability management** (`/companies/:id/agents`): list of agents → click into an agent → weekly schedule editor (a day × time-block list, add/remove rows) with a timezone dropdown (IANA list). Saves via `PUT .../availability`. Same screen (or a small settings panel) exposes the company's capacity cap via `GET`/`PUT .../config`, since the PRD calls it configurable.
2. **Team coverage** (`/companies/:id/coverage`): a horizontal bar per agent plus a combined "Team" bar, all on one reference timezone (selectable, default UTC); gaps rendered as a highlighted break with the day/time range labeled underneath. This is the "lead sees when availability doesn't cover all needed times" requirement made concrete.
3. **Pending tickets** (`/companies/:id/pending`): table of `assignment_pending` tickets, oldest first, each row showing effort, time pending, and the latest decision's exclusion reasons — this is what makes the team lead's accountability role real rather than passive.
4. **Assignment explainability** (ticket detail view): shows the full `AssignmentDecision` — every candidate considered, included/excluded and why, and the winner — satisfying "understand why a ticket was assigned to a particular person."

## Edge cases

| Case | Handling |
|---|---|
| DST transition mid-week | Timezone library conversion, not manual offsets — schedule blocks shift correctly across the transition |
| Concurrent requests, same ticket | Row lock + idempotency check short-circuits the second request |
| Concurrent requests, same company, different tickets | Company-level advisory lock serializes them |
| Capacity cap boundary | Projected load (current + incoming weight) checked, not current load alone |
| Tie on current load | `last_assigned_at ASC NULLS FIRST`, then `agent.id ASC` |
| Zero eligible agents | `assignment_pending`, decision logged, lead-visible, no fabricated owner |
| Sustained zero capacity | Stays `assignment_pending` indefinitely (no guaranteed bound — matches the corrected `prd.md` wording), re-checked every reconciliation tick |
| Multiple pending tickets, one freed slot | Oldest-first ordering in the retry pass |
| Agent with no availability blocks | Always excluded as unavailable — no special-case needed |
| Ticket already closed, assign called anyway | Defensive rejection (shouldn't occur via normal flow) |
| `POST .../companies/A/tickets/:id/assign` where ticket actually belongs to company B | 404 — the ticket lookup filters on `company_id = companyId`, not just ticket id, so a mismatched pair never resolves |

## Test plan

**Unit**
- `isWithinSchedule` across a DST boundary (block that's 9am–5pm local before and after a spring-forward/fall-back date).
- Capacity cap: agent at `cap - weight` is included; agent at `cap - weight + 1` is excluded.
- Tie-break order: equal load → `last_assigned_at` (including null-vs-set) → `agent.id`.
- Coverage gap detection: a schedule with a known gap returns exactly that gap; a fully-covered schedule returns none.

**Integration**
- `assign(ticket)` twice → same agent both times, load incremented once, exactly one `AssignmentDecision` row.
- Zero eligible agents → `assignment_pending`, decision logged with a reason per excluded candidate.
- Two concurrent requests for the *same* ticket → one assignment, one decision row (no duplicate).
- Two concurrent requests for *different* tickets in the same company, contending for one eligible agent → exactly one of the two tickets gets that agent; outcome is deterministic under the company lock.
- Closing a ticket that frees capacity → a pending ticket for that company resolves without a new external request (event trigger, not reconciliation).
- A pending ticket with no qualifying event for 2+ reconciliation intervals still gets re-checked each tick (safety net fires even without an event).
- Multiple pending tickets, one slot frees → oldest one is assigned, not an arbitrary one.
