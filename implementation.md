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
  status        enum('assignment_pending','assigned','closed')   -- default 'assignment_pending'
  assigned_agent_id  uuid fk -> Agent, nullable
  created_at    timestamptz    -- also the start of "pending age" while status = 'assignment_pending'
  assigned_at   timestamptz nullable

AssignmentDecision
  id            uuid pk
  ticket_id     uuid fk -> Ticket
  created_at    timestamptz
  trigger       enum('request','event_load_drop','reconciliation')
  outcome       enum('assigned','no_eligible_agent')
  assigned_agent_id  uuid nullable
  candidates    jsonb    -- [{agent_id, included: bool, reason, current_load, projected_load, last_assigned_at}]
                          -- last_assigned_at is each candidate's value *before* this decision's write, so the
                          -- log can actually explain a recency tie-break instead of showing post-mutation state
```

`effort → weight`: `small=1, medium=2, huge=4` — a plain in-code constant, not a stored column. It was originally denormalized onto `Ticket` "for auditability," but that's exactly the second-source-of-truth problem avoided for active load below: a stored `weight` can silently drift from `effort`. `AssignmentDecision.candidates` already snapshots the weight used in each decision, which covers the audit need without a column that can go stale.

An agent's **active load** is not a stored column — it's computed as `COALESCE(SUM(ticket.weight_for(effort)), 0) WHERE ticket.assigned_agent_id = agent.id AND ticket.status = 'assigned'`. The `COALESCE` matters: `SUM` over zero matching rows returns `NULL` in SQL, not `0` — without it, an agent with no active tickets (the normal starting state for every agent) would compute `current_load = NULL`, `projected_load = NULL`, and `NULL <= max_active_load` is never true, so a fresh agent could never receive a ticket.

**Data integrity constraints** (enforced at the DB level, not just application logic): `Ticket.assigned_agent_id`, when set, must reference an `Agent` with the same `company_id` as the ticket (a company-scoped FK, or a check constraint joining through `Agent`) — two independent FKs alone don't prevent an agent from company B ending up on a company A ticket. A `CHECK` constraint also enforces the valid state/owner combinations: `status='assigned'` requires `assigned_agent_id IS NOT NULL`; `status IN ('assignment_pending','closed')` requires it `IS NULL` (a closed ticket's owner, if needed for history, lives in `AssignmentDecision`, not on the ticket row).

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
  "gaps": [{ "start": "2026-11-02T01:00:00Z", "end": "2026-11-02T09:00:00Z" }] }
```
Endpoints are offset-bearing timestamps within the anchored reference week (see Coverage gap detection below), not day-of-week + local-time pairs — necessary to stay unambiguous across a DST fall-back's repeated hour.

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
    weight = WEIGHT_BY_EFFORT[ticket.effort]     -- plain constant lookup, not a stored column
    agents = SELECT * FROM Agent WHERE company_id = ticket.company_id
    candidates = []
    for agent in agents:
      available = isWithinSchedule(agent, now)                -- see Availability check
      current_load = computeActiveLoad(agent.id)               -- COALESCE'd, see Data model
      projected_load = current_load + weight
      included = available AND projected_load <= company.max_active_load
      -- snapshot last_assigned_at as a plain value here, before any write below can change it
      candidates.push({agent_id: agent.id, available, current_load, projected_load, included,
                        last_assigned_at: agent.last_assigned_at,
                        reason: !available ? 'unavailable' : !included ? 'at_capacity' : 'eligible'})

    eligible = candidates.filter(c => c.included)
    if eligible.empty:
      UPDATE Ticket SET status='assignment_pending' WHERE id=ticketId   -- explicit write, not just a response value
      INSERT AssignmentDecision(outcome='no_eligible_agent', candidates, trigger)
      COMMIT
      return { status: 'assignment_pending' }

    winner = eligible.sort_by(current_load ASC, last_assigned_at ASC NULLS FIRST, agent_id ASC)[0]
    UPDATE Ticket SET status='assigned', assigned_agent_id=winner.agent_id, assigned_at=now
    UPDATE Agent SET last_assigned_at=now WHERE id=winner.agent_id
    INSERT AssignmentDecision(outcome='assigned', assigned_agent_id=winner.agent_id, candidates, trigger)
  COMMIT
  return { status: 'assigned', agent_id: winner.agent_id }
```

Tickets are created with `status='assignment_pending'` by default (there is no separate "unattempted" state) — that's what makes `created_at` a valid start point for "pending age," and what lets a freshly seeded ticket already show up in `GET /tickets/pending` and get picked up by reconciliation even before its first `assign` call.

### Availability check

`isWithinSchedule(agent, instantUtc)`: convert `instantUtc` into `agent.timezone` (via a timezone library — `luxon` or `date-fns-tz`, not manual offset math, so DST is handled automatically), get the resulting local day-of-week and time-of-day, and check whether it falls within any of the agent's `AvailabilityBlock` rows for that day, using half-open `[start_time, end_time)` semantics.

**Overnight blocks are rejected, not silently wrong.** A block like Sunday `22:00–02:00` (wrapping past midnight) can't be matched correctly by a same-day check — Monday `01:00` would never match a row stored under `day_of_week=Sunday`. Rather than build wraparound matching logic, `PUT .../availability` validates `end_time > start_time` and rejects any block that would cross midnight; the UI and API require an overnight shift to be entered as two rows (e.g. Sun `22:00–23:59` + Mon `00:00–02:00`). Simpler to implement correctly than wraparound matching, and still expressible from the UI.

### Concurrency

Two tickets can arrive for the same company at nearly the same instant. To guarantee no double-assignment and no stale-load reads (the guarantee stated in `prd.md`), `attemptAssign` takes a **Postgres advisory lock keyed by `company_id`** for the duration of the transaction — not just a row lock on the ticket. A row lock on the ticket alone would only prevent two requests for the *same* ticket from racing; it wouldn't stop two different tickets in the same company from both reading Alice's load as 2 and both assigning to her. The company-level lock serializes every assignment attempt for that company, so load reads are always fresh relative to the previous request's writes.

Trade-off: assignment for one company can't run in parallel — acceptable at trial scale (single company, low request volume); a production version at higher scale would need finer-grained locking (e.g. per-agent optimistic concurrency with retry) instead of a whole-company lock.

### Coverage gap detection

A fixed slot count doesn't actually work here: a real calendar week isn't always `7 × 24 × 4 = 672` fifteen-minute slots once DST is involved — a spring-forward week has one fewer hour (668 slots), a fall-back week has one extra (676), and a `{day, start, end}` result expressed only in local day+time can't distinguish fall-back's two occurrences of the repeated hour (1:30am happens twice, at two different real instants, and could have different coverage in each).

So coverage is computed as **true interval union on real instants**, anchored to a concrete reference week, not an abstract 672-slot grid:

1. Pick a concrete reference week (e.g. the current real week) and express it as an actual UTC instant range — not a repeating abstract pattern.
2. For each agent, convert each `AvailabilityBlock` into the one or more real UTC instant-intervals it produces within that reference week, using the timezone library against real dates (so DST shifts are handled correctly, including which of the two fall-back occurrences a block covers).
3. Merge all agents' intervals with a standard interval-union sweep (sort by start, merge overlapping/adjacent).
4. Any gap between merged intervals, within the reference week's instant range, is a coverage gap.
5. Return gaps as **offset-bearing timestamps** (e.g. `2026-11-01T01:30:00-04:00`), not ambiguous local day+time pairs — this is what makes the repeated fall-back hour unambiguous.

Availability inputs (`start_time`/`end_time`) accept arbitrary minute values — there's no 15-minute alignment requirement to enforce, since interval math has no fixed granularity to violate. Recomputed on request (`GET /coverage`) rather than maintained incrementally — cheap enough at this scale (agents × blocks is small) and avoids a second consistency problem.

### Event triggers

- **On ticket close**: the `PATCH .../tickets/:ticketId/close` handler, after committing the status change, synchronously loops over that company's `assignment_pending` tickets in `created_at` order and calls `attemptAssign` on each, trigger=`event_load_drop`. This is the only path that reduces an agent's active load, so it's the only place this hook needs to live.
- **Reconciliation** (a single sweep, not two separate loops): every `RECONCILIATION_INTERVAL_MINUTES` (a fixed constant, `1`), for every company with at least one `assignment_pending` ticket, run the same oldest-first pass regardless of cause, trigger=`reconciliation`. A separate "did a shift just start" polling check was considered and dropped — a 1-minute reconciliation sweep already covers shift starts, manual cap/schedule edits, and anything else that changes eligibility, without a second piece of crossing-detection machinery to maintain. Implemented as an in-process interval timer for the trial; a production deployment would use a durable job scheduler so retries survive process restarts and work across multiple instances.

**"Oldest-first" is best-effort, not a hard guarantee.** The retry pass processes a company's pending tickets in `created_at` order, but a *direct* `POST .../assign` call on a specific newer ticket can acquire the company lock and consume the one available slot before a reconciliation/event pass reaches an older pending ticket — the company lock serializes individual transactions, not a whole batch pass against a snapshot. Enforcing strict ordering under concurrent direct retries would require holding the lock across the entire pending list, which is more contention than this trial's scope needs. Documented here as an accepted limitation rather than silently assumed away.

The reconciliation interval is the only one of these values — fixed constant, not lead-configurable, since it's an internal safety mechanism, not something the brief asks the product to expose.

## UI flow

1. **Availability management** (`/companies/:id/agents`): list of agents → click into an agent → weekly schedule editor (a day × time-block list, add/remove rows) with a timezone dropdown (IANA list). Saves via `PUT .../availability`. Same screen (or a small settings panel) exposes the company's capacity cap via `GET`/`PUT .../config`, since the PRD calls it configurable.
2. **Team coverage** (`/companies/:id/coverage`): a horizontal bar per agent plus a combined "Team" bar, all on one reference timezone (selectable, default UTC); gaps rendered as a highlighted break with the day/time range labeled underneath. This is the "lead sees when availability doesn't cover all needed times" requirement made concrete.
3. **Pending tickets** (`/companies/:id/pending`): table of `assignment_pending` tickets, sorted longest-pending first, each row showing effort, time pending, and the latest decision's exclusion reasons. This is a **drill-down/audit view, not proactive alerting** — worth being explicit about that distinction: a table the lead has to remember to check doesn't fully solve the PRD's original "lead watches the queue" problem on its own, it just makes checking fast and explainable once they do. A true push-notification/escalation mechanism is out of scope (no third-party integrations, per the brief) and already named as future work.
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
| Multiple pending tickets, one freed slot | Oldest-first in the retry pass; best-effort under concurrent direct retries (see Event triggers) |
| Agent with no availability blocks | Always excluded as unavailable — no special-case needed |
| Ticket already closed, assign called anyway | Defensive rejection (shouldn't occur via normal flow) |
| `POST .../companies/A/tickets/:id/assign` where ticket actually belongs to company B | 404 — the ticket lookup filters on `company_id = companyId`, not just ticket id, so a mismatched pair never resolves |
| Agent's first-ever ticket (zero prior active load) | `COALESCE(SUM(...), 0)` — a `NULL` load would otherwise wrongly exclude every agent with no active tickets |
| Overnight/wraparound availability block (e.g. Sun 22:00–02:00) | Rejected at `PUT .../availability`; must be entered as two same-day blocks |
| DST fall-back's repeated hour | Interval union on real instants with offset-bearing endpoints disambiguates the two occurrences; a fixed slot count cannot |
| Recency tie-break needs explaining | `AssignmentDecision.candidates` snapshots each candidate's pre-decision `last_assigned_at`, captured before the winner's is overwritten |

## Test plan

**Unit**
- `isWithinSchedule` across a DST boundary (block that's 9am–5pm local before and after a spring-forward/fall-back date).
- `isWithinSchedule` rejects/never matches an overnight block; a shift split into two same-day rows matches correctly across midnight.
- Capacity cap: agent at `cap - weight` is included; agent at `cap - weight + 1` is excluded.
- **An agent with zero active tickets is correctly eligible** — `computeActiveLoad` returns `0`, not `NULL`, and is included in capacity comparisons.
- Tie-break order: equal load → `last_assigned_at` (including null-vs-set) → `agent.id`; `AssignmentDecision.candidates` for that case contains each candidate's pre-decision `last_assigned_at`, not the post-write value.
- Coverage gap detection: a schedule with a known gap returns exactly that gap, as offset-bearing timestamps; a fully-covered schedule returns none.
- Coverage gap detection across a DST spring-forward and fall-back week, including that the two occurrences of fall-back's repeated hour are distinguishable in the result.

**Integration**
- `assign(ticket)` twice → same agent both times, load incremented once, exactly one `AssignmentDecision` row.
- Zero eligible agents → ticket's `status` is actually persisted as `assignment_pending` (not just returned in the response), decision logged with a reason per excluded candidate, and the ticket subsequently appears in `GET /tickets/pending`.
- A freshly seeded ticket (never had `assign` called) already appears in `GET /tickets/pending` and gets picked up by the next reconciliation tick.
- Two concurrent requests for the *same* ticket → one assignment, one decision row (no duplicate).
- Two concurrent requests for *different* tickets in the same company, contending for one eligible agent → exactly one of the two tickets gets that agent; outcome is deterministic under the company lock.
- Cross-company guard: `assign` on a ticket/company-id pair that don't match returns 404 and touches no data.
- Closing a ticket that frees capacity → a pending ticket for that company resolves without a new external request (event trigger).
- A pending ticket with no qualifying event for 2+ reconciliation intervals still gets re-checked each tick (safety net fires even without an event) — including recovery after a manual cap increase or schedule edit, which no event hook watches for.
- Multiple pending tickets, one slot frees via the retry pass → oldest one is assigned; a concurrent direct retry on a newer ticket is allowed to win the race instead (documented best-effort behavior, not a bug).
- Sustained zero capacity: ticket stays `assignment_pending` across many reconciliation ticks with no guaranteed resolution — asserts the *absence* of a false "eventually assigned" guarantee.
- At least one UI/E2E flow: edit an agent's availability → coverage view reflects it and any gap it closes/opens → a pending ticket for that slot resolves and its explanation is visible.
