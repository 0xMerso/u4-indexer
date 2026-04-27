# Uniswap V4 indexing engine + LP rewards

The test asks for:
- an indexer that saves Uniswap V4 data in a relational database
- a way to use that data to reward LPs

Here is my work: a roadmap (and a Prisma schema in `schema.prisma`).

## Overview

- An indexer runs all the time, reads every V4 `PoolManager` event for the pools we track and saves them to Postgres. It can start at any block.
- A user creates a campaign via UI or API: `(poolId, rewardToken, amount, startBlock, endBlock)` and N epochs.
- Each epoch has its own Merkle root.
- A scorer runs every 2 hours, once per epoch. It computes time-weighted active liquidity per position, writes `Leaf` rows, builds the Merkle tree and sends the root on chain.
- Users claim via the `Distributor`. One claim pays all they earned since the campaign started.

## Scope

- 1 campaign = 1 pool.
- Reward token: any ERC20 on the same chain as the pool.
- Indexes only pools having a Campaign in DB. Per pool, runs from the earliest `Campaign.startBlock` forward and never rewinds
- A campaign added later with an earlier `startBlock` or on a previously untracked pool isn't handled, but can be implement that with a backfill program later.

## Positions minted outside a known PositionManager

ALMs (Gamma, Arrakis), custom routers and bots open positions via their own `unlock` callbacks. Their `sender` is not in our list, so they get no `Leaf` row. Fix later via per-sender `PositionResolver` adapters: ALMs map vault shares to positions; custom routers use an opt-in allowlist with one ownership rule each.

## Architecture

```
┌─────┐  ┌─────────┐  ┌──────────┐  ┌────────────┐  ┌─────────────┐
│ RPC │->│ Indexer │->│ Postgres │<>│   Scorer   │->│ Distributor │
└─────┘  └─────────┘  └──────────┘  └────────────┘  └─────────────┘
                           |                                |
                           │        ┌───────────┐           │
                           └────────│ Claim API │<──── user claim
                                    └───────────┘
```

- Indexer: reads `Initialize`, `Swap` and `ModifyLiquidity` from the `PoolManager`. Uses Envio (faster than `eth_getLogs`, handles reorgs). A `topics[1]` filter on the `LogFilter` scopes Envio to pools with an active Campaign.
- Postgres: one database with events, campaigns, epochs and leaves.
- Scorer: Rust, runs every 2 hours. Reads finished epochs, scores the closing epoch, builds the Merkle tree, saves proofs and root, writes the root on chain. Replays events from the DB only, no RPC call during scoring.
- Distributor (Solidity): one contract shared by all campaigns. Stores `rootByCampaign[bytes32]` and `paid[positionManager][tokenId][token]`.
- Claim API: HTTP. The client sends a list of tokenIds. The API returns matching leaves and proofs across all campaigns.

Each service runs in its own Docker container, on a shared docker-compose network.

## Data model

See `schema.prisma`, 6 models.

- `Pool`: one row per V4 pool, written on `Initialize`. `hookPerms` holds the 14 permission flags from the hook address. `initialTick` is set by `Initialize`; if the pool was already Initialize'd before we started indexing, `seed` reads the current tick once via `StateView.getSlot0(poolId)`. The scorer itself makes no RPC call.
- `SwapEvent`: raw Swap logs. Holds `tickBefore` and `feeAmount`.
- `ModifyLiquidityEvent`: raw logs. V4 merges mint and burn into one signed `liquidityDelta`. `salt` encodes the tokenId.
- `Campaign`: one reward program. Creator, pool, reward token, budget, window.
- `Epoch`: a time slice of a campaign, one Merkle root each.
- `Leaf`: one row per `(epoch, positionManager, tokenId)`. `cumulative` is the running total hashed into the leaf.

No `Position` table. We rebuild positions from `ModifyLiquidityEvent` on each scoring run.

## Hook management

What we index is the same whether the pool has a hook or not. `Swap.tick`, `Swap.amount0/1`, `Swap.fee` and `ModifyLiquidity.liquidityDelta` come out after the hook callbacks run. Dynamic fees: `Pool.fee` keeps the uint24 with the `0x800000` flag, the real per-swap fee is in `SwapEvent.feeAmount`. Gap: we don't index `Donate` events (matters only for fee-growth scoring).

## Scoring: TWAL (Time-Weighted Active Liquidity)

Time based, not fee based. LPs out of range earn zero. Tight ranges win: the same money buys more `L` when the range is smaller. No JIT risk, since it's time based.

For each position `i`, sum across every block `b` of the epoch the position's liquidity, but only when the pool's tick is inside the position's range:

```
score_i = Σ over blocks b in the epoch:  L_i(b) × 1[tick_pool(b) ∈ range_i]
```

`L_i(b)` changes only on `ModifyLiquidity`. `tick_pool(b)` changes only on `Swap`. Between events both are constant, so the sum collapses to `Σ L_i × segment_blocks` for each in-range segment. Same events in, same scores out.

This matches the `C` (Liquidity Contribution) part of Merkl's current formula with weights `(0, 0, 1)`.

Per epoch:

```
epoch_budget   = budget × (epoch_blocks / total_blocks)
position_delta = epoch_budget × score_position / Σ scores
cumulative     = previous_cumulative + position_delta
```

Rules the scorer must follow:

1. `new_cumulative ≥ previous_cumulative` for every `(positionManager, tokenId)`. If it goes down, `Distributor.claim`'s subtraction goes below zero and the user cannot claim. Stop the run.
2. Positions that earned in a past epoch but not in this one still get a row. `delta = 0`, same cumulative.

## Distribution: Merkle with running totals

Leaf preimage: `keccak256(abi.encodePacked(positionManager, tokenId, rewardToken, cumulative))`. Leaves sorted by `(positionManager, tokenId)` so the same data always gives the same root.

Each epoch: score, group per position, update cumulatives, build the tree, send the root. The new root replaces the previous one. The Distributor stores only the current root per campaign and `paid[positionManager][tokenId][token]`.

`cumulative` is hashed into the leaf so claims are O(1): the contract rebuilds the leaf hash, verifies against `rootByCampaign` with the proof, and pays `cumulative − paid[posMgr][tokenId][token]`. Users always claim against the latest root.

I didn't sketch a withdraw path. We'd want one for unclaimed rewards after the campaign ends.

## Known limitations

- **`initialTick` fallback can bias the first segment of the first epoch.** Fix: re-seed with a working RPC.
- **Forward-only per pool.** Retroactive campaigns on untracked pools cannot be scored correctly today. Fixes: track `(indexed_from, indexed_to)` per pool, or add a one-shot `backfill` worker started on campaign creation.
- **ALMs and non-PositionManager senders produce no leaves.** Events are indexed but dropped at scoring. Fix: pluggable `PositionResolver` per sender type.
- **`Donate` events are not indexed.** Matters only for fee-growth scoring. Fix: add a `DonateEvent` model.

## Uniswap V3 vs V4, from the tech test point of view

V3: one contract per pool from a factory. Separate `Mint` and `Burn`. Position key is `(owner, tickLower, tickUpper)`. WETH needed for ETH pairs.

V4: one singleton `PoolManager` for every pool, one address to index. Any `uint24` fee. `tickSpacing` is set separately from the fee. Dynamic fees: a hook can change the fee per swap; the fee uint24 has a `0x800000` flag, the real fee is in each `Swap` event. One `ModifyLiquidity` event with a signed delta. The position key includes `salt`, so many positions per owner and range; the PositionManager uses `salt` to encode the tokenId. Native ETH via `address(0)`. Hooks add lifecycle callbacks, with permissions in the low 14 bits of the hook address. Flash accounting and ERC6909 are runtime optimisations, not relevant here.
