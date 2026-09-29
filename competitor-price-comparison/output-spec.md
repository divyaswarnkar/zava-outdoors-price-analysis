# Output spec

`recommendations.json` is parsed by the workflow, so field names are not negotiable. Prices
are numbers, not strings. Dates are ISO 8601 UTC.

The approval summary's format lives in SKILL.md, since it is the output that matters most.

## recommendations.json

```json
{
  "run": {
    "run_id": "2026-09-28",
    "generated_at": "2026-09-28T06:41:12Z",
    "competitors_expected": 3,
    "competitors_received": 3,
    "previous_run_found": true
  },
  "recommendations": [
    {
      "sku": "ZO-HDL-400",
      "product_name": "Lumen 400 Headlamp",
      "match_type": "comparable",
      "confidence": "high",
      "status": "reprice",
      "current_price": 39.00,
      "recommended_price": 35.00,
      "lowest_eligible_price": 35.00,
      "lowest_eligible_competitor": "Basecamp Supply Co.",
      "price_position_pct": 11.4,
      "margin_now_pct": 59.0,
      "margin_after_pct": 54.3,
      "rationale": "Basecamp cut to $35.00 overnight, leaving us the highest of three.",
      "changed_since_yesterday": true
    }
  ],
  "near_misses": [
    {
      "sku": "ZO-BR-AG-SP60",
      "competitor": "Basecamp Supply Co.",
      "listed_name": "Summit Pro 60 Pack (2025)",
      "their_model_number": "AG-SP60-25",
      "our_model_number": "AG-SP60-26",
      "price": 219.00,
      "reason": "Prior-season model; suffix differs. Likely clearance, not like-for-like."
    }
  ],
  "matches_used": [
    {
      "sku": "ZO-HDL-400",
      "competitor": "Basecamp Supply Co.",
      "listing_id": "basecamp-0011",
      "listed_name": "Beam 400 Headlamp",
      "price_used": 35.00,
      "price_field": "list_price",
      "match_type": "comparable",
      "confidence": "high",
      "reasoning": "Both 400 lumen USB-C rechargeable headlamps.",
      "eligible": true,
      "exclusion_reason": null
    }
  ],
  "changes_since_yesterday": [
    {
      "competitor": "Basecamp Supply Co.",
      "listed_name": "Beam 400 Headlamp",
      "previous_price": 39.00,
      "current_price": 35.00,
      "change_pct": -10.3,
      "affects_sku": "ZO-HDL-400"
    }
  ],
  "no_match": [
    { "sku": "ZO-POL-AL", "reason": "No comparable pole set at any competitor today." }
  ],
  "warnings": [
    { "code": "W2", "sku": "ZO-TNT-2P", "detail": "Driven by a single competitor listing." }
  ],
  "missing_snapshots": []
}
```

### status

| Value | Meaning | `recommended_price` |
|---|---|---|
| `reprice` | A change is recommended | Differs from `current_price` |
| `hold` | No change; `rationale` says why | Equals `current_price` |
| `review` | Needs a human decision | `null` |

Use `hold` when the rules produce no change or a constraint blocks one. Use `review` when
the matching itself is uncertain.

### Number rules

- `price_position_pct` is positive when the catalogue price is the more expensive.
- Margin percentages are computed from cost, but **cost itself never appears in the file**.
  Nor does the margin floor.
- Money to two decimals, percentages to one.

## price-comparison.xlsx

Build with `openpyxl`. Four sheets, in this order. The reviewer opens this only when the
summary raises a question, so optimise for answering "why did it say that".

**Recommendations** — one row per catalogue item with a status:
SKU, product, match type, confidence, current price, recommended price, change, status,
lowest competitor, price position, margin now, margin after, rationale.
Freeze the header. Colour the status cell: green `reprice`, amber `hold`, grey `review`.

**Price position** — one row per SKU, one column per competitor, cells holding the matched
price. Em dash where there is no match, and a note where a price was excluded. Final column
holds our own price.

**Exact matches** — only items with a model number: SKU, brand, our model number, each
competitor's model number and price, and a flag where a near-miss was found. This is the
sheet that makes the exact/comparable distinction visible.

**Sources** — every listing used or considered: competitor, listed name, price used, which
price field it came from, source URL, matched SKU, confidence, exclusion reason. Every
number anywhere in the workbook traces to a row here.

Formatting: currency format on money columns, percentage format on percentages, column
widths set so nothing truncates, wrap text on rationale columns.
