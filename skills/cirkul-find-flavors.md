---
name: Find and compare Cirkul flavors and bottles
description: Search the Cirkul storefront catalog for bottles and flavor cartridges,
  inspect a specific product and variant, and answer store-policy questions — all
  read-only, no credential, no purchase.
api: mcp/cirkul-mcp.yml
surface: https://drinkcirkul.com/api/mcp
transport: MCP (JSON-RPC 2.0) over HTTP
auth: none
operations:
- search_catalog
- get_product_details
- search_shop_policies_and_faqs
generated: '2026-08-02'
method: generated
source: mcp/cirkul-mcp-tools-list.json
---

# Find and compare Cirkul flavors and bottles

Cirkul sells a reusable bottle plus flavor cartridges ("Sips") that clip into the
lid. This skill covers the read-only half of Cirkul's agent surface. Nothing here
places an order or moves money.

## Before you start

- Endpoint: `POST https://drinkcirkul.com/api/mcp`, `Content-Type: application/json`,
  `Accept: application/json, text/event-stream`.
- No credential is required. `tools/list` and all three tools below answer anonymously.
- The endpoint is rate-limited **per IP**. On `429`, back off and retry — do not
  parallelize aggressively.
- Do not scrape `/cart.js` or `/recommendations/products`; Cirkul's `robots.txt`
  disallows both and directs agents here instead.

## Step 1 — Search the catalog (`search_catalog`)

Pass a free-text query, filters, or both — at least one is required.

- `catalog.query` — free-text, e.g. "zero sugar iced tea cartridge".
- `catalog.filters.categories` — array, OR logic.
- `catalog.filters.price.min` / `.max` — **ISO 4217 minor units**. `5000` is $50.00,
  not 5000 dollars. Getting this wrong is the most common failure on this tool.
- `catalog.context.address_country` and `catalog.context.currency` — always send
  these. Cirkul's published agent instructions state pricing and availability are
  only accurate when buyer context is supplied. Country is ISO 3166-1 alpha-2
  ("US"), currency is ISO 4217 ("USD").
- `catalog.context.intent` — a sentence of background about what the buyer actually
  wants; it feeds relevance.
- `catalog.pagination.limit` — defaults to `10`, maximum `250`, and the server may
  clamp lower. Do not assume you got everything back.

Take `pagination.cursor` from the response and pass it back as
`catalog.pagination.cursor` only when the user asks for more results. The response
conforms to the UCP capability `dev.ucp.shopping.catalog.search`.

## Step 2 — Inspect one product (`get_product_details`)

Required: `product_id`.

- Without `options`, you get the first available variant. That is usually **not**
  the variant the buyer described (bottle size, color, flavor pack count).
- Pass `options` to pin the exact variant before you quote a price or report stock.
- `country` and `language` localize the result; keep them consistent with the
  `context` you used in step 1 or prices will disagree between steps.

Carry the variant id forward — cart operations key on `product_variant_id`, not on
`product_id`.

## Step 3 — Answer policy and FAQ questions (`search_shop_policies_and_faqs`)

Required: `query`.

Use this rather than fetching and summarizing the HTML policy pages. It is the only
structured policy surface Cirkul exposes, and it is the tool to reach for on
questions about subscriptions and the autoship plan, shipping and delivery,
returns and cancellation, and recycling of used cartridges.

## Stop here

This skill ends at "here is what it costs and whether it is in stock." Building a
cart is a separate skill, and completing a purchase requires a human — see
`cirkul-build-cart.md`.
