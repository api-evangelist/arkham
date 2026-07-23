---
name: arkham-address-investigation
description: Investigate a blockchain address or entity with Arkham — attribution, balances, counterparties, flows, and risk — before trading or for compliance due diligence.
api: openapi/arkham-openapi-original.json
operations: [IntelligenceSearchHandler, GetAddressIntelligence, GetAllAddressIntelligence, GetAddressBalances, GetAddressCounterParty, GetAddressFlow, GetRiskScoreAddress]
---

# Investigate an address or entity with Arkham

Use the Arkham Intel API to attribute an on-chain address to a real-world entity and assess it before
trading or for compliance. Base URL `https://api.arkm.com`. All requests require an `API-Key` header
(NOT `Authorization`). Failed (4xx/5xx) requests are never charged.

## Steps

1. **Resolve the subject.** If you have a name or ticker rather than an address, call
   `IntelligenceSearchHandler` (`GET /intelligence/search`) to find the matching address, entity, token,
   or pool. This is a Heavy endpoint (1 req/s) and costs 30 credits per call.

2. **Attribute the address.** Call `GetAddressIntelligence`
   (`GET /intelligence/address/{address}`) for entity labels and tags on one chain, or
   `GetAllAddressIntelligence` (`GET /intelligence/address/{address}/all`) to attribute across all chains.
   Attribution is confidence-scored, not a binary claim — read the confidence.

3. **Size the wallet.** Call `GetAddressBalances` (`GET /balances/address/{address}`) for current token
   balances and USD value.

4. **Map the network.** Call `GetAddressCounterParty` (`GET /counterparties/address/{address}`) for the
   top counterparties. Heavy endpoint (1 req/s); `limit` max is 1000.

5. **Trace the money.** Call `GetAddressFlow` (`GET /flow/address/{address}`) for historical USD inflow/
   outflow. Use `timeGte`/`timeLte` (unix ms) to bound the window — do not combine `timeLast` with
   `timeGte`.

6. **Score risk.** Call `GetRiskScoreAddress` (`GET /risk-score/address/...`, beta) for a risk signal to
   gate a copy-trade or onboarding decision.

## Conventions

- **Pagination:** offset-based (`limit` + `offset`); responses carry `count`/`total`. Recommended
  page size 50-500 for list endpoints. See `conventions/arkham-conventions.yml`.
- **Rate limits:** 20 req/s standard, 1 req/s on Heavy endpoints (search, counterparties, flow).
  On `429`, back off with `Retry-After` + jitter. Intel label limits are tracked via the
  `X-Intel-Datapoints-*` response headers.
- **Errors:** simple `{ "error": string }` envelope. `403 not allowed to access endpoint` means the
  endpoint is not in your plan (`api@arkm.com`). See `errors/arkham-error-codes.yml`.
- **No account?** The same reads are available pay-per-request at `https://api.arkm.com/x402` (POST,
  params in the JSON body, USDC-on-Base settlement) — see `skills/arkham-x402-agent-skill.md`.
