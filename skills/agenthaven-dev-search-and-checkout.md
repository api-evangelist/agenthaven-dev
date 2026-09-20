---
name: agenthaven-dev-search-and-checkout
description: Quote a flight from the Agent Bench Travel Merchant and, with a bearer authorization, turn one quote into a test-mode payment intent — using the two tools the agent actually exposes, within its ten-minute quote window and single-use mandate rules.
api: mcp/agenthaven-dev-mcp.yml
operations: [search_flights, create_checkout]
protocols: [MCP 2025-06-18 at https://travel.agenthaven.dev/mcp, A2A 0.3 message/send at https://travel.agenthaven.dev/a2a]
method: generated
generated: '2026-09-19'
source: tools/list from https://travel.agenthaven.dev/mcp (verbatim in mcp/agenthaven-dev-mcp-tools.json) and https://travel.agenthaven.dev/
---

# Search a flight and check out with Agent Bench

Agent Bench's travel merchant is a proof of concept: fares are real (Google Flights via SerpApi), the checkout is a Stripe **test-mode** payment intent, no money moves and no ticket is issued. It exposes exactly two operations, and they must be called in order. Both tool names below are the real names returned by `tools/list`; the same two names are the A2A `action` values.

## Before you start

- Discover the agent: `GET https://travel.agenthaven.dev/.well-known/agent-card.json`. Its digest is pinned in the DNSSEC-signed SVCB record at `travel.agenthaven.dev` (DNS-AID); `dns-aid verify travel.agenthaven.dev` checks it.
- Get the schemas: `POST https://travel.agenthaven.dev/mcp` with `{"jsonrpc":"2.0","id":1,"method":"tools/list"}` — no authorization. Both tools carry `inputSchema` and `outputSchema`.
- Mint one `correlation_id` (UUID v4, lowercase) per transaction and send it on both calls; the merchant echoes it into every ledger entry so you can audit the run afterwards at `GET /ledger`.

## Step 1 — `search_flights` (no authorization)

Input: `origin` and `destination` as **uppercase IATA codes** (`"MAD"`, not `"Madrid"`), `departure_date` as `YYYY-MM-DD` in the future, optional `adults` (default 1).

MCP: `{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"search_flights","arguments":{"origin":"MAD","destination":"LHR","departure_date":"2026-10-20"}}}`

A2A: `message/send` with one data part `{"action":"search_flights","input":{"origin":"MAD","destination":"LHR","departure_date":"2026-10-20"}}` — or a text part exactly `search_flights MAD LHR 2026-10-20`.

Read `options[]` (1–2 entries, labelled `cheapest` and/or `fastest`). Each is a signed quote: keep `id` (the `quote_id` for step 2), `quote_jws`, `quote_hash`, `total` (EUR **minor units**), `expires_at` and the `itinerary`. Recompute `quote_hash` yourself as SHA-256 lowercase hex over the raw decoded payload octets of `quote_jws` — the schema says "Recompute it; never trust it."

Budget: anonymous searches are limited to **40 per day and 120 per month, shared by everyone**; an authorised search does not count. Check `GET /health` → `counters.anonymous_searches` before a batch. Exhaustion is recorded in the ledger as `request.refused` / `search_budget_exhausted`.

## Step 2 — `create_checkout` (bearer JWT required)

You have **ten minutes** from step 1. Input: `quote_id` (one `options[].id`), `merchant_id` **must be** `"demo-travel-seller"`, `currency` **must be** `"EUR"`; optionally your recomputed `quote_hash`, the same `correlation_id`, and `mandate_jws` when you are using a signed request envelope.

Authorization, from the card's `securitySchemes.bearer` — there is **no public token endpoint**:
1. **Request envelope + mandate**: sign each call as a compact JWS (`typ request+jwt`) under a key attested at `/.well-known/agent-keys.json` on *your own* domain, and send `mandate_jws` (`typ mandate+jwt`) from an authority the merchant trusts (`https://sealedby.dev`), bound to `quote_hash` and to your envelope key.
2. **Operator-issued JWT**: ask the operator; `iss agent-bench-demo-issuer`, `aud agent-bench`, `scope commerce:purchase`, with an EUR spending cap.
3. **Guest**: publish a P-256 key in a DNSSEC-signed TXT record at `_agent-keys.<your domain>`, sign envelopes with `iss spiffe://<your domain>/<path>`, and self-issue a mandate living at most 10 minutes. Ceiling 1000.00 EUR per purchase; 20 guest purchases a day shared. Contract and reference buyer: https://provedby.dev/contracts/guest-buyer.md

MCP: `{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"create_checkout","arguments":{"quote_id":"<options[].id>","merchant_id":"demo-travel-seller","currency":"EUR","correlation_id":"<same uuid>"}}}` with header `Authorization: Bearer <token>`.

A2A: `message/send` with data part `{"action":"create_checkout","input":{"quote_id":"<options[].id>","merchant_id":"demo-travel-seller","currency":"EUR"}}` — or text `create_checkout <quote_id>` — with the same header.

Output: `checkout` (amount, currency, itinerary), `payment_intent {source, id, status, idempotency_key}`, `settlement {rail, outcome, payment_status, charged}` (on `stripe-test` the outcome is `captured` and `charged` is `true` — a test card, never real money), `receipt_jws` (`typ receipt+jwt`), and `evidence` with `audit_hash` and `completion_hash` you can find in `GET /ledger`.

## Rules that will bite

- **Order and window.** A `quote_id` the merchant did not mint, or one older than ten minutes, is refused as `quote_unavailable`. Search again.
- **Idempotency is the mandate.** A mandate's `jti` buys once; a second checkout on it is refused as `mandate_consumed`. If the payment rail did not answer (ledger type `payment.ambiguous`), retry with the **same** mandate and quote — "the mandate stays retryable with the same idempotency key, so a retry finds the intent if one exists." Do not mint a new mandate to retry.
- **No way back.** There is no cancel, refund or void operation. The spending cap, currency and deadline inside your mandate are your only controls, and they act *before* the call.
- **Constants.** `merchant_id` other than `demo-travel-seller` or `currency` other than `EUR` is refused; both tools set `additionalProperties: false`.
- **Refusals are normal.** They come back as JSON-RPC errors and are written to the public ledger as `request.refused` with `facts.reason` and `facts.step` (see `errors/agenthaven-dev-problem-types.yml`). Cite `request_id` (also the `x-request-id` header) when reporting a problem.
- **Verify the responder.** Every output carries `evidence.signer {spiffe_id, kid, key_url}` and `evidence.agent_card_sha256`; check the key against `/.well-known/spiffe-bundle.json` and the card digest against the DNS record before trusting a receipt.
- **Rehearsals.** Add `exercise: "R-06"`-style labels to mark a run as a test vector; it is recorded as your claim and changes nothing.
