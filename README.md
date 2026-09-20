# qa-skills

**Agent Skills for software testers — the craft of QA, not just test generation.**

Most AI testing tools help you *write tests faster*. This one helps you decide
**what is worth testing, whether a test actually verifies anything, and how to
report what you found** so somebody can act on it.

Framework-agnostic. Examples lean on JUnit/TestNG and NUnit/xUnit, because that
is where most enterprise QA work actually happens and where the existing
JavaScript-flavoured tooling has the least to say.

---

## Why this exists

The Agent Skills ecosystem has grown quickly, and QA is close to absent from it.
Anthropic's public skills repository ships exactly one testing skill. The community
directories list a handful of generic "qa-testing" entries, most of them a single
paragraph of prompt.

Meanwhile the need went the other way. Roughly 40% of new code is now AI-generated,
carrying a measurably higher defect rate than human-written code, and 61% of teams
report increased QA demand as a result. The ICSE 2026 systematic review of 101 sources
named QA the most frequently overlooked dimension of AI coding workflows.

The gap is not "generate more tests". It is judgement:

- A test suite at 90% line coverage and 20% mutation score verifies nothing, and
  looks green while doing it.
- A test plan that lists what it covers, and never what it skips, hides every
  decision worth reviewing.
- A bug report that costs a round trip costs more than it saved.

These skills encode that judgement.

---

## Install

### Claude Code (CLI)

```
/plugin marketplace add rcptrkr/qa-skills
/plugin install qa-essentials@qa-skills
```

To work from a local clone instead:

```
git clone https://github.com/rcptrkr/qa-skills.git
```

```
/plugin marketplace add ./qa-skills
```

### Claude desktop app

The app manages plugins through its own Plugins panel rather than the slash
command. Open it and add `rcptrkr/qa-skills` as a marketplace, then install
**qa-essentials** from it.

### Any agent that reads SKILL.md

The skills are plain Markdown with YAML frontmatter and depend on nothing else,
so you can drop them in directly:

```
git clone https://github.com/rcptrkr/qa-skills.git
mkdir -p ~/.claude/skills
cp -R qa-skills/plugins/qa-essentials/skills/* ~/.claude/skills/
```

They then load as `/bug-report-forensics`, `/risk-based-test-plan` and
`/exploratory-charter`. The same folders work with Cursor, Codex, Gemini CLI and
anything else that reads the Agent Skills format — adjust the destination path.

---

## What's inside

### `bug-report-forensics`
Turns a vague observation into a report whose first developer action is *fix*, not *ask*.

- Three gates before writing: is it a bug (name the oracle), is it a duplicate,
  is the repro minimal
- Reproduction minimization bisected along four axes — steps, data, environment, time —
  producing a **narrowest condition** line
- Severity and priority separated, each with written justification; full rubric with
  a decision table and the severity-inflation anti-patterns
- Evidence checklists per stack: Java/Spring (full `Caused by:` chains, thread dumps),
  .NET (`InnerException`, TraceId), web (HAR + console), API, data
- Three before/after rewrites, including the flaky test that should never have been
  fixed with a retry

### `risk-based-test-plan`
A plan ranked by risk, whose most valuable section is what it **won't** test.

- Likelihood and impact scored from observable signals — churn, complexity, defect
  history, concurrency, time/locale/money, AI authorship — not intuition
- Depth chosen per area and pushed to the cheapest level that can still find the defect
- Ten non-functional dimensions walked explicitly, each marked relevant or
  *not applicable, because —*
- An out-of-scope table with a real mitigation per row, so accepted risk is recorded
  rather than absorbed silently
- Worked example: idempotency keys on a payment endpoint, eight areas scored
- Three-point estimation tied to stopping points instead of a single invented number

### `exploratory-charter`
Session-based exploratory testing that produces evidence, not anecdotes.

- Charters in the `Explore … with … to discover …` form, where the third clause
  must be a question, not "bugs"
- 5–8 ranked charters per area using SFDIPOT product elements and quality criteria
- Heuristic library: HICCUPPS oracles, data attacks (including Turkish İ/ı casing,
  decimal comma, DST boundaries), state and flow models, failure injection, eight tours
- An accessibility section covering the ~60% of barriers automated scanners miss
- Session sheet with the task breakdown — where a setup percentage consistently
  above 30% is your evidence that the real problem is testability

---

## These skills are measured

Every skill here was run against a no-skill control on six realistic prompts and graded
by an independent agent on 63 assertions drawn from the skills' own promises:
**98.3% with the skill, 63.8% without**, winning all six cases.

More usefully, the measurement found three defects in the skills — including one where
`bug-report-forensics` was *suppressing* something the control produced, because the
report template had no field for how a fix would be verified. All three are fixed.

It also showed that only 35% of the assertions discriminate at all, which narrows the
honest claim: these skills do not supply good judgement, they stop good judgement from
being left soft.

Method, full results, the defects found and the limits of the measurement are in
[`evals/RESULTS.md`](evals/RESULTS.md). Prompts and assertions are in
[`evals/evals.json`](evals/evals.json).

---

## Design principles

1. **Judgement over generation.** Anything that just wraps a prompt around
   "write tests for this" is out of scope.
2. **Every skill states what it will not do.** Out-of-scope sections are features.
3. **Real examples, real stacks.** Every rubric ships with a worked example that
   names a build, an endpoint, and a line of code.
4. **Anti-patterns are part of the skill.** Knowing the wrong shape is half the craft.
5. **Companion files carry the depth.** `SKILL.md` stays readable; the rubric,
   the heuristic library and the examples live beside it.

---

## Roadmap

Shipping roughly one skill a week.

- [x] `bug-report-forensics`
- [x] `risk-based-test-plan`
- [x] `exploratory-charter`
- [ ] `flake-triage` — classify a flaky failure by root-cause family, then fix rather than retry
- [ ] `api-contract-diff` — breaking-change detection across OpenAPI versions, with a consumer blast radius
- [ ] `test-data-synth` — realistic test data with no PII; KVKK/GDPR-safe by construction
- [ ] `a11y-audit` — keyboard traversal, focus order and announcement checks mapped to WCAG 2.2 / EN 301 549
- [ ] `requirement-ambiguity-linter` — find untestable acceptance criteria before code is written
- [ ] `boundary-value-analysis` — equivalence partitions and boundaries for a given signature
- [ ] `regression-impact-map` — what a diff can actually reach, including via shared state
- [ ] `oracle-review` — does this test verify anything, or does it assert its own mock back to itself?

Issues and PRs welcome, particularly worked examples from stacks other than
Java and .NET.

---

## License

MIT
