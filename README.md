# Zava Outdoors — price analysis

A GitHub Copilot **skill** that turns a day's competitor price snapshots into a short set of
repricing recommendations a person can approve in a couple of minutes.

The skill recommends; a person decides. Nothing it produces goes live on its own, and it
never fetches a web page — it works only from snapshots the extraction agents captured
earlier in the run.

## What's here

```
competitor-price-comparison/
    price-comparison-skill.md    # the workflow: match, apply rules, validate, summarise
    price-snapshot-schema.md     # input contract — the competitor snapshot format
    output-spec.md               # output contract — recommendations.json + spreadsheet layout
    pricing-rules.json           # pricing policy: target position, rounding, change cap, exclusions
```

All four files live in a single folder; the skill links to the other three by filename.

## How it works

1. **Merge** every competitor's `price-snapshot.json` into one view.
2. **Match** each catalogue item — exact (same brand and model number) or comparable (a
   judgment from specifications, carrying a confidence level).
3. **Apply the rules** in `pricing-rules.json` — lowest eligible price, target position, the
   cap on a single change, exclusions, rounding — never below the margin floor.
4. **Validate** with a script the run writes and executes (blocking checks V1–V10).
5. **Produce** four outputs: `approval-summary.md` (the one a reviewer reads),
   `approval-card.json` (the same summary as a Teams Adaptive Card with Approve/Reject and a
   comment box), `recommendations.json`, and `price-comparison.xlsx`.

## Inputs the caller supplies at run time

The skill lives here; the run-time data does not. A caller provides:

- **Price snapshots** — one per competitor, matching `price-snapshot-schema.md`.
- **The catalogue** — the products to price, carrying `sku`, `current_price`, `unit_cost`,
  `margin_floor_price` and `match_type`. Costs and margin floors are confidential and never
  appear in any output.
- **The rules** — `pricing-rules.json`.
