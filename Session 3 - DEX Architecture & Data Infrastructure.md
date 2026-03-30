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

### Type 5: DEX Aggregator (OKX DEX model)
Does not operate its own liquidity pools. Instead, aggregates liquidity from hundreds of existing DEXes and routes user trades to achieve the best net execution.

```
User wants: PEPE → USDC

Aggregator scans: 400+ DEX pools across 26 chains
→ splits order across Uniswap V3, Curve, and PancakeSwap
→ submits through private mempool (Flashbots) to block builder
→ user receives best net price, MEV-protected
```

**OKX DEX** is this model: a pure aggregator backed by OKX exchange infrastructure, integrated directly into the OKX self-custody wallet.

| Property | OKX DEX |
|---|---|
| Own liquidity pools | None |
| DEXes aggregated | 400+ across 26 chains |
| Cross-chain swaps | 17 chains, one click |
| Routing algorithm | X-Routing (proprietary) |
| MEV protection | Flashbots private relay |
| Wallet integration | OKX App (passkey-secured self-custody) |

**Why this model:** An aggregator's value proposition is best execution, not liquidity ownership. By routing across all available pools, it consistently outperforms any single DEX on price — especially for large trades where split routing reduces price impact. The moat is the routing algorithm and the data infrastructure powering it, not the liquidity itself.

**Comparison to competitors:**

| | OKX DEX | 1inch | Jupiter |
|---|---|---|---|
| Chain coverage | 26 chains | 9+ chains | Solana only |
| DEXes aggregated | 400+ | 100+ | Multiple Solana DEXes |
| MEV protection | Flashbots | Yes | Limited |
| Wallet integration | OKX App (native) | External wallets | Solana-native |
| Unique angle | Cross-chain breadth + OKX ecosystem | Smart routing maturity | Best UX on Solana |

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

For an aggregator like OKX DEX — which sources from 400+ DEXes across 26 chains — the data problem is compounded: it must maintain accurate, real-time state for every upstream pool it routes through, across chains with different block times, finality models, and event schemas. The data layer is not just supporting the product; it *is* the product.

### Five reasons data infra is non-negotiable

**1. Trade execution quality**
The X-Routing algorithm needs live reserve data from every pool it considers. Stale data = suboptimal path = user gets worse price than a competitor's aggregator would give. For OKX DEX, this means maintaining fresh pool state across 400+ DEX sources simultaneously.

**2. User safety**
Across 26 chains, there are millions of token contracts — the vast majority scams or worthless forks. Without a verified token registry, a user searching "USDC" on an obscure chain could be routed to a honeypot. OKX DEX's token labeling system is the trust layer preventing this.

**3. MEV protection**
OKX DEX integrates Flashbots private relay to shield users from sandwich attacks. This requires real-time mempool monitoring to detect active bots, and routing logic that decides when private relay is warranted vs standard submission.

**4. LP economics**
Not directly applicable to OKX DEX (it doesn't run pools), but relevant for the upstream DEXes it routes through — pool health (TVL, fee APY, IL) feeds into the routing decision. A draining pool is deprioritised.

**5. Platform trust**
OKX DEX's credibility depends on surfacing accurate price quotes, realistic slippage estimates, and transparent fee breakdowns before trade submission. Bad data = users getting worse execution than quoted = trust eroded.

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

**OKX DEX context:** 26 chains with heterogeneous block times (Ethereum ~12s, Solana ~400ms, BSC ~3s). Each chain requires a separate indexing pipeline with chain-specific event schemas. Solana doesn't use EVM-style event logs — its program logs require a different parser entirely.

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

**OKX DEX context:** X-Routing queries pool state from 400+ sources before computing a path. Each quote shown to the user is only valid for a few seconds. The pool state database must be updated continuously across all upstream DEXes — a stale record in a high-volume pool is enough to produce a bad quote that reverts on-chain, degrading user experience.

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

**OKX DEX context:** Operating across 26 chains means maintaining a verified token registry at massive scale. The registry also feeds cross-chain deduplication — the same logical asset (USDC) has a different contract address on every chain, and the registry is what allows OKX Wallet to show "total USDC across all chains" as a single balance.

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

**OKX DEX context:** X-Routing is OKX DEX's proprietary routing engine. It extends the standard model with cross-chain routing — when a user swaps Token A on Chain X for Token B on Chain Y, the engine must simultaneously optimise the single-chain leg, the bridge selection, and the destination-chain leg as one unified path. This requires pool state data, bridge fee/latency data, and gas estimates across two chains at once.

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

**OKX DEX context:** OKX DEX integrates Flashbots Protect to route transactions through a private relay, bypassing the public mempool. The decision of when to use private routing (adding latency) vs standard submission (faster but exposed) is itself a data product — it should be driven by real-time MEV risk signals per pool and per trade size, not applied uniformly.

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

## 7. Real-Time On-Chain Data as Alpha

On-chain data is fully public — but extracting signal faster and more accurately than the market is a competitive edge. Here are three concrete examples of how real-time on-chain data generates alpha for bots and traders.

### Example 1: Liquidation Cascade Prediction

**The data:** On-chain lending protocols (Aave, Compound) expose every collateral position publicly — collateral amount, debt, and liquidation threshold.

```
ETH collateral position:
  Deposited:  100 ETH ($200,000 at $2,000/ETH)
  Borrowed:   $130,000 USDC
  Liquidation threshold: 80% LTV → liquidated if ETH < $1,625

Aggregated across all positions:
  $40M of ETH collateral liquidated if ETH drops to $1,800
  $120M more liquidated if ETH drops to $1,600
```

**The alpha:** Build a "liquidation heatmap" — a price ladder of forced selling at each ETH level. When ETH starts falling toward a dense cluster, a bot can:
1. Short ETH on a perp DEX (anticipating cascaded sell pressure)
2. Position to be the liquidator (liquidators earn a bonus for closing underwater positions)

**Why on-chain only:** CEX order books don't expose pending liquidations — they show current bids/asks only. The liquidation supply is invisible until it hits the market. On-chain, it's fully transparent and queryable in advance.

---

### Example 2: DEX/CEX Price Arbitrage

**The data:** Real-time pool reserve ratios from DEX contracts vs price feeds from centralized exchanges.

```
Binance ETH/USDC:  $2,005
Uniswap V3 pool:   $1,998  ← $7 spread
```

**Why the spread exists:** CEX prices update in milliseconds via order book matching. On-chain pools only update when a transaction is included in a block (~12 seconds on Ethereum). Any macro price move creates a window where the DEX is mispriced.

**The alpha:** Detect the spread faster than other arbitrageurs, buy on the cheap DEX side, sell on the CEX (or hedge with a perp), and capture ~$7/ETH minus gas and fees. Repeated thousands of times per day.

**The infrastructure edge:** The bot that wins is the one with the lowest-latency indexer. Milliseconds matter — the first to see the spread and get their transaction included earns the profit. This is why trading firms run their own Ethereum nodes co-located near block builders rather than relying on third-party APIs.

---

### Example 3: Smart Wallet Tracking

**The data:** Every wallet's full transaction history is public. Historical buys and sells are readable for any address.

**The signal:** Identify wallets with a strong track record — e.g., a wallet that bought early into multiple tokens before they 10×, and consistently exited before the peak. Label it a "smart wallet."

```
Smart wallet 0xabc...:
  Block 18,200,100: bought $50K of unknown token X → 10× three days later
  Block 18,450,200: bought $80K of unknown token Y → 7× five days later
  Block 19,100,050: NOW buying $120K of token Z
  → bot mirrors the trade immediately
```

**The alpha:** Copy a signal wallet before the broader market notices. The window is often minutes — the bot must detect the buy in the mempool (before confirmation) or in the first block it lands, then execute immediately.

**The infrastructure edge:** Requires (a) a historical database of labeled wallets, (b) a real-time transaction stream that triggers on any activity from tracked wallets, and (c) fast execution before other copy-trading bots act on the same signal.

---

### The Common Thread

All three examples follow the same structure:

```
On-chain state (public, real-time)
    → Pattern detected by data layer
        → Predictive signal about future price pressure
            → Trade executed before market fully prices it in
```

The alpha is not from secret information — everything is public. The edge is **infrastructure speed and analytical depth**: faster indexing, richer models (liquidation ladders, wallet reputation scores), and tighter feedback loops between data and execution. This is why data infrastructure is a competitive moat, not just a back-office cost.

---

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| DEX | Decentralized exchange — on-chain trading without a central custodian |
| AMM | Pricing mechanism inside most DEXes; sets price via reserve ratio formula |
| Order book DEX | DEX using bids/asks instead of liquidity pools |
| Intent / RFQ | User declares desired trade; solvers compete to fill it off-chain, settle on-chain |
| DEX aggregator | Routes trades across 100s of DEX pools to find best execution; owns no liquidity itself |
| X-Routing | OKX DEX's proprietary routing algorithm; optimises across 400+ DEXes and 26 chains |
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
