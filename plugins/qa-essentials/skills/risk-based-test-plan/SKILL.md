---
name: risk-based-test-plan
description: Build a test plan ranked by risk rather than by enumeration — for a release, a feature, a pull request, or a legacy area nobody wants to touch. Produces a risk register, test depth per area, and an explicit list of what will NOT be tested and why. Use when asked what to test, how much testing is enough, how to fit testing into a deadline, or where to focus limited QA time.
argument-hint: "[feature, release, PR, or area to plan for]"
---

# Risk-Based Test Plan

Exhaustive testing is impossible, so every test plan is a set of choices about
what to skip. A plan that lists only what it *will* cover hides those choices.
A plan that also states what it will **not** cover, and why, is a decision the
team can review, disagree with, and own together.

That out-of-scope list is the most valuable section of the document. Write it
last, but never omit it.

## 1. Establish the mission

Do not start listing test cases. Answer these first; each answer changes the plan
materially:

- **What decision does this testing inform?** Ship / don't ship? Where to spend the
  next two days? Whether a refactor was safe? Different decisions justify different
  evidence.
- **What is the time box?** Two hours and two weeks produce different plans, not the
  same plan at different speeds.
- **What changed?** Get the diff, the release notes, the migration list, the
  dependency bumps, the config changes. Testing an unchanged area is only justified
  when the change can reach it.
- **What is the cost of being wrong?** A B2B tool used by 30 internal analysts and a
  payment flow are not on the same scale, and should not get the same rigour.
- **What already covers this?** Existing unit/integration tests, monitoring, feature
  flags, canary releases, a reversible deploy. Testing has to add information the
  team does not already have. A change behind a flag with a fast rollback needs
  materially less pre-release testing than one that cannot be undone.

## 2. Decompose the change surface

List the areas that could break. Do not list features — list **surfaces**, because
that is where defects live.

- Directly changed code paths
- Callers of changed code (fan-in), and everything the changed code now calls (fan-out)
- Shared state the change touches: caches, sessions, singletons, static fields, DB rows
- Contracts crossing a boundary: API schemas, message payloads, DB schema, file formats
- Configuration and feature flags, in every combination that ships
- Data already in production that the new code must read — the migration path, not
  just the fresh-install path
- The rollback path. It is code too, and it is usually untested.

`RISK_MODEL.md` contains the full scoring model and worked examples.

## 3. Score each area

Risk = **likelihood × impact**. Score each 1–5 from observable signals, not intuition.

**Likelihood — is a defect probable here?**

| Signal | Weight |
|---|---|
| High code churn in this area over the last 90 days | ++ |
| Defect history: this area produced bugs before | +++ |
| High cyclomatic complexity, deep nesting, long methods | ++ |
| Concurrency, async, scheduling, retries, or ordering | +++ |
| New or upgraded third-party dependency | ++ |
| Author unfamiliar with this area, or the original author has left | ++ |
| Time, timezone, locale, currency, or encoding involved | +++ |
| Written or heavily assisted by an AI coding tool | ++ |
| Thin or tautological existing tests (high coverage, low mutation score) | +++ |

**Impact — how bad if it does break?**

| Signal | Weight |
|---|---|
| Money moves, or a monetary figure is shown | +++ |
| Data can be lost or silently corrupted | +++ |
| Authentication, authorization, or tenant isolation | +++ |
| Regulatory or contractual obligation (KVKK, GDPR, PCI, EAA, SOC 2) | +++ |
| On the primary revenue path | +++ |
| Failure is silent — no error, no log, no alert | ++ |
| Irreversible: cannot be rolled back or corrected after the fact | +++ |
| Affects all users rather than a segment | ++ |

The AI-authorship signal is not decoration. AI-generated code carries a measurably
higher defect rate, and when the same tool wrote both the code and its tests, the
two share a blind spot: the tests assert what the code *does*, not what it *should*
do. Treat high coverage on AI-written code as unverified until something independent
of that tool has checked it.

## 4. Choose depth, not just scope

For each area, pick a depth and justify it. Depth is where the plan becomes real.

| Depth | What it means | Fits |
|---|---|---|
| **Deep** | Boundaries, negatives, concurrency, failure injection, data variants, recovery | Risk ≥ 15 |
| **Standard** | Happy path plus the obvious negatives and one boundary per input | Risk 8–14 |
| **Shallow** | One smoke check that the area still responds | Risk 4–7 |
| **None** | Explicitly not tested, with a reason | Risk < 4 |

Then choose the **level** for each: unit, integration, contract, end-to-end, manual
exploratory, or production monitoring. Push each check to the cheapest level that can
still find the defect. An end-to-end test that verifies a boundary condition is a
unit test wearing an expensive costume.

## 5. Go beyond functional correctness

Most plans stop at "does it do the right thing". Walk this list and mark each one
*relevant* or *not applicable, because —*. The value is in the deliberate dismissal.

- **Performance** — under realistic data volume, not an empty test database
- **Concurrency** — two users, the same record, the same instant
- **Security** — authorization at the object level, not just the route; injection; secrets in logs
- **Data integrity** — partial failure, retries, idempotency, duplicate messages
- **Recovery** — what happens when the dependency is down, slow, or returns garbage
- **Compatibility** — old clients against the new server, and the reverse during a rolling deploy
- **Accessibility** — keyboard-only completion of the flow, focus order, announced state changes
- **Observability** — when this fails in production, will anyone find out? From what signal?
- **Localization** — Turkish dotted/dotless İ/ı casing, RTL, non-Gregorian dates, decimal comma
- **Migration** — existing production-shaped data through the new code, not fresh fixtures

## 6. Write the plan

Keep it to one page plus the register. Nobody reads more.

```
## Mission
  One paragraph: what decision this informs, the time box, what changed.

## Risk register
  | # | Area | Likelihood | Impact | Risk | Depth | Level | Owner |

## Test approach per high-risk area
  For each area scoring >= 15, three to six lines: what you will actually do,
  which oracle tells you it is wrong, and what data you need.

## Out of scope
  | Area | Why not | What we accept if we are wrong | Mitigation |
  Mitigation is real: a feature flag, a canary, a monitor, a fast rollback,
  or "none — we are accepting this risk".

## Entry and exit
  Entry: what must be true before testing starts (environment, data, build).
  Exit: what must be true to call it done. Never "all tests pass" — that is
  automatic. Use "no open S1/S2, the three highest-risk areas tested to planned
  depth, rollback rehearsed once."

## Assumptions and dependencies
  What you believe to be true and have not verified. State them; they are the
  places the plan silently breaks.
```

## Anti-patterns

- **A plan that is a test-case list.** Cases are the output of a plan, not the plan.
- **Uniform depth everywhere.** If everything is equally important, nothing was prioritized.
- **Regression = re-run everything.** Re-run what the change can reach; the rest is
  ritual, and it eats the time the high-risk area needed.
- **Missing out-of-scope section.** Then the risk was accepted silently, by one person,
  with no record.
- **Coverage percentage as an exit criterion.** Coverage measures execution, not
  verification. Pair it with a mutation score, or drop it.
- **Scoring from feelings.** If a likelihood score cannot be traced to churn, complexity,
  defect history, or a named property of the change, it is a guess wearing a number.

## Output

Produce the plan as Markdown. Then state, in three lines:

1. The single highest-risk area and why it beat the others.
2. The riskiest thing you are deliberately not testing.
3. The information you lacked that would most change the plan.

Companion file: `RISK_MODEL.md` — scoring bands, a worked example, and the
regression-scope heuristics.

See also: `exploratory-charter` to turn each high-risk area into a session with a
stated mission, and `bug-report-forensics` for what those sessions produce. Neither
is required to use this skill.
