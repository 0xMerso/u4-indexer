# Uniswap V4 indexing engine + LP rewards

The test asks for:
- an indexer that saves Uniswap V4 data in a relational database
- a way to use that data to reward LPs

Here is my work: a roadmap (and a Prisma schema in `schema.prisma`).

## Contents

- [Uniswap V4 indexing engine + LP rewards](#uniswap-v4-indexing-engine--lp-rewards)
  - [Contents](#contents)
  - [Overview](#overview)
  - [Scope](#scope)
  - [Out of scope (fix paths in `Known limitations`)](#out-of-scope-fix-paths-in-known-limitations)
  - [Handling NFT positions](#handling-nft-positions)
    - [Positions minted outside a known PositionManager](#positions-minted-outside-a-known-positionmanager)
  - [Architecture](#architecture)
  - [Data model](#data-model)
  - [Hook management](#hook-management)
  - [Scoring: TWAL (Time-Weighted Active Liquidity)](#scoring-twal-time-weighted-active-liquidity)
  - [Distribution: Merkle with running totals](#distribution-merkle-with-running-totals)
  - [Phases](#phases)
  - [Known limitations](#known-limitations)
  - [Uniswap V3 vs V4, from the tech test point of view](#uniswap-v3-vs-v4-from-the-tech-test-point-of-view)

## Overview

- An indexer runs all the time. It reads every V4 `PoolManager` event for the pools we track and saves them to Postgres. It can
  start at any block, past or current.
- A user creates a campaign via a UI or an API: `(poolId, rewardToken, amount, startBlock, endBlock)` and N epochs.
- Each epoch has its own Merkle root.
- A scorer runs every 2 hours, once per epoch. It computes a time-weighted active liquidity per position, writes `Leaf` rows,
  builds the Merkle tree and sends the root on chain.
- Users claim via the `Distributor`. One claim pays all they earned since the campaign started.

## Scope

- 1 campaign = 1 pool.
- Reward token: any ERC20 on the same chain as the pool.
- Forward-only indexer. On empty DB, backfills from the earliest `Campaign.startBlock`. No gap-fill for late-added past campaigns.
  See Known limitations section

## Out of scope (fix paths in `Known limitations`)

- ALM vaults (Gamma, Arrakis and similar).
- Claim API (shape only).
- Rules based on hook permissions.
- Campaigns across many chains.
- Fee-weighted scoring.
- JIT consideration.

## Handling NFT positions

`ModifyLiquidity.sender` is always a contract. It runs inside a V4 `unlock` callback. For most users, that contract is the Uniswap
`PositionManager` (ERC 721), not their own wallet. If we used `sender` as the reward target, all rewards would end up on the
PositionManager (or the ALM vaults), and nobody could claim them.

The trick: V4's PositionManager always passes `salt = bytes32(tokenId)` when it calls `modifyLiquidity`. So the tokenId is already
in `ModifyLiquidityEvent.salt`. No Transfer indexing, no extra table.

Rule at scoring time:

```
if sender is in the known PositionManagers config:
    (positionManager, tokenId) = (sender, uint256(salt))   // score it
else:
    skip                                                   // see below
```

Each `Leaf` is keyed by `(epoch, positionManager, tokenId)`. At claim time the Distributor checks
`IERC721(positionManager).ownerOf(tokenId) == msg.sender`. The NFT is the reward ticket. If it is transferred or burned, the
rewards go with it.

### Positions minted outside a known PositionManager

ALMs (Gamma, Arrakis etc.) or custom routers and bots open positions via their own `unlock` callbacks. Their `sender` is not in
our list, so they get no `Leaf` row. Rewards go only to LPs who used a known PositionManager. It is a limitation we can fix
later, but case by case per sender.

Ways to add them later, one piece of code per sender type, same `(positionManager, tokenId)` output:

- ALMs: one resolver per vault. Map vault shares to the real V4 position using the vault's own accounting. Score the vault, then
  split the leaf across share holders off chain.
- Custom routers or bots: opt-in allowlist, one ownership rule each (EOA sender, NFT wrapper, etc.).
- Shared shape: a pluggable `PositionResolver` trait in the Rust scorer, one impl per `sender` contract.

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

- Indexer: reads `Initialize`, `Swap` and `ModifyLiquidity` from the `PoolManager`. I use Envio. It is faster than `eth_getLogs`
  and handles reorgs, but it is an API dependency. The Uniswap V4 subgraph is another option, faster to set up but less reliable
  than a direct node. I added a `topics[1]` filter on the `LogFilter` so Envio only streams events for pools with an active
  Campaign.
- Postgres: one database with events, campaigns, epochs and leaves.
- Scorer: Rust, runs every 2 hours. Reads finished epochs, scores the closing epoch (writes `Leaf` rows), builds the Merkle tree,
  saves proofs and root, writes the root on chain. It only replays events from the DB. No RPC call during scoring.
- Distributor (Solidity): one contract shared by all campaigns. Stores `rootByCampaign[bytes32]` and
  `paid[positionManager][tokenId][token]`. Interface below.
- Claim API: HTTP, or IPFS if we want something more auditable. The client sends a list of tokenIds. The API returns matching
  leaves and proofs across all campaigns. `GET /claim?address=0xalice&tokenIds=[...]`.

Each service runs in its own Docker container, on a shared docker-compose network.

```
t:       0                         2h                        4h
indexer  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  live, every block
scorer                             ★                         ★  every 2h: score, tree, send root
claim    on demand, against the last root
```

## Data model

See `schema.prisma`, 6 models.

- `Pool`: one row per V4 pool, written on `Initialize`. `hook` is null for pools without a hook. `hookPerms` holds the 14
  permission flags from the hook address. Stored for later use, not read by the scorer today. `initialTick` is set by
  `Initialize`. If the pool was already Initialize'd before we started indexing, `seed` reads the current tick once via
  `StateView.getSlot0(poolId)`. The scorer itself makes no RPC call.
- `SwapEvent`: raw Swap logs. Holds `tickBefore` and `feeAmount`.
- `ModifyLiquidityEvent`: raw logs. V4 merges mint and burn into one signed `liquidityDelta`. `salt` encodes the tokenId.
- `Campaign`: one reward program. Creator, pool, reward token, budget, window.
- `Epoch`: a time slice of a campaign, one Merkle root each. `publishedAt` and `publishedTxHash` kept for audit.
- `Leaf`: one row per `(epoch, positionManager, tokenId)`. `delta` is this epoch's share, `cumulative` is the running total and
  what gets hashed into the leaf. Always tied to one NFT position.

No `Position` table. We rebuild positions from `ModifyLiquidityEvent` on each scoring run. A cache could help for long campaigns.

## Hook management

What we index is the same whether the pool has a hook or not.

- `Swap.tick`, `Swap.amount0/1`, `Swap.fee` and `ModifyLiquidity.liquidityDelta` come out after the hook callbacks run. This
  includes the `beforeSwapReturnDelta` and `afterAddLiquidityReturnDelta` variants. The saved values match the real pool state.
- Dynamic fees: `Pool.fee` keeps the uint24 with the `0x800000` flag. The real per-swap fee is in `SwapEvent.feeAmount`.
- Gap: we don't index `Donate` events. This only matters if we move to fee-growth scoring. Not a blocker for TWAL.

## Scoring: TWAL (Time-Weighted Active Liquidity)

Time based, not fee based. LPs out of range earn zero. Tight ranges win: the same money buys more `L` when the range is smaller.

For each position `i`, sum across every block `b` of the epoch the position's liquidity, but only when the pool's tick is inside
the position's range:

```
score_i = Σ over blocks b in the epoch:  L_i(b) × 1[tick_pool(b) ∈ range_i]
```

- `L_i(b)`: the position's liquidity at block `b`. Changes only on `ModifyLiquidity`.
- `tick_pool(b)`: the pool's tick at block `b`. Changes only on `Swap`.
- `1[...]`: 1 if in range, 0 otherwise.

Both values stay the same between events. So the sum turns into a loop over segments: `Σ L_i × segment_blocks` for each segment
where the position is in range. Running it twice on the same events gives the same score.

This matches the `C` (Liquidity Contribution) part of Merkl's current formula with weights `(0, 0, 1)`. The `A·x + B·y`
token-held parts are the next step. They help creators who want to reward only one token of the pair.

Per epoch:

```
epoch_budget   = budget × (epoch_blocks / total_blocks)
position_delta = epoch_budget × score_position / Σ scores
cumulative     = previous_cumulative + position_delta
```

Rules the scorer must follow:

1. `new_cumulative ≥ previous_cumulative` for every `(positionManager, tokenId)`. If it ever goes down, `Distributor.claim`'s
   subtraction goes below zero and the user cannot claim. Stop the run.
2. Positions that earned in a past epoch but not in this one still get a row. `delta = 0`, same cumulative. Keeps their proof
   valid against the latest root.

No JIT risk, since it's time based.

## Distribution: Merkle with running totals

Leaf preimage: `keccak256(abi.encodePacked(positionManager, tokenId, rewardToken, cumulative))`. We sort leaves by
`(positionManager, tokenId)` so the same data always gives the same root.

Each epoch: score, group per position, update cumulatives, build the tree, send the root. The new root replaces the previous
one. The Distributor stores only the current root per campaign and `paid[positionManager][tokenId][token]`.

```solidity
interface IDistributor {
    function updateRoot(bytes32 campaignId, bytes32 newRoot) external; // owner only

    function claim(
        bytes32 campaignId,
        address positionManager,
        uint256 tokenId,
        address token,
        uint256 cumulative,
        bytes32[] calldata proof
    ) external; // open, caller supplies cumulative and proof from the Claim API
}
```

`cumulative` is a parameter because the contract only stores the root. At claim time, it rebuilds the leaf hash, checks it
against `rootByCampaign` with `proof`, and pays `cumulative − paid[posMgr][tokenId][token]`. One root per campaign at a time.
Users always claim against the latest root.

I didn't sketch a withdraw path. We'd want one for unclaimed rewards after the campaign ends.

## Phases

- P0, Skeleton: indexer reads PoolManager events for one pool. Rows land in local Postgres. Prisma schema applied.
- P1, Indexer: full backfill from the deploy block. Follow the chain head with a finality buffer. Reorg drill.
- P2, Scorer: Rust scorer, runs every 2h. Replay events, match each to its position, build timelines, compute TWAL, update
  running totals. Unit and replay tests.
- P3, Distribution: Merkle tree builder inside the same scorer run. Deploy the Distributor. End-to-end test.

About one week for a Rust engineer who knows some V4 and some reward systems. A few days if they already know both.

## Known limitations

- **`initialTick` falls back to 0 if the `StateView` RPC fails at seed time.** `seed` reads the current tick once via
  `StateView.getSlot0(poolId)`. If that call fails, the stub stays at 0 and the first segment of the first epoch is marked as
  out of range when it should be in range. Fix: re-seed with a working RPC, or turn the warning into a hard error.
- **Forward-only per pool.** Only pools with a Campaign row get indexed. If you add a campaign with a `startBlock` earlier than
  what we already indexed, the gap is not filled. So a retroactive campaign on a pool we never tracked cannot be scored
  correctly today. In production, backfill is needed from day one. Creators want to launch retroactively, and new pools show up
  all the time. Two fixes:
  - Single program: track `(indexed_from, indexed_to)` per pool in the main indexer. Less code, fine for v1.
  - Two programs: keep the live indexer forward-only, add a one-shot `backfill` worker started on campaign creation for a past
    `startBlock`. Needs a `PoolCoverage` table and a scorer completeness check. Scales better, can run in parallel, and the
    same code handles reorg recovery.
- **ALMs and non-PositionManager senders produce no leaves.** Events are indexed but dropped at scoring. Fix: pluggable
  `PositionResolver` per sender type, see `Handling NFT positions`.
- **`Donate` events are not indexed.** Not usefull for now. only matters if we move to fee-growth scoring. Fix: add a `DonateEvent` model and index
  it alongside the rest.

## Uniswap V3 vs V4, from the tech test point of view

V3: one contract per pool from a factory. Separate `Mint` and `Burn`. Position key is `(owner, tickLower, tickUpper)`. WETH
needed for ETH pairs.

V4: one singleton `PoolManager` for every pool, one address to index. Any `uint24` fee. `tickSpacing` is set separately from
the fee. Dynamic fees: a hook can change the fee per swap. The fee uint24 has a `0x800000` flag, and the real fee is in each
`Swap` event, not in pool state. One `ModifyLiquidity` event with a signed delta. The position key includes `salt`, so you can
have many positions per owner and range. The PositionManager uses `salt` to encode the tokenId. Native ETH via `address(0)`.
Hooks add lifecycle callbacks, with permissions in the low 14 bits of the hook address. Flash accounting and ERC6909 are
runtime optimisations, not relevant here.

