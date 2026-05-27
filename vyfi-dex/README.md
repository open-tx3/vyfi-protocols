# VyFi DEX

[VyFinance](https://vyfi.io/) is a Cardano-native DeFi platform offering an AMM-based decentralised exchange, liquid staking, and yield-bearing vaults. The DEX is the foundational product: users swap between ADA and Cardano-native tokens, and liquidity providers deposit token pairs into pools to earn a share of swap fees. VyFi runs its own batcher infrastructure — users submit order intents on-chain, and VyFi's batchers execute them in batches against the pool UTxOs.

This tx3 covers the **user-facing order submission and cancellation** surface of the DEX: swap and liquidity intents, sent to per-pool order addresses. Batcher-side execution (consuming orders, updating pool state, distributing results) is not in scope.

## Overview

VyFi uses a two-step order model:

1. **User submits the order** — the user sends funds + an inline datum to a pool-specific order script address. No validators execute on submission.
2. **Batcher processes the order** — VyFi infrastructure consumes the order alongside the pool UTxO and distributes the results back to users.

Each pool has its own order address; the correct address must be passed dynamically per call via the `OrderScript` party. Pool addresses, token identifiers, and current ratios are queried from the VyFi public API.

Scope of this tx3:

- Implemented: swap (both directions), add liquidity, remove liquidity — all user-side order submission.
- Not implemented: batcher consumption of orders, pool-state updates.

## Transactions

| Transaction | Description |
|---|---|
| `swap_a_to_b` | Swap ADA for a token |
| `swap_b_to_a` | Swap a token for ADA |
| `add_liquidity` | Deposit ADA + tokens into a liquidity pool |
| `remove_liquidity` | Withdraw liquidity by sending LP tokens |

## Important considerations

- **No script execution on submission.** Order submissions are plain payments with inline datums — no validators are invoked. No collateral is needed.
- **Pool-specific order address.** The `OrderScript` party must be set to the correct order address for each pool. Pool data is available from the VyFi API.
- **Process fee.** VyFi batchers charge a process fee (currently 1.9 ADA) included in each order.
- **User credentials format.** The `user_creds` parameter is a 56-byte value: payment credential (28 bytes) concatenated with staking credential (28 bytes).
- **Batcher-side not implemented.** This tx3 covers user-facing order submission only. Batcher operations (consuming orders, updating pool state) are out of scope.

## Caller preparation

### All transactions

| Parameter | Source |
|---|---|
| `user_creds: Bytes` | 56-byte hex value: payment credential (28 bytes) + staking credential (28 bytes). Constructed from the user's wallet address by extracting the raw credential hashes and concatenating them. |
| `OrderScript` party address | The correct **pool-specific** order address. Query the VyFi API (`/lp?networkId=1`) for pool data including the order address. |

### `swap_b_to_a` / `remove_liquidity`

| Parameter | Source |
|---|---|
| `order_ada: Int` | ADA amount to include in the order UTxO (min UTxO + process fee). Must account for the tokens being sent. |

### `add_liquidity`

| Parameter | Source |
|---|---|
| `desired_lp: Int` | Desired LP token amount. Calculated from the current pool ratio. |

### Pool data (query VyFi API before any transaction)

| Value | Source |
|---|---|
| Pool order address (for `OrderScript` party) | VyFi API `/lp?networkId=1` |
| Token policy ID and asset name | VyFi API |
| Current pool ratio (for swap amounts, LP tokens) | VyFi API |
| Process fee | VyFi API (currently 1.9 ADA) |

## References

- **Homepage / app:** [vyfi.io](https://vyfi.io/)
- **Pool data API:** [`/lp?networkId=1`](https://api.vyfi.io/lp?networkId=1)
