---
name: exploratory-charter
description: Generate session-based exploratory testing charters and run the session properly — a ranked set of missions with heuristics and oracles, plus a session sheet that records what was actually learned. Use when scripted cases are not enough, when testing something new or poorly specified, when asked to "have a look at" a feature, or when a bug is suspected but not yet found.
argument-hint: "[feature, screen, API, or area to explore]"
---

# Exploratory Charter

Exploratory testing is not unstructured clicking, and it is not a substitute for
having thought. It is **simultaneous learning, test design, and execution**, kept
accountable by two artifacts: a *charter* that states the mission before you start,
and a *session sheet* that records what you found after.

Scripted tests confirm what someone already imagined. Exploration finds what nobody
imagined — which is where the defects that reach production actually come from.

## Charter format

Three parts, one sentence:

> **Explore** _target_ **with** _resources_ **to discover** _information_.

The third part is the one people get wrong. "To discover bugs" is not information —
it is a wish. Name the *question* the session answers.

Good:
- Explore **the bulk CSV import** with **files that are malformed in one way each** to discover **how partial failures are reported and whether partial data is committed**.
- Explore **session expiry** with **two browser tabs and a clock skewed forward** to discover **whether an expired session in one tab can still act through the other**.
- Explore **the order search API** with **Turkish locale data (İ/ı, ç, ş) and 10k-row result sets** to discover **where casing and paging assumptions break**.

Bad:
- Explore the checkout page to find bugs. *(no question, no resources, unbounded)*
- Test the new report screen. *(not a charter — a task)*

## Producing a charter set

Given an area, generate **5–8 charters**, each 60–90 minutes, ranked by risk.
Use these lenses so the set covers different failure classes rather than eight
variations of the happy path.

**Product elements — SFDIPOT** (what is there to test?)
- **S**tructure — files, modules, config, the physical thing
- **F**unction — what it does, including what it does when it fails
- **D**ata — inputs, outputs, stored state, size, cardinality, lifetime
- **I**nterfaces — UI, API, CLI, imports/exports, other systems
- **P**latform — OS, browser, device, locale, timezone, dependencies you do not own
- **O**perations — how it is really used: by whom, how often, under what pressure
- **T**ime — concurrency, ordering, timeouts, expiry, scheduling, daylight saving

**Quality criteria** (what does "good" mean here?)
Capability, reliability, usability, **accessibility**, security, scalability,
performance, installability, compatibility, supportability, testability,
maintainability, portability, localizability.

Pick the two or three that matter for this area and let them drive charters. An
accessibility charter belongs in most sets and is almost always missing from them.

`HEURISTICS.md` holds the full heuristic library — oracles, data attack patterns,
state and boundary models, and per-interface tours.

## Rank the set

Order the charters so that if the session budget is cut in half, the top half is
still the right half. Rank by:

1. **Uncertainty** — where do you know least? Exploration buys information; spend
   it where you have none. An area with solid tests is a poor charter target.
2. **Consequence** — what costs the most if wrong (money, data, isolation, compliance).
3. **Novelty** — new code, new dependency, new author, new integration.
4. **Reachability** — a defect only reachable by a path nobody takes matters less.

## Running the session

**Time box it.** 90 minutes, one charter, no interruptions. When the box ends, stop
and write the sheet — even mid-thread. An unbounded session produces no record and
no comparable data.

**Follow the interesting thing, but log the debt.** When something odd appears that
is outside the charter, note it as a lead and continue. Do not silently redefine the
mission. If the lead is clearly more valuable, end the session, say so on the sheet,
and start a new charter for it.

**Vary deliberately, not randomly.** Change one thing at a time so you can attribute
the result. Random clicking finds defects you cannot reproduce, which are defects you
cannot report.

**Name your oracle before you call something a bug.** How do you know this is wrong?
See the HICCUPPS list in `HEURISTICS.md`. "It felt off" is a lead worth pursuing, not
a finding worth filing.

## Session sheet

Record these. Comparable across sessions, and it makes exploratory work *reportable* —
which is the only reason managers who distrust exploratory testing come to trust it.

```
Charter:      Explore ... with ... to discover ...
Tester:       Recep
Date/Start:   2026-09-06 14:00     Duration: 90 min
Build:        4.12.3-rc2 (a1b2c3d)      Environment: staging-eu

Task breakdown (must total 100%)
  Test design & execution:  65%
  Bug investigation & reporting:  25%
  Session setup:  10%

Data & tooling used
  Postman collection X, a 10k-row CSV with three deliberate encoding faults,
  Chrome with the locale forced to tr-TR

Notes
  Chronological. What you tried, what you saw, what you concluded, and where you
  changed direction. Enough that someone else can retrace the path.

Bugs        (filed, with ids)
Issues      (obstacles to testing: missing data, broken environment, no way to
             observe an internal state — these are findings too, and often the
             most actionable ones)
Leads       (threads not pulled; the seed of the next charter)

Coverage achieved vs charter
  One honest sentence. "Covered malformed headers and encoding; did not reach
  the 10k-row volume case — the import times out at 2k on staging, filed QA-4502."
```

The **setup percentage** is quietly the most useful number on the sheet. When setup
is consistently over 30%, the team's real problem is testability, not test coverage,
and you now have the evidence to say so.

## Anti-patterns

- **"Explore X to find bugs."** No question, so no way to know when the session is done.
- **No time box.** Produces one endless session and no comparable data.
- **No session sheet.** The work becomes invisible, and invisible work is the first
  thing cut when the schedule tightens.
- **Charters that duplicate the scripted suite.** Explore where the script *cannot* go.
- **Exploring without a build/environment record.** An unattributable finding is
  unreproducible and will be closed.
- **Treating "no bugs found" as a failed session.** A session that establishes
  confidence in a high-risk area is a successful session. Record what you covered
  and how deeply.

## Output

Produce:

1. **A ranked charter set** (5–8), each with target, resources, information sought,
   the heuristics you intend to apply, and a one-line risk rationale for its position.
2. **A ready-to-fill session sheet** for the top charter.
3. **One line** naming which quality criteria the set does *not* cover, so the gap is
   a decision rather than an oversight.

Companion file: `HEURISTICS.md` — oracles, data attacks, state models, tours.
