# Session 2 — Cross-Chain & MEV Protection
*Date: 2026-03-29*

---

## 1. Cross-Chain Mechanics

### Five bridge mechanisms

| Mechanism | How it works | Trust model | Risk |
|---|---|---|---|
| Lock-and-mint | Lock on Chain A → mint wrapped token on Chain B | Bridge contract keys | Locked assets = honeypot |
| Burn-and-mint | Burn on Chain A → mint fresh on Chain B | Token issuer | Centralized issuer required |
| Liquidity pools | Deposit into pool on A → relayer pays from pool on B | Relayer network | Pool imbalance → delays |
| Messaging protocols | Pass arbitrary messages between chains | Validator/relayer set | Validator collusion |
| Canonical rollup bridge | Native L2↔L1 bridge using rollup proof system | Rollup proof system | Slow withdrawal (7 days for optimistic) |

### Lock-and-mint (most common)
```
User locks 1 ETH in bridge contract on Ethereum
→ Bridge mints 1 wETH on BSC

To return: burn wETH on BSC → bridge releases ETH on Ethereum
```

The destination asset is a **synthetic** — an IOU created by the bridge, only redeemable if the source contract still holds the locked collateral. The core invariant: `tokens locked == wrapped tokens in circulation`. If an attacker breaks this (by forging a mint without a corresponding lock), the wrapped token becomes worthless.

**Honeypot risk:** All locked assets accumulate in a single source contract — a high-value target. Historical losses: Ronin ($625M), Wormhole ($320M), Nomad ($190M).

**How the hack works:** The bridge contract accepts a signed attestation from a validator set before minting. Three ways this breaks:
- **Validator key theft** — attacker steals enough keys to forge a valid signature set (Ronin: 5-of-9 keys compromised)
- **Signature verification bug** — contract logic flaw allows forged attestation to pass (Wormhole: missing account validation on Solana)
- **Bad initialisation** — misconfiguration makes all messages pass verification by default (Nomad: trusted root accidentally set to `0x0`)

**Fragmented wrapped assets:** Each bridge that lock-and-mints creates its own wrapped token. `wETH via Ronin ≠ wETH via Wormhole` — they are different contracts. If the issuing bridge is hacked, that specific wrapped token becomes worthless even though native ETH is unaffected.

**Validators are humans, not smart contracts.** The on-chain contract only checks signatures cryptographically. The private keys belong to off-chain operators — companies or individuals who run validator software. The smart contract logic can be flawless and the bridge still gets hacked via key management failures.

**How users establish trust:** audited open-source contracts, multi-sig with credible independent validators, time-locks on admin actions, and immutable contracts. The emerging trustless alternative is ZK light client bridges — destination chain verifies a ZK proof of source chain state, removing the human validator layer entirely.

### Burn-and-mint (safest for stablecoins)
Circle's CCTP for USDC: issuer burns on source chain, mints fresh on destination. No locked pool to exploit. Only possible when the token issuer controls minting on all chains.

### Liquidity pool bridges
```
User deposits 1 ETH on Ethereum → Hop Protocol
→ Relayer pays 0.999 ETH from Hop's Arbitrum pool (minus fee)
```

The destination asset **already exists natively** on the destination chain — no synthetic is created. The user receives real ETH on Arbitrum, not a bridge-specific wrapper. LPs pre-fund pools on both sides and earn fees for providing this liquidity.

**Key difference from lock-and-mint:**

| | Lock-and-mint | Liquidity pool |
|---|---|---|
| Destination asset | Synthetic/wrapped | Native asset |
| Honeypot risk | Yes — single locked contract | No — liquidity spread across pools |
| Pool imbalance risk | No | Yes — one side drains under high demand |
| Speed | Validator confirmation time | Fast — relayer pays from existing pool |

**Pool imbalance:** if everyone bridges ETH from Ethereum → Arbitrum, the Arbitrum pool drains faster than LPs refill it. New bridge requests fail or queue until rebalanced.

### Cross-chain messaging protocols
LayerZero, Axelar, Wormhole, Chainlink CCIP — the **infrastructure layer** that bridges and other cross-chain apps build on. They don't move assets themselves; they deliver arbitrary data between chains.

**What they actually do:**
```
Chain A: any event occurs
→ Validators attest: "this event happened at block N"
→ Signed message delivered to Chain B contract
→ Chain B contract receives payload and does whatever you programmed
```

The payload can be anything: mint a token, release from a pool, update a price oracle, execute a governance vote, trigger a liquidation. Lock-and-mint and liquidity pool bridges are just two specific applications built on top of this transport layer.

**Why this matters — composability:**
```
Cross-chain governance:   DAO votes on Ethereum → outcome executed on Polygon
Cross-chain lending:      collateral on Arbitrum → borrow against it on Ethereum
Cross-chain NFT:          lock NFT on Ethereum → playable version minted on gaming chain
```

None of these move assets — they move information. Messaging protocols make this possible without each app building its own validator set.

**The trust problem doesn't go away.** Messaging protocols still rely on a validator/relayer set. Every application built on LayerZero inherits LayerZero's security assumptions. A compromised messaging layer puts all applications on top of it at risk simultaneously — broader blast radius than a single bridge.

**Relationship to ZK bridges:** ZK light client bridges are an alternative architecture where the message is verified via a cryptographic proof of chain state rather than by a trusted validator set. They achieve the same delivery guarantee without the human layer.

### Canonical rollup bridges
- Deposit: Ethereum → L2 in ~10 minutes
- Withdrawal: L2 → Ethereum takes **7 days** for optimistic rollups (Arbitrum, Optimism, Base) due to the fraud proof challenge window
- ZK rollups: faster (minutes to hours) — validity proven cryptographically, no challenge window needed

**Canonical vs ZK light client bridges:** A canonical rollup bridge is built into one specific L2↔L1 pair by the rollup team. A ZK light client bridge is a standalone protocol applying ZK proofs to arbitrary chain pairs. A ZK rollup's canonical bridge uses ZK proofs — but only for that one L2↔L1 corridor, not cross-chain generally.

---

## 2. Cross-Chain Data Products

### The core problem
The same asset has a different contract address on every chain. Liquidity is fragmented. Standard on-chain indexing (per-chain) cannot answer: "what is my total ETH across all chains?"

### Product 1: Canonical token registry
Maps one logical asset to its verified addresses on every chain:
```
USDC:
  ethereum: 0xa0b86991...
  solana:   EPjFWdd5...
  arbitrum: 0xff970a61...
  base:     0x833589...
```
The root of trust for cross-chain data. Without it, search returns thousands of fake tokens per chain. CoinGecko's `/coins/list` + `/asset_platforms` is the public reference implementation.

### Product 2: Cross-chain price consistency monitoring
Same token should trade at the same price on all chains (arbitrageurs maintain parity). When prices diverge, it signals bridge congestion or pool imbalance:
```
USDC on Ethereum: $1.000
USDC on distressed L2: $0.997  ← bridge withdrawal backed up
```
Both a risk signal and a trading opportunity signal.

### Product 3: Bridge flow monitoring
Track net asset flows between chains to surface capital rotation narratives:
```
ETH: Ethereum → Base (net outflow)  ← Base ecosystem gaining traction
BNB: BSC → Ethereum (net outflow)   ← capital leaving BSC
```
Reference: DefiLlama's bridges dashboard. Not covered by CoinGecko.

### Product 4: Cross-chain TVL aggregation
A protocol like Aave exists on 10+ chains. True TVL = sum across all deployments. Requires canonical registry + per-chain indexing. Reference: DefiLlama.

### Product 5: Unified portfolio view
User-facing: show all assets across all chains as one portfolio.
Requires:
- Balance indexer running on 100+ chains simultaneously
- Canonical registry to deduplicate (1 ETH on Arbitrum + 0.5 ETH on Base = 1.5 ETH total)
- Bridge position tracking (assets in transit between chains)

This is one of the hardest data engineering problems in Web3, and a flagship feature of leading multi-chain wallets.

---

## 3. Sandwich Attacks

### What it is
A sandwich attack is a form of **MEV (Maximal Extractable Value)** — value extracted by manipulating transaction ordering.

### Mechanics
```
Pool: 1,000 ETH / 2,000,000 USDC  → spot price $2,000/ETH

Victim: pending transaction to buy 50 ETH, 2% slippage tolerance (max $2,040)

Step 1 — Frontrun: attacker buys 30 ETH first (higher gas fee)
  → price moves to $2,061

Step 2 — Victim executes at $2,061
  → within 2% tolerance so doesn't revert

Step 3 — Backrun: attacker immediately sells 30 ETH
  → price returns to ~$2,000
  → attacker profit: ~$1,800

Victim's loss: paid $2,061 instead of $2,000 — extracted, not earned by pool
```

The attacker needs no special access — the public mempool broadcasts all pending transactions.

---

## 4. Sandwich Attack Prevention

### Mechanism 1: Slippage tolerance
Every swap includes `minAmountOut` in the contract. If frontrun pushes price beyond tolerance, transaction reverts:
```
Victim sets 0.5% slippage → minAmountOut = 49.75 ETH
Attacker's frontrun → victim would only receive 49 ETH
→ Transaction reverts → sandwich fails → attacker loses gas
```
Tradeoff: too tight = frequent reverts on volatile tokens; too loose = sandwich target.

### Mechanism 2: Private mempools
Route transaction directly to a block builder — never enters public mempool:
```
Standard: User → Public mempool → All bots see it → Block
Protected: User → Flashbots Protect / MEV Blocker → Direct to builder → Block
```
Transaction revealed only when included in block — too late to sandwich.
Examples: Flashbots Protect, MEV Blocker, 1inch Fusion mode.

### Mechanism 3: Batch auctions
Collect orders in a time window, match off-chain:
```
30-second batch:
  User A: buy ETH
  User B: sell ETH
  → Match peer-to-peer → zero slippage, zero sandwich risk
  → Only net imbalance hits the AMM on-chain
```
Attacker can't sandwich because they don't know net direction until settled.
Examples: CoW Protocol, 1inch Fusion.

### Mechanism 4: TWAP execution
Split large trade into small pieces over time:
```
Instead of buying 500 ETH at once → buy 50 ETH every 5 minutes
Each small trade: minimal price impact → not worth sandwiching (gas cost > profit)
```

### Mechanism 5: Deadline parameter
Every swap includes a block deadline. Transaction reverts if not included in time:
```
deadline = current_block + 3  → expires if not included within 3 blocks
Limits attacker's window to hold a frontrun position
```

---

## 5. How Data Products Help Prevent Sandwiches

This is the key product angle for the Infra PM role.

### Data product 1: Slippage recommendation engine
Instead of requiring users to set slippage manually, compute optimal tolerance per token + trade:
```
USDC/ETH (deep pool, stable):      recommend 0.1%
New meme coin (shallow, volatile): recommend 3–5%
Pool with active sandwich bots:    warn user + recommend private routing
```
Data needed: historical price volatility per pool, sandwich frequency, pool depth.

### Data product 2: MEV risk scoring per pool
Index historical MEV events — how often each pool is sandwiched and typical extracted value:
```
WETH/USDC ($500M liquidity):  MEV risk LOW  — deep pool, hard to move price
New meme coin ($50K):         MEV risk HIGH — shallow, frequently sandwiched
```
Surface as a risk label before trade submission. Proprietary exchange signal — CoinGecko doesn't have this.

### Data product 3: Real-time price impact calculator
Show user exactly what they'll receive before submitting:
```
You want to buy 50 ETH
Current price:           $2,000
Price impact:            1.2%
Expected execution:      $2,024
At your 0.5% slippage:   $2,010  ← trade will likely fail — recommend 1.5%
```
Requires: live pool reserve data (from GeckoTerminal-equivalent) + AMM formula applied to specific trade size.

### Data product 4: Mempool monitoring
Index pending transactions to detect active sandwich attempts:
```
User submits buy of 100 ETH
System detects: bot frontrun submitted in same block
→ Alert: "MEV bot detected — route through private mempool?"
→ User confirms → transaction rerouted
```
Requires: separate mempool indexing infrastructure (not block data — pending transactions only).

### Data product 5: MEV-aware route optimization
DEX aggregators split trades across pools. Add MEV risk as an optimization dimension:
```
Standard routing:     minimize price impact + fees
MEV-aware routing:    minimize (price impact + fees + expected MEV extraction)

Prefer pools with:
  - Lower sandwich history
  - Deeper liquidity
  - Less bot activity
Automatically split large trades to reduce per-pool impact
```

### Data product 6: Post-trade MEV transparency
After confirmation, analyze whether the user was sandwiched:
```
Compare:
  Fair price (pre-frontrun): $2,000
  Your execution price:      $2,040
  Post-backrun price:        $2,002
  Extracted MEV:             $40/ETH → $2,000 total on a 50 ETH trade
```
Surface to user: "You were sandwiched. Enable MEV protection to prevent this next time."
Both a user protection feature and a trust/retention mechanism.

---

## Key Concepts Cheat Sheet

| Term | One-line definition |
|---|---|
| MEV | Maximal Extractable Value — profit from manipulating transaction ordering |
| Sandwich attack | Frontrun victim's trade + backrun → extract value from their slippage tolerance |
| Frontrunning | Submitting a transaction ahead of another by paying higher gas |
| Backrunning | Submitting a transaction immediately after another (same block, next position) |
| Private mempool | Transaction routing that bypasses public mempool → bots can't see it |
| Batch auction | Orders collected in time window, matched off-chain → no individual trade to sandwich |
| TWAP | Time-Weighted Average Price — split trade over time to minimize per-trade impact |
| Bridge | Protocol for moving assets between chains |
| Lock-and-mint | Lock on source chain → mint wrapped token on destination |
| Burn-and-mint | Destroy on source, mint fresh on destination (requires issuer control) |
| Liquidity bridge | Pool on each chain, relayer transfers between them |
| Canonical bridge | Native L1↔L2 bridge secured by rollup proof system |
| Fraud proof window | 7-day withdrawal delay on optimistic rollups for challenge period |
| Wrapped/synthetic asset | Bridge-created token representing a locked asset — only redeemable via that specific bridge |
| Bridge invariant | Tokens locked on source == wrapped tokens in circulation; broken invariant = insolvency |
| Validator (bridge) | Off-chain human/org holding a private key that attests to cross-chain events |
| Honeypot | Single contract holding all locked assets — high-value hack target in lock-and-mint bridges |
| Messaging protocol | Infrastructure layer that delivers arbitrary data between chains; bridges built on top |
| ZK light client bridge | Bridge using ZK proofs to verify source chain state — no human validator set required |
| Cross-chain registry | Verified mapping of same token address across all chains |
| Bridge flow | Net movement of assets between chains — signals capital rotation |
| Unified portfolio | Aggregated view of all assets across all chains for one wallet |
