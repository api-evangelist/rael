---
name: rael-agent-purchase
description: Buy Rael period-care, intimate-care and skincare products on behalf of a buyer through Rael's anonymous UCP commerce MCP endpoint, from catalog search to a completed order, without ever completing payment the buyer has not approved.
api: Rael UCP Commerce MCP
endpoint: https://www.getrael.com/api/ucp/mcp
transport: MCP over HTTP (JSON-RPC 2.0)
operations:
  - search_catalog
  - get_product
  - lookup_catalog
  - create_cart
  - update_cart
  - get_cart
  - cancel_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
generated: '2026-08-26'
method: generated
source: mcp/rael-mcp-tools.json (verbatim tools/list, probed 2026-08-26) + https://www.getrael.com/llms.txt
---

# Buying from Rael as an agent

Rael's storefront speaks the Universal Commerce Protocol over MCP. The endpoint is anonymous —
there is no API key and no OAuth flow — so the access control that matters is the **buyer**, not a
credential. Everything below uses tool names verified against a live `tools/list` on 2026-08-26.

## Before you call anything

1. `GET https://www.getrael.com/.well-known/ucp` to confirm the protocol version the store is on
   (currently `2026-04-08`; `2026-01-23` is still supported).
2. Every single tool requires `meta.ucp-agent.profile` — your agent profile URI. Omit it and the
   endpoint returns HTTP 422 with `-32001 / invalid_profile_url`, before your tool ever runs. This
   is the most common failure on this surface.
3. Pass `context.address_country` and `context.currency` on catalog and cart calls so prices and
   availability come back correct for your buyer.

## Find the product

- `search_catalog` — natural-language `catalog.query`, optional `catalog.filters.categories`,
  `catalog.filters.price.min` / `.max` (minor units), `catalog.filters.available` (defaults to true,
  i.e. sale-ready only). Page with `catalog.pagination.cursor` and `catalog.pagination.limit`.
- `get_product` — full detail for one `catalog.id`, with `catalog.selected[]` to pin variant options.
- `lookup_catalog` — resolve several known product or variant identifiers at once.

Rael's line is period care, intimate care, skincare and cycle supplements; queries phrased in the
buyer's own words ("organic cotton tampons", "PMS supplement") are what `search_catalog` expects.

## Build the cart

- `create_cart` with `cart.line_items[]`, each `{item: {id: <variant id>}, quantity: n}`. The id is a
  **product variant** id, not a product id.
- `update_cart` with the cart `id` to change quantities or add lines.
- `get_cart` to re-read state before you show the buyer anything.
- `cancel_cart` to abandon it. Do this rather than leaving carts open.

## Checkout

- `create_checkout` — either pass `checkout.cart_id` to convert an existing cart, or supply
  `checkout.line_items[]` directly. Add `checkout.buyer.email`, fulfillment via
  `checkout.fulfillment.methods[]`, and any `checkout.discounts.codes[]`.
- `update_checkout` — set or change the shipping destination and method. **Discount codes replace
  the previous set on every submission**, so resend the full list, and only prompt for a code if the
  buyer mentions having one.
- `get_checkout` — read line items, totals, discounts and taxes back.
- `cancel_checkout` — the reversal for everything above. It works right up until completion.

Prices are integers in ISO 4217 minor units paired with a currency code: `{"amount": 600,
"currency": "USD"}` is $6.00. Divide by 100 for two-decimal currencies before you quote anything to
a human. Getting this wrong quotes a buyer 100x the price.

## Complete — the one step you cannot take back

`complete_checkout` requires **two** things you must get right:

- `meta.idempotency-key` is **required**. Generate one per completion attempt and reuse the same key
  on any retry, so a network timeout does not become a second charge.
- Contemporaneous buyer approval. Rael states plainly in its own `llms.txt`: agents must not
  complete payment without explicit buyer consent. If you cannot get approval at the moment of
  payment, do not complete — route the purchase through Shop Pay via `https://shop.app/SKILL.md`
  instead.

There is no `refund` or `void` tool. Once `complete_checkout` returns an order id, the only reversal
is Rael's human return process: email `support@getrael.com` within **30 days of purchase**, items
unopened, unused and in original packaging; accepted returns are refunded as **store credit**, less
the return-label cost, and sale items and gift cards are not returnable
(https://www.getrael.com/policies/refund-policy). Tell the buyer this *before* they approve, not
after.

## Afterwards

`get_order` with the order id (`gid://shopify/Order/123`) reads the order back.

## Errors and limits

- Errors are JSON-RPC error objects at **HTTP 422**, shaped
  `{error: {code, message, data: {code, content, continue_url}}}`. Branch on the status before you
  parse. `data.continue_url` is a human-usable store URL to hand the buyer when you are stuck.
- The endpoint is rate limited per IP with no published number and no `Retry-After`. Back off
  exponentially on 429. Watch `shopify-complexity-score-v2` on responses — it is the only
  forward-looking cost signal you get.
- Only `complete_checkout` is idempotent. Nothing else takes a key, so do not assume a retry of
  `create_cart` or `create_checkout` is free of side effects — read back with `get_cart` /
  `get_checkout` instead.
