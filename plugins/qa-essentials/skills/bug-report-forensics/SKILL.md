---
name: bug-report-forensics
description: Turn a vague observation ("it broke", "login is weird") into a bug report a developer can act on without asking a single follow-up question. Minimizes the reproduction, separates severity from priority with written justification, and collects stack-appropriate evidence. Use when filing a bug, cleaning up an existing ticket, triaging a failing test, or when someone describes something that went wrong.
argument-hint: "[bug description, ticket id, or failure output]"
---

# Bug Report Forensics

A bug report is not a message. It is an **argument** that (a) something is wrong,
(b) here is proof, (c) here is the cheapest path to see it again, and (d) here is
what it costs. Reports that skip any of the four generate a round trip, and a
round trip costs more than the report did.

Your job is to produce a report where **the developer's first action is to fix,
not to ask.**

## Before writing anything: three gates

Run these in order. Failing a gate changes the output, so do not skip ahead.

**Gate 1 — Is it a bug?**
Something can be unexpected without being wrong. Ask which oracle is violated:
a written requirement, a documented API contract, a consistent pattern elsewhere
in the product, a standard (HTTP, WCAG, ISO date), a user's reasonable expectation,
or the product's own prior behaviour. If you cannot name an oracle, this is a
*question* or a *change request*, and it must be labelled as one. Say so plainly
rather than filing it as a defect.

**Gate 2 — Is it already known?**
Search the tracker for the error string, the endpoint, the screen name, and the
symptom in the reporter's own words. A duplicate filed as new fragments the
history of a defect and delays the fix. If a related ticket exists, decide: is
this the same defect (comment there), or a different defect with a shared root
cause (file new, link with "related to")?

**Gate 3 — Is the reproduction minimal?**
See the next section. An unminimized repro is the single most common reason a
report bounces back.

## Minimize the reproduction

Do not report the path you happened to take. Report the shortest path that still
fails. Bisect along four independent axes, one at a time, keeping the others fixed:

| Axis | Question | Method |
|---|---|---|
| **Steps** | Which steps are load-bearing? | Remove the last step before the failure. Still fails? Remove the next. Restore any step whose removal makes it pass. |
| **Data** | Which field actually matters? | Replace each input with a trivial known-good value. The one that makes it pass is the trigger. Halve long strings/collections until the failure disappears. |
| **Environment** | Is it universal or conditional? | Vary one at a time: browser, OS, locale, timezone, screen size, account role, tenant, feature flag, data volume. |
| **Time** | When did it start? | Bisect over builds/commits. If a build boundary exists, name it — this is the single most valuable line in the report. |

Then state the result in the exact shape below. Numbers, one action per step,
and the state each step leaves the system in when it is not obvious.

```
Preconditions:
  - Build: 4.12.3-rc2 (commit a1b2c3d)
  - Account: standard user, no admin role, tenant "acme-eu"
  - Locale: tr-TR, timezone Europe/Istanbul
  - Feature flag NEW_CHECKOUT: off

Steps:
  1. POST /api/v2/orders with quantity = 0
  2. Read the response

Expected: 400 Bad Request, body { "error": "quantity must be >= 1" }
Actual:   201 Created, order persisted with quantity 0 and total -12.50

Frequency: 5/5 attempts
Narrowest condition: only when quantity = 0. Negative values are correctly rejected;
  the guard is `quantity < 0` where it should be `quantity <= 0`.
```

That last line — the *narrowest condition* — is what separates a forensic report
from a description. Always try to produce it.

## Severity and priority are different fields

They are decided by different people for different reasons, and collapsing them
is how a cosmetic typo ends up blocking a release while a silent data-corruption
bug sits in the backlog.

- **Severity** = technical damage if it occurs. **You own this.** It is an
  observation, and it must be justified in one sentence.
- **Priority** = when it gets fixed, given business context. **The product owner
  owns this.** You may *recommend* it; never assert it.

A high-severity/low-priority bug is legitimate (catastrophic but only on a
deprecated flow used by two internal accounts). So is low-severity/high-priority
(a typo in the company name on the landing page, an hour before a launch).

See `SEVERITY.md` for the full rubric, the impact-vs-reach decision table, and
the list of severity-inflation anti-patterns.

## Evidence, by stack

Attach proof, not prose. Missing evidence is the second most common bounce reason.

**Always**
- Exact build/version/commit, environment name, timestamp with timezone
- The precise error text, copied — never paraphrased, never screenshotted if it is text
- Frequency: 5/5, or 2/10 (a flaky bug is a *different* and usually more urgent bug)

**Java / Spring**
- Full stack trace including every `Caused by:` chain — truncating it discards the cause
- Correlation/trace id if the service emits one; the matching log window either side
- For a hang or deadlock: a thread dump (`jstack <pid>`), not a screenshot of a spinner
- For a suspected leak: heap usage over time, plus the GC log window

**.NET**
- Full exception with `InnerException` chain and stack trace
- Activity/TraceId from the distributed trace
- For async issues: whether the failure survives `ConfigureAwait(false)`, and the
  synchronization context involved

**Web UI**
- HAR file for the failing request, plus the browser console (errors *and* warnings)
- A short recording only if the bug is temporal (animation, race, focus); otherwise
  screenshots with the relevant region marked
- The DOM snapshot of the failing element when the issue is state, not rendering

**API / integration**
- Full request: method, URL, headers (redact secrets — replace the value, keep the key), body
- Full response: status, headers, body
- A ready-to-run `curl` reproducing it against a non-production environment

**Data**
- The minimal row/document that triggers it, anonymized
- The schema version or migration state

## Write the report

Use this structure. Keep the title to one line that names the *behaviour*, not
the feeling — a title someone can recognize in a list of two hundred.

```
Title: [component] <observable wrong behaviour> when <narrowest condition>

Summary
  Two sentences. What is wrong, and what it costs.

Preconditions / Steps / Expected / Actual / Frequency
  (as minimized above)

Evidence
  (attachments, logs, trace ids)

Severity: <level> — <one-sentence justification tied to the rubric>
Suggested priority: <level> — <one sentence, framed as a recommendation>

Scope
  What else is likely affected, and what you checked and found unaffected.
  "Also reproduces on PUT /api/v2/orders. Does NOT reproduce on the v1 endpoint."

Regression
  First bad build, last good build, and the suspected change if identified.

Workaround
  For support and for the users waiting on the fix. "None found" is a valid and
  useful answer — it tells the triager something.
```

## Anti-patterns to reject

If the input you were given contains any of these, fix it rather than passing it through.

- **"Doesn't work" / "is broken" / "acts weird"** — names no observable behaviour.
- **Expected behaviour left implicit.** "The total is wrong" without saying what it
  should be forces the developer to guess the requirement.
- **Screenshot of text.** Unsearchable, uncopyable, and it hides the rest of the log.
- **Multiple defects in one ticket.** They will get different owners, different fixes
  and different release trains. Split them, and cross-link.
- **Severity asserted with no justification.** "Critical" with no sentence after it is
  a preference, not an assessment.
- **A repro that starts at "log in".** Start at the narrowest precondition that matters.
- **"Sometimes"** — replace with a measured frequency, or say the count is unknown and
  state how many attempts were made.

## Output

Produce the finished report as copy-pasteable Markdown. Then, separately and briefly:

1. State which oracle Gate 1 identified.
2. List what you could not determine and what the reporter must supply — as specific
   questions, each with why it changes the diagnosis, not a generic "more info needed".

Companion files: `SEVERITY.md` (rubric and decision table), `EXAMPLES.md`
(three before/after rewrites).

See also: `exploratory-charter`, where most reports worth writing originate, and
`risk-based-test-plan` when a defect suggests an area was under-tested rather than
unlucky. Neither is required to use this skill.
