# Product Requirements: Support Ticket Assignment

## The problem

Support teams currently rely on a team lead manually watching the incoming ticket queue and assigning each new ticket to whoever they know is available. This breaks down as the team grows:

- Tickets that arrive while the lead is offline sit unassigned for hours.
- The lead becomes a bottleneck, spending their day triaging instead of doing their own work.
- Work isn't shared evenly — some agents get buried while others sit idle.

We're building a system that automatically assigns each new ticket to the right person on the right team, respecting who is actually available at that moment and spreading work fairly, without requiring a human to watch the queue.

## Target users

- **Team lead** — sets up and maintains their team's availability, and needs to trust the assignment logic enough to stop manually triaging. Needs visibility into whether the team's schedule actually covers the hours tickets arrive in, and needs to be able to understand why any given ticket landed with a specific agent.
- **Support agent** — the person tickets get assigned to. Not a direct user of the UI in this scope, but their workload (active ticket count/weight) is the primary input the system reasons about.
- **Calling system / API consumer** — whatever creates tickets (out of scope to build) calls the assignment API with a `company_id` and `ticket_id` and expects back a single agent to own it.

## Scope

**In scope**

- UI for a company to define and maintain agent availability: recurring weekly schedule blocks (day of week + start/end time) per agent, each tagged with that agent's timezone.
- Assignment API: given `company_id` + `ticket_id`, returns the agent who should own the ticket.
- Fair, capacity-aware assignment that accounts for agents' current active workload, weighted by ticket effort (small/medium/huge), not raw ticket count.
- A view for the team lead to see whether the team's combined availability covers all hours that need coverage (coverage gap visibility).
- A logged, human-readable reason for every assignment decision.
- Automatic re-attempt of `assignment_pending` tickets: when an agent's active load drops (a ticket closes) or their schedule window opens (a shift starts), the system re-runs the assignment check for that company's pending tickets — no manual re-trigger or lead action required for the common case.

**Out of scope** (per brief, plus decisions made during design)

- Login, auth, roles/permissions, billing, account management.
- Mobile support.
- Holiday calendars and one-off availability overrides (a single agent being out sick/on vacation for a day).
- Third-party integrations (PagerDuty, Opsgenie, etc.).
- Creating companies, agents, or tickets — these are assumed to already exist (via seed data / fixtures).
- Ticket urgency/priority as a routing factor. Considered during design and explicitly dropped: any time-windowed or delayed assignment scheme conflicts with the "never leave a ticket unowned" requirement, and priority-based routing added complexity disproportionate to the trial's scope. Every ticket — regardless of how urgent it might be — is assigned instantly using the same logic.
- Reassignment if an agent goes offline mid-ticket, or if a better-fit agent becomes available after assignment.
- Auto-classification of ticket effort from ticket content (NLP/keyword rules).

## Core behavior

**Availability** is defined per agent as a set of recurring weekly time blocks in the agent's own timezone (IANA identifier, DST-aware). A ticket's arrival instant is checked against each agent's blocks, converted into that agent's local time, to determine who is "available right now."

**Fairness** is capacity-aware, not purely round-robin. Each ticket carries an effort level (small/medium/huge, set at creation) mapped to a numeric weight (e.g. 1/2/4). An agent's "active load" is the sum of the weights of their currently open tickets — not a raw count — so a person holding one huge ticket isn't treated as less busy than someone holding three small ones. Concretely: **fairness means minimizing the difference in current weighted active load among eligible agents at assignment time** — it's a live snapshot comparison, not a historical quota or an equal-count guarantee.

**Assignment algorithm** (runs instantly, per ticket, at creation time):

1. Filter the company's agents to those available right now, per their recurring schedule.
2. Drop any agent whose **projected** load — current active load plus the incoming ticket's weight — would exceed the max active-load cap. This is a hard cap, not a stop-after threshold: an agent at 9/10 is excluded from a "huge" (weight 4) ticket, because 9 + 4 = 13 > 10, even though 9 alone is under the cap. The cap is a single **company-wide** configurable threshold, expressed in the same units as ticket weights — not per-agent or role-based, to keep the trial's model simple.
3. Among the remainder, pick the agent with the lowest **current** active weighted load — ranking uses current load, not projected load, so fairness reflects who's least busy right now; the projected-load check in step 2 is purely a safety bound, not a ranking signal. (E.g. cap=10, incoming ticket weight=4: Alice at current load 2 and Bob at current load 5 are both eligible since their projected loads, 6 and 9, are under the cap — Alice wins on lower *current* load.)
4. Among agents tied at that minimum load (e.g. Alice=2, Bob=2, both below Charlie=5), tie-break by least-recently-assigned — the tie-break only ever compares agents already tied on load, never the full candidate pool.
5. Final tie-break by agent ID, so the outcome is always deterministic and reproducible.

**Idempotency.** Assignment is idempotent per `company_id` + `ticket_id`. If a ticket already has an owner, a repeat assignment request for that same ticket returns the existing owner without re-running the algorithm — it must not select a different agent, modify any agent's active-load calculation, update `last_assigned_at`, or record a second assignment decision. This is what keeps retries (client timeouts, network failures) from silently acting as reassignment, which would otherwise contradict the "no reassignment" scope decision above. A repeat request after a `no_eligible_agent` result is different — since no assignment happened, it's legitimate to re-run the algorithm against current state.

**Concurrency.** Two tickets can arrive at nearly the same instant. If both assignment requests read agent load independently before either write lands, they can both see the same "Alice is least loaded" snapshot and both route to Alice — silently breaking the fairness guarantee. The assignment decision and the resulting load update are therefore performed as a single atomic/transactional operation: within one transaction, (1) load the ticket, (2) if it's already assigned, return the existing owner and stop, (3) otherwise read eligible agents and their current loads, (4) compute projected loads and select an agent, (5) write the assignment, (6) record the assignment decision log, (7) commit. Folding the idempotency check into the same transaction as the load read means a second concurrent request always observes either "already assigned" or the first request's updated load — never a stale snapshot. This is the simplest correctness guarantee the trial needs — full distributed-systems-grade concurrency handling is out of scope.

Every decision is logged with the reason (who was considered, who was excluded and why, why the winner won), satisfying the "lead should understand why" requirement and making the logic testable.

**Coverage gaps vs. assignment-time capacity.** These are two different problems and the design treats them separately:

- A **coverage gap** is a recurring period in the team's weekly schedule where no agent is ever scheduled to be available (e.g. nobody covers Monday 01:00–09:00 IST, every week). It's detected by converting every agent's recurring local availability into a common reference timezone and taking the union of all their intervals — any uncovered period is a gap. This is a scheduling problem, and the fix is the team lead adjusting agent schedules. The lead-facing UI shows both each agent's individual schedule and the team's combined coverage (normalized to one timezone) so gaps are visible before they ever cause a real assignment problem.
- **Assignment-time zero eligible candidates** is different: the schedule may have full coverage, but at the moment a specific ticket arrives, every scheduled agent happens to be at or over their capacity cap. This is a capacity problem, not a coverage problem, and it's transient rather than structural.

The two failure modes (plus the normal cases) that the assignment logic must distinguish:

| Situation | Meaning | Behavior |
|---|---|---|
| No agent scheduled at all | Coverage gap | Flagged to the lead via the availability UI |
| Agents scheduled, but ticket arrives outside their hours | Unavailable | Excluded from candidates |
| Agents scheduled and available, but at/over capacity | Capacity exhaustion | Excluded from candidates; explained in the assignment log |
| Agents scheduled, available, under capacity | Normal | Assigned fairly per the algorithm above |
| Zero candidates remain after all filters | Assignment-time failure | Ticket enters `assignment_pending`; API returns an explicit `no_eligible_agent` result; reason is logged; team lead is accountable; auto-retried on the next relevant event (agent load drops or shift starts) |

**Known deviation from a stated requirement.** The brief states "a new ticket should not be left without an owner." Under normal operation this holds — every ticket is assigned immediately. But if zero eligible agents remain after the availability and capacity filters, we deliberately relax that requirement rather than violate a different one: we do not silently assign the ticket to someone unavailable or over capacity just to guarantee an owner. Instead, the ticket's state becomes `assignment_pending`: it stays unassigned, the API returns an explicit `no_eligible_agent` result, and the failure (with which candidates were considered and why each was excluded) is logged. This is a defined fallback behavior, not an undefined failure mode. Note that this case is distinct from a recurring coverage gap: a team can have continuous scheduled coverage and still temporarily hit zero eligible agents because everyone available is at their capacity cap — schedule coverage alone doesn't prevent capacity exhaustion.

**Pending assignment accountability and auto-retry.** For `assignment_pending` tickets, the team lead is the accountable party — responsible for noticing the ticket if it isn't resolved automatically, but not the working-agent owner. Resolution doesn't require the lead to act in the common case: the system automatically re-attempts assignment for a company's pending tickets whenever a relevant event occurs — an agent's active load decreases (a ticket closes) or an agent's schedule window opens (a shift starts). Each re-attempt reuses the same idempotent, transactional assignment operation from the algorithm above, which is safe to call repeatedly since it only acts while the ticket remains unassigned. If multiple pending tickets exist for a company, they're re-attempted oldest-first, so the longest-waiting ticket gets first claim on newly freed capacity. A caller explicitly retrying the same still-unassigned ticket also re-runs the check, as before. Once an eligible agent is found, that agent becomes the ticket's owner and `assignment_pending` clears. What's still not built is a designated backup/overflow agent as an alternate resolution path — event-triggered retry uses the same fairness algorithm as any other ticket, not a special-case routing rule.

## Assumptions

- Ticket effort (small/medium/huge) is provided at ticket creation (via seed data in this build); the system does not infer it.
- Demo/seed data is set up so the team's combined availability covers all hours across timezones — i.e., no genuine recurring coverage gap exists in the sample data, and the gap-detection/visibility feature would surface one if it existed. This assumption only rules out the *coverage-gap* case, not the *capacity-exhaustion* case: the seed data provides continuous schedule coverage, but zero eligible agents (everyone at their cap) can still occur depending on active ticket state at the moment a ticket arrives — that path is real and its behavior (`assignment_pending`) is defined above, not assumed away.
- "Active" ticket, for load purposes, means open/in-progress; resolved/closed tickets drop out of an agent's load immediately. This is also what bounds the case of an agent being away for several days without the system knowing (one-off absences are out of scope, below): their open tickets stay open and their load stays high, so the capacity cap naturally stops new tickets from piling onto them — it doesn't require any leave-tracking to avoid unbounded pileup.
- Assignment and ticket ownership updates are performed transactionally, so concurrent assignment requests observe a consistent workload rather than racing on a stale load snapshot.
- Whoever uses the availability UI is assumed authorized (no roles/permissions, per FAQ).
- The system runs locally; no deployment is required.

## What's simplified or stubbed, and why

- **No urgency/priority routing** — every ticket uses the same instant, weighted-load logic regardless of stated urgency, to avoid reintroducing "ticket sits unassigned" delays via a windowed-assignment scheme.
- **No one-off absences or holiday overrides** — out of scope per brief; only the recurring weekly schedule is modeled.
- **No designated backup/overflow agent** — `assignment_pending` tickets resolve automatically via event-triggered retry (on agent load dropping or a shift starting), reusing the standard fairness algorithm. What's not built is a separate routing path to a special backup/overflow agent as an alternative — that's a distinct design decision, out of scope for the trial.
- **Static effort weights** (e.g. 1/2/4) — a fixed heuristic, not calibrated against real resolution-time data.
- **No reassignment** — if an agent goes offline mid-ticket, their load isn't rebalanced; the ticket stays with them.
- **No fairness memory beyond "current active load" and "last assignment time"** — the system doesn't track cumulative volume over a longer window (e.g., a week), so it optimizes instantaneous fairness rather than long-run fairness.

