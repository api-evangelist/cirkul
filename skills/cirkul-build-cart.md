---
name: Build a Cirkul cart and hand off for buyer approval
description: Assemble a Cirkul cart — line items, buyer identity, delivery address,
  shipping selection, discount and gift-card codes — then hand the checkout URL to
  the human. Payment is never completed by the agent.
api: mcp/cirkul-mcp.yml
surface: https://drinkcirkul.com/api/mcp
transport: MCP (JSON-RPC 2.0) over HTTP
auth: none
operations:
- search_catalog
- get_product_details
- update_cart
- get_cart
generated: '2026-08-02'
method: generated
source: mcp/cirkul-mcp-tools-list.json
---

# Build a Cirkul cart and hand off for buyer approval

## The rule that governs this whole skill

Cirkul states in three separate published places — `/agents.md`, `/llms.txt` and
`/robots.txt` — that checkout, payment and order placement must not be completed
automatically. No scripted form fills, no browser automation, no end-to-end flow
that finalizes payment without an explicit, contemporaneous human approval step.

You may build the cart. You may not buy. If you cannot get the buyer in front of
the approval moment, Cirkul directs agents to install `https://shop.app/SKILL.md`
and route the purchase through Shop Pay instead of driving the storefront yourself.

## Step 1 — Resolve variants

Use `search_catalog` and `get_product_details` (see `cirkul-find-flavors.md`) until
you hold a concrete `product_variant_id` and quantity for every line. Cart
operations key on the **variant** id.

Keep `context.address_country` and `context.currency` identical to what you will use
for the cart, or the totals will not match what you quoted.

## Step 2 — Create the cart (`update_cart`)

`update_cart` is the single consolidated mutation — there is no separate create
tool on this endpoint. **Omit `cart_id` and the server creates a new cart.**

- `add_items` — array of `{product_variant_id, quantity}`. Required when creating.
  Quantity minimum is 1.

Keep the returned `cart_id`. It is a Shopify GID, shaped like
`gid://shopify/Cart/{id}?key={key}` — the `key` query parameter is part of the
identifier, not decoration. Strip it and subsequent calls fail.

## Step 3 — Add buyer identity and delivery address (`update_cart`)

Same tool, now with `cart_id`.

- `buyer_identity` — `email`, `phone`, `country_code`. The country code drives
  regional pricing.
- `delivery_addresses_to_add` — each entry wraps a `delivery_address` with
  `first_name`, `last_name`, `phone`, `address1`, `address2`, `city`,
  `province_code`, `zip`, `country_code`, plus a `selected` boolean. The first
  address added defaults to selected.
- `delivery_addresses_to_replace` replaces the whole set; passing it empty removes
  every address. Use `_to_add` unless you intend that.

Only ever send an address and email the buyer actually gave you for this purchase.

## Step 4 — Select shipping (`update_cart`)

Shipping options do not exist until the cart has **both** items and a delivery
address. Call `get_cart` after step 3 to read the available options, then set
`selected_delivery_options`.

## Step 5 — Apply codes (`update_cart`)

- `discount_codes` — array of codes.
- `gift_card_codes` — array of codes.
- `note` — free text on the order.

Never invent or brute-force discount codes. Apply only codes the buyer supplied or
that Cirkul is publicly advertising, and re-read the total afterwards — a rejected
code changes nothing but is not always loud about it.

## Step 6 — Adjust or remove lines (`update_cart`)

- `update_items` — `{id, quantity}` against the **line item** id (from `get_cart`),
  not the variant id. Quantity `0` removes the line.
- `remove_line_ids` — explicit removal by line id.

## Step 7 — Read back and hand off (`get_cart`)

Call `get_cart` with `cart_id`. It returns items, shipping options, discount info
and the **checkout URL**.

Present the human with the itemized cart, the shipping method, the discounts
applied and the final total, then give them the checkout URL and stop. Do not open
it, fill it, or click through it on their behalf.

## Errors

- `429` — per-IP rate limit. Back off.
- JSON-RPC error objects carry `error.code`, `error.message` and `error.data.code`.
  See `errors/cirkul-problem-types.yml`.
- There is **no idempotency key** on this API. A retried `update_cart` with
  `add_items` is not deduplicated and will add the items again. On an ambiguous
  failure, call `get_cart` and reconcile before retrying.

## If you need checkout, order status, or batch lookup

Those live on the UCP shopping endpoint at `https://drinkcirkul.com/api/ucp/mcp`
(`create_checkout`, `update_checkout`, `complete_checkout`, `get_order`,
`lookup_catalog`), which requires a resolvable UCP agent profile URI before any
method will run. See `mcp/cirkul-tool-crosswalk.yml` for what lives where. The
human-approval rule applies there too.
