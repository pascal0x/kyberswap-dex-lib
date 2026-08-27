# Add Synthereum (Jarvis) liquidity source — Base

## Background

[Synthereum](https://jarvis.money/) is the synthetic-asset protocol behind Jarvis Network. It issues
fiat-pegged synthetic tokens (jFIATs, e.g. **jEUR**) fully backed by collateral, priced by a Chainlink
oracle instead of an AMM curve. Swaps therefore execute at the oracle FX rate with zero slippage,
subject to a flat protocol fee and hard capacity limits.

This PR adds the `synthereum` liquidity source with **two statically listed pools on Base (chainId 8453)**:

| Pool | Address | Pair |
|---|---|---|
| MultiLP liquidity pool | `0x67aefc812ec0a83a327c05d6e7913c35b48bfb94` | USDC ↔ jEUR |
| Fixed-rate wrapper | `0x41b0667ea45a5401d95f9a5d281287630704b798` | EURC ↔ jEUR |

Tokens:

- USDC `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913` (6 decimals)
- EURC `0x60a3e35cc302bfa44cb288bc5a4f316fdb1adb42` (6 decimals)
- jEUR `0x4154550f4db74dc38d1fe98e1f3f28ed6dad627d` (18 decimals)

## Pricing logic

### MultiLP pool (`SynthereumMultiLpLiquidityPool`, jEUR/USDC)

The pool mints jEUR against USDC (**mint**) and burns jEUR for USDC (**redeem**) at the Chainlink
EUR/USD rate, charging `feePercentage()` (1e18-scaled, currently 0.15%) on the collateral leg.
For a fixed oracle price and fee, mint and redeem outputs are **linear in the input amount**, so the
tracker probes the on-chain quoter with exactly one whole unit of the input token and the simulator
scales linearly (pure, no RPC):

- Tracker probes `getMintTradeInfo(1e6)` → `(synthTokensReceived, feePaid)` and
  `getRedeemTradeInfo(1e18)` → `(collateralAmountReceived, feePaid)`, and reads
  `feePercentage()`, `maxTokensCapacity()`, `totalSyntheticTokens()`. All calls go through one
  `TryAggregate` so a reverting probe (e.g. the redeem probe when outstanding jEUR supply is below
  1 jEUR) only disables that trade side instead of failing the refresh.
- Simulator: `amountOut = amountIn * probeOut / probeIn` (floor), fee scaled the same way from the
  probe's `feePaid`.

Limits enforced by the simulator (mirroring on-chain reverts):

- **Mint**: output jEUR must not exceed `maxTokensCapacity()` (jEUR-denominated mint headroom,
  driven by LP collateralization).
- **Redeem**: input jEUR must not exceed `totalSyntheticTokens()` (outstanding supply — currently
  small on Base, so the code treats it generically).

`UpdateBalance` moves both caps after a swap (mint decreases mint headroom and increases the
redeemable supply; redeem decreases the redeemable supply; the mint-headroom increase on redeem is
conservatively ignored).

### Fixed-rate wrapper (jEUR/EURC)

The wrapper converts EURC ↔ jEUR at **exactly 1:1 in value** (×/÷ 10^12 for decimals), zero fee.

- **Wrap** (EURC→jEUR): unbounded capacity.
- **Unwrap** (jEUR→EURC): bounded by the collateral the wrapper can redeem from the Morpho vault it
  deposits into (`0xbeef086b8807dc5e5a1740c5e3a7c4c366ea6ab5`). The tracker reads
  `vault.previewRedeem(vault.balanceOf(wrapper))` (~607k EURC at the time of writing), with the
  `previewRedeem` call pinned to the same block as `balanceOf`.
  Note: reading the wrapper's raw EURC balance would always return 0 — the wrapper keeps no idle
  collateral — hence the vault-based read.

Dust unwraps (< 10^12 wei jEUR) round to zero output and are refused.

## Implementation notes

- Pools are embedded statically (`pools/base.json`), listed with zero reserves; the tracker fills
  reserves as "collateral payable / synth payable" (the unbounded wrap side uses a large placeholder).
- All amounts use `uint256.Int`; addresses are lowercase.
- Registration: `ExchangeSynthereum` in `pkg/valueobject/exchange.go`, `Synthereum` in
  `pkg/pooltypes/pooltypes.go`, msgpack bindings regenerated via `go generate ./pkg/msgpack/...`.
- Executor-side encoding is left to the KyberSwap team (out of scope for this repo). On-chain entry
  points: `mint((minNumTokens, collateralAmount, expiration, recipient))` /
  `redeem((numTokens, minCollateral, expiration, recipient))` on the MultiLP pool, and
  `wrap(uint256 amount, address recipient)` / `unwrap(uint256 amount, address recipient)` on the
  wrapper. The approval address is the pool/wrapper itself (exposed via `GetApprovalAddress`).

## Tests

```
$ go build ./...          # OK
$ go test ./pkg/liquidity-source/synthereum/... ./pkg/pooltypes/... ./pkg/msgpack/...
ok    github.com/KyberNetwork/kyberswap-dex-lib/pkg/liquidity-source/synthereum   0.401s
ok    github.com/KyberNetwork/kyberswap-dex-lib/pkg/pooltypes                     0.607s
?     github.com/KyberNetwork/kyberswap-dex-lib/pkg/msgpack        [no test files]
?     github.com/KyberNetwork/kyberswap-dex-lib/pkg/msgpack/generate [no test files]
```

Simulator coverage: mint with fee, redeem with fee, mint/redeem capacity refusals, wrap/unwrap ×10^12,
unwrap bounded by vault reserve, dust rounding, missing tracked state (fresh listing), state updates
(`UpdateBalance`) and clone isolation (`CloneState`).

🤖 Generated with [Claude Code](https://claude.com/claude-code)
