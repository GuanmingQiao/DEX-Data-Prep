# Session 4 — Smart Money, Signals & AI Trading Harness
*Date: 2026-03-30*

---

## 1. What Is a Signal

A signal is any observable data point with **predictive value** for future price movement. The key word is predictive — not descriptive. Price right now is data. The pattern that historically precedes a price move is a signal.

On-chain signals are uniquely powerful because they are:
- **Verifiable** — every signal is backed by an immutable on-chain event
- **Timestamped** — precise to the block
- **Leading** — on-chain activity often precedes CEX price moves

By the time a move shows up on Binance, it frequently already happened on-chain first.

---

## 2. Signal Taxonomy for DEX Data

### Wallet flow signals

Accumulation and distribution patterns reveal intent before price reacts:

```
Accumulation:
  Wallet 0xabc buys $20K of TOKEN_X
  30 min later: buys $25K more
  1h later: buys $30K more
  → Increasing-size, staged buys = conviction building, not a one-off

Distribution:
  Wallet 0xdef sells 5% of position
  Next day: sells 8% more
  → Staged exit = smart money leaving without crashing their own position
```

### Liquidity signals

LP behaviour reveals expectations about future price volatility:

```
Uniswap V3 LP adds a tight range: ETH $1,980–$2,020
→ LP is betting price stays in this narrow band
→ If LP has a strong track record: low volatility expected

Large LP removal:
  Largest LP (60% of pool) withdraws all liquidity
→ Pool depth collapses → remaining trades face high slippage
→ Signal: insider knew something; or price about to leave their range
```

### Pool state signals

```
New pool launched with $500K initial liquidity (unusual for a new token)
→ Either backed project (VC-seeded LP), or team staging a pump

Pool imbalance:
  WETH/USDC: reserve ratio shifts from 50/50 → 40/60
  → More USDC than ETH = net ETH selling into pool
  → DEX spot price now above CEX price
  → Arb bots haven't closed the gap yet — short-lived signal
```

### Cross-market signals

```
Perp funding rate on ETH: +0.15% per 8h (very high)
→ Longs paying shorts heavily = crowded long trade → vulnerable to unwind

CEX inflows: 50,000 ETH moved to Binance hot wallets
→ Historically precedes selling pressure
→ Needs volume confirmation (can be wash trading)
```

### MEV activity as a signal

MEV bots are signal generators, not just extractors:

```
Sandwich activity on MEME_TOKEN doubles in 1 hour
→ Bots sandwich because retail is actively trading
→ Rising sandwich frequency = rising retail interest = momentum signal

Arb bot frequency:
  Normal: closes DEX/CEX spread 20× per hour
  Today:  200× per hour
→ Unusual price volatility or heavy directional flow on DEX
→ Active price discovery happening
```

### Lending protocol signals

```
ETH borrowing utilisation on Aave: 40% → 85% in 6 hours
→ People borrowing to short, or leveraging long with stablecoin loans
→ Rate spike incoming → LP yield changes → capital rotation follows

Liquidation heatmap:
  $200M of leveraged ETH positions liquidated if ETH drops to $1,900
  → Structural support level — also a cascade trigger if breached
```

---

## 3. Smart Money

"Smart money" refers to wallets with a demonstrated track record of being right before the market. The premise: some participants have analytical edge (or material information) that manifests in their on-chain behaviour before it shows up in price.

### What makes a wallet "smart"

Smart money identification requires building a multi-dimensional reputation model:

```
Scoring dimensions:
  Win rate:       % of token positions that were profitable
  Timing alpha:   how early did they enter relative to price discovery?
                  (bought at $0.001, peak was $0.10 = entered at 1% of peak)
  Exit quality:   did they sell near the top, or hold through the decline?
  Risk control:   do they cut losses, or average down into disasters?
  Size scaling:   do their wins hold at larger position sizes? (rules out luck)
  Consistency:    are they right across different market conditions?
```

### Categories of smart money

| Category | Behaviour | What they signal |
|---|---|---|
| Early token buyers | Buy in first blocks of new liquidity | Upcoming price discovery |
| Swing traders | Consistent entry/exit timing across cycles | Directional momentum |
| Liquidation hunters | Watch lending positions, position near cascade levels | Structural price levels |
| Yield optimisers | Rotate capital to highest-yield pools before the crowd | Where yield is migrating |
| MEV searchers | Exploit execution inefficiency | Active retail flow, arb opportunities |

### Building and maintaining the wallet database

```
Step 1 — Index all historical DEX trades by wallet address

Step 2 — For each wallet:
  - Extract every token buy (first purchase of a token)
  - Track peak price after entry
  - Track exit price and timing
  - Compute: entry-to-peak multiple, exit quality score

Step 3 — Filter for statistical validity:
  - Minimum $100K traded (rules out luck at small size)
  - Minimum 20 token positions (statistical significance)
  - Active in last 90 days (not a dormant historical wallet)

Step 4 — Score and label:
  - Tier 1: >70% win rate, >3× avg entry-to-exit, strong timing alpha
  - Tier 2: >55% win rate, decent exit quality
  - Watch list: promising but insufficient track record

Step 5 — Maintain continuously:
  - Decay scores over time (past performance fades)
  - Update on every new trade
  - Flag sudden behaviour changes (wallet compromised? strategy shifted?)
```

---

## 4. The Fundamental Challenge: LLMs and DEX Data

Before building the harness, understand the mismatch between DEX data and LLMs:

| DEX data | LLM characteristics |
|---|---|
| Numerical, time-series | Reasons over language, not numbers |
| Updates every ~12 seconds | Context window is static |
| Millions of pools and tokens | Cannot hold all state in context |
| Raw hex — addresses, wei amounts | Needs human-readable representation |
| Execution is irreversible | Prone to hallucination and arithmetic errors |

The harness exists to bridge this gap: translating on-chain reality into something the LLM can reason over, and translating LLM decisions back into safe, executable on-chain actions.

---

## 5. Context Engineering

### Selection — only inject what's relevant

An agent monitoring an ETH/USDC pool on Arbitrum doesn't need Solana meme coin prices. Context selection requires a semantic model of the active strategy to determine relevance. Irrelevant context is not harmless — it increases hallucination risk and dilutes signal.

### Transformation — from raw to readable

```
Raw on-chain:
  reserve0: 12450000000000000000000  (wei)
  reserve1: 24900000000  (USDC 6-decimal)
  block: 19482301

Transformed for LLM context:
  Pool: WETH/USDC (Uniswap V3, 0.05% fee, Arbitrum)
  ETH reserve: 12,450 ETH
  USDC reserve: $24,900,000
  Spot price: $2,000/ETH
  TVL: $49.8M
  Data age: 4 seconds (block 19,482,301)
```

Never make the LLM do unit conversions or address lookups. That is where arithmetic errors happen.

### Inject signals, not raw data

```
Bad (raw events):
  "0x7a3b bought 450000000000000000 of 0xd3a7... at block 19482301"

Good (pre-computed signal):
  SIGNAL [HIGH CONFIDENCE — 8 seconds ago]:
  Tier 1 smart wallet (78% win rate, $12M traded) purchased $180K
  of TOKEN_X across 4 transactions over 22 minutes (accumulation pattern).
  Last 3 similar patterns preceded 3×–8× moves within 48–72 hours.
  Pool TVL: $800K. Your max safe size at 1% price impact: ~$8K.
```

The agent reasons over the interpretation, not the event.

### Freshness — staleness is dangerous

Market data older than 30 seconds can be materially wrong. Use delta updates: only push what changed meaningfully since last injection. Always attach data age explicitly — without it, the LLM treats all context as equally current.

**Inject in layers:**
```
Layer 1 (always present):
  ETH: $2,000, up 0.3% in 1h. Your trigger: $1,960. Gap: 2.0%.

Layer 2 (injected when relevant):
  Pool depth: $49.8M TVL. Trade size ($50K) = 0.1% of pool.
  Estimated slippage: 0.05%. MEV risk: LOW.

Layer 3 (on tool call only):
  Full OHLCV history, individual swap events, wallet history.
```

---

## 6. Intent Recognition

User: *"buy the dip on ETH if it drops another 3%"*

This is deeply ambiguous. The harness extracts a structured intent object before anything executes:

```json
{
  "asset": "ETH",
  "chain": null,              ← must clarify
  "direction": "buy",
  "trigger": {
    "type": "price_drop_pct",
    "reference_price": "$2,000",
    "threshold": "3%",
    "target_price": "$1,940"
  },
  "size": null,               ← must clarify
  "slippage_tolerance": null, ← use default?
  "expiry": null,             ← does this expire?
  "pool": null                ← which DEX/aggregator?
}
```

**Two approaches to ambiguity:**

*Clarification dialog* — ask for every null before registering the intent. Safe, higher friction. Right for large trades.

*Conservative defaults* — fill nulls with small size, tight slippage, short expiry, and require confirmation before execution. Right for small exploratory trades.

**Compound intents require decomposition:**

*"If ETH breaks $2,100, sell half and put it into stETH"*

This is a DAG of dependent actions: (1) price trigger, (2) partial sell, (3) follow-on DEX swap, (4) Lido deposit — not just a DEX swap. Intent recognition must decompose this into ordered, dependent steps and track execution state across all of them.

---

## 7. Signal Stacking — Conviction from Convergence

A single signal is weak. Multiple independent signals in the same direction build conviction:

```
TOKEN_X conviction stack:

  Signal 1 [MEDIUM]: Tier 1 wallet accumulating — $180K over 22 min
  Signal 2 [MEDIUM]: Pool TVL up 40% in 6h — new capital entering
  Signal 3 [LOW]:    Pool is 3 days old — still early price discovery
  Signal 4 [HIGH]:   Bridge inflows to this chain up 3× this week
  Signal 5 [MEDIUM]: No sandwich bot activity yet — retail hasn't arrived

Composite: HIGH conviction. Five independent signals align.
Recommended: small initial position, scale in if signals hold.
```

The agent weights signals against the active strategy — it does not blindly follow any single one.

---

## 8. Signal Decay — Timing Is Everything

Signals have half-lives. The harness must track when a signal is still actionable:

```
Smart wallet buy detected:
  0–5 min:   HOT — likely before market notices
  5–15 min:  WARM — some participants may have seen it
  15–60 min: COOLING — partially priced in
  >1h:       STALE — likely fully reflected in price

Liquidation heatmap ($200M at $1,900):
  Valid until: ETH price moves away, or positions are closed/liquidated
  Not time-decaying — condition-decaying
```

Every signal in the agent's context should carry an explicit freshness score. Stale signals should be downweighted automatically.

---

## 9. Drift Prevention

Three types of drift, each requiring a different fix:

**Strategy drift** — the agent gradually forgets the original intent as conversation history grows.

*Fix:* Store intent as a structured object outside conversation history. Re-inject it as a system prompt anchor at every decision point:
```
[System — always present]:
  Active strategy: BUY 0.5 ETH when price ≤ $1,940 on Arbitrum.
  Max slippage: 0.5%. Expires: 2026-03-30 18:00 UTC.
  Status: MONITORING. Trigger not hit.
```

**Market drift** — conditions changed materially but the agent is still reasoning from stale context.

*Fix:* Force a live data fetch before any execution decision. If data is older than N seconds, abort and surface to the user — never execute on stale state.

**Hallucination drift** — the LLM confabulates market data when context hasn't been updated recently. ("ETH looks like it's trending up" with no actual data backing it.)

*Fix:* Require every market claim to be traceable to a tool call result within the last N seconds. If the agent makes a claim with no recent tool call backing it, treat it as potentially hallucinated. Block execution unless grounded.

**Instruction drift** — user gives conflicting updates and the agent loses track of net intent.

*Fix:* Surface conflicts explicitly before updating the strategy anchor:
*"Your new instruction (sell if ETH > $2,100) conflicts with your active strategy (buy if ETH < $1,940). Replace, or run both?"*

---

## 10. The Harness Architecture

```
User natural language
        ↓
[Intent Parser]
  → structured intent object + ambiguity flags
        ↓
[Strategy Anchor]
  → canonical store, injected as system prompt at every step
        ↓
        ┌──────────────────────────────┐
        │         LLM Agent            │
        │  - monitors conditions        │
        │  - calls tools to ground      │
        │    all market claims          │
        │  - stacks signals             │
        │  - decides when to execute    │
        └─────────────┬────────────────┘
                      │ tool calls
        ┌─────────────▼────────────────┐
        │          Tool Layer           │
        │  get_price(token, chain)      │
        │  get_pool_state(pool)         │
        │  get_signals(token, chain)    │
        │  get_smart_money(token)       │
        │  check_balance(wallet)        │
        │  estimate_swap(params)        │
        │  execute_swap(params)         │  ← requires guardrail pass
        │  get_tx_status(hash)          │
        └─────────────┬────────────────┘
                      │
        ┌─────────────▼────────────────┐
        │        Guardrail Layer        │
        │  - max position size          │
        │  - max slippage               │
        │  - data freshness check       │
        │  - duplicate execution block  │
        │  - human confirmation gate    │  ← for trades above threshold
        └─────────────┬────────────────┘
                      │
        ┌─────────────▼────────────────┐
        │       Execution Layer         │
        │  - simulate trade first       │
        │  - construct calldata         │
        │  - route (private or public)  │
        │  - submit + monitor           │
        │  - report actual result back  │
        └──────────────────────────────┘
```

**Why simulate before executing:**
Between the LLM's decision and the transaction landing on-chain, pool state can move. Always simulate at execution time — not decision time. If simulation shows worse execution than decided, abort and re-surface rather than proceeding.

---

## 11. The Feedback Loop Risk

If enough agents follow the same smart money signals, the signal self-destructs:

```
Feedback cascade:
  Smart wallet buys TOKEN_X
  → 1,000 AI agents detect signal simultaneously
  → All 1,000 submit buys in the next block
  → Price spikes before wallet finishes accumulating
  → Signal is front-run by the agents following it
  → Smart wallet learns to obscure (smaller trades, time delays, multi-wallet splits)
```

At system scale, synchronised agent behaviour creates new risks:
```
Flash crash pattern:
  Smart wallet sells (routine profit-taking)
  → Agents detect sell signal → all sell simultaneously
  → Price crashes beyond wallet's intent
  → Liquidations triggered → cascade
  → More agents detect cascade → more selling
```

Mitigations the harness must enforce:
- Cap position size relative to pool TVL (agent never exceeds X% of pool)
- Randomise execution timing slightly to break synchronisation
- Circuit breaker: if N% of monitored pools are moving in the same direction simultaneously, pause and require human confirmation before acting

---

## 12. Infrastructure Requirements

| Layer | What it produces |
|---|---|
| Wallet DB | Labeled wallets with reputation scores, trade history, win rates, tier classification |
| Real-time tx stream | Every on-chain transaction — mempool + confirmed — fed to signal engine |
| Signal engine | Processes raw events → typed signals with confidence scores and freshness timestamps |
| Signal API | Tool functions for the LLM: `get_signals()`, `get_smart_money_activity()`, `get_pool_momentum()` |
| Strategy anchor store | Persists structured intent objects outside conversation context |
| Execution receipt feed | Post-trade: actual price, gas paid, MEV detected — closes the loop for signal quality improvement |

The competitive moat is in the **wallet database and signal engine** — not raw indexing. Any team can index swap events. The edge is in correctly identifying which wallets are actually smart, which signals have historical predictive value, and delivering those signals to the agent before they decay.

---

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| Signal | Observable data with predictive value for future price — not just description of current state |
| Smart money | Wallets with a demonstrated track record of entering/exiting before price moves |
| Timing alpha | How early a wallet entered relative to peak price — measures information or analytical edge |
| Accumulation pattern | Increasing-size staged buys over time — signals building conviction, not a one-off |
| Distribution pattern | Staged partial sells over time — smart money exiting without crashing their own position |
| Signal stacking | Multiple independent signals pointing the same direction — raises conviction for action |
| Signal decay | Signals lose predictive value over time as the market prices them in; each has a half-life |
| Liquidation heatmap | Price ladder showing how much forced selling occurs at each price level — structural signal |
| Funding rate | Fee longs pay shorts (or vice versa) on perps — high positive rate = crowded long, unwind risk |
| Strategy anchor | Canonical structured intent stored outside conversation history; re-injected at every decision |
| Strategy drift | Agent gradually deviates from original intent as conversation history grows |
| Market drift | Agent reasons from stale context after conditions have materially changed |
| Hallucination drift | Agent confabulates market data without a grounding tool call backing the claim |
| Grounding check | Live data fetch that must precede any execution decision — confirms context is still valid |
| Intent decomposition | Breaking compound user instructions into a DAG of ordered, dependent executable steps |
| Circuit breaker | Halts execution when many positions move in the same direction simultaneously — prevents cascade |
| Feedback loop | When many agents follow the same signal simultaneously, the signal self-destructs and can cause cascades |
| Execution receipt | Post-trade record of actual price, gas, and MEV outcome — fed back to agent to close the loop |
