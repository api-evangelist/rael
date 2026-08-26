---
name: rael-catalog-lookup
description: Read Rael's product catalog — search, look up and price organic period-care, intimate-care and skincare products — without creating a cart, a checkout or any other state on the store.
api: Rael UCP Commerce MCP
endpoint: https://www.getrael.com/api/ucp/mcp
transport: MCP over HTTP (JSON-RPC 2.0)
operations:
  - search_catalog
  - get_product
  - lookup_catalog
generated: '2026-08-26'
method: generated
source: mcp/rael-mcp-tools.json (verbatim tools/list, probed 2026-08-26) + https://www.getrael.com/llms.txt
---

# Reading Rael's catalog

Three tools, all read-only, all anonymous. Use this skill when the buyer is comparing options,
checking availability, or asking about price — and stop here. Do not create a cart to answer a
question about a price.

## Required on every call

`meta.ucp-agent.profile` — your UCP agent profile URI. Without it the endpoint answers HTTP 422 with
`-32001 / invalid_profile_url` regardless of which tool you named.

## The tools

- **`search_catalog`** — `catalog.query` is a plain search string. Narrow with
  `catalog.filters.categories[]` (OR logic), `catalog.filters.price.min` / `.max` in **minor
  currency units**, and `catalog.filters.available` (defaults true — sale-ready items only; set it
  false if the buyer wants to see out-of-stock items too). Page with `catalog.pagination.cursor` and
  `catalog.pagination.limit`.
- **`get_product`** — one `catalog.id`, plus `catalog.selected[]` `{name, label}` pairs to resolve a
  specific variant, and `catalog.preferences[]`.
- **`lookup_catalog`** — several known product or variant identifiers in one call. Prefer this over
  looping `get_product`.

## Localization

Send `catalog.context`: `address_country` (ISO 3166-1 alpha-2), `address_region`, `postal_code`,
`language` (BCP 47) and `currency` (ISO 4217). Rael ships to 13 countries, and price and
availability both move with these fields. Omitting them gives you a default-market answer that may
be wrong for your buyer.

## Reading prices correctly

Every amount is an integer in the currency's minor units, paired with a currency code —
`{"amount": 2500, "currency": "USD"}` is $25.00. Convert before quoting. Zero-decimal currencies
such as JPY are already whole units.

## The no-MCP fallback

Rael's own `llms.txt` documents unauthenticated read-only storefront routes if you cannot speak MCP:
`GET /products/{handle}.json`, `GET /collections/{handle}/products.json`,
`GET /search?q={query}&type=product` and `GET /sitemap.xml` on `https://www.getrael.com`. These are
plain JSON and need no agent profile, but they carry no localized pricing and no pagination cursor.
