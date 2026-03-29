# Session 3 — DEX Architecture & Data Infrastructure
*Date: 2026-03-29*

---

## 1. AMM vs DEX — Clarifying the Terms

These two terms are often used interchangeably but describe different things:

| | AMM | DEX |
|---|---|---|
| What it is | A pricing mechanism | A trading platform/protocol |
| Level | Component | System |
| Defines | How price is calculated | How trades are facilitated without a central party |
| Examples | Constant product (x×y=k), StableSwap, CLMM | Uniswap, Curve, dYdX, CoW Protocol |

**An AMM is one way to build a DEX.** Most DEXes today use an AMM under the hood, but a DEX can also use an order book or RFQ system.

The defining property of a DEX is non-custody: users keep their tokens in their own wallets. A smart contract executes the trade atomically — there is no central party who holds your funds or matches your order.

---

## 2. DEX Types

### Type 1: AMM DEX (dominant model)
Liquidity sits in on-chain pools. Price is set by the AMM formula. Anyone can LP. Trade executes against the pool.

Examples: Uniswap V2/V3, Curve, PancakeSwap, Balancer

```
User wallet → Router contract → Pool contract → User wallet
              (finds best path)  (executes swap, emits Swap event)
```

### Type 2: Order book DEX
On-chain order book — bids and asks posted as transactions. Matching happens on-chain or off-chain with on-chain settlement.

Examples: dYdX (off-chain matching, on-chain settlement), Serum (on-chain)

Tradeoff: more familiar to traders; expensive on L1 (every order = gas); better suited to L2s or app-specific chains.

### Type 3: RFQ / Intent-based DEX
User submits an "intent" (I want to swap X for Y). Off-chain solvers compete to fill it at the best price, then settle on-chain.

Examples: CoW Protocol, 1inch Fusion, UniswapX

```
User signs intent → Solver network competes → Best solver settles on-chain
```

Key advantage: solvers can source liquidity from CEXes, other DEXes, private inventory — user gets best price without touching AMM pools directly. Also naturally MEV-resistant (batch settlement, no public pending trade).

### Type 4: Concentrated liquidity AMM (Uniswap V3 model)
LPs provide liquidity only within a chosen price range (e.g., ETH between $1,800–$2,200). Capital is deployed more efficiently — deeper liquidity near the current price.

Tradeoff: higher LP returns when price stays in range; 100% impermanent loss (full exposure to one token) if price exits range entirely.

---

## 3. DEX Smart Contract Architecture

A production DEX is not one contract — it is a system:

| Contract | Role |
|---|---|
| **Factory** | Deploys new pool contracts; maintains registry of all pools |
| **Pool** | Holds reserves; executes swaps; emits events; issues LP tokens |
| **Router** | Accepts user trade request; finds best path across pools; calls pool(s) |
| **Quoter** | Read-only; simulates a trade to return expected output (used by UIs) |
| **Position Manager** | For V3-style DEXes: manages LP positions with price ranges |

### How a multi-hop trade works
```
User wants: SHIB → USDC (no direct pool exists)

Router finds path:  SHIB → WETH → USDC  (two pools)

Step 1: Router calls SHIB/WETH pool → receives WETH
Step 2: Router calls WETH/USDC pool → receives USDC
Step 3: USDC sent to user wallet

All atomic — if step 2 fails, step 1 reverts too.
```

---

## 4. Why DEXes Need Data Infrastructure

A DEX's smart contracts handle execution but produce no intelligence on their own. The contracts just emit raw events. Everything a user sees — prices, charts, routing, risk warnings — requires a data layer built on top.

### The raw data problem
```
What the chain gives you:
  Swap event: { pool: 0x8ad..., amount0In: 1000000, amount1Out: 499, sender: 0x... }

What the user needs:
  "You bought 0.499 ETH for 1,000 USDC at $2,004/ETH
   Price impact: 0.2% | Fee: $3.00 | MEV risk: LOW"
```

Bridging that gap is the data infrastructure problem.

### Five reasons data infra is non-negotiable

**1. Trade execution quality**
The router needs to know live pool reserves to compute price impact and find the optimal path. Stale data = worse execution = users get less than they should.

**2. User safety**
Without a token registry and trust scoring layer, users can be routed to honeypot pools or buy scam tokens with the same ticker as a legitimate asset.

**3. MEV protection**
Detecting sandwich bots in the mempool, scoring pools by MEV risk, and recommending private routing all require a real-time data layer the contracts don't provide.

**4. LP economics**
LPs need to know their fees earned, current IL, and yield vs alternatives to make rational capital allocation decisions. The contracts track balances but not P&L.

**5. Platform trust**
Charts, volume, TVL, and market data are how users evaluate whether a pool or token is legitimate. A DEX without analytics looks like a scam itself.

---

## 5. Types of Data Infrastructure

### Layer 1: Blockchain Indexing

The foundation. Reads raw data from chain nodes and makes it queryable.

**What it does:**
- Connects to full nodes (own nodes or managed providers: Alchemy, Infura, QuickNode)
- Streams every new block; parses transaction receipts for event logs
- Indexes Swap, Mint, Burn, Transfer events by pool address and block number
- Tracks block finality — waits for confirmations before treating events as final
- Handles chain reorganizations (reorgs) — rolls back indexed data if a block is orphaned

**Output:** A queryable database of all historical and live on-chain events.

**Scale:** For a multi-chain DEX — indexing 20+ chains simultaneously, each at 1–12 blocks/second, each block containing hundreds of events.

---

### Layer 2: Pool State Database

Real-time snapshot of every pool's reserves, fees, and liquidity.

**What it does:**
- After every Swap/Mint/Burn event, recalculates pool reserves
- Stores current `reserve0`, `reserve1`, `fee_tier`, `TVL_usd` for every pool
- Powers: routing engine, price impact calculator, slippage recommendations

**Example state record:**
```
pool: WETH/USDC (Uniswap V3, 0.05% fee)
reserve_eth:  12,450 ETH
reserve_usdc: 24,900,000 USDC
spot_price:   $2,000/ETH
TVL:          $49,800,000
last_updated: block 19,482,301
```

**Why freshness matters:** A router using 10-second-old reserves may compute a different output amount than the pool will actually deliver — causing the transaction to revert (slippage exceeded).

---

### Layer 3: Token Registry

Human-verified mapping of canonical token identities to contract addresses across chains.

**What it does:**
- Maps logical asset ("USDC") to its verified contract address on every chain
- Assigns trust labels: verified, unverified, flagged, honeypot
- Powers: search, routing (prevent routing to fake tokens), wallet balance display

**How trust is established:**
- Cross-reference official project announcements
- Volume and age heuristics (established tokens have long history)
- Security checks: is the contract upgradeable? Is liquidity locked? Is there a mint backdoor?
- Manual review for high-value assets

**Without this layer:** A user searching "USDC" would see thousands of contracts named "USD Coin" — most fake, some honeypots that allow buying but not selling.

---

### Layer 4: Price Feeds & Oracles

Aggregated, manipulation-resistant price data.

**Two types:**

| Type | How it works | Use case |
|---|---|---|
| **Real-time spot** | Latest pool reserve ratio | UI display, routing |
| **TWAP oracle** | Time-weighted average of recent prices | On-chain use (lending protocols, derivatives) — resistant to single-block manipulation |

**TWAP example:**
```
Instead of: price = current reserve ratio (manipulable in one block)
Use: average price over last 30 minutes of blocks

Attacker would need to sustain manipulation for 30 minutes at enormous cost
→ economically infeasible for most pools
```

**Cross-source aggregation:**
For major tokens, aggregate prices from multiple DEX pools + CEX feeds. Divergence between sources signals bridge congestion, pool imbalance, or active manipulation.

---

### Layer 5: Routing Engine

Finds the optimal execution path for a trade across all pools and chains.

**Inputs:**
- Current pool state database (reserves, fees, TVL)
- Gas cost estimates per chain
- MEV risk scores per pool (optional)

**What it optimizes:**
```
Standard:    minimize (price impact + trading fees + gas)
MEV-aware:   minimize (price impact + fees + gas + expected MEV extraction)
```

**Split routing:**
For large trades, splitting across multiple pools reduces per-pool price impact:
```
Buy 1,000 ETH total:
  → 400 ETH from Pool A (Uniswap V3, 0.05%)
  → 350 ETH from Pool B (Curve)
  → 250 ETH from Pool C (Uniswap V2)
Better net price than hitting any single pool for the full amount.
```

This is the core product of DEX aggregators (1inch, Paraswap, Jupiter on Solana).

---

### Layer 6: Mempool Monitoring

Indexes pending (unconfirmed) transactions — the layer before blocks.

**What it does:**
- Subscribes to the public mempool via node connections
- Detects sandwich bot patterns: a pending frontrun directly followed by a target trade
- Feeds MEV risk alerts in real time

**Infrastructure difference from block indexing:**
Block indexing is append-only and permanent. Mempool is ephemeral — transactions appear, may be replaced, and disappear when included in a block or dropped. Requires a separate streaming pipeline, not a standard database.

**Output:** Real-time MEV alerts; feeds into private mempool routing recommendations.

---

### Layer 7: Analytics Pipeline

Aggregates raw event data into business and market metrics.

**Metrics produced:**

| Metric | How it's computed |
|---|---|
| Volume (24h) | Sum of `amount_usd` across all Swap events in window |
| TVL | Sum of `reserve_usd` across all active pools |
| Fees earned (pool) | Volume × fee_tier |
| LP return | Fees earned − impermanent loss |
| Unique traders | Distinct sender addresses in Swap events |
| Token price | Reserve ratio × reference stablecoin price |
| OHLCV candles | Bucket Swap events by time interval |

**Update frequency tradeoff:**
Real-time metrics (price, volume) need sub-second updates. Historical analytics (weekly TVL trend, LP APY) can be batch-computed hourly. Running everything in real time is expensive — a mature data platform separates hot and cold pipelines.

---

### Layer 8: Risk & Anomaly Detection

Monitors for abnormal behavior that signals attacks, scams, or market stress.

**Signal types:**

| Signal | What it indicates |
|---|---|
| Sudden TVL drop (>50% in one block) | Possible rug pull or exploit |
| Price divergence from cross-chain reference | Bridge congestion or oracle manipulation |
| Single wallet providing >80% of pool liquidity | Concentration risk — easy rug |
| Swap volume spike with no external news | Possible wash trading or coordinated pump |
| New pool with copy-cat token name | Scam token attempting to intercept search traffic |
| Contract with mint/pause function in owner hands | Honeypot risk |

**Output:** Risk labels surfaced to users pre-trade; automated alerts to the ops team; input to the token registry trust score.

---

## 6. How the Layers Connect

```
Chain nodes
    │
    ▼
[Layer 1: Block Indexer] ──────────────────────────────────────────┐
    │                                                               │
    ├──► [Layer 2: Pool State DB] ──► [Layer 5: Routing Engine]    │
    │                                        │                     │
    ├──► [Layer 3: Token Registry] ──────────┤                     │
    │                                        │                     │
    ├──► [Layer 4: Price Feeds] ─────────────┤                     │
    │                                        ▼                     │
    ├──► [Layer 7: Analytics Pipeline]    DEX UI / API             │
    │                                                               │
[Layer 6: Mempool Monitor] ──► [Layer 8: Risk Detection] ──────────┘
```

Every layer downstream is only as good as the indexing layer above it. Node reliability, event parsing correctness, and finality handling are the root dependencies for the entire stack.

---

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| DEX | Decentralized exchange — on-chain trading without a central custodian |
| AMM | Pricing mechanism inside most DEXes; sets price via reserve ratio formula |
| Order book DEX | DEX using bids/asks instead of liquidity pools |
| Intent / RFQ | User declares desired trade; solvers compete to fill it off-chain, settle on-chain |
| Factory contract | Deploys and tracks all pool contracts for a DEX protocol |
| Router contract | Finds best multi-hop path and calls pool contracts on user's behalf |
| Quoter contract | Read-only simulation of a trade — used by UIs to preview output |
| Multi-hop trade | Trade routed through 2+ pools because no direct pool exists |
| Concentrated liquidity | LPs provide liquidity only within a price range (Uniswap V3 model) |
| TWAP oracle | Time-weighted average price — manipulation-resistant on-chain price feed |
| Routing engine | Software that finds optimal execution path using live pool state |
| MEV risk score | Per-pool signal for how often it's sandwiched — input to routing decisions |
| Mempool | Pending transaction queue — public, readable, the source of MEV opportunity |
| Rug pull signal | Sudden large TVL withdrawal by a concentrated LP — detectable via pool event monitoring |
| Hot pipeline | Real-time data processing (price, mempool) — latency-sensitive |
| Cold pipeline | Batch analytics (TVL trend, LP APY) — latency-tolerant, cheaper to run |
