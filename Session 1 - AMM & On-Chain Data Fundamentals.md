# Session 1 — AMM & On-Chain Data Fundamentals
*Date: 2026-03-26*

---

## 1. CoinGecko Product Structure

Two distinct product lines under one "Pro API":

| Product | Data source | API prefix |
|---|---|---|
| **CoinGecko** | CEX — aggregates exchange APIs | `/coins/...`, `/global/...` |
| **GeckoTerminal** | On-chain — indexes blockchain nodes | `/onchain/...` |

"Pro API" refers to the **subscription tier**, not the data source. The Demo/free tier covers most CEX endpoints but almost no on-chain endpoints.

---

## 2. What is OHLCV

**O**pen, **H**igh, **L**ow, **C**lose, **V**olume — five data points describing price action within a time interval (a "candle").

| Field | Meaning |
|---|---|
| Open | Price of the first swap in the interval |
| High | Highest swap price in the interval |
| Low | Lowest swap price in the interval |
| Close | Price of the last swap in the interval |
| Volume | Sum of USD value of all swaps in the interval |

**How on-chain OHLCV is built:** GeckoTerminal indexes individual swap events from DEX pool contracts and buckets them into time intervals (1s / 1m / 5m / 15m / 1h / 2h / 4h / 8h / 12h / 1d). Each swap carries an implied price (ratio of tokens exchanged) and a USD value. Intervals with zero swaps are skipped by default.

---

## 3. AMM Mechanics (Constant Product)

### The formula
Every Uniswap V2-style pool enforces:
```
x × y = k   (constant product)

x = reserve of Token A
y = reserve of Token B
k = constant (only during swaps)
```

### Price derivation
Price is the ratio of reserves:
```
Pool: 1,000 ETH + 2,000,000 USDC
Spot price = 2,000,000 / 1,000 = $2,000/ETH
```

### Two ways to read the same price
- **Reserve ratio** — the spot price *right now*, between swaps
- **Swap-implied price** — the average price *of a specific trade* (USDC paid / ETH received)

For small trades these are nearly identical. For large trades they diverge — that divergence is **slippage**.

### What k actually does
- `k` stays constant **during swaps** — this is the AMM pricing rule
- `k` changes **during LP deposits/withdrawals** — deposits increase k, withdrawals decrease k
- `k` grows slightly **with every fee-bearing trade** — fees accumulate in reserves

---

## 4. Slippage

The extra cost paid due to your trade moving the price against you.

**Formula:**
```
Slippage ≈ trade_size / (pool_reserve + trade_size)
```

**Example (buying 10 ETH from pool of 1,000 ETH / 2,000,000 USDC):**
```
Expected cost (spot price × amount): 10 × $2,000 = $20,000

Actual cost (enforcing k):
  New ETH reserve = 990
  New USDC reserve = 2,000,000,000 / 990 = 2,020,202
  Actual cost = 2,020,202 - 2,000,000 = $20,202

Slippage = (20,202 - 20,000) / 20,000 = 1.01%
```

Slippage is why `reserve_in_usd` (pool depth) is one of the most important signals in on-chain data — a deeper pool means less slippage for the same trade size.

---

## 5. Trading Fees vs Slippage

These are separate costs that are easy to conflate:

| | Trading fee | Slippage |
|---|---|---|
| Who sets it | Pool contract (fixed %) | AMM formula (variable) |
| Who receives it | LPs (stays in pool) | Nobody |
| Depends on trade size | No | Yes |
| Can you avoid it | No | Yes — use a bigger pool |

**Combined cost example:**
```
Spot price:                    $2,000/ETH
Buy 10 ETH

Cost without fee or slippage:  $20,000
+ Slippage:                    +$202
+ Fee (0.3% on $20,202):       +$61
─────────────────────────────────────
Total paid:                    $20,263
```

**Fee mechanics:**
- Fee is charged in the **input token** (the token you put into the pool)
- Fee is collected by withholding it from the AMM calculation — pool receives full input but calculates output on the post-fee amount
- This causes k to grow slightly after every trade — the compounding mechanism behind LP returns

---

## 6. Smart Contract Architecture

### Tokens are smart contracts
Each token is an ERC-20 contract — essentially a ledger mapping addresses to balances:
```
balances = {
  wallet_A:    1000,
  wallet_B:    500,
  pool_address: 2,000,000   ← pool "holds" tokens here
}
```

"Holding" tokens = having a non-zero entry in a token contract's balance mapping. There is no physical custody.

### Pools are smart contracts
A pool contract has its own address, which appears in the balance mappings of both token contracts it holds.

### How a swap works (step by step)
```
1. User calls approve() on Token A contract
   → authorizes pool to pull Token A from user's wallet

2. User calls swap() on pool contract

3. Pool contract internally:
   a. Calls transferFrom() on Token A contract → moves user's tokens in
   b. Verifies new_x × new_y ≥ k (enforces AMM formula)
   c. Calls transfer() on Token B contract → moves tokens out to user

4. Pool emits Swap event with all amounts
   → GeckoTerminal indexes this event
```

All steps are atomic — either all succeed or none do.

---

## 7. LP Token Economics

### How LPs are tracked
Pools issue **LP tokens** — a third ERC-20 contract — to track proportional ownership:
```
You deposit 1 ETH + 2,000 USDC (= 0.1% of pool)
You receive 1,000 LP tokens out of 1,000,000 total (= 0.1%)
```

LP tokens represent a claim on your share of the pool's reserves at withdrawal time.

### LP returns = fees earned − impermanent loss

**Fees earned:** Every trade leaves fee residue in the pool, growing k. LP tokens appreciate as k grows.

**Impermanent loss (IL):** When token prices diverge from your entry ratio, the AMM automatically rebalances — selling the appreciating token and buying the depreciating one. You exit with less of the winner than if you had just held.

```
IL example (ETH doubles from $2,000 to $4,000):
  Deposited:  1.000 ETH + 2,000 USDC = $4,000
  Withdrawn:  0.707 ETH + 2,828 USDC = $5,656
  Just held:  1.000 ETH + 2,000 USDC = $6,000
  IL = ($6,000 - $5,656) / $6,000 = 5.7%
```

IL scale with price divergence:
```
2×  price move → ~5.7% IL
4×  price move → ~20% IL
10× price move → ~42% IL
```

### Withdrawal
LPs always receive **both tokens** in the current reserve ratio — you cannot withdraw a single token. If prices moved since deposit, you get less of the token that appreciated and more of the one that fell.

### Deposit rule
You must deposit in the **exact current reserve ratio**. An imbalanced deposit would change the price, which would be a price manipulation attack vector.

---

## 8. Data Trust & Node Infrastructure

### Multi-node reconciliation
CoinGecko connects to multiple nodes per chain for reliability and correctness. Discrepancies are resolved using **block hashes** — a cryptographic commitment to the entire chain state. Two nodes reporting the same block hash are guaranteed to have identical state; no need to compare individual records.

In practice: managed node providers (Alchemy, Infura, QuickNode) handle node clusters internally. CoinGecko likely runs own nodes for highest-volume chains.

**Finality risk:** On chains with probabilistic finality, blocks can be reorganized shortly after mining. CoinGecko waits for confirmations before treating swap events as final — trading freshness for correctness.

### Token authenticity — the curated registry problem
Token name and symbol fields are **self-declared** in the smart contract. Anyone can deploy a contract called "USD Coin" with symbol "USDC." On-chain data alone cannot distinguish fake from real.

**CoinGecko's solution: human-curated registry**
```
"usdc" (CoinGecko ID) → {
  ethereum: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48  ← verified
  solana:   EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v
  ...
}
```

Verified by cross-referencing official project announcements, on-chain activity volume, and manual team review.

**For unverified tokens (GeckoTerminal):** Everything gets indexed, including fakes. Trust is signaled via:
- GT Score (0–100): composite of engagement, volume, liquidity, security checks
- `checks=on_coingecko` megafilter: only show pools with CoinGecko-verified tokens
- Contract age, locked liquidity, honeypot detection

**OKX implication:** The curated token registry is the trust foundation for the entire product stack. Without it: search surfaces scams, wallets show fake balances, DEX routes to honeypot pools. The "labeling system" in the JD is essentially this registry plus the trust scoring layer on top of it.

---

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| AMM | Smart contract that prices tokens using a mathematical formula instead of an order book |
| Constant product | x × y = k — the formula that governs price in Uniswap V2-style pools |
| Spot price | Current price = ratio of pool reserves |
| Slippage | Extra cost from your trade moving the price; grows with trade size, shrinks with pool depth |
| Trading fee | Fixed % charged on every swap; stays in pool as LP reward |
| LP token | ERC-20 token representing your proportional ownership of a pool |
| Impermanent loss | Loss vs holding from AMM auto-rebalancing when prices diverge |
| OHLCV | Open/High/Low/Close/Volume — price action summary for a time interval |
| Swap event | Blockchain log emitted by pool contract on every trade; primary data source for GeckoTerminal |
| Curated registry | Human-verified mapping of canonical token contract addresses; root of trust for search and labeling |
| GT Score | CoinGecko's composite trust score for DEX tokens (0–100) |
| FDV | Fully Diluted Valuation = price × total supply (vs market cap = price × circulating supply) |
