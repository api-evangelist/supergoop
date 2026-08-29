---
name: supergoop-find-sunscreen
description: >-
  Search the Supergoop! catalog for a sunscreen matching a shopper's stated
  needs (SPF level, mineral vs chemical, face vs body, finish, water resistance)
  and return real products with real prices, using either the store's UCP MCP
  server or its anonymous Storefront GraphQL API.
api: Supergoop! UCP Shopping MCP Server
generated: '2026-08-29'
method: generated
source: mcp/supergoop-ucp-mcp-tools-list.json ; graphql/supergoop-storefront-2026-07.graphql
operations:
  - search_catalog
  - lookup_catalog
  - get_product
  - 'graphql:QueryRoot.products'
  - 'graphql:QueryRoot.search'
  - 'graphql:QueryRoot.productByHandle'
---

# Find a Supergoop! sunscreen

## When to use this
The shopper wants a sunscreen and has said something about what they need — an
SPF number, "mineral", "for my face", "won't leave a white cast", "reef safe",
"for my kids". You need real products, real variants and real prices from
supergoop.com.

## Two paths, pick one

### Path A — UCP MCP (preferred when you are a commerce agent)
Endpoint: `POST https://supergoop.com/api/ucp/mcp`
Headers: `Content-Type: application/json`, `Accept: application/json, text/event-stream`

Every call carries a `meta` object:

```json
{"meta": {"ucp-agent": {"profile": "https://your-agent.example/ucp-profile"}}}
```

**The profile URI must be publicly fetchable.** The server GETs it. If it cannot,
you get HTTP 422 and JSON-RPC error `-32001` with `data.code =
profile_unreachable`, and nothing else will work. Verify your profile is live
before you start.

1. `tools/list` — confirm the tool set (13 tools) and the negotiated protocol
   version in the `x-shopify-ucp-mcp-api-version` response header.
2. `search_catalog` — pass `catalog.query` (natural language) and/or
   `catalog.filters`. At least one is required. Also pass buyer context
   (`context.address_country`, `context.currency`) or prices and availability
   may be wrong for your shopper.
3. Page with `pagination.cursor` from the response, only when the shopper asks
   for more.
4. `get_product` for full detail on one item; `lookup_catalog` to batch several
   identifiers at once.

### Path B — Storefront GraphQL (preferred when you only need to read)
Endpoint: `POST https://supergoop.com/api/2026-07/graphql.json`, no auth, no
agent profile required.

```graphql
{
  products(first: 10, query: "mineral SPF 40") {
    edges { node {
      id title handle productType tags
      priceRange { minVariantPrice { amount currencyCode } }
      variants(first: 5) { edges { node { id title sku availableForSale price { amount currencyCode } } } }
    } }
  }
}
```

Use `search(query:, types: PRODUCT)` for relevance ranking, or
`predictiveSearch` for typeahead.

## Reading the results

- **Money is represented differently on each surface.** UCP returns integer
  minor units (`{"amount": 2000, "currency": "USD"}` = $20.00). GraphQL returns
  decimal strings (`{"amount": "20.0", "currencyCode": "USD"}`). Do not share a
  parser. Convert before quoting a price.
- Availability lives on the **variant**, not the product
  (`availableForSale`). A product can be listed while every size is out of stock.
- The store's own merchandising vocabulary is in `tags`, namespaced:
  `Body Part:Face`, `Finish:Matte`, `Preferences:Mineral formula`,
  `Preferences:Sweat & water-resistant`. Match the shopper's words against these
  rather than guessing from the title.

## Errors you will actually hit
- `-32001 / profile_unreachable`, HTTP 422 — your agent profile URI is not
  fetchable. Fix the profile; retrying will not help.
- GraphQL returns **HTTP 200 with an `errors[]` array** on a malformed query.
  Status code alone is not a success signal on that surface.
- Any `text/html` body from a JSON path is the storefront's 404 page.

## Rate limits
No published quota. The MCP endpoint is rate-limited per IP; back off on 429.
Cost is reported to you — `shopify-complexity-score` headers on MCP,
`extensions.cost.requestedQueryCost` in the GraphQL body — but remaining budget
and reset time are not, so pace yourself.

## Do not
Do not proceed to checkout from this skill. Purchasing is a separate skill and
requires explicit human approval at the moment of payment.
