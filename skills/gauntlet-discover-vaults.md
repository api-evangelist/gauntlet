---
name: Discover and compare Gauntlet vaults
description: Find Gauntlet-curated vaults, read current metrics and definitions, and pull historical timeseries to compare yield and risk.
api: openapi/gauntlet-openapi-original.json
operations: [list_vaults, list_featured_vaults, get_vault, get_vault_definition, get_vault_timeseries]
---

# Discover and compare Gauntlet vaults

Read-only flow over the Gauntlet API (`https://api.gauntlet.xyz`). Authenticate every
request with a partner-provisioned API key: `Authorization: Bearer <API_KEY>`
(see `authentication/gauntlet-authentication.yml`).

## Steps

1. **List vaults** — call `list_vaults` to get identification-only rows for every
   Gauntlet vault the indexer knows about (ordered most-recent-first). Use
   `list_featured_vaults` for the curated highlight set. Vault ids are CAIP-10
   (`"{chainId}:{address}"`).
2. **Read current metrics** — for a chosen `vault_id`, call `get_vault` for the
   current metrics snapshot (TVL, APY, unit price). This is the "current point"; it
   shares the metric shape emitted per point by the timeseries.
3. **Read the definition** — call `get_vault_definition` for the vault's identity and
   protocol-specific definition (name, owner, numeraire token, hooks, fees, curator),
   merged inline by `vault_type` (Aera / Morpho V1/V2 / Kamino / Drift / Symbiotic).
4. **Pull history** — call `get_vault_timeseries` for historical metric points.
   Default order is `asc` (oldest-first, chart-friendly); pass `?order=desc` for
   newest-first. Paginate by passing `meta.next_cursor` back as `?next=`.

## Rules

- Respect rate limits (60/min, 10,000/day). On `429`, back off until
  `X-RateLimit-Reset` (`rate-limits/gauntlet-rate-limits.yml`).
- Every response carries `meta.request_id` (include when contacting support) and
  `meta.refreshed_at` (data freshness). See `conventions/gauntlet-conventions.yml`.
- Errors return `{ "error": { "code", "message", "details" } }` — handle `404`
  (`NOT_FOUND`) and `401` (`UNAUTHORIZED`) explicitly (`errors/gauntlet-problem-types.yml`).
