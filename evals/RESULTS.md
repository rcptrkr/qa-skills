# Eval results

These skills are measured, not asserted. This is what the measurement found, including
where it found the skills wanting.

## Method

Six test cases, each phrased the way a tester would actually ask, never naming the
skill. Two are written in Turkish to check the skills hold up on non-English input.
Every case ran twice: once with the skill, once with no skill at all as a control.
Outputs were graded by an independent agent against 63 assertions derived from the
skills' own stated promises — not from an outside idea of quality.

Test prompts and assertions are in `evals.json`.

## Iteration 1

| Case | Skill | With skill | Control |
|---|---|---|---|
| undocumented-api-turkish | exploratory-charter | 11/11 | 6/11 |
| dotnet-exception-dump | bug-report-forensics | 11/11 | 7/11 |
| feature-tight-deadline | risk-based-test-plan | 10/10 | 6/10 |
| legacy-migration-turkish | risk-based-test-plan | 12/12 | 8/12 |
| vague-support-complaint | bug-report-forensics | 9/9 | 6/9 |
| new-screen-no-spec | exploratory-charter | 9/10 | 7/10 |

**98.3% with the skill, 63.8% without.** The skill won every case; no assertion went
the other way. Cost: roughly 20% more tokens and 30% more wall time per run.

### What the skills actually add

Only 22 of 63 assertion instances (35%) discriminated. The other 63% passed in both
configurations — those assertions confirm the model is competent, not that the skill
helps. Reading the 22 that did discriminate, they fall into one pattern:

- **Process steps the model skips unprompted** — checking for a duplicate ticket,
  framing priority as the product owner's decision rather than the tester's
- **Structure that forces a commitment** — a scored risk register, an out-of-scope
  table where every row carries a mitigation, a session sheet whose percentages
  have to add up
- **Named vocabulary** — heuristic frameworks, stack-specific evidence
- **A bounded answer** — 5 to 8 charters, not 13

The assertions that did *not* discriminate are all things a capable model does anyway:
naming an oracle, separating severity from priority, respecting a time box, ranking.

So the honest claim is narrow and worth stating plainly: **these skills do not supply
good judgement. They stop good judgement from being left soft.**

## Iteration 2: three defects found in the skills

The eval was more useful for what it found wrong than for the score.

**1. `bug-report-forensics` suppressed something the control produced.** Neither
skill-guided report said how anyone would know the bug was fixed. Both controls did.
The report template had no such field, and the model followed the template instead of
its own instinct — the skill was actively subtracting value. Fixed by adding a **Fix
verification** field covering the check that must pass and the regression test that
should exist afterwards.

**2. Length did not track knowledge.** The skill produced a ~2,600 word report for a
defect nobody had reproduced, padded with ranked hypotheses that made it look
investigated when it was not. Fixed by making proportionality explicit. Re-run: 2,574
→ 1,590 words and 2,705 → 1,420 words, with the substance kept.

**3. `exploratory-charter` broke its own rule.** The skill forbids "to discover bugs",
so the model wrote "to discover presentation and localisation defects" instead — a
defect *category*, which fails for the same reason. Fixed with a concrete test: if you
can append "defects" to the clause and it still reads naturally, it is a category, not
a question. Re-run: category clauses went from 1 to 0 across both charter cases, and
the Turkish case showed the principle transferring to a different domain rather than
the example being copied.

## Limits of this measurement

- **One run per cell.** The ± figures in `benchmark.json` are spread across cases, not
  run-to-run variance. Nothing here establishes reliability; repeated runs would.
- **Two thirds of the assertions measure nothing.** They are kept for now because a
  non-discriminating assertion still guards against regression, but they inflate the
  headline pass rate and should be sharpened.
- **Trigger accuracy is untested.** These evals hand the skill to the model directly.
  Whether the model reaches for the skill unprompted — which decides whether any of
  this matters in practice — is a separate measurement.
