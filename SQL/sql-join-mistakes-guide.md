# SQL JOIN Mistakes: A Field Guide for Working Developers

> A practical, example-driven reference for intermediate-to-senior developers who write JOINs
> against production databases. Every pattern below has a corresponding real-world failure mode —
> the kind that passes code review, passes a manual spot-check, and then quietly corrupts a
> dashboard, doubles a billing report, or drops rows from an export three weeks later.

---

## Table of Contents

1. [Why JOIN bugs are uniquely dangerous](#1-why-join-bugs-are-uniquely-dangerous)
2. [Mistake 1: Joining on the wrong (or almost-right) column](#2-mistake-1-joining-on-the-wrong-or-almost-right-column)
3. [Mistake 2: Missing or incomplete join conditions (accidental Cartesian products)](#3-mistake-2-missing-or-incomplete-join-conditions-accidental-cartesian-products)
4. [Mistake 3: Choosing the wrong JOIN type (INNER vs. LEFT vs. FULL)](#4-mistake-3-choosing-the-wrong-join-type-inner-vs-left-vs-full)
5. [Mistake 4: Filtering in WHERE instead of ON (silently undoing a LEFT JOIN)](#5-mistake-4-filtering-in-where-instead-of-on-silently-undoing-a-left-join)
6. [Mistake 5: Ignoring row multiplication (fan-out) in one-to-many joins](#6-mistake-5-ignoring-row-multiplication-fan-out-in-one-to-many-joins)
7. [Mistake 6: Joining on descriptive/natural keys instead of stable surrogate keys](#7-mistake-6-joining-on-descriptivenatural-keys-instead-of-stable-surrogate-keys)
8. [Mistake 7: Column ambiguity and missing qualification](#8-mistake-7-column-ambiguity-and-missing-qualification)
9. [Mistake 8: NULL-unsafe join conditions](#9-mistake-8-null-unsafe-join-conditions)
10. [Mistake 9: Duplicate rows on the "one" side (the silent fan-out's evil twin)](#10-mistake-9-duplicate-rows-on-the-one-side-the-silent-fan-outs-evil-twin)
11. [Mistake 10: Joining across mismatched grains without aggregating first](#11-mistake-10-joining-across-mismatched-grains-without-aggregating-first)
12. [Mistake 11: Non-equi joins and range joins done carelessly](#12-mistake-11-non-equi-joins-and-range-joins-done-carelessly)
13. [Mistake 12: Self-joins that double-count or miss the "first/last" row](#13-mistake-12-self-joins-that-double-count-or-miss-the-firstlast-row)
14. [Mistake 13: Performance traps — joining before filtering, and non-sargable ON clauses](#14-mistake-13-performance-traps--joining-before-filtering-and-non-sargable-on-clauses)
15. [A systematic validation checklist](#15-a-systematic-validation-checklist)
16. [Incremental debugging method](#16-incremental-debugging-method)
17. [Quick-reference decision table](#17-quick-reference-decision-table)

---

## 1. Why JOIN bugs are uniquely dangerous

Most bugs fail loudly: a stack trace, a 500 error, a crash. JOIN bugs usually don't. The database
executes exactly the query you wrote — it has no concept of what "customer" or "order" is supposed
to mean in your business. If your ON clause is syntactically valid but semantically wrong, you get
back a result set that:

- **Looks structurally correct** (right columns, plausible-looking values)
- **Often returns a non-empty, non-obviously-broken row count**
- **Passes a casual `SELECT * LIMIT 10` sanity check**
- **Fails only under specific data conditions** (a customer with zero orders, a product renamed
  last quarter, an order with three line items instead of one)

This is why JOIN mistakes are disproportionately responsible for incidents like "the revenue
dashboard says we made 2.3x more than we did" or "the export dropped 40% of customers" — bugs that
surface in a QBR or a customer complaint, not in CI.

**Schema used throughout this guide** (a simplified e-commerce schema, referenced consistently so
examples build on each other):

```sql
customers(customer_id PK, name, email, created_at, region_id FK)
orders(order_id PK, customer_id FK, order_date, status, discount_code)
order_items(order_item_id PK, order_id FK, product_id FK, quantity, unit_price)
products(product_id PK, name, sku, category_id FK, current_price)
regions(region_id PK, region_name)
refunds(refund_id PK, order_id FK, amount, refunded_at)
```

---

## 2. Mistake 1: Joining on the wrong (or almost-right) column

### The failure mode
The columns exist, the names are plausible, and the query runs without error — but the relationship
being expressed isn't the one the schema actually models.

```sql
-- WRONG: joins customers to orders using region_id, not customer_id.
-- Looks fine — both tables have a "region_id"-shaped relationship in the analyst's head —
-- but this pairs every customer with every order placed by anyone in their region.
SELECT c.name, o.order_id, o.order_date
FROM customers c
JOIN orders o ON c.region_id = o.customer_id;
```

### Real-world case: the "duplicate customer emails" incident
A support team built a query to find customers who share an email address with another account
(fraud detection). The intended join was `a.email = b.email AND a.customer_id <> b.customer_id`.
An engineer under time pressure copy-pasted a similar query and joined on `a.customer_id =
b.region_id` instead — a leftover from a different report. The query returned ~400 "duplicate"
accounts. Fraud ops froze 400 real customer accounts before someone noticed the joined rows had no
relationship to email at all. **Root cause: the column existed on both sides, so autocomplete and a
quick glance made it look legitimate.**

### How to avoid it
- Before writing the `ON` clause, open the schema (or an ERD) and identify the actual foreign key.
  Don't rely on column name similarity as a proxy for a real relationship.
- If your schema has a data dictionary or dbt model docs, check the declared relationships there.
- Add foreign key constraints in the database where possible — they won't stop a bad JOIN, but
  they document intent and catch a class of orphaned-row problems at insert time.
- Write the join condition as a sentence first: *"Each order belongs to exactly one customer, via
  orders.customer_id referencing customers.customer_id."* If you can't say that sentence about your
  ON clause, don't write the query yet.

---

## 3. Mistake 2: Missing or incomplete join conditions (accidental Cartesian products)

### The failure mode
A `JOIN` with no `ON` (or a comma-join with a missing `WHERE` filter) pairs every row on the left
with every row on the right.

```sql
-- WRONG: no join condition between order_items and products.
-- 1,000 order_items × 5,000 products = 5,000,000 rows.
SELECT o.order_id, p.name, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p;  -- <-- missing ON products.product_id = oi.product_id
```

### Real-world case: the invoice generator that billed 5,000 line items per order
A billing service had a report-generation job with a typo: a trailing comma turned an intended
three-table JOIN into a comma-join for the last table, with the `ON` clause accidentally dropped
during a refactor from implicit to explicit JOIN syntax. In a staging environment with 20 products
it was invisible — 20 extra rows per order, nobody looked closely. In production with 12,000
products, each invoice PDF tried to render 12,000 line items. The job didn't crash; it just ran for
six hours and produced invoices with a single real line item repeated alongside thousands of
nonsensical ones, because a later `WHERE` clause happened to filter most (but not all) of the
Cartesian rows out.

### How to avoid it
- Treat a missing `ON` clause as a compile error, not a style issue. Some SQL linters
  (`sqlfluff`, `sqlint`) can flag `JOIN` without `ON`/`USING` — turn that rule on.
- After any JOIN, sanity-check the row count against expectations: if you expect roughly one row
  per order and you're getting hundreds, stop and investigate before adding more joins on top.
- Never leave a raw comma-join (`FROM a, b`) in production code — always use explicit
  `JOIN ... ON`, which makes a missing condition visually obvious.

---

## 4. Mistake 3: Choosing the wrong JOIN type (INNER vs. LEFT vs. FULL)

### The failure mode
`INNER JOIN` silently discards any row on either side that has no match. This is invisible unless
you specifically go looking for what disappeared.

```sql
-- WRONG intent: "find customers who have never ordered."
-- INNER JOIN can never answer this — customers without orders are removed
-- before they can be identified as having no match.
SELECT c.customer_id, c.name
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;   -- this condition can NEVER be true after an INNER JOIN
```

```sql
-- RIGHT: preserve every customer, then check which ones had no matching order.
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

### Real-world case: the "active users" metric that excluded new signups
A product analytics query computed "active users this week" by inner-joining `users` to
`event_logs`. It was correct for its original purpose. Months later, someone reused it as the base
query for a "signups this week" report by changing the date filter, assuming the join was neutral.
Because it was an `INNER JOIN`, any user who signed up but hadn't yet triggered an event (i.e.,
essentially all same-day signups) vanished from the report. Leadership got a "signups" number that
was consistently ~30% lower than reality for over two months before a discrepancy against Stripe
signup counts triggered an investigation.

### How to avoid it
| You want... | Use... |
|---|---|
| Only rows that match on both sides | `INNER JOIN` |
| All rows from the left table, matched data where available | `LEFT JOIN` |
| All rows from the right table, matched data where available | `RIGHT JOIN` (or flip table order and use `LEFT JOIN` — more readable) |
| All rows from both sides, matched where possible | `FULL OUTER JOIN` |
| Rows that exist on the left with no match on the right | `LEFT JOIN ... WHERE right.key IS NULL` (an "anti-join") |

Ask explicitly: *"Do I need to keep rows that have no match?"* If the answer is yes for either side,
`INNER JOIN` is wrong by construction — no amount of `WHERE` filtering fixes it, because the
non-matching rows are gone before `WHERE` ever runs.

---

## 5. Mistake 4: Filtering in WHERE instead of ON (silently undoing a LEFT JOIN)

### The failure mode
This is the single most common "the LEFT JOIN isn't working" bug developers post to Stack Overflow.
A condition on the *right-hand table*, placed in `WHERE`, converts a `LEFT JOIN` back into the
equivalent of an `INNER JOIN` — because `WHERE` runs after the join and drops any row where the
right-side columns are `NULL`.

```sql
-- WRONG: intent is "all customers, and their refunds from 2026 if any."
-- Because the filter is in WHERE, customers with zero refunds (refund.amount = NULL)
-- get dropped — the LEFT JOIN's whole purpose is defeated.
SELECT c.customer_id, c.name, r.amount
FROM customers c
LEFT JOIN refunds r ON c.customer_id = (SELECT customer_id FROM orders WHERE order_id = r.order_id)
WHERE r.refunded_at >= '2026-01-01';
```

```sql
-- RIGHT: move the right-side filter into the ON clause so it only restricts
-- WHICH refunds match, not WHICH customers survive the join.
SELECT c.customer_id, c.name, r.amount
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.customer_id
LEFT JOIN refunds r ON r.order_id = o.order_id AND r.refunded_at >= '2026-01-01'
```

### Real-world case: the churn report that hid churned customers
A subscription company's churn dashboard used a `LEFT JOIN` from `customers` to
`subscription_cancellations` intending to show all customers with cancellation date if applicable.
An engineer added `WHERE cancellations.reason != 'test_account'` to exclude internal test data. This
silently removed every customer who had *never cancelled* (their `reason` column was `NULL`, and
`NULL != 'test_account'` evaluates to `NULL`, not `TRUE`, so they were filtered out too). The
dashboard's "active customer" count dropped by 60% overnight. The fix required moving the exclusion
into the join condition (`AND (cancellations.reason != 'test_account' OR cancellations.reason IS
NULL)`) or, more robustly, handling it in a `CASE`/`COALESCE` after the join.

### How to avoid it
- **Rule of thumb:** conditions that determine *whether a right-side row matches* belong in `ON`.
  Conditions that determine *whether a result row should exist at all* belong in `WHERE` — but only
  apply `WHERE` to left-side (or post-join, NULL-safe) columns when a `LEFT JOIN` is involved.
- If you added a `LEFT JOIN` specifically to preserve unmatched rows, immediately ask: "does my
  `WHERE` clause reference a column from the right-hand table?" If yes, that condition almost
  certainly belongs in `ON` instead.
- Watch out for `!=`/`<>` and `NOT IN` against nullable columns — they don't behave the way `!=`
  does in application code. `NULL != 'x'` is `NULL` (falsy for filtering purposes), not `TRUE`.

---

## 6. Mistake 5: Ignoring row multiplication (fan-out) in one-to-many joins

### The failure mode
Joining a "one" table to a "many" table multiplies every row on the one side by however many
matches it has on the many side. This is *correct* relational behavior — the bug is downstream,
when someone aggregates the result as if the "one" side were still at its original grain.

```sql
-- one order can have many order_items — this is expected and correct at THIS grain:
SELECT o.order_id, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
-- an order with 3 line items now appears as 3 rows. Fine, if you know that.
```

### Real-world case: the revenue report that was 2.4x too high
A finance analyst needed "total revenue and total discount amount per order." They wrote:

```sql
-- WRONG: joining order_items causes each order row to repeat once per line item,
-- so orders.discount_amount gets summed once PER LINE ITEM instead of once per order.
SELECT
    SUM(oi.quantity * oi.unit_price) AS revenue,
    SUM(o.discount_amount) AS total_discount   -- BUG: multiplied by item count
FROM orders o
JOIN order_items oi ON oi.order_id = o.order_id;
```

Orders in this dataset averaged 2.4 line items. The `revenue` figure was correct (it's naturally
computed at the line-item grain). The `total_discount` figure was silently inflated by ~2.4x,
because `discount_amount` lives on the order and got repeated once per line item, then summed as if
each repetition were a distinct discount. This number went into a board deck before a controller
cross-checked it against Stripe's actual discount ledger and found a mismatch.

### How to avoid it
1. **Identify the grain of your final result before writing the query.** Is it one row per order?
   Per line item? Per customer per month?
2. **Never `SUM`/`COUNT` a column from the "one" side after joining to the "many" side** without
   either (a) aggregating the many side into a subquery first, or (b) using `SUM(DISTINCT ...)` /
   window functions carefully, or (c) dividing by the count of matches (fragile, avoid).
3. **Correct version**, aggregating order_items to the order grain *before* joining:

```sql
SELECT
    o.order_id,
    li.revenue,
    o.discount_amount
FROM orders o
JOIN (
    SELECT order_id, SUM(quantity * unit_price) AS revenue
    FROM order_items
    GROUP BY order_id
) li ON li.order_id = o.order_id;
```

4. As a smoke test, always compare `COUNT(*)` before and after adding a join to a "many" table
   against `COUNT(DISTINCT <left-table-primary-key>)`. If they diverge, you have fan-out, and any
   `SUM`/`AVG` on left-side columns downstream is suspect.

---

## 7. Mistake 6: Joining on descriptive/natural keys instead of stable surrogate keys

### The failure mode
Joining on a human-readable column (`product.name`, `customer.email`, `category.label`) instead of
a primary/foreign key. These columns can be non-unique, can change over time, and often have
whitespace/casing inconsistencies that make matches fail silently.

```sql
-- FRAGILE: two different products can share a name ("iPhone Case" from two suppliers),
-- and a rename breaks all historical joins.
SELECT o.order_id, p.current_price
FROM order_items oi
JOIN products p ON p.name = oi.product_name_at_purchase;
```

### Real-world case: the pricing report broken by a rebrand
A retailer joined historical `order_items` to the current `products` table by product name to
compute "price change since purchase." When marketing renamed "Classic Tee" to "Heritage Tee" as
part of a rebrand, every historical order for that product silently stopped matching — no error, no
warning, just missing rows in the price-change report starting the day of the rename. Nobody
noticed for a full sales cycle because the report didn't fail, it just under-reported.
Separately, the same company briefly had two distinct SKUs both temporarily labeled "Wireless
Charger" during a supplier transition, and the name-based join fan-out silently doubled the row
count for that product during the overlap window.

### How to avoid it
- Always join on the foreign key / primary key relationship (`product_id`), not a label.
- If you need the *label as it existed at the time of the transaction* (a common and legitimate
  need — e.g., historical order line-item display should show the name the customer actually saw),
  **store that label directly on the transactional row at write time** (`order_items.product_name_snapshot`)
  rather than re-deriving it by joining to the current dimension table. This is the standard
  data-warehousing pattern for "slowly changing dimensions."
- Treat any join condition using a `VARCHAR` business-facing label as a code smell worth a comment
  explaining why a key wasn't available.

---

## 8. Mistake 7: Column ambiguity and missing qualification

### The failure mode
Multiple tables in a join share a column name (`id`, `name`, `created_at`, `status`). Without table
aliases and explicit qualification, the query either throws an "ambiguous column" error (best case)
or silently resolves to the wrong table's column (worse case, in engines/contexts with lenient
resolution rules, or when the ambiguity is only in the output column list, not the WHERE clause).

```sql
-- Both orders and refunds have a "status" and "created_at" column.
-- Unqualified references make it unclear (to both the engine and the next reader)
-- which table's status is meant, and some engines will pick one silently.
SELECT order_id, status, created_at
FROM orders
JOIN refunds ON refunds.order_id = orders.order_id;
```

### Real-world case: the wrong "status" silently used in a WHERE clause
An engineer wrote a report joining `orders` and `shipments`, both of which have a `status` column
(`orders.status` ∈ {pending, paid, cancelled}; `shipments.status` ∈ {label_created, in_transit,
delivered}). The query used `WHERE status = 'delivered'` with no qualification. The database
resolved `status` to `orders.status` because of column resolution order in that particular
warehouse engine, `orders.status` never contains `'delivered'`, and the query silently returned
zero rows for weeks. It was assumed to be correct ("no orders are stuck," everyone thought) until a
customer complaint about an undelivered order revealed the dashboard had never worked.

### How to avoid it
- **Always alias every table** in a multi-table query (`orders o`, `refunds r`) and **always qualify
  every column** in the `SELECT`, `WHERE`, `ON`, `GROUP BY`, and `ORDER BY` clauses once more than
  one table is involved — even columns that aren't currently ambiguous, because a future schema
  change can make them ambiguous silently.
- Turn on strict-mode ambiguous-column errors where your engine supports it, and treat any
  "ambiguous column" warning in logs as a build-breaking issue, not a warning to ignore.
- In code review, flag any unqualified column reference inside a query with 2+ tables.

---

## 9. Mistake 8: NULL-unsafe join conditions

### The failure mode
`NULL = NULL` evaluates to `NULL` (not `TRUE`) in standard SQL, so an equality-based `ON` clause
will never match two `NULL` values, even when a "match on NULL" is exactly what the business logic
requires (e.g., "match orders to discount codes, treating no-code and no-code as the same").

```sql
-- WRONG: if BOTH orders.discount_code and promotions.code are NULL
-- (i.e., "no discount" on both sides), this ON clause will NOT match them,
-- because NULL = NULL is NULL, not TRUE.
SELECT o.order_id, pr.discount_pct
FROM orders o
LEFT JOIN promotions pr ON o.discount_code = pr.code;
```

This specific example is usually *fine* (you don't want a fake "no discount" promotion row to
match), but the same pattern becomes a real bug when the nullable column is a genuine foreign key
that can legitimately be null on both sides and should be treated as "unassigned = unassigned."

### Real-world case: the region-based access control gap
An internal tool restricted data access using `LEFT JOIN user_region_overrides ON
users.manager_id = overrides.manager_id AND users.region_id = overrides.region_id`. Some users
and some override rows legitimately had `NULL` region_id (meaning "applies to all regions" /
"unassigned"). The intent was for null-region users to match null-region override rows. Because
`NULL = NULL` is never true, those rows never matched, and a small set of users silently fell
through to a default-deny path instead of getting the override permissions they were supposed to
have — a security-relevant availability bug (users locked out of data they should have seen)
discovered only when someone filed a support ticket.

### How to avoid it
- Use `IS NOT DISTINCT FROM` (Postgres, SQL Server via different syntax) instead of `=` when you
  specifically want NULL-to-NULL to count as a match: `ON a.col IS NOT DISTINCT FROM b.col`.
- In engines without that operator, use `ON (a.col = b.col OR (a.col IS NULL AND b.col IS NULL))`.
- More often, the *real* fix is to avoid nullable foreign keys in join conditions altogether by
  using a sentinel value (e.g., `region_id = 0` meaning "global") — nulls in join keys are a
  frequent source of subtle bugs and are worth designing out of the schema where feasible.

---

## 10. Mistake 9: Duplicate rows on the "one" side (the silent fan-out's evil twin)

### The failure mode
Sometimes fan-out isn't caused by a legitimate one-to-many relationship — it's caused by the
"one" side not actually being unique due to bad data, a missing constraint, or an unexpected
historical duplicate.

```sql
-- Expectation: one row per customer per order.
-- Reality: a data migration once inserted a duplicate customer row with the same email
-- but a different customer_id, so this join returns 2 rows for affected orders.
SELECT o.order_id, c.name
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id;
```

### Real-world case: doubled email sends
A marketing automation job joined `customers` to `campaign_targets` to generate a send list. A
historical data import bug had created duplicate customer records for ~2% of the customer base
(same person, two `customer_id` values, both still active due to an incomplete dedup job). The join
was written correctly, but because the "unique" side wasn't actually unique, ~2% of recipients
received every campaign email twice. It took several customer complaints about "why did I get this
twice" before anyone traced it back to duplicate rows rather than a join logic error.

### How to avoid it
- Don't assume a table is deduplicated on its intended key just because it *should* be. Verify:
  `SELECT customer_id, COUNT(*) FROM customers GROUP BY customer_id HAVING COUNT(*) > 1;`
- Enforce uniqueness with a database constraint (`UNIQUE`/primary key) wherever the business rule
  requires it — this converts a silent data bug into a loud insert-time failure, which is much
  cheaper to fix.
- When you can't fix the underlying duplication immediately, deduplicate defensively at query time
  (`ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) = 1`) and leave a comment
  explaining why, with a ticket link to the real fix.

---

## 11. Mistake 10: Joining across mismatched grains without aggregating first

### The failure mode
A generalization of the fan-out issue: joining two tables that are at genuinely different levels
of granularity (e.g., "per order" vs. "per order per day" vs. "per customer per month") without
first bringing both sides to a common grain produces numbers that are technically computed but
meaningless.

### Real-world case: the "average order value" that wasn't
A team wanted average order value per customer per month, joined against a `daily_active_flags`
table (one row per customer per active day) to filter to "customers active that month." The join
between `orders` (per order) and `daily_active_flags` (per customer per day) fanned every order out
by the number of active days that customer had that month, then `AVG(order_total)` was computed
over the fanned-out result — silently weighting customers with more active days more heavily in the
average, with no error and a plausible-looking number.

### How to avoid it
- Before joining, write down the grain of each input table in a comment: `-- orders: 1 row per order`,
  `-- daily_active_flags: 1 row per customer per day`.
- If grains don't match, aggregate one side to the other's grain *first* (e.g.,
  `SELECT customer_id, MAX(active_date) FROM daily_active_flags GROUP BY customer_id` to get to a
  per-customer grain) rather than joining raw and hoping the aggregation downstream cancels it out.
- Any aggregate function (`AVG`, `SUM`, `COUNT`) applied after a multi-grain join deserves a second
  look before it ships.

---

## 12. Mistake 11: Non-equi joins and range joins done carelessly

### The failure mode
Not all joins are equality joins. Date-range and tier-lookup joins (`BETWEEN`, `<`, `>=`) are
common and easy to get subtly wrong at the boundaries.

```sql
-- Intent: match each order to the price tier effective on the order date.
-- WRONG: this can match MULTIPLE overlapping tiers if effective_end is NULL
-- for the current tier and a prior tier's end date overlaps the order date.
SELECT o.order_id, t.tier_name
FROM orders o
JOIN price_tiers t
  ON o.order_date >= t.effective_start
  AND o.order_date <= t.effective_end;
```

### Real-world case: double-counted commission tiers
A sales commission calculator joined deals to commission tiers using `deal_amount BETWEEN
tier_min AND tier_max`, where tier boundaries were defined inclusively on both ends by different
team members over time (`tier_max` of one row equal to `tier_min` of the next). Deals landing
exactly on a boundary value matched two tiers simultaneously, and the downstream `SUM` doubled the
commission for those specific deal amounts — a bug that only manifested for deals at exact
round-number boundaries ($10,000, $25,000), which happened disproportionately often because sales
reps tend to close deals at round numbers.

### How to avoid it
- For range joins, always make boundaries **half-open** (`>= start AND < end`) rather than
  fully-closed (`BETWEEN start AND end`, which is inclusive on both ends) unless you've explicitly
  verified there's no overlap risk at the boundary.
- Add a validation query that checks for overlapping ranges in the "lookup" table itself:
  `SELECT * FROM price_tiers a JOIN price_tiers b ON a.tier_id != b.tier_id AND a.effective_start < b.effective_end AND b.effective_start < a.effective_end;` — any result means your tiers overlap.
- Test range joins explicitly at boundary values, not just "normal" mid-range values.

---

## 13. Mistake 12: Self-joins that double-count or miss the "first/last" row

### The failure mode
Self-joins used to find "the most recent related row" (e.g., latest order per customer) without a
tiebreaker can return multiple rows when there are ties, or the wrong row when timestamps aren't
unique enough.

```sql
-- Intent: get each customer's most recent order.
-- WRONG: if a customer placed two orders in the same second (batch import,
-- clock resolution), this self-join returns BOTH as "most recent."
SELECT o1.*
FROM orders o1
JOIN (
    SELECT customer_id, MAX(order_date) AS max_date
    FROM orders
    GROUP BY customer_id
) latest ON o1.customer_id = latest.customer_id AND o1.order_date = latest.max_date;
```

### Real-world case: duplicate "most recent order" emails
A re-engagement campaign selected "each customer's most recent order" using the pattern above to
personalize an email with the last-purchased product. A batch data-import event had inserted several
orders with an identical `order_date` timestamp (truncated to the second) for a subset of customers.
Those customers received an email listing multiple "most recent" products, because the self-join
matched every row tied for the max timestamp — a bug invisible in normal testing because normal
order creation timestamps are rarely exactly identical.

### How to avoid it
- Use a window function with a **unique, deterministic tiebreaker** instead of a self-join against
  `MAX()`:

```sql
SELECT *
FROM (
    SELECT o.*,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC, order_id DESC) AS rn
    FROM orders o
) ranked
WHERE rn = 1;
```

  `ORDER BY order_date DESC, order_id DESC` guarantees exactly one winner even on exact timestamp
  ties, because `order_id` is unique.
- Whenever you self-join to find an "extreme" row (first, last, max, min), ask: "what happens if
  there's a tie?" and make sure your tiebreaker column is actually unique.

---

## 14. Mistake 13: Performance traps — joining before filtering, and non-sargable ON clauses

### The failure mode
Correctness aside, two join-adjacent performance mistakes recur constantly:

1. **Wrapping a joined column in a function** in the `ON` or `WHERE` clause, which prevents index
   usage:

```sql
-- Non-sargable: the function call on orders.order_date prevents the optimizer
-- from using an index on order_date for this filter.
SELECT *
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE DATE(o.order_date) = '2026-09-01';
```

2. **Joining large tables before filtering them**, forcing the engine to build and then discard a
   much larger intermediate result than necessary (modern optimizers often reorder this
   automatically, but not always — especially across CTE boundaries with `MATERIALIZED` hints, or
   in engines with weaker cost-based optimization).

### Real-world case: the report that timed out only in production
A query joined `orders` (50M rows) to `order_items` (200M rows) and *then* filtered to
`WHERE o.order_date >= CURRENT_DATE - INTERVAL '7 days'`. In a dev database with a week of seed
data, this was fast because the join itself was already small. In production, the engine — under
memory pressure and without a clear enough statistics-based reason to push the predicate down on
its own — built the full historical join before applying the date filter, and the report started
timing out. Rewriting to filter `orders` down to the last 7 days in a CTE *before* joining to
`order_items` cut runtime from several minutes (timeout) to under two seconds.

### How to avoid it
- Filter each table down to the smallest relevant set *before* joining wherever the logic allows,
  especially in engines/query patterns where you can't fully trust automatic predicate pushdown.
- Avoid wrapping indexed join/filter columns in functions; rewrite `DATE(col) = 'x'` as
  `col >= 'x' AND col < 'x' + INTERVAL '1 day'` so the underlying index remains usable.
- Check `EXPLAIN`/`EXPLAIN ANALYZE` for any join step whose estimated vs. actual row counts diverge
  wildly — that's usually the first sign of either a cardinality misunderstanding (see Mistakes 5
  and 10) or a missing/unusable index.

---

## 15. A systematic validation checklist

Run through this before shipping any non-trivial JOIN, especially one feeding a report, a billing
calculation, or an access-control decision:

- [ ] **Relationship**: Does the `ON` clause reference an actual foreign key relationship, verified
      against the schema — not just a plausibly-named column?
- [ ] **Join type**: Does the business question require preserving unmatched rows on either side?
      If yes, is that side using `LEFT`/`RIGHT`/`FULL`, not `INNER`?
- [ ] **WHERE vs. ON**: For every `LEFT`/`RIGHT` join, does any `WHERE` clause reference a column
      from the "preserved-optional" side in a way that would silently convert it back to an INNER
      JOIN?
- [ ] **Grain**: What is the grain of the output you expect (one row per X)? Does every join in the
      chain either preserve that grain or is the multiplication intentional and downstream
      aggregation-safe?
- [ ] **Cardinality check**: Does `COUNT(*)` after the join match `COUNT(DISTINCT <left key>)` when
      you expect a 1:1 relationship? If not, is the divergence understood and intended?
- [ ] **NULL safety**: Can either side of the join key be `NULL` in real data, and if so, does the
      chosen join semantics (match-nothing vs. match-null-to-null) match the business intent?
- [ ] **Uniqueness of "the one side"**: Have you verified (not assumed) that the table you're
      treating as unique per key is actually unique per key in production data?
- [ ] **Stable keys**: Are you joining on primary/foreign keys, not on names, labels, or other
      values that can change or collide?
- [ ] **Qualification**: Is every column in `SELECT`/`WHERE`/`ON`/`GROUP BY` explicitly qualified
      with a table alias?
- [ ] **Boundary values** (for range/non-equi joins): Have you tested exact-boundary values for
      double-matches or gaps?
- [ ] **Performance**: Have you filtered each table down before joining where possible, and checked
      `EXPLAIN` for row-count estimate mismatches?

---

## 16. Incremental debugging method

When a multi-table query gives an unexpected result, don't stare at the whole thing — rebuild it
one join at a time:

1. Run the query against just the first table with the intended `WHERE` filters. Confirm the row
   count and a few sample rows are what you expect.
2. Add the second table's `JOIN`. Re-check the row count. If it changed in a way you didn't
   predict (grew when you expected 1:1, or shrank when you expected all rows preserved), stop here
   — the newly added join is the bug, not something further down the query.
3. Add the next join. Repeat.
4. Only add `GROUP BY`/aggregation once the row-level join result (before aggregation) looks
   correct. Aggregating over a broken join just hides the break behind a plausible-looking number.

This single habit — "the most recently added join is the first place to look when something's
wrong" — resolves the majority of JOIN bugs faster than reading the finished query top to bottom.

---

## 17. Quick-reference decision table

| Symptom | Likely mistake | Section |
|---|---|---|
| Result set is enormous / query never finishes | Missing/incomplete `ON` clause (Cartesian product) | §3 |
| Rows I know exist are missing from an `INNER JOIN` | Wrong join type — need `LEFT`/`FULL` | §4 |
| `LEFT JOIN` isn't preserving unmatched rows | Right-side filter placed in `WHERE` instead of `ON` | §5 |
| `SUM`/`AVG`/`COUNT` numbers look inflated | Fan-out from a one-to-many join, aggregated at the wrong grain | §6, §11 |
| A report broke after a rename/relabel with no code change | Joined on a name/label instead of a stable key | §7 |
| "Ambiguous column" error, or a filter that mysteriously never matches | Missing table qualification | §8 |
| Rows with `NULL` keys on both sides don't match when they should | `NULL = NULL` join semantics | §9 |
| Some records appear twice for no obvious reason | Duplicate rows on the assumed-unique side | §10 |
| A value near a threshold gets double-counted or dropped | Inclusive/overlapping range join boundaries | §12 |
| "Most recent" query returns more than one row for some keys | Self-join without a unique tiebreaker | §13 |
| Query is fast in dev, times out in production | Joining before filtering, or non-sargable predicates | §14 |

---

### Closing principle

A JOIN is a claim about how two things relate in the real world. SQL will faithfully execute that
claim whether or not it's true. Every mistake above comes down to the same root cause: **the query
was syntactically valid but semantically mismatched with the actual schema, actual data, or actual
business question** — and because the failure is silent, the cost of catching it late is almost
always higher than the cost of the checklist in §15 up front.
