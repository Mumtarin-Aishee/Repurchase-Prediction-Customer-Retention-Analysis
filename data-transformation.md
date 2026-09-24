## Data Transformation (Power Query)

Built a Power Query pipeline to turn the raw order export into an analysis-ready table. Beyond standard cleanup (headers, types, date parts), every one of the seven issues catalogued in [`data-quality-issues.md`](./data-quality-issues.md) is accounted for below — six resolved in the pipeline itself, one deliberately scoped out to the reporting layer instead:

| Step | Issue | Resolution |
|---|---|---|
| 1 | — | Promote headers, set column types, extract Month/Year from `Date` |
| 2 | [Issue 1](./data-quality-issues.md#1-revenue-is-stored-as-formatted-text-not-a-number) — revenue stored as text | Extract numeric revenue from the currency-prefixed text field |
| 3 | [Issue 2](./data-quality-issues.md#2-n-revenue-formatted-and-net-sales-dont-agree) — Revenue vs. Net Sales disagree | Pick `Revenue` as the canonical value field; document why |
| 4 | [Issue 4](./data-quality-issues.md#4-internaloperational-accounts-are-mixed-into-the-customer-list) — internal accounts in `Customer` | Filter out internal/operational accounts via a maintained exclude-list |
| 5 | [Issue 5](./data-quality-issues.md#5-the-same-customer-appears-under-multiple-name-spellings) — name/account variants | Normalize customer names; merge known multi-account customers |
| 6 | [Issue 3](./data-quality-issues.md#3-products-is-blank-on-orders-that-clearly-have-items) — blank `Product(s)` | Flag blank rows, then split and unpivot into one row per line item |
| 7 | [Issue 6](./data-quality-issues.md#6-attribution-values-are-inconsistently-formatted) — inconsistent `Attribution` | Standardize into a clean set of channels via a mapping table |
| 8 | — | Filter remaining junk rows; reorder final columns |
| — | [Issue 7](./data-quality-issues.md#7-the-customer-base-is-heavily-skewed-toward-one-time-buyers) — one-time-buyer skew | **Not fixed here.** Not a data error, so there's nothing to transform — handled at the reporting layer instead, where the repurchase-interval model scopes explicitly to the repeat-purchase segment rather than treating the full customer list as representative. |

### 1. Headers, types, date parts

```m
#"Promoted Headers" = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
#"Changed Type" = Table.TransformColumnTypes(#"Promoted Headers",
    {{"Date", type datetime}, {"Order #", Int64.Type}, {"Items sold", Int64.Type},
     {"Net Sales", Int64.Type}, {"Invoice Number", Int64.Type}}),
#"Extracted Date Parts" = Table.AddColumn(
    Table.AddColumn(#"Changed Type", "Month", each Date.Month([Date]), Int64.Type),
    "Year", each Date.Year([Date]), Int64.Type)
```

### 2. Extract numeric revenue

`N. Revenue (formatted)` arrives as `"৳2,999"` — currency symbol and thousands separator baked into text. Split on the symbol, strip the remaining comma, cast to number.

```m
#"Split Column by Delimiter1" = Table.SplitColumn(#"Changed Type1", "N. Revenue (formatted)",
    Splitter.SplitTextByDelimiter("৳", QuoteStyle.Csv),
    {"N. Revenue (formatted).1", "N. Revenue (formatted).2"}),
#"Extracted Revenue Value" = Table.TransformColumns(
    Table.RenameColumns(#"Split Column by Delimiter1",
        {{"N. Revenue (formatted).2", "Revenue"}}),
    {{"Revenue", each Number.From(Text.Remove(_, ",")), type number}})
```

### 3. Pick the canonical revenue field

`Revenue` and `Net Sales` disagree on most rows by a small, consistent amount — most likely a delivery charge included in one and netted out of the other. Rather than reconciling them, the pipeline picks `Revenue` for every downstream calculation and documents that choice, so nothing later mixes the two definitions.

```m
#"Documented Revenue Basis" = Table.AddColumn(#"Extracted Revenue Value",
    "Revenue Basis Note", each
        "Order-level revenue uses [Revenue] (gross). [Net Sales] is retained " &
        "for reference only and is not used in customer-value calculations.",
    type text)
```

### 4. Filter internal and operational accounts

A handful of `Customer` values are store-side accounts, not paying customers. Kept in a small maintained reference table rather than caught by an order-count threshold, which would either miss them or wrongly exclude a genuine high-frequency customer.

```m
// Separate query: "Excluded Accounts" — one column, Customer, maintained manually
#"Filtered Internal Accounts" = Table.SelectRows(#"Documented Revenue Basis",
    each not List.Contains(ExcludedAccounts[Customer], [Customer]))
```

### 5. Normalize and merge customer identities

A cheap normalization pass catches casing/whitespace variants of the same name. A second pass joins against a manually maintained mapping table ([`customer-mapping.example.csv`](../phase-2-ai-automation/config/customer-mapping.example.csv)) to merge the deeper cases — one person ordering under two genuinely different names or emails, which normalization alone can't catch.

```m
#"Trimmed Customer Name" = Table.TransformColumns(#"Filtered Internal Accounts",
    {{"Customer", each Text.Trim(Text.Proper(_)), type text}}),

// Separate query: "Customer Mapping" — Customer, CanonicalName
#"Merged Customer Mapping" = Table.NestedJoin(#"Trimmed Customer Name", {"Customer"},
    CustomerMapping, {"Customer"}, "MappingMatch", JoinKind.LeftOuter),
#"Applied Canonical Name" = Table.TransformColumns(
    Table.ExpandTableColumn(#"Merged Customer Mapping", "MappingMatch", {"CanonicalName"}),
    {{"CanonicalName", each if _ = null then [Customer] else _, type text}})
```

### 6. Handle blanks, then split and unpivot products

About 1 in 9 orders has no itemized `Product(s)` value despite a non-zero `Items sold` count. Flagging and placeholding these *before* the unpivot keeps them visible in product-level output instead of silently vanishing.

```m
#"Flagged Unitemized Orders" = Table.AddColumn(#"Applied Canonical Name",
    "HasItemizedProducts", each [Product(s)] <> null and [Product(s)] <> "", type logical),
#"Replaced Blank Products" = Table.ReplaceValue(#"Flagged Unitemized Orders",
    null, "(Unspecified — no itemized products in export)",
    Replacer.ReplaceValue, {"Product(s)"}),

#"Split Product Slots" = Table.SplitColumn(#"Replaced Blank Products", "Product(s)",
    Splitter.SplitTextByDelimiter(", ", QuoteStyle.Csv),
    List.Transform({1..14}, each "Product(s)." & Text.From(_))),
#"Renamed Columns1" = Table.RenameColumns(#"Split Product Slots", {}),

#"Unpivoted Columns" = Table.UnpivotOtherColumns(#"Renamed Columns1",
    {"Date","Order #","Revenue","Status","Customer","Customer type","Items sold",
     "Coupon(s)","Net Sales","Attribution","Invoice Number","Month","Year"},
    "Attribute", "Value"),

#"Parsed Product Quantity and Name" = Table.SplitColumn(
    Table.SelectRows(#"Unpivoted Columns", each [Value] <> null), "Value",
    Splitter.SplitTextByDelimiter("× ", QuoteStyle.Csv), {"Quantity", "Product Name"})
```

### 7. Standardize marketing channels

Over 20 raw `Attribution` values collapse into a much smaller set of real channels once casing and near-duplicate platform variants are mapped together.

```m
// Separate query: "Channel Mapping" — RawAttribution, CleanChannel
#"Merged Channel Mapping" = Table.NestedJoin(#"Parsed Product Quantity and Name",
    {"Attribution"}, ChannelMapping, {"RawAttribution"}, "ChannelMatch", JoinKind.LeftOuter),
#"Applied Clean Channel" = Table.TransformColumns(
    Table.ExpandTableColumn(#"Merged Channel Mapping", "ChannelMatch", {"CleanChannel"}),
    {{"CleanChannel", each if _ = null then [Attribution] else _, type text}})
```

### 8. Final filter and column order

```m
#"Filtered Junk Rows" = Table.SelectRows(#"Applied Clean Channel",
    each [Order #] <> null and [Order #] > 0),
#"Reordered Final Columns" = Table.ReorderColumns(#"Filtered Junk Rows",
    {"Date","Year","Month","Order #","Customer","CanonicalName","Customer type",
     "Product Name","Quantity","HasItemizedProducts","Revenue","Net Sales",
     "CleanChannel","Coupon(s)","Status","Invoice Number"})
```
