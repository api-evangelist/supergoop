---
name: supergoop-agent-checkout
description: >-
  Build a cart and prepare a checkout at supergoop.com through the store's UCP
  MCP server, stopping at the point of payment for explicit human approval, and
  knowing exactly what can and cannot be undone afterwards.
api: Supergoop! UCP Shopping MCP Server
generated: '2026-08-29'
method: generated
source: mcp/supergoop-ucp-mcp-tools-list.json ; https://supergoop.com/llms.txt ; https://supergoop.com/robots.txt ; https://supergoop.com/policies/refund-policy
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
---

# Check out at Supergoop! as an agent

## The rule that governs this whole skill

Supergoop!'s `robots.txt` and `llms.txt` both say it plainly:

> Checkouts are for humans. Do NOT complete checkout, payment, or order
> placement automatically — no scripted form fills, browser automation, or
> end-to-end agent flows that finalize payment without an explicit,
> contemporaneous human approval step.

You may build the cart. You may prepare the checkout. You may present totals.
You **stop** before `complete_checkout` and hand the decision to your user, at
that moment, with the total in front of them.

## Endpoint and preconditions

`POST https://supergoop.com/api/ucp/mcp`

Every call carries `meta["ucp-agent"].profile` — a publicly fetchable URI
identifying you. The server fetches it. An unreachable profile fails the call
with HTTP 422 / `-32001 profile_unreachable` and returns a `continue_url` you
can hand the shopper so they can finish in a browser instead.

## Sequence

1. **`create_cart`** — start the basket.
2. **`update_cart`** — add, change or remove lines. This one tool covers what
   the GraphQL surface splits across seven mutations.
3. **`get_cart`** — re-read before you quote anything to the shopper.
4. **`create_checkout`** — converts the basket into a priced checkout with
   totals, taxes and any discounts applied.
5. **`update_checkout`** — set the shipping address and delivery method.
6. **`get_checkout`** — read the final total. Show it to the human.
7. **STOP.** Get explicit, contemporaneous approval.
8. **`complete_checkout`** — requires `meta["idempotency-key"]` in addition to
   `meta["ucp-agent"]`. Generate one key per intended purchase and reuse the
   *same* key on any retry: this is the only tool on the server with idempotency
   protection, and it exists because this is the only call that spends money.
   Returns the order id and a Thank You Page URL.

If you cannot get approval at the moment of payment, do not proceed. Hand the
shopper the `continue_url`, or route the purchase through the Shop skill at
`https://shop.app/SKILL.md`, which Supergoop!'s own agent document recommends
for exactly this case.

## Undoing things

| Stage | Reversal | Window |
|---|---|---|
| Cart | `cancel_cart` | Any time before checkout |
| Checkout | `cancel_checkout` | Any time before `complete_checkout` |
| Order (paid) | **No API call exists** | Returns within 30 days of purchase, self-served at returns.supergoop.com |

Tell the shopper this before step 8. Everything up to `complete_checkout` is
machine-undoable; nothing after it is. A paid order comes back only through the
human returns flow — the shopper enters their email at returns.supergoop.com,
picks items from their order history, prints a prepaid label, and is issued an
Instant Refund Gift Card while the return is in transit. Exchanges ship within
24 hours of selection.

## Money

UCP returns integer minor units with a currency code: `{"amount": 2500,
"currency": "USD"}` is $25.00. Divide by 100 for two-decimal currencies before
quoting. JPY and other zero-decimal currencies are already whole units. Never
show a shopper a raw minor-unit integer.

## Errors

- `-32001` / `profile_unreachable` (HTTP 422) — publish a reachable agent
  profile. Includes `continue_url` for a human handoff.
- `-32000` / `AuthenticationRequired` (HTTP 403) — a JWT is required. This is
  what `get_order` returns anonymously; obtain a token per the pointer in the
  error message.
- HTTP 429 — per-IP rate limit. Back off.

## Order status afterwards
`get_order` requires a JWT. If you do not hold one, the Thank You Page URL
returned by `complete_checkout` is the shopper's way to see their order.
