# Cirkul

Cirkul, Inc. is a Tampa, Florida direct-to-consumer beverage company founded in 2016.
It sells a reusable bottle system paired with patented flavor cartridges ("Sips") that
sit in the lid behind a dial, letting a drinker set flavor intensity without pre-mixing.

- Website — https://drinkcirkul.com/
- Agent instructions — https://drinkcirkul.com/agents.md
- GitHub — https://github.com/drinkcirkul
- Secondary market listing — https://forgeglobal.com/cirkul_stock/

## API surface

Cirkul runs **no developer program**: no OpenAPI, no GraphQL, no SDKs, no CLI, no
status page, no changelog, no `security.txt`. What it does have is an
**agent-commerce surface**, served from its own `drinkcirkul.com` host:

| Surface | Path | State |
|---|---|---|
| Storefront MCP server | `POST /api/mcp` | live, anonymous, 5 tools with real JSON Schema inputs |
| UCP shopping MCP | `POST /api/ucp/mcp` | live, gated behind a UCP agent-profile URI |
| UCP merchant profile | `GET /.well-known/ucp` | 200 — 8 capabilities, 3 payment handlers |
| Agent instructions | `GET /agents.md`, `GET /llms.txt` | 200 |
| Agent discovery sitemap | `GET /sitemap_agentic_discovery.xml` | 200 |
| OAuth 2.0 / OIDC discovery | `GET /.well-known/openid-configuration` | 200 — Shopify Customer Account API |
| A2A agent card | `/.well-known/agent-card.json` | 404 |

The storefront is built on Shopify, and these are Shopify-platform surfaces — but
they are served on Cirkul's own domain, advertised by Cirkul's own `robots.txt`, and
the UCP profile carries Cirkul-specific merchant and payment-handler configuration.

Cirkul states in three places that **checkout requires contemporaneous human
approval**; agents may build a cart but must not finalize payment.

See `apis.yml` for the full artifact index.
