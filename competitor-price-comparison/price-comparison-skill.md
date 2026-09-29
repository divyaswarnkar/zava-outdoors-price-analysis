---
name: competitor-price-comparison
description: Compare a catalogue against competitor price snapshots and recommend which products to reprice. Use this skill whenever the task involves price matching, repricing recommendations, price position against competitors, margin impact of a competitor's move, or preparing a pricing decision for someone to approve — including when the request is only "what should we do about their price drop" or "run today's price check". Covers exact matching by model number, comparable matching by specification, margin-floor rules, and the approval summary the reviewer reads.
---

# Competitor price comparison

Turn the day's competitor price snapshots into a short set of recommendations a person can
approve or reject in a couple of minutes.

**You recommend. A person decides.** Never write "new price" or "updated" — everything is a
recommendation until someone approves it.

## Inputs

The caller supplies these however it chooses to pass them:

| Input | What it holds |
|---|---|
| Price snapshots | One per competitor, from today's extraction runs. Schema: `price-snapshot-schema.md` |
| Catalogue | Products to price: sku, name, brand, model_number, match_type, cost, current price, margin floor |
| Rules | Target price position, rounding, cap on a single change, exclusions. Policy: `pricing-rules.json` |

You never fetch a web page. If a competitor's snapshot is missing, note it and carry on
with the rest.

Costs and margin floors are confidential. Use them; never write them into an output.

## Environment setup

Set the environment up before you start — the run writes `recommendations.json`,
`approval-summary.md`, `approval-card.json` and `price-comparison.xlsx`, plus the
`validate_run.py` you author and execute.

- **Python 3.9+.** The matching, pricing maths and the JSON/Markdown outputs use only the
  standard library (`json`, `math`, `pathlib`, `re`, `datetime`) — no install needed.
- **`openpyxl`** is the one third-party dependency, used to build `price-comparison.xlsx`.
  Check for it first and install only if missing:

  ```bash
  python -c "import openpyxl" 2>/dev/null || python -m pip install openpyxl
  ```

- Pin it for repeatable runs by writing a `requirements.txt` next to your outputs with a
  single line, `openpyxl`, and installing with `python -m pip install -r requirements.txt`.
- No network access is required at this stage: you never fetch a page, and every input
  (snapshots, catalogue, rules) is a local file the caller supplies.
- If `openpyxl` cannot be installed, still write `recommendations.json`,
  `approval-summary.md` and `approval-card.json`, and say in the summary that the spreadsheet
  could not be generated — a named gap beats a missing file.

## Workflow

**1. Merge the snapshots.** Read every competitor's listings into one set. Check each has
today's `captured_at` — a stale snapshot is worse than a missing one, so flag it rather
than using it.

**2. Match.** Two standards, below. Record what you matched, what you rejected and why.

**3. Apply the rules.** Lowest eligible price per catalogue item, price position, the
recommended price under the rules, and margin at that price.

**4. Write the outputs.** Four files — see "What you produce".

**5. Validate.** Write `validate_run.py` alongside your outputs, checking the rules under
"Validation" below. Run it, fix the **data** it flags — never the script, never the rule —
and run again. Stop after three rounds and say plainly what is still failing.

## Matching

### Exact

For catalogue items with a `model_number`. A listing matches only when the brand matches
and the model number matches character for character after trimming whitespace and casing.

**Any difference at all is a near-miss, not a match** — including a year or revision suffix
(`AG-SP60-25` against `AG-SP60-26`). Record it with the competitor's model number, their
price, and one line on why it was rejected. Never price against a near-miss.

Model numbers usually sit in `model_number`. A `retailer_item_number` is the store's own
number and means nothing across retailers — never match on it.

This strictness matters: a differing suffix usually means a different season being cleared,
and pricing this year's stock against last year's clearance destroys margin on a false
match.

### Comparable

For everything else. No shared identifier exists, so judge from specification and intended
use.

1. **Filter by category and use.** A tent competes with a tent. An 18 L daypack does not
   compete with a 45 L backpacking pack.
2. **Line up the defining specs.** Capacity for packs, temperature rating and length for
   sleeping bags, sleeps-how-many for tents, burner count for stoves, volume for dry bags,
   lumens for headlamps, sold-singly-or-as-a-pair for poles.
3. **Pick one per competitor.** Two plausible candidates: take the closer one and note the
   other. Never average them.
4. **State the reasoning** — one sentence naming the specs that make them competitors. This
   is what the reviewer checks.

Confidence:

| Level | Means | Effect |
|---|---|---|
| `high` | Specs align, same category and use | Recommendation proceeds |
| `medium` | Clearly competing, one meaningful spec differs | Proceeds, difference named |
| `low` | Same category only | No price. Status `review`, with the question |

Be honest about confidence. An inflated `high` a reviewer catches costs more than an
accurate `low`. A catalogue item with no comparable listing is a normal result, not a
failure.

### Eligibility

Skip a listing, whatever the match type, when it is out of stock or backordered, down to
limited sizes or low stock, bundle-only, or gated behind membership. Record every exclusion
with its reason — what you ignored is as informative as what you used.

Use `sale_price` when present, otherwise `list_price`. Never `member_price`.

Clearance pricing is **not** automatically excluded — it is a real price a shopper can pay.
Include it, and say in the reasoning that it looks like clearance so the reviewer can judge
whether chasing it makes sense.

### Constraints

- **Never recommend below the margin floor.** When the rules would take you there,
  recommend **hold** and name which rule conflicted with which floor.
- Never exceed the rules' cap on a single change.
- Low-confidence comparable matches go to the reviewer as `review` with no price.

## What you produce

Four files. The approval summary is the one a person actually reads.

| File | For |
|---|---|
| `approval-summary.md` | The reviewer. Posted to chat for approval |
| `approval-card.json` | The same summary as a Teams Adaptive Card, with Approve/Reject buttons and a comment box. Layout: `output-spec.md` |
| `recommendations.json` | The workflow. Schema: `output-spec.md` |
| `price-comparison.xlsx` | The detail behind the summary. Layout: `output-spec.md` |

### The approval summary

Someone will read this on a phone, between meetings, and decide. Everything they need to
approve the easy items is in the message; anything needing thought points at the
spreadsheet.

Rules for it:

- **Under 300 words.** If it is longer, detail belongs in the spreadsheet.
- **Lead with the count**, so the reviewer knows the size of the decision before reading.
- **One line per item**, with the reason on the same line as the number.
- **Three sections, in this order:** what you recommend changing, what you are holding, what
  needs their judgment. Never mix them.
- **Every price change shows the margin effect.** A price without its margin consequence is
  not a decision.
- **Name the competitor and the price.** "Basecamp is lowest at $35" beats "a
  competitor is cheaper".
- **No preamble.** Do not restate the task or explain what the pipeline is.
- **Say when nothing happened.** A quiet day is a valid result and a short message.

Shape:

```markdown
**Daily price check — 28 Sep**
2 to reprice · 1 holding · 1 needs your call

**Recommend repricing**

| Product | Now | Recommend | Why | Margin |
|---|---|---|---|---|
| Lumen 400 Headlamp | $39.00 | $35.00 | Basecamp lowest at $35, we're now highest | 59% → 54% |
| Trailhead 2P Tent | $219.00 | $198.00 | Timberline clearance at $198, 10% below us | 46% → 40% |

**Holding**

- **Emberlite Stove** — Timberline is at $39.00. Matching would put us at ~$41,
  below our floor. Holding at $59.00.

**Needs your call**

- **Alpenglow Summit Pro 60** — Basecamp lists $219 against our $279, but it's model
  AG-SP60-25, last season's. Ours is AG-SP60-26. Timberline has the current model at $265.
  Match the current model, or treat theirs as clearance?

_No change on 8 other products. Full workings in the attached spreadsheet._
```

Adjust the sections to the day: drop any that is empty, and say so in the count line.

## Validation

Write the script. Fix the data, not the rule.

**Blocking**

- **V1** No recommended price is below its margin floor.
- **V2** Every exact match has identical brand and model number. A differing suffix is a
  near-miss, never a match.
- **V3** Every listing used traces to a snapshot with a source URL.
- **V4** Status agrees with price: `reprice` changes it, `hold` equals the current price,
  `review` has none.
- **V5** No change exceeds the rules' cap.
- **V6** One recommendation per SKU, and every SKU is in the catalogue.
- **V7** Every comparable match has reasoning; every recommendation has a rationale; every
  exclusion has a reason.
- **V8** No cost or margin-floor figure appears in any output.
- **V9** The summary's counts match the recommendations. If it says two repricings, there
  are two.
- **V10** All four outputs exist and are non-empty.

**Warnings** — report, and mention in the summary when they fire:

- **W1** A catalogue item with no match at any competitor.
- **W2** A recommendation driven by exactly one competitor listing.
- **W3** More than a third of comparable matches rated low confidence.
- **W4** A recommendation moving a price more than 10%, even inside the cap.
- **W5** A recommendation driven by a listing that looks like clearance.
- **W6** A missing or stale competitor snapshot.

**Not checkable in code** — state these in the summary instead: whether a comparable match
is genuinely comparable, whether chasing a clearance price makes commercial sense, and
whether a competitor's move is a promotion or a permanent reposition.

## Judgment worth getting right

**Explain, don't just report.** "Competitor dropped $40" is the observation. "Competitor
dropped $40 on last season's model, which they're likely clearing" is the analysis. The
second is what the reviewer needs.

**A cheaper competitor is not always a reason to move.** Clearance, a discontinued model, a
thin-stock listing — each is a reason to hold. Say so.

**Flag what you are unsure about** rather than picking a plausible answer. Three solid
recommendations and two flagged questions beat five confident guesses.

## Reference files

- `price-snapshot-schema.md` — the input contract from the extraction agents
- `output-spec.md` — `recommendations.json` fields and the spreadsheet layout
- `pricing-rules.json` — the pricing policy: target position, rounding, the cap on a single change, and exclusions
