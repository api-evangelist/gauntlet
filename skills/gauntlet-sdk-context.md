> ## Documentation Index
> Fetch the complete documentation index at: https://docs.gauntlet.xyz/llms.txt
> Use this file to discover all available pages before exploring further.

# CONTEXT.md

> Copy this into your LLM context window for accurate, grounded answers about the Gauntlet SDK.

Copy the block below into your system prompt or context window. It gives an LLM everything it needs to answer questions about the SDK, generate correct integration code, and avoid common mistakes.

````markdown theme={null}
# Gauntlet SDK — CONTEXT.md

## Package

- Name: `@gauntlet-xyz/sdk`
- Install: `npm install @gauntlet-xyz/sdk` (peer dep: `viem >=2.0.0`)
- Import paths:
  - `@gauntlet-xyz/sdk` — root: GauntletClient, AttributionMode, all functions, types, errors
  - `@gauntlet-xyz/sdk/evm` — EVM subpath: same functions + types (preferred for tree-shaking)

---

## GauntletClient

Main entry point. Pass one instance to every SDK function.

```typescript
import { GauntletClient, AttributionMode } from '@gauntlet-xyz/sdk'

const client = new GauntletClient({
  evmClients: { [chainId]: publicClient }, // viem PublicClient per chain
  wallet: walletClient,                    // viem WalletClient — SDK reads account address only, never signs
  apiKey: '<YOUR_API_KEY>',                // optional
  builderCode: 'your-code',               // optional — MUST be issued by Gauntlet, not self-serve
  attributionMode: AttributionMode.PUBLIC, // optional — default PUBLIC
})
```

| Param | Type | Required | Notes |
|-------|------|----------|-------|
| `evmClients` | `Record<number\|string, PublicClient>` | For tx methods | chainId → viem PublicClient |
| `wallet` | `WalletClient` | For tx methods | Only reads `wallet.account.address` |
| `apiKey` | `string` | No | Gauntlet API key |
| `builderCode` | `string` | No | Must be issued by Gauntlet. Without it, transactions are unattributed. |
| `attributionMode` | `AttributionMode` | No | `PUBLIC` (default). `ENCODED`/`PRIVATE` throw `UnimplementedFeatureError`. |

---

## Functions

### getDepositTx(client, params) → Promise<PreparedTx[]>

Builds ordered EVM transactions to deposit into a vault. Returns 1–2 steps: optional ERC-20 `approve` (only if allowance insufficient) + `deposit` or `requestDeposit`.

```typescript
import { getDepositTx, VaultId } from '@gauntlet-xyz/sdk/evm'

const steps = await getDepositTx(client, {
  vaultId: VaultId.AeraUsdAlpha, // or any VaultId value or raw string
  amount: 1_000_000n,            // token base units
  chainId: 8453,                 // optional; defaults to Base (8453) for multi-chain vaults
  assetSymbol: 'USDC',          // optional; required for multi-asset vaults
  depositMode: 'async',         // optional; 'async' | 'sync'. Omit to use vault native mode.
  receiver: '0x...',            // optional; defaults to wallet.account. Aera: MUST equal signer.
  slippageBps: 100,             // optional; integer 0–10000. Default 100 (1%).
})
```

Throws: `VaultNotFoundError`, `AccountRequiredError`, `UnsupportedDepositModeError`,
`UnsupportedAssetError`, `InvalidSlippageBPSError`, `RpcNotConfiguredError`

---

### getWithdrawTx(client, params) → Promise<PreparedTx[]>

Builds ordered EVM transactions to withdraw from a vault. Exactly one of `shares`, `amount`, or `entireAmount` is required.

```typescript
import { getWithdrawTx, VaultId } from '@gauntlet-xyz/sdk/evm'

const steps = await getWithdrawTx(client, {
  vaultId: VaultId.AeraUsdAlpha,
  entireAmount: true,   // OR: shares: 500_000000000000000000n  OR: amount: 1_000_000n
  chainId: 8453,        // optional; defaults to Base
  assetSymbol: 'USDC', // optional; required for multi-asset vaults
  depositMode: 'async', // optional; 'async' | 'sync'
  receiver: '0x...',   // optional; defaults to wallet.account
  slippageBps: 100,    // optional; integer 0–10000. Default 100 (1%).
})
```

Throws: `VaultNotFoundError`, `AccountRequiredError`, `UnsupportedDepositModeError`,
`UnsupportedAssetError`, `InvalidSlippageBPSError`, `InvalidWithdrawParamsError`, `RpcNotConfiguredError`

---

### getVaults(client, filter?) → Promise<VaultInfo[]>

Returns vaults from bundled manifest. No network request.

```typescript
import { getVaults } from '@gauntlet-xyz/sdk/evm'

const vaults = await getVaults(client, { chainId: 8453, protocol: 'aera' })
// filter is optional — omit to return all vaults
```

---

### getUserCurrentBalance(client, params) → Promise<UserCurrentBalance[]>

Returns balance breakdown for a user across all deployments of a vault. Only works for Aera multi-depositor vaults (those with a `provisionerAddress`). Makes on-chain RPC calls.

```typescript
import { getUserCurrentBalance } from '@gauntlet-xyz/sdk'

const balances = await getUserCurrentBalance(client, {
  vaultId: 'gtusda',
  address: '0xUser',
  chainId: 8453,  // optional; omit to return one entry per chain the vault is deployed on
})
```

Throws: `VaultNotFoundError`, `UnsupportedProtocolError`, `ChainMismatchError`, `RpcNotConfiguredError`

---

## Transaction Submission

`getDepositTx` and `getWithdrawTx` return `PreparedTx[]`. Steps MUST be executed in order.
An `approve` step, when present, always comes first.

### PreparedTx type

```typescript
type PreparedTx = {
  payload: {
    type: string    // 'approve' | 'deposit' | 'requestDeposit' | 'redeem' | 'requestRedeem' | 'withdraw'
    to: Address     // contract address
    data: Hex       // ABI-encoded calldata + attribution suffix pre-concatenated
    account?: Address
  }
  tx: EvmTxStep   // structured ABI fields + raw attribution bytes
}

type EvmTxStep = {
  type: 'approve' | 'deposit' | 'requestDeposit' | 'redeem' | 'requestRedeem' | 'withdraw'
  address: Address
  abi: Abi
  functionName: string
  args: readonly unknown[]
  account: Address
  attribution?: Hex  // raw ERC-8021 suffix bytes only — NOT the full calldata
}
```

---

### Path 1: step.payload + sendTransaction

**Use for:** backend scripts, embedded wallets (Privy, Dynamic), server-side signing, EVM simulation.
Attribution is baked into `payload.data` — cannot be dropped regardless of wallet or provider.

```typescript
for (const step of steps) {
  // estimateGas simulates exact calldata before broadcast — catches reverts before spending gas
  const gas = await publicClient.estimateGas({
    to: step.payload.to,
    data: step.payload.data,
    account: step.payload.account,
  })
  // gas limit prevents out-of-gas failures on-chain
  const hash = await walletClient.sendTransaction({ ...step.payload, gas })
  // wait for confirmation before next step — deposit reverts if preceding approve is not mined
  const receipt = await publicClient.waitForTransactionReceipt({ hash, confirmations: 2 })
  if (receipt.status !== 'success') throw new Error(`Reverted: ${step.payload.type}`)
}
```

---

### Path 2: step.tx + writeContract

**Use for:** browser wallets via wagmi (MetaMask, Coinbase Wallet, WalletConnect).

CRITICAL: `dataSuffix` MUST be `step.tx.attribution`. Omitting it silently drops attribution —
the transaction succeeds on-chain but volume is NOT tracked.

```typescript
import { useWriteContract } from 'wagmi'

const { writeContractAsync } = useWriteContract()

for (const step of steps) {
  await writeContractAsync({
    address: step.tx.address,
    abi: step.tx.abi,
    functionName: step.tx.functionName,
    args: step.tx.args,
    account: step.tx.account,
    dataSuffix: step.tx.attribution, // REQUIRED — omitting silently drops attribution
  })
}
```

---

## Attribution

- Format: ERC-8021 — `0x8021` + UTF-8 hex of builderCode, e.g. `"acme"` → `0x802161636d65`
- Appended as suffix to calldata. Does not affect contract execution.
- `builderCode` MUST be issued by Gauntlet. Unregistered codes append bytes but are not counted.
- Without `builderCode`, no bytes are appended and transactions are unattributed.
- `AttributionMode.ENCODED` and `AttributionMode.PRIVATE` throw `UnimplementedFeatureError`.

---

## VaultId Enum

```typescript
enum VaultId {
  BaseUsdcPrime             = 'baseUsdcPrime',
  EthUsdcPrime              = 'ethUsdcPrime',
  AeraUsdAlpha              = 'gtusda',        // gtUSDa — multi-chain USDC, Aera async
  EthUsdcPrimeV2            = 'ethUsdcPrimeV2',
  AeraUsdAlphaStaging       = 'stgusda',
  AeraUsdAlphaDev           = 'devusda',
  AeraUsdAlphaDevDeux       = 'devusda2',
  AeraLeveredFalconX        = 'gpaafalconx',
  AeraLeveredFalconXStaging = 'pytstg',
  AeraSyrupUsdc             = 'gpsyrupusdc',
  AeraSyrupUsdcStaging      = 'syruppytstg',
  AeraBtcYield              = 'gtbtc',
  AeraBtcYieldStaging       = 'gtbtcstaging',
  AeraLeveredUsccStaging    = 'gtusccstg',
  AeraLeveredUscc           = 'gtuscc',
  AeraLend                  = 'gtlend',
  AeraKastEth               = 'kasteth',
}
```

---

## Types

```typescript
type VaultInfo = {
  vaultId: string
  name: string
  protocol: 'aera' | 'morpho'
  strategy: string
  deployments: VaultDeployment[]
}

type VaultDeployment = {
  chain: 'evm'
  chainId: number
  vaultAddress: Address
  provisionerAddress?: Address   // present for Aera multi-depositor only
  vaultType: 'single-depositor' | 'multi-depositor'
  depositMode: 'sync' | 'async' | 'both'
  supplyToken: TokenInfo[]
}

type TokenInfo = {
  symbol: string
  address: Address
  decimals: number
}

type UserCurrentBalance = {
  chain: string           // 'base' | 'ethereum' | 'arbitrum' | 'optimism'
  token: Address
  decimals: number
  pendingDeposit: bigint  // assets locked in provisioner after async deposit — awaiting solver, not yet earning yield; 0n if none
  balance: bigint         // assets actively earning in the vault; 0n if no position
  pendingWithdraw: bigint // assets redeemed but not yet claimable after async withdraw — awaiting solver, no longer earning; 0n if none
}
// All three bigint fields are always present. 0n means no position — not an error.

type VaultFilter = {
  chainId?: number
  protocol?: string
}

enum AttributionMode {
  PUBLIC  = 'public',   // ERC-8021 builder code — default
  ENCODED = 'encoded',  // NOT IMPLEMENTED — throws UnimplementedFeatureError
  PRIVATE = 'private',  // NOT IMPLEMENTED — throws UnimplementedFeatureError
}
```

---

## Balance Lifecycle

Pending states represent funds in transit — locked in the provisioner contract and awaiting the vault solver. During `pendingDeposit`, assets are not yet earning yield. During `pendingWithdraw`, vault shares have been redeemed and assets are no longer earning yield but have not yet been transferred. Funds are safe in both pending states; the delay (~2h, up to 12h) is operational, not a risk.

| Event | Effect on balance fields |
|-------|--------------------------|
| Async deposit submitted | Amount enters `pendingDeposit` — locked in provisioner, not yet earning |
| Solver processes deposit (~2h, up to 12h) | `pendingDeposit` → `balance` — now earning yield |
| Sync deposit | Goes directly to `balance` — earning immediately |
| Async withdraw submitted | `balance` → `pendingWithdraw` — shares redeemed, no longer earning, not yet claimable |
| Solver processes withdraw (~2h, up to 12h) | Leaves `pendingWithdraw`; claimable as ERC-20 in receiver wallet |
| Sync withdraw | Removed from `balance` immediately; claimable in receiver wallet |

---

## Errors

All extend `GauntletSDKError extends Error`.

| Class | Properties | Thrown when |
|-------|------------|-------------|
| `VaultNotFoundError` | `.vaultId`, `.chainId?` | Vault ID not in manifest or not on requested chain |
| `UnsupportedAssetError` | `.asset`, `.vaultId` | Token not accepted by vault |
| `ChainMismatchError` | `.expected`, `.received` | chainId param doesn't match deployment |
| `UnsupportedDepositModeError` | `.vaultId`, `.requested`, `.available` | Requested sync/async not supported |
| `RpcNotConfiguredError` | `.chainId` | No evmClients entry for required chain |
| `AccountRequiredError` | — | No wallet in client config |
| `UnsupportedProtocolError` | `.protocol` | getUserCurrentBalance called on non-Aera or single-depositor vault |
| `InvalidWithdrawParamsError` | — | Not exactly one of shares/amount/entireAmount provided |
| `InvalidSlippageBPSError` | `.slippage` | slippageBps not an integer in 0–10000 |
| `UnimplementedFeatureError` | `.feature` | ENCODED/PRIVATE attribution modes |
| `UnsupportedFeatureError` | `.feature` | Feature not supported |
| `UnitConversionError` | `.vaultAddress` | Fee calculator unavailable on-chain |

---

## Key Constraints

- The SDK never signs. `wallet` is read-only for `wallet.account.address`.
- `getDepositTx`/`getWithdrawTx` require `evmClients` + `wallet` (check allowance on-chain).
- `getVaults` reads bundled manifest — no network call required.
- `getUserCurrentBalance` scans ~3 days of provisioner events — requires `evmClients` for each chain queried.
- **Aera vaults:** `receiver` MUST equal the transaction signer. A different address will cause an on-chain revert.
- **Multi-chain vaults** (e.g. gtUSDa): `chainId` defaults to Base (8453) when omitted.
- **Single-chain vaults:** provide `chainId` matching the deployment, or omit it. A mismatched chainId throws `ChainMismatchError`.
- **depositMode 'both':** defaults to `'async'` when `depositMode` param is not specified.
- Steps from `getDepositTx`/`getWithdrawTx` must be sent in order. Deposit reverts if preceding approve is not mined first.
- Using Path 2 (writeContract): `dataSuffix` must be `step.tx.attribution` or attribution is silently lost.
````
