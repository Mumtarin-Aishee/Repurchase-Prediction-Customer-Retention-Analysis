# Repurchase Prediction & Customer Retention Analysis — DTC Home Fragrance Brand

`Power BI` `DAX` `Power Query (M)` `Excel`


## Problem

The client had no systematic way to know when a repeat customer was due to reorder. Order tracking depended on manual review of large order dataset, so lapsing high-value accounts could go unnoticed.

## Data & Sources

Raw order-level CSV export from Woocommerce (Jan '25–Aug '26) with Date, Order #, Revenue (currency-formatted text), Customer, a multi-product field (up to 14 products per order, delimited), Coupons, Net Sales, Attribution, and Invoice Number.

## Data Transformation (Power Query)

Built a Power Query pipeline to turn a messy multi-product order export into an analysis-ready table:

- Promoted headers and set column types
- Parsed `Date` into Month/Year
- Extracted numeric Revenue from a currency-prefixed text field
- Split the delimited Product(s) column into up to 14 slots, then **unpivoted** them into one row per product line
- Parsed each product entry into ordered quantity + product name
- Filtered out non-order junk rows and reordered the final columns

```
#"Split Column by Delimiter1" = Table.SplitColumn(#"Changed Type1", "N. Revenue (formatted)",
    Splitter.SplitTextByDelimiter("৳", QuoteStyle.Csv), {"N. Revenue (formatted).1", "N. Revenue (formatted).2"}),
#"Unpivoted Columns" = Table.UnpivotOtherColumns(#"Renamed Columns1",
    {"Date","Order #","Revenue","Status","Customer","Customer type","Items sold","Coupon(s)","Net Sales","Attribution","Invoice Number","Month","Year"},
    "Attribute", "Value"),
```

## Data Modeling (DAX)

**Calculated column — Previous Purchase Date** (per-customer self-lookup):

```DAX
VAR CurrentCustomer = 'Customer Order'[Customer]
VAR CurrentDate = 'Customer Order'[Purchase Date]
RETURN
CALCULATE(
    MAX('Customer Order'[Purchase Date]),
    FILTER(ALL('Customer Order'), 'Customer Order'[Customer] = CurrentCustomer && 'Customer Order'[Purchase Date] < CurrentDate)
)
```

**Calculated column — Repurchase Interval:**

```DAX
VAR CurrentPurchaseDate = 'Customer Order'[Purchase Date]
VAR PreviousPurchaseDate = 'Customer Order'[Previous Purchase Date]
RETURN
IF(ISBLANK(PreviousPurchaseDate), BLANK(), DATEDIFF(PreviousPurchaseDate, CurrentPurchaseDate, DAY))
```

**Measure — Repurchase Interval (Days):**

```DAX
MEDIANX(
    FILTER('Customer Order', NOT ISBLANK('Customer Order'[Repurchase Interval])),
    'Customer Order'[Repurchase Interval]
)
```

Uses **median**, not average — a single unusually long gap (a customer who churned and returned once) would otherwise distort that customer's reminder timing.

**Measure — Total Revenue 2026:**

```DAX
CALCULATE(SUM(Orders[Revenue]), Orders[Year] = "2026")
```

## Findings

- 16 repeat customers identified with documented repurchase patterns (intervals from 9 to 196 days), representing ৳6,456,289 in combined 2026 revenue.
- Built a proactive reminder schedule (Sep–Dec 2026): `Reminder Date = Last Purchase + Repurchase Interval − 3-day processing buffer`.
- Identified the top 10 new high-value accounts by 2026 revenue with no repeat purchase yet, flagged for retention outreach.

## Recommendations

- Automate the reminder schedule into a recurring trigger rather than a one-off manual run.
- Prioritize the two highest-frequency buyers (9–12 day intervals) first — they generate the most reminder touchpoints per quarter.
- Track actual reorder conversion from reminders sent, to replace the interval estimate with a real response-rate model over time.

## Assumptions & Limitations

The median-interval estimate assumes fairly regular buying behavior; it will be less reliable for a customer who returns after an unusually long gap (e.g. the 196-day case). The model currently relies on Power BI's auto-generated date tables rather than a single shared, explicitly marked Date dimension — a cleanup item for the next iteration.

## Privacy note

Client name, customer names, and exact revenue figures are anonymized or replaced with illustrative sample data throughout this project's public files. The underlying `.pbix` and raw Excel exports contain real customer data and are intentionally **not** included in this repository.

