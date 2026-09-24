# Data Quality Issues — Order Report Export

Data-quality findings from profiling the raw order-level export that feeds the [repurchase-interval analysis](./README.md). Companion to [`data-dictionary.md`](./data-dictionary.md), which covers schema; this file covers what's actually wrong or surprising in the data, and what to do about it.

> **Privacy note:** as with the data dictionary, nothing below is a real customer name, order number, invoice number, or exact revenue figure. Findings are described at the pattern level (what the issue is, roughly how common it is) rather than tied to specific rows.

## Summary

| # | Issue | Severity | Affects |
|---|---|---|---|
| 1 | Revenue stored as formatted text, not numeric | Blocking | Any aggregation |
| 2 | Revenue field and Net Sales field disagree | High | Any revenue rollup |
| 3 | `Product(s)` blank on ~11% of orders despite items sold | Medium | Product-level analysis |
| 4 | Internal/operational accounts mixed into `Customer` | High | Customer-level analysis |
| 5 | Same customer under multiple name spellings/casing | Medium | Customer-level analysis |
| 6 | `Attribution` inconsistently formatted | Low | Marketing/channel reporting |
| 7 | Customer base heavily skewed toward one-time buyers | Informational | Interpretation of results |

## 1. Revenue is stored as formatted text, not a number

**What's happening:** `N. Revenue (formatted)` is a text field with a currency symbol (৳) and comma thousands separators baked in, e.g. `"৳2,999"` rather than `2999`. Every single row is affected — this isn't intermittent, it's the column's native format.

**Why it matters:** Any spreadsheet formula, BI measure, or script that treats this column as numeric without first stripping it will either error out or silently coerce to zero/null, depending on the tool.

**Fix:** Strip the currency symbol and thousands separators, then cast to numeric. This is the first transformation step in the Power Query pipeline documented in [`phase-1-bi-repurchase-model/README.md`](./README.md).

## 2. `N. Revenue (formatted)` and `Net Sales` don't agree

**What's happening:** These two columns look like they should represent the same thing — order revenue — but on the majority of rows they don't match. The gap between them is small and strikingly consistent: across thousands of rows, it clusters tightly around one or two specific values rather than varying randomly.

**Likely explanation:** This pattern is most consistent with `Net Sales` having a flat delivery/shipping charge subtracted out, while `N. Revenue (formatted)` includes it. Rows where the two fields *do* match are plausibly orders with no delivery charge applied (self-pickup, in-store, etc.).

**Why it matters:** Depending on which field you sum, you'll get two different "total revenue" numbers for the same period — and the gap compounds across thousands of orders into a non-trivial discrepancy. Neither field is simply wrong; they're answering slightly different questions.

**Fix:** Decide explicitly which field represents the metric you want (gross order value vs. net of delivery) before building any revenue rollup, and document that choice next to the resulting figure so nobody downstream assumes the other definition.

## 3. `Product(s)` is blank on orders that clearly have items

**What's happening:** About 1 in 9 rows has an empty `Product(s)` field, but `Items sold` is populated and non-zero on every one of them — meaning the order unambiguously contains items, they're just not itemized in this export.

**Likely explanation:** These orders were probably placed through a channel, order type, or legacy import path that doesn't populate itemized line items in this particular report format, rather than being genuinely empty orders.

**Why it matters:** If you filter to rows with a non-empty `Product(s)` field before doing product-level analysis (which the unpivot step requires), these orders silently disappear from "what do customers buy" analysis while still counting toward that same customer's order count and revenue in other parts of the model — creating an inconsistency between "orders we know this customer placed" and "products we can attribute to them."

**Fix:** Keep these rows in customer-level and revenue-level analysis; explicitly flag or exclude them only from product-mix analysis, rather than dropping them from the dataset entirely.

## 4. Internal/operational accounts are mixed into the customer list

**What's happening:** A small number of distinct `Customer` values are clearly store-side accounts — internal test orders, a customer-service account, the brand's own account — rather than real paying customers. At least one of these has an order count far higher than any genuine customer, which will visibly distort any "orders per customer" distribution if left in.

**Why it matters:** This is the kind of thing that looks like a data anomaly (a statistical outlier) but is actually a data *definition* problem — the record shouldn't be in the customer population at all. An automated outlier filter based purely on order count risks either missing it (if the threshold is too high) or incorrectly excluding a genuine high-frequency VIP customer (if the threshold is too aggressive).

**Fix:** Build an explicit exclude-list of known internal/operational account names rather than relying on statistical outlier detection alone. Apply this filter before any customer-level aggregation (order counts, repurchase intervals, revenue-per-customer).

## 5. The same customer appears under multiple name spellings

**What's happening:** Two related but distinct problems:

- **Surface-level variants:** the same person's name appears with different capitalization or extra whitespace across different orders (a normalization problem — case-folding and whitespace-trimming catches this).
- **Deeper account fragmentation:** the same person orders under two genuinely different identities — for example, a personal account and a shared staff/business email — which no amount of text normalization will catch, because the strings are legitimately different.

**Why it matters:** Both inflate the apparent number of distinct customers and, more importantly, fragment a single customer's purchase history across multiple records — which directly breaks the repurchase-interval calculation, since that calculation depends on seeing one customer's *complete* order history in one place.

**Fix:**
- Normalize casing/whitespace on `Customer` before any grouping — cheap, mechanical, catches the first case.
- For the deeper fragmentation problem, a manual mapping step is required. See `customer-mapping.example.csv` in [`phase-2-ai-automation/config/`](../phase-2-ai-automation/config/) for the format used to merge known multi-account customers in this project.

## 6. `Attribution` values are inconsistently formatted

**What's happening:** The marketing-channel field follows a loose `Source: X` / `Referral: X` / `Direct` / `Organic: X` convention, but with over 20 distinct raw values, several are effectively duplicates (different casing or subdomain variants of the same platform), and a handful are malformed or truncated.

**Why it matters:** Lower stakes than the issues above — this doesn't corrupt the repurchase analysis — but it does mean a naive "count of orders by channel" will over-count the number of distinct channels and under-count any single real channel, since its orders are split across near-duplicate labels.

**Fix:** Build a small mapping table collapsing raw `Attribution` values into a clean, fixed set of channel categories before using this field in any reporting.

## 7. The customer base is heavily skewed toward one-time buyers

**What's happening:** The large majority of distinct customers in this export have exactly one order on record. A small minority of customers account for a disproportionate share of total order volume.

**Why it matters:** This isn't a data error — it's the normal shape of a store's full customer list — but it's worth stating explicitly because it's easy to conflate with the *output* of the repurchase-interval analysis. The "top customers" identified by that analysis are a small, specifically-selected high-frequency segment, not a representative sample of the customer base as a whole. Any claim about "average repurchase behavior" drawn from the full dataset would be misleading without accounting for this skew.

**Fix:** No fix needed — this is a framing note. When presenting repurchase-interval results, be explicit that they describe the repeat-purchase segment, not the average customer.

## Suggested cleaning order

Putting the fixes above in the sequence actually used in the Power Query pipeline ([`phase-1-bi-repurchase-model/README.md`](./README.md)):

1. Parse `Date` into a proper datetime.
2. Strip `N. Revenue (formatted)` down to a numeric value (issue 1).
3. Decide which of `N. Revenue (formatted)` vs. `Net Sales` is the intended revenue metric, and document the choice (issue 2).
4. Filter out internal/operational accounts from `Customer` (issue 4).
5. Normalize `Customer` casing/whitespace, then apply the manual multi-account mapping (issue 5).
6. Split and unpivot `Product(s)` into one row per line item, explicitly handling the blank-but-nonzero-items case rather than dropping those orders (issue 3).
7. Optionally, collapse `Attribution` into a clean set of channel categories (issue 6).

---

*Findings above come from profiling the real client dataset; the dataset itself is not included in this repository. See [`phase-2-ai-automation/sample-data/`](../phase-2-ai-automation/sample-data/) for a synthetic dataset in this same format, and [`data-dictionary.md`](./data-dictionary.md) for the full column-by-column schema.*
