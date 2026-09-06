# Heuristic library

Heuristics are fallible shortcuts, not rules. Their value is that they make you
consider something you would otherwise walk past.

## Oracles — how do you know it is wrong?

**HICCUPPS.** Before filing anything, name which of these it violates.

| | Oracle | Question |
|---|---|---|
| **H** | History | Did it behave differently before? Is this a regression? |
| **I** | Image | Does it damage how the organization wants to be seen? |
| **C** | Comparable products | Do competitors, or our other products, do it differently? |
| **C** | Claims | Does it contradict docs, the spec, marketing, a sales promise, the ticket? |
| **U** | User expectations | Would a reasonable user be surprised, and to their cost? |
| **P** | Product | Is it inconsistent with the rest of *this* product? |
| **P** | Purpose | Does it defeat what the feature exists to do? |
| **S** | Standards | Does it violate HTTP, RFC, ISO, WCAG, a platform HIG, or a house convention? |

If none applies, you have a *lead* or a *question*, not a bug. Say which.

## Data attacks

The fastest defect-per-minute lens there is. For every input:

**Strings**
- Empty; a single space; leading and trailing whitespace
- Maximum length; maximum length + 1; far beyond maximum
- Unicode: emoji, combining accents, zero-width characters, RTL marks
- **Turkish casing:** `İstanbul`.toLowerCase() in an English locale yields `i̇stanbul`;
  `i`.toUpperCase() in a Turkish locale yields `İ`. Any case-insensitive comparison
  done without an invariant locale is a real defect. In Java, `toLowerCase()` without
  `Locale.ROOT` breaks on a Turkish JVM; in .NET, `ToLower()` without
  `CultureInfo.InvariantCulture` does the same.
- Injection shapes: `'`, `--`, `<script>`, `${}`, `{{}}`, `../`, null bytes
- Format strings: `%s`, `%n`, `{0}`
- Names that break assumptions: one character, no surname, apostrophes, hyphens

**Numbers**
- 0, -1, 1; the minimum and maximum of the type; min−1 and max+1
- Floating point where money is involved: 0.1 + 0.2, values requiring 3+ decimals
- Very large values that overflow after arithmetic rather than on input
- Numeric strings with leading zeros, `+`, thousands separators, a decimal **comma**
  (`12,50` is valid input in Turkish and German locales)

**Dates and time**
- Leap day; the last day of a month; 31st in a 30-day month
- Daylight-saving transitions — the hour that does not exist, and the hour that happens twice
- Timezone boundaries: an event at 23:30 UTC belongs to a different day for the user
- Server, database, and client in three different zones
- Expiry exactly at the boundary, and one millisecond either side
- Dates far in the past and future; the year 2038; a non-Gregorian display calendar

**Collections and files**
- Empty; exactly one; exactly the page size; page size + 1; far beyond
- Duplicates; nulls inside the collection; ordering assumptions
- Files: 0 bytes, exactly the size limit, over the limit, wrong extension, correct
  extension with wrong content, a valid file that is a zip bomb, a filename with
  spaces/Unicode/traversal

**Nothing at all**
- Omit an optional field; omit a required one; send `null` explicitly versus omitting
  the key; send an empty string where `null` is meant. These are four distinct inputs
  and most APIs treat at least two of them inconsistently.

## State and flow

- **CRUD round trip** — create, read back, update, read back, delete, read back.
  The second read is where the defect usually is.
- **Interrupt everything** — close the tab, hit back, refresh, lose the network,
  time out, kill the app mid-write. Then check what state was left behind.
- **Double submit** — click twice, submit two identical requests concurrently.
  Is the operation idempotent, and does it *claim* to be?
- **Out-of-order** — do step 3 before step 2; navigate straight to a deep URL without
  the preceding flow; replay an old webhook after a newer one.
- **Two actors** — two users editing the same record; the same user in two tabs;
  one session logged out while the other acts.
- **Stale state** — leave a page open for the token lifetime, then act.
- **Back button after a state change** — the single most under-tested browser control.

## Failure and dependency

- The dependency is **down**; **slow** (add 30 s); returns **garbage**; returns a
  **200 with an error body**; returns **half a response** then drops the connection
- The retry storm: does the client retry a non-idempotent operation?
- Circuit breaker open — what does the user actually see?
- Disk full, connection pool exhausted, queue backed up
- Partial success in a batch: is anything committed, and is it reported?

## Accessibility

Automated scanners cover roughly 30–40% of barriers. These are in the other 60%,
and each is a charter of its own:

- Complete the entire flow **using only the keyboard**. Any step you cannot finish
  is a blocker-level defect.
- Is the focus order the same as the visual order? Does focus move into a modal when
  it opens, and back to the trigger when it closes?
- Is there a focus trap you cannot escape? Is focus ever lost to `<body>`?
- Are dynamic changes announced (live regions), or do they happen silently?
- Do interactive elements have an accessible name that matches their visible label?
- Does the page still work at 200% zoom, and at 320 px width with reflow?
- Is state (expanded, selected, invalid, busy) conveyed by something other than colour?

## Tours

Each is a different way to walk the product; each surfaces a different defect class.

- **Money tour** — every path where a value, price, tax, discount or refund is computed
- **Landmark tour** — hit the major features once, in an order no user would use
- **Back-alley tour** — the least-used, least-loved features; where the tests are thinnest
- **Supermodel tour** — look only at the surface: layout, truncation, alignment, i18n
- **Saboteur tour** — actively try to break it: interrupt, corrupt, exhaust, deny
- **Configuration tour** — every setting toggled, especially the non-default combinations
- **Data tour** — follow one record through its whole lifecycle across every screen and
  export that touches it
- **Rested tour** — leave everything idle overnight, come back and act on the stale page

## Escalation questions

When something looks wrong but you are unsure it matters:

1. Which oracle does it violate? (HICCUPPS)
2. Who would notice, and would they know it was wrong?
3. What is the worst plausible consequence, not the worst imaginable one?
4. Is it a symptom of something structural, or is it local?
5. If nobody fixes it, what happens in six months as data accumulates?

Question 5 catches the slow defects: the unbounded table, the leaking handle, the
counter that overflows at scale — the ones that never fail in a test environment
because a test environment never lasts long enough.
