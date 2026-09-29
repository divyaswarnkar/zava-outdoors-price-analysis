# price-snapshot.json schema

The contract with the analysis stage. Field names are parsed downstream, so they are not
negotiable. Prices are numbers, not strings. Timestamps are ISO 8601 UTC.

## Shape

```json
{
  "run": {
    "competitor": "Timberline Trading Post",
    "run_id": "2026-09-28",
    "captured_at": "2026-09-28T06:04:12Z",
    "urls_given": 1,
    "urls_read": 1,
    "pages_read": 1,
    "listing_total_stated": 13,
    "products_recorded": 13
  },
  "listings": [
    {
      "listing_id": "timberline-0007",
      "product_name": "Wanderlight 2P Tent",
      "brand": null,
      "model_number": null,
      "retailer_item_number": null,
      "category_shown": "Camp & trail clearance",
      "configuration": null,
      "description_shown": "Three-season backpacking tent · two doors · 4 lb 4 oz",
      "list_price": 249.00,
      "sale_price": 198.00,
      "member_price": 178.20,
      "currency": "USD",
      "price_note": "Clearance badge shown on the listing",
      "is_bundle": false,
      "bundle_contents": null,
      "availability": "in_stock",
      "availability_text": "In stock",
      "source_url": "https://example.com/timberline-trading/",
      "captured_at": "2026-09-28T06:04:12Z"
    }
  ],
  "errors": [
    {
      "url": "https://example.com/timberline-trading/page2",
      "problem": "Returned 503 on both attempts",
      "attempts": 2
    }
  ]
}
```

## Fields

### run

| Field | Type | Notes |
|---|---|---|
| `competitor` | string | Exactly as the caller named it |
| `run_id` | string | The date of the run, `YYYY-MM-DD` |
| `captured_at` | string | When the run started, ISO 8601 UTC |
| `urls_given` | number | How many URLs you were handed |
| `urls_read` | number | How many you successfully read |
| `pages_read` | number | Including pagination |
| `listing_total_stated` | number or null | The count the page claimed, if any ("showing 1–24 of 60" → 60) |
| `products_recorded` | number | Must equal the length of `listings` |

### listings

| Field | Type | Notes |
|---|---|---|
| `listing_id` | string | Unique within the run. Competitor slug plus a counter is fine |
| `product_name` | string | Verbatim from the page |
| `brand` | string or null | Only when shown separately from the name |
| `model_number` | string or null | Manufacturer model, character for character |
| `retailer_item_number` | string or null | The store's own number |
| `category_shown` | string or null | The section or heading it appeared under |
| `configuration` | string or null | Null when the product has one configuration |
| `description_shown` | string or null | The specification line, verbatim |
| `list_price` | number or null | See the price rules below |
| `sale_price` | number or null | |
| `member_price` | number or null | |
| `currency` | string | ISO code, `USD` unless the page says otherwise |
| `price_note` | string or null | Qualifiers, badges, or why a price is null |
| `is_bundle` | boolean | True only when the page signals a bundle/kit/set of multiple separately-named products, or a multi-pack quantity. A product whose description lists its own parts ("pots and lids") is not a bundle |
| `bundle_contents` | array of strings or null | Components as named on the page; never invented |
| `availability` | string | One of `in_stock`, `limited`, `out_of_stock`, `backordered`, `unknown` |
| `availability_text` | string or null | Verbatim |
| `source_url` | string | The page this came from |
| `captured_at` | string | ISO 8601 UTC |

### errors

| Field | Type | Notes |
|---|---|---|
| `url` | string | The URL that failed |
| `problem` | string | What happened, plainly |
| `attempts` | number | How many times you tried |

## Price rules

**One price shown** → `list_price`. The other two null.

**Struck-through plus current** → the struck-through in `list_price`, the current in
`sale_price`.

**Member price present** → always its own field, never in place of another.

**No usable price** → all three null, with the reason in `price_note`. Record the product
regardless.

Never compute a price. No unit prices from multi-packs, no totals from per-item figures,
no currency conversion, no applying a percentage the banner advertises.

## Configuration rows

One listing per configuration. Rows sharing a product repeat `product_name` and differ
in `configuration`, `price` and possibly `availability`.

```json
{ "product_name": "Northfork Backcountry Tent", "configuration": "2 person", "list_price": 244.00, "availability": "in_stock" },
{ "product_name": "Northfork Backcountry Tent", "configuration": "3 person", "list_price": 289.00, "availability": "in_stock" },
{ "product_name": "Northfork Backcountry Tent", "configuration": "4 person", "list_price": 339.00, "availability": "backordered" }
```

Keep every dimension the page distinguishes, joined with a slash: `"20°F / Long"`,
`"45 L / M–L torso"`.

## Fields that must never appear

Adding these corrupts the evidence, because the analyst cannot separate your guess from a
fact on the page:

`matched_sku`, `confidence`, `eligible`, `excluded`, `exclusion_reason`, `comparable_to`,
`price_position`, `recommendation`, or any field naming another retailer's product.
