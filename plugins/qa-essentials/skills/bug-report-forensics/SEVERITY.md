# Severity rubric

Severity answers one question: **if this defect occurs in production, how much
damage does it do?** It does not ask how likely it is, how many users hit it, or
how close the release is — those belong to priority.

## Levels

### S1 — Critical
Irreversible or unbounded damage. No workaround.

- Data loss or silent data corruption (the silent case is worse: nobody knows to restore)
- Money moves incorrectly, or a financial figure shown to a customer is wrong
- Authentication or authorization bypass; one tenant can read another tenant's data
- Complete outage of a primary flow for all users
- A regulatory breach that is reportable by law (PII exposure, audit trail gap)

Test: *if this shipped and nobody noticed for a week, could the damage still be undone?*
If no, it is S1.

### S2 — Major
A primary flow is blocked or produces a wrong result, but the damage is bounded
and reversible.

- A core use case cannot be completed by an identifiable group of users
- Wrong output that the user can see and therefore will not act on blindly
- A workaround exists but is unreasonable to expect (contact support, use a different browser)
- Persistent performance failure: an operation that must be interactive takes minutes

### S3 — Moderate
The flow completes. Something along the way is wrong, degraded, or confusing.

- A secondary feature is broken; the primary path is intact
- Wrong behaviour with a reasonable workaround the user can find unaided
- Validation accepts bad input but the system handles it safely downstream
- Noticeable but non-blocking performance degradation

### S4 — Minor
Cosmetic or informational. Nothing behaves incorrectly.

- Layout, spacing, alignment, truncation that does not hide information
- Wording, capitalization, an untranslated string in a non-critical surface
- Console warnings with no user-visible effect

## Decision table

Work left to right. The first row that matches sets the floor.

| Condition | Minimum severity |
|---|---|
| Data lost or corrupted without an error being shown | S1 |
| A user can access data belonging to another user or tenant | S1 |
| A monetary amount is computed or displayed incorrectly | S1 |
| The system records an audit/compliance event incorrectly or not at all | S1 |
| A primary flow cannot be completed at all | S2 |
| Wrong result shown to the user in a primary flow | S2 |
| Only workaround requires leaving the product | S2 |
| A secondary feature is unusable | S3 |
| Correct behaviour, poor experience | S3 |
| Nothing behaves incorrectly | S4 |

## Silence is an aggravating factor

Raise severity by one level when the failure is **silent** — no error, no log, no
alert. A visible failure is contained by the user noticing it. A silent one
compounds until something else surfaces it, and by then the blast radius has grown
and the trail has gone cold.

A wrong total shown on screen is S2. The same wrong total written to the ledger
with no error is S1.

## Frequency belongs in the report, not the severity

An intermittent S1 is still S1. Record frequency as its own line (`2/10 attempts`)
and let priority weigh it. Downgrading severity because "it only happens sometimes"
hides catastrophic-but-rare defects — exactly the ones that survive to production.

If the intermittency itself is unexplained, that is a *second* finding worth
recording: an unpredictable system is a testability defect in its own right.

## Recommending a priority

You do not set priority, but a recommendation with reasoning is far more useful
than a bare number. Give the product owner the four inputs:

1. **Severity** — the damage, from above.
2. **Reach** — how many users, which segment, how often they hit the trigger.
3. **Detectability** — will the user notice? Will monitoring notice? If neither, argue up.
4. **Cost of delay** — is there a date (launch, audit, contract, regulation) attached?

Then write one sentence:

> Suggested priority: High — S2 severity, but it hits every checkout using a saved
> card (roughly 40% of orders by volume) and the failure is silent from the user's
> side, so support will not receive tickets that map to the cause.

## Severity inflation — do not do these

- **Marking everything S1 to get attention.** It works once. After that the field
  carries no information and the whole team stops reading it.
- **Confusing "I found it" with "it matters".** Effort spent finding a defect has
  no bearing on its severity.
- **Rating by how loudly it fails.** A dramatic stack trace on an admin-only debug
  screen is not more severe than a quietly wrong tax calculation.
- **Rating by how easy the fix looks.** Fix cost is an input to priority, never to severity.
- **Deferring the rating.** A bug filed without severity gets triaged by whoever
  shouts loudest. Rate it, justify it in one sentence, and let it be argued down.
