# Before / after

Three real-shaped rewrites. The "before" versions are the forms that actually get
filed; each one costs at least one round trip.

---

## 1. Web UI — the vague report

**Before**

> Checkout is broken, cart total is wrong sometimes. Screenshot attached.

Every field a developer needs is missing: which total, wrong how, which build,
which conditions, how often.

**After**

```
Title: [checkout] Cart total omits shipping when a discount code is applied after items are added

Summary
  Applying a discount code recalculates the total but drops the shipping line.
  The customer is charged the displayed (understated) amount, so the shortfall is
  absorbed as a loss on every affected order.

Preconditions
  - Build: 4.12.3-rc2 (a1b2c3d), staging-eu
  - Standard user, tenant "acme-eu", locale tr-TR
  - Cart contains >= 1 physical item (shipping applies)

Steps
  1. Add any physical item to the cart
  2. Proceed to checkout — note the total includes a shipping line
  3. Enter discount code SAVE10 and apply

Expected: total = (items - 10%) + shipping
Actual:   total = (items - 10%); the shipping line disappears from the summary
          and from the order payload sent to /api/v2/orders

Frequency: 10/10
Narrowest condition: only when the code is applied *after* reaching checkout.
  Applying the same code from the cart page keeps shipping correct.

Evidence
  - HAR: checkout-discount.har (see request #14, "shipping" key absent from body)
  - Console: no errors
  - Order 4471 in staging shows total 89.90 where 99.90 was owed

Severity: S1 — a monetary amount is computed incorrectly and persisted to the
  order without any error surfaced.
Suggested priority: Highest — S1, reproduces every time, affects any order with a
  discount code, and it is silent: neither the customer nor support has a reason
  to report it.

Scope
  Also reproduces with FREESHIP. Does NOT reproduce on the v1 checkout page.
  Gift-card redemption on the same screen is unaffected.

Regression
  Last good: 4.11.8. First bad: 4.12.0-rc1. Suspect the pricing refactor in
  PricingEngine.recalculate() which appears to rebuild the line list from
  scratch rather than mutating it.

Workaround
  Apply discount codes from the cart page before proceeding to checkout.
```

---

## 2. Java / Spring — the stack trace dump

**Before**

> NullPointerException in prod, see attached log. Please fix ASAP.

A stack trace is evidence, not a report. It says where the program stopped, not
what was expected, what triggered it, or what it costs.

**After**

```
Title: [orders-api] NPE on GET /api/v2/orders/{id} when the order has no shipping address

Summary
  Orders created through the partner integration have a null shippingAddress.
  Fetching one returns 500 instead of the order. Partner reconciliation is blocked
  for every such order.

Preconditions
  - orders-api 2.8.1 (commit 9f4e2a1), prod and staging
  - An order created via POST /partner/v1/import with shippingAddress omitted
    (the partner schema permits it; ours assumes it)

Steps
  1. Import an order with no shippingAddress (payload attached: partner-order.json)
  2. GET /api/v2/orders/{id}

Expected: 200 with shippingAddress serialized as null
Actual:   500, NullPointerException in OrderMapper.toDto (line 84)

Frequency: 5/5 for affected orders. 0/5 for orders with an address — the defect
  is fully determined by the null field, not intermittent.

Evidence
  - Full stack trace with both Caused by: frames — orders-api-npe.log
  - traceId 0af7651916cd43dd8448eb211c80319c; the surrounding 30s log window is attached
  - The failing line: `dto.setCity(order.getShippingAddress().getCity())`

Severity: S2 — a primary read flow fails for an identifiable set of orders.
  Not S1: no data is lost or corrupted, and the failure is loud.
Suggested priority: High — 312 orders currently in this state (query attached) and
  the count grows with every partner import; reconciliation cannot proceed on any
  of them.

Scope
  Same mapper is used by GET /api/v2/orders (list). The list endpoint fails the
  whole page, not just the affected row — arguably a second defect; filed as QA-4482.
  The v1 endpoint returns these orders correctly.

Regression
  Not a regression. The partner import path was added in 2.8.0; the mapper predates
  it and has always assumed the field is present.

Workaround
  None for the API. Support can read the order through the admin console, which
  uses a different mapper.
```

---

## 3. Flaky test — the one that gets ignored

**Before**

> CheckoutIT is flaky again, re-ran it and it passed. Adding a retry.

Adding a retry converts a signal into silence. If the test is flaky because the
*system* is racy, the retry hides a real production defect.

**After**

```
Title: [CheckoutIT.appliesDiscountBeforeTax] fails ~30% on CI, passes locally —
       shared static PricingContext across parallel test classes

Summary
  The test asserts a tax figure computed from a static PricingContext that a
  parallel test class mutates. This is a test defect, but it exposes a genuine
  question about the production singleton (see Scope).

Preconditions
  - main @ 9f4e2a1, CI runner (4 parallel forks), surefire parallel=classes
  - Reproduces locally with -Dsurefire.parallel=classes; never with a single fork

Steps
  1. Run CheckoutIT and TaxRateIT in parallel forks
  2. Repeat 10 times

Expected: 10/10 pass
Actual:   3/10 fail with expected 18.00 but was 20.00

Frequency: 3/10 on CI. 0/10 single-threaded. 4/10 locally with parallel forks.

Evidence
  - Two full runs, one pass one fail, with thread names visible — checkout-flake.zip
  - Interleaving confirmed: TaxRateIT sets PricingContext.rate = 0.20 between this
    test's arrange and assert steps

Severity: S3 as a test defect — no production behaviour is wrong.
Suggested priority: High — it is currently the top cause of red builds, and the
  team has begun re-running CI reflexively, which will hide the next real failure.

Scope
  PricingContext is also a singleton in production. It is only written at startup
  there, so no production race exists today — but nothing enforces that. Filed
  QA-4491 to make the field final or move it behind a request-scoped bean.

Regression
  Started when TaxRateIT was added in #2214. The race has existed since the
  singleton was introduced; that PR was the first parallel writer.

Fix, not a retry
  Give each test its own PricingContext, or annotate both classes @NotThreadSafe.
  A retry here would mask a design flaw one refactor away from becoming a
  production defect.
```
