---
name: Track a wallet's Gauntlet positions and returns
description: Read a wallet's current vault positions, PnL/ROI over time, and on-chain activity log using the Gauntlet API.
api: openapi/gauntlet-openapi-original.json
operations: [get_user_all_positions, get_user_vault_position, get_user_vault_position_timeseries, get_user_activity]
---

# Track a wallet's Gauntlet positions and returns

Read-only flow over the Gauntlet API (`https://api.gauntlet.xyz`). Authenticate with
`Authorization: Bearer <API_KEY>` (see `authentication/gauntlet-authentication.yml`).
All endpoints take a `wallet_address` path parameter.

## Steps

1. **All positions** — call `get_user_all_positions` with the `wallet_address` to get
   every current position (value, cost basis, PnL, ROI) across vaults.
2. **Single position** — call `get_user_vault_position` with `wallet_address` and a
   `vault_id` for the current snapshot in one vault. Historical points live on the
   timeseries endpoint (below).
3. **Position history** — call `get_user_vault_position_timeseries` for historical
   value and ROI for a single vault position. Default order `asc`; pass `?order=desc`
   for newest-first list views. Cursor-paginate with `meta.next_cursor` → `?next=`.
4. **Activity log** — call `get_user_activity` for the immutable on-chain event log
   (deposits, withdrawals, transfers). Each row is a frozen-in-time record and never
   mutates after emission.

## Rules

- Vault ids are CAIP-10 (`"{chainId}:{address}"`); monetary values are human-unit
  decimal strings.
- Respect rate limits (60/min, 10,000/day); back off on `429` until
  `X-RateLimit-Reset` (`rate-limits/gauntlet-rate-limits.yml`).
- Use `meta.refreshed_at` for freshness and `meta.request_id` for support/tracing
  (`conventions/gauntlet-conventions.yml`).
- Errors use `{ "error": { "code", "message", "details" } }`
  (`errors/gauntlet-problem-types.yml`).
