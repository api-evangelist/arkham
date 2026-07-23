---
name: arkham-intelligence-api
description: Pay per request for Arkham wallet intelligence and market data when an agent needs trading due diligence, token flows, portfolios, markets, or Polymarket analytics.
---

# Arkham Intelligence API

## What this is

Arkham wallet intelligence and market data, paid per request through x402. No account or Arkham API key is needed: the agent pays USDC on Base for each successful call.

## When to use

- Agentic-trading due diligence: investigate a wallet or entity before copy-trading and analyze counterparties before trading with them.
- Token-flow and smart-money tracking: inspect transfers, swaps, counterparties, historical flows, and top token flow.
- Balance and portfolio lookups for addresses or Arkham entities.
- Token prices, volume, holders, trends, and broader market data.
- Polymarket events, prices, positions, activity, holders, and leaderboard analytics.

## How to call

All Arkham endpoints are POST requests with a JSON body. Pass Arkham path parameters as body fields; for example:

    POST /balances/address
    {"address":"0x..."}

The service base URL is https://api.arkm.com/x402; all paths here are relative to it. Discover all routes, required fields, prices, and per-endpoint JSON Schemas at GET /openapi.json. Full Arkham API documentation for agents (endpoint references, guides, and schemas) is available at https://arkm.com/llms.txt.

## Payment (x402)

1. Make the request without payment. The service returns HTTP 402 and a PAYMENT-REQUIRED header containing base64-encoded x402 v2 JSON.
2. Decode the challenge and select the accepts[] entry for the network the challenge advertises (Base mainnet, eip155:8453, in production), with scheme exact, the USDC asset, amount in atomic units, and payTo.
3. Sign an EIP-3009 transferWithAuthorization for that amount. This authorization is gasless for the agent.
4. Retry the identical request with the signed payload in PAYMENT-SIGNATURE.
5. On success, read PAYMENT-RESPONSE for the settlement receipt and transaction hash.

Settlement happens only after the upstream Arkham request succeeds. Failed upstream requests are not charged.

## Pricing

Pricing is $0.20 per Arkham credit. Representative prices from the route configuration:

| Category | Endpoint | Price |
| --- | --- | ---: |
| Intelligence | /intelligence/address | $0.20 |
| Intelligence | /intelligence/entity-summary | $0.20 |
| Intelligence | /counterparties/address | $10.00 |
| Balances and portfolios | /balances/address | $0.20 |
| Balances and portfolios | /portfolio/address | $0.20 |
| Flows | /transfers | $0.40 × limit |
| Flows | /swaps | $0.40 × limit |
| Flows | /flow/address | $0.40 |
| Flows | /token/top-flow/by-address | $2.00 |
| Market data | /token/market | $0.20 |
| Market data | /token/price/history | $0.20 |
| Market data | /marketdata/altcoin-index | $0.20 |
| Polymarket | /polymarket/event | $0.20 |
| Polymarket | /polymarket/positions | $0.60 |
| Polymarket | /polymarket/leaderboard | $2.00 |

Per-row endpoints such as transfers and swaps cost credits × limit; always pass the smallest useful limit. The free /chains, /networks/status, and /arkm/circulating endpoints use SIWX sign-in instead of payment.

## Error handling

- 402 with PAYMENT-REQUIRED: this is the payment challenge, not an application error. Pay and retry the identical request with PAYMENT-SIGNATURE.
- 402 without PAYMENT-REQUIRED after payment: verification failed. Read the new challenge and re-sign; retry once.
- 400: validation failed or required body fields are missing. Fatal for the request as written; fix it before retrying. Not charged.
- 404: the upstream Arkham resource does not exist. Fatal for the same input. Not charged.
- 429: the payer wallet exceeded its endpoint rate limit. Retry with at least 1 second of backoff; keep heavy endpoints at no more than 1 request/second.
- 5xx, including 502 and 504: Arkham failed or timed out. Retry with backoff. Payment is not settled for failed requests, so the client is not charged.

## Limits

Rate limits apply per payer wallet and endpoint. The default is 20 requests/second. These heavy upstream paths are limited to 1 request/second: transfers, transfers/unenriched, transfers/histogram, swaps, token/top, both token/top_flow forms, both token/volume forms, intelligence/search, both counterparties forms, and both flow forms.