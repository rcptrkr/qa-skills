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

## Trigger accuracy

Output quality only matters if the skill is reached for in the first place. That is a
separate measurement, and it needed its own harness.

14 queries, two runs each, phrased the way a tester would actually write them and never
naming a skill. Four should route to `bug-report-forensics`, four to
`risk-based-test-plan`, three to `exploratory-charter`, and three should trigger nothing
at all. Three of the queries are in Turkish. Because all three skills live in the same
domain and share vocabulary, the interesting failure is not silence — it is one skill
answering another's question.

**28/28.** Every positive query reached the correct skill on both runs, Turkish
included, with no cross-triggering. All three negatives — "write a junit 5 test for this
method", "this NullPointerException is coming from OrderMapper line 84, fix it", "write
the postmortem doc for yesterday's outage" — left every skill dormant.

### The tooling was the least reliable thing measured

The standard description-optimization loop first reported 0% recall, then 11–17% after a
partial fix. Both figures were wrong, and finding out why took longer than the
measurement itself. Two faults:

1. **It tested a command, not a skill.** The harness writes the candidate description to
   `.claude/commands/`. Commands are user-invoked; the model does not reach for them on
   its own. Placing a command and a skill with identical descriptions side by side, the
   model called the skill and ignored the command every time.
2. **It did not isolate from the installed copy.** With `qa-essentials` installed, every
   run had the real skill competing with the harness's temporary duplicate. The model
   picked the real one — and the harness, which only counts its own uuid-suffixed copy,
   recorded that as "not triggered". The thing under test was breaking its own test.

Even with both faults fixed, the harness's synthetic skill — a uuid-suffixed name and a
one-line body — triggered on 1 of 3 runs where the real installed skill triggered on 4
of 4. So its absolute numbers stay untrustworthy, and the figures above come from
running the real skills and recording which one the model actually called.

The loop's own conclusion was to change nothing, which was the right non-action on a
broken signal.

## Limits of this measurement

- **One run per cell.** The ± figures in `benchmark.json` are spread across cases, not
  run-to-run variance. Nothing here establishes reliability; repeated runs would.
- **Two thirds of the assertions measure nothing.** They are kept for now because a
  non-discriminating assertion still guards against regression, but they inflate the
  headline pass rate and should be sharpened.
- **The trigger queries were written by the same person who wrote the descriptions.**
  That is the weakest point in the trigger result. Queries written by someone who has
  not read the skills would be a fairer test, and 28/28 should be read with that in mind.
- **14 queries, two runs, one model.** Enough to show the skills route correctly and stay
  quiet on near-misses; not enough to put a confidence interval on it.
- **Quality and triggering were measured separately.** Nothing here measures the two
  together — whether a skill that fires on a real, messy, half-stated request still
  produces the output these evals graded.
