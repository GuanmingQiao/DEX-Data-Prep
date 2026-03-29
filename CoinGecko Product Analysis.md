# CoinGecko Product Analysis
*Last updated: 2026-03-26. Source: llms-full.txt (official CoinGecko AI documentation)*

---

## 1. What CoinGecko Is

CoinGecko is the world's largest independent crypto data aggregator. It operates two distinct data products under one API:

| Product | Focus | Data source |
|---|---|---|
| **CoinGecko** | CEX market data — coins, exchanges, NFTs, global | Exchange APIs + web scraping |
| **GeckoTerminal** | DEX / on-chain data — pools, tokens, swaps | Blockchain node indexing |

These were originally separate products and remain distinct in the API (different base paths: `/coins/...` vs `/onchain/...`). Understanding this split is key — they have different data collection methods, update frequencies, and business models.

---

## 2. Full API Surface Map

### 2.1 CoinGecko (CEX-side)

**Price primitives**
- `GET /simple/price` — current price of any coin in any currency; 20s cache
- `GET /simple/token_price/{id}` — price by contract address on a specific chain; 20s cache
- `GET /simple/supported_vs_currencies` — list of supported quote currencies

**Coin data**
- `GET /coins/list` — full list of all coins with IDs (18,000+)
- `GET /coins/markets` — paginated coin list with price, market cap, volume, 24h change; sortable; 30s cache
- `GET /coins/{id}` — full coin detail: market data, tickers, community data, developer data; 60s cache
- `GET /coins/{id}/tickers` — exchange-by-exchange price feeds for one coin
- `GET /coins/{id}/history` — historical snapshot for a specific date
- `GET /coins/{id}/market_chart` — time-series: price, market cap, volume (auto-granularity: 5m/hourly/daily)
- `GET /coins/{id}/market_chart/range` — time-series with custom date range
- `GET /coins/{id}/ohlc` — candlestick OHLCV data
- `GET /coins/{id}/ohlc/range` — OHLCV with date range; historical data for inactive/delisted coins included
- `GET /coins/{id}/circulating_supply_chart` — historical circulating supply over time (Enterprise only)
- `GET /coins/top_gainers_losers` — top movers with >$50K daily volume; 5min cache (Paid only)
- `GET /coins/list/new` — recently added coins

**Contract address lookup**
- `GET /coins/{id}/contract/{contract_address}` — full coin data resolved from a contract address
- `GET /coins/{id}/contract/{contract_address}/market_chart` — time-series by contract address
- `GET /coins/{id}/contract/{contract_address}/market_chart/range` — ranged time-series by contract

**Categories (CEX-side)**
- `GET /coins/categories/list` — flat list of all category IDs + names (200+ categories)
- `GET /coins/categories` — categories with market cap, volume, 24h change, top coins; 5min cache

**Exchanges**
- `GET /exchanges` — list of all tracked exchanges with trust score, volume
- `GET /exchanges/{id}` — exchange detail: volume, tickers, year established, country
- `GET /exchanges/{id}/tickers` — all trading pairs on a given exchange with price and spread
- `GET /exchanges/{id}/volume_chart` — exchange volume over time
- `GET /exchanges/{id}/volume_chart/range` — exchange volume for a date range
- `GET /exchanges/list` — flat ID+name list

**NFTs**
- `GET /nfts/list` — all tracked NFT collections
- `GET /nfts/markets` — NFTs ranked by market cap with floor price, volume
- `GET /nfts/{id}` — collection detail: floor price, holders, market cap, volume
- `GET /nfts/{id}/market_chart` — floor price + volume over time

**Global & macro**
- `GET /global` — total crypto market cap, volume, BTC/ETH dominance, active coins; ~60s cache
- `GET /global/decentralized_finance_defi` — DeFi-specific global metrics (TVL, volume, dominance)

**Discovery**
- `GET /search` — search coins, exchanges, NFTs by name/symbol; ranked by market cap desc; 15min cache
- `GET /search/trending` — top 15 trending coins (by search volume), top 7 NFTs, top 6 categories; 10min cache

**Treasury (institutional)**
- `GET /public_treasury/{entity_id}` — holdings data for a named institution (e.g. Strategy/MicroStrategy)
- `GET /public_treasury/{entity_id}/{coin_id}/holding_chart` — historical holding quantity over time
- `GET /public_treasury/{entity_id}/transaction_history` — buy/sell transaction log
- `GET /{entity}/public_treasury/{coin_id}` — all companies holding a given coin (e.g. all BTC holders)

**Asset platforms**
- `GET /asset_platforms` — list of all blockchains supported as asset platforms for token resolution

---

### 2.2 GeckoTerminal (On-chain/DEX side)

**Networks**
- `GET /onchain/networks` — all supported blockchain networks with IDs (100+ chains)
- `GET /onchain/networks/{network}/dexes` — all DEXes indexed on a specific network

**Token data**
- `GET /onchain/networks/{network}/tokens/{address}` — token price, market cap, FDV, volume, top pool; 10s cache
- `GET /onchain/networks/{network}/tokens/multi/{addresses}` — batch up to 50 tokens; 10s cache
- `GET /onchain/networks/{network}/tokens/{address}/info` — metadata: name, symbol, socials, description, GT Score, holder distribution, bonding curve/launchpad status; 60s cache
- `GET /onchain/networks/{network}/tokens/{address}/ohlcv/{timeframe}` — OHLCV from most liquid pool; timeframes: minute/hour/day; data from Sep 2021; 10s cache
- `GET /onchain/networks/{network}/tokens/{address}/trades` — last 24h trade feed for a token; real-time cacheless
- `GET /onchain/networks/{network}/tokens/{address}/holders_chart` — historical holder count over time (Beta; 7d/30d/max)
- `GET /onchain/networks/{network}/tokens/{address}/top_holders` — top holders ranked by balance, with PnL details (avg buy price, realized/unrealized PnL, buy/sell counts)
- `GET /onchain/networks/{network}/tokens/{address}/top_traders` — top traders ranked by realized PnL
- `GET /onchain/simple/networks/{network}/token_price/{addresses}` — lightweight price lookup by address; real-time cacheless; up to 100 addresses

**Pool data**
- `GET /onchain/networks/{network}/pools/{address}` — pool detail: reserves, volume, tx count, fee tier, price, locked liquidity %, launchpad graduation status; 10s cache
- `GET /onchain/networks/{network}/pools/multi/{addresses}` — batch up to 50 pools; 10s cache
- `GET /onchain/networks/{network}/pools/{pool_address}/ohlcv/{timeframe}` — OHLCV for a specific pool; supports pools with 3+ tokens; 10s cache
- `GET /onchain/networks/{network}/pools/{pool_address}/trades` — real-time trade feed for a pool; cacheless
- `GET /onchain/networks/{network}/pools/{pool_address}/info` — pool token metadata, GT Score, sentiment votes, community sus reports, holder distribution; 60s cache

**Pool discovery**
- `GET /onchain/networks/trending_pools` — trending pools across all networks; 30s cache
- `GET /onchain/networks/{network}/trending_pools` — trending pools on a specific network; 30s cache
- `GET /onchain/networks/{network}/new_pools` — most recently created pools on a network
- `GET /onchain/networks/{network}/pools` — top pools by network (ranked by liquidity × volume)
- `GET /onchain/networks/{network}/dexes/{dex}/pools` — top pools on a specific DEX
- `GET /onchain/pools/megafilter` — advanced pool screener: filter by network, DEX, liquidity range, volume range, price change (5m/1h/6h/24h), tx count, security checks; 30s cache
- `GET /onchain/pools/trending_search` — pools trending in search (Paid only); 60s cache
- `GET /onchain/categories` — GeckoTerminal-native on-chain categories (e.g. pump.fun, Raydium ecosystem)
- `GET /onchain/categories/{category_id}/pools` — pools within a specific on-chain category

**Token lists**
- `GET /token_lists/{asset_platform_id}/all.json` — CoinGecko-verified token list per chain; 5min cache
- `GET /tokens/info_recently_updated` — tokens with recently updated metadata; 30s cache

**Search (DEX)**
- `GET /onchain/search/pools` — search pools by address, token name, symbol, or contract; 30s cache

---

### 2.3 WebSocket Streaming (Pro only)

Three real-time channels, all via `wss://stream.coingecko.com/v1`:

| Channel | Code | Trigger | Payload | Max frequency |
|---|---|---|---|---|
| **OnchainSimpleTokenPrice** | G1 | By network + token address | Price, 24h change, market cap, volume | 1s |
| **OnchainTrade** | G2 | By network + pool address | Trade type (buy/sell), token amount, quote amount, volume USD, price, tx hash | 0.1s |
| **OnchainOHLCV** | G3 | By network + pool address + interval | Open, High, Low, Close, Volume per candle interval | 1s |

OHLCV intervals: `1s / 1m / 5m / 15m / 1h / 2h / 4h / 8h / 12h / 1d`

Limits (self-serve paid plans): 10 concurrent socket connections, 100 subscriptions per channel per socket.

---

## 3. How the Data Is Actually Collected

### 3.1 CEX Market Data

**Source: Exchange APIs and websockets**

CoinGecko connects to each exchange's public REST API and/or WebSocket feed to collect ticker data (last price, bid, ask, 24h volume, spread). The aggregation process:

1. **Ticker collection**: Pull all trading pairs from each exchange (e.g. BTC/USDT on Binance, BTC/USD on Coinbase).
2. **Price normalization**: Convert all prices to a common quote currency (USD) using FX rates.
3. **Volume-weighted aggregation**: The global price is a volume-weighted average across all active exchange tickers. This is why CoinGecko's price may differ slightly from any single exchange.
4. **Staleness filtering**: Tickers that haven't updated recently are excluded from the aggregation to prevent stale data from distorting the price (the deprecated `trust_score` field was part of this filtering logic).
5. **Market cap calculation**: `price × circulating_supply`. FDV = `price × total_supply`.

**Key data quality decision — rehypothecated tokens:** CoinGecko excludes tokens like wETH, stETH from market cap rankings by default (effective Feb 2026), since counting both ETH and wETH would double-count the same underlying asset. The field `market_cap_rank_with_rehypothecated` exposes the inclusive ranking.

### 3.2 On-Chain / DEX Data (GeckoTerminal)

**Source: Blockchain node subscription + event indexing**

This is a significantly more complex pipeline than CEX data collection.

**Step 1 — Node connections**
GeckoTerminal maintains connections to full nodes (or uses node providers like Alchemy/Infura/QuickNode) for each supported blockchain. For 100+ chains, this means a large fleet of node subscriptions.

**Step 2 — Swap event indexing**
Every DEX pool is a smart contract that emits events on each trade. GeckoTerminal subscribes to these events in real time:
- Uniswap V2/V3 pools emit `Swap(sender, amount0In, amount1In, amount0Out, amount1Out, to)`
- From `amount0In/Out` and `amount1In/Out`, the exchange ratio is derived → this IS the price
- The pool address, block number, and transaction hash are indexed for each swap

**Step 3 — Price derivation**
Price of Token A in USD = (amount of Token B received) / (amount of Token A given) × (price of Token B in USD). For most pools, Token B is a stablecoin (USDC, USDT) or a native token with a known USD price (ETH, BNB, SOL).

For tokens with no direct stablecoin pair, CoinGecko routes through intermediate pools: Token A → ETH → USDC, deriving the price from two hops.

**Step 4 — Top pool selection**
A token can exist in many pools. CoinGecko picks the "top pool" using a composite score of `reserve_in_usd` (liquidity) + `volume_usd` (24h trading activity). This top pool's price is the canonical price returned by the token endpoints. This is configurable: users can force a specific pool via `/networks/{network}/pools/{address}`.

**Step 5 — OHLCV aggregation**
Individual swap events are bucketed into time intervals (1s, 1m, 5m, etc.) to produce OHLCV candles:
- **Open**: price at the first swap in the interval
- **High/Low**: highest/lowest swap price within the interval
- **Close**: price at the last swap in the interval
- **Volume**: sum of USD value of all swaps in the interval
- Intervals with zero swaps are skipped by default (or filled with previous close if `include_empty_intervals=true`)

**Step 6 — Token metadata resolution**
- **On-chain metadata**: CoinGecko reads the ERC-20 (or equivalent) contract: `name()`, `symbol()`, `decimals()`, `totalSupply()`
- **Social metadata**: sourced from token info contracts or community submissions; not always vetted (flagged as `not vetted by CoinGecko team` in the docs)
- **Holder distribution**: indexed from `Transfer` events — tracking every token transfer to reconstruct current holder balances. This is computationally expensive and explains why holder data is Beta-only on limited chains.

**Step 7 — Launchpad / Bonding curve tracking**
For pump.fun-style launchpads, tokens start on a bonding curve (a single-sided price curve), not a standard AMM pool. GeckoTerminal indexes the launchpad contract separately to track:
- Bonding curve graduation status
- Migration details (when a token graduates to a DEX pool)
- This data is returned in `launchpad_details` object in pool/token endpoints

**Step 8 — Security checks (GT Score)**
- **Honeypot detection**: uses GoPlus Token Security and De.Fi Scanner APIs to check if a token contract prevents selling (a honeypot scam)
- **Locked liquidity**: checks if LP tokens are locked or burned (reduces rug pull risk)
- **GT Score** (0–100): composite of user engagement, trading activity, liquidity, and security checks
- **Community sus reports**: user-reported suspicious activity count surfaced in pool info

---

## 4. Business Model

### 4.1 Tiered API Access

CoinGecko monetizes through API subscriptions. Two key dimensions: **rate limits** and **data access**.

| Tier | Price | Key unlocks |
|---|---|---|
| **Demo (free)** | Free | `api.coingecko.com`; limited rate limits; 1 yr historical data; no on-chain pro endpoints |
| **Analyst** | ~$129/mo | On-chain data from Sep 2021; WebSocket; top gainers/losers; trending search pools; 2yr history |
| **Lite / Pro** | Higher | Higher rate limits; longer history; more concurrent WebSocket connections |
| **Enterprise** | Custom | Full history; 5-minute/hourly interval params (bypass auto-granularity); circulating supply charts; dedicated SLA |

**Key gating strategy:** Real-time on-chain data (trades feed, token prices cacheless) requires paid plans. Historical depth also gates by plan. The Demo API is generous enough for learning/prototyping but insufficient for production trading applications.

### 4.2 Business Cases by API Segment

**Simple Price API** — *Highest volume, lowest margin*
The most called endpoint in the ecosystem. Used by thousands of DeFi protocols, wallets, portfolio trackers to display prices. Business case: high call volume × credit billing model. Demo users get limited calls/minute; paid plans scale up.

**Coins/Markets API** — *Core product for dashboards and apps*
Powers most CoinGecko-style market overview pages. Business case: any app showing a ranked list of coins (CoinMarketCap competitor) needs this. Used by news sites, analytics platforms, aggregators.

**On-Chain Token/Pool APIs** — *Fastest growing, highest value segment*
The GeckoTerminal data product. Business case: DEX trading platforms, meme coin snipers, on-chain analytics tools, and AI trading bots all need this data. These clients pay Pro/Enterprise rates because the data directly generates alpha. The real-time trades feed (0.1s for OnchainTrade WebSocket) is the highest-value endpoint — a trading firm's latency advantage depends on it.

**Trending & Search APIs** — *Discovery layer for consumer apps*
Used by wallets and consumer crypto apps to power their home screen discovery feed. Business case: any app that wants to surface "what's hot" without building its own trending algorithm. CoinGecko's trending signal (search volume + price momentum) is trusted by millions of retail users.

**Categories API** — *Content layer for editorial and analytics*
Used by analytics platforms and media to segment the market. Business case: institutional research (e.g. "how did the AI token category perform this week?"). Also powers automated newsletter generation and portfolio construction tools.

**Treasury API** — *Institutional and compliance use case*
Tracks public company and government crypto holdings (e.g. Strategy/MicroStrategy's BTC). Business case: financial journalists, compliance teams, ETF analysts monitoring institutional exposure.

**WebSocket Streaming** — *Trading and real-time monitoring*
The most defensible, highest-margin product. Trading firms and market makers need sub-second price feeds. Business case: replacing expensive custom node infrastructure with a managed streaming service. CoinGecko monetizes this per-response (0.1 credit per WebSocket message).

---

## 5. Key Product Design Decisions

### 5.1 Caching as a Pricing Mechanism
CoinGecko's caching tiers are a deliberate pricing tool, not just a technical constraint:
- Free/Demo: longer caches (30s–60s), coarser data
- Paid: shorter caches (10s), some endpoints go fully cacheless (real-time)
- Enterprise: bypass auto-granularity for custom time intervals

This means the same endpoint returns fundamentally different data quality depending on your plan tier — the product differentiates on freshness, not just rate limits.

### 5.2 Two Category Systems
CoinGecko maintains **two separate, incompatible category hierarchies**:
- **CoinGecko categories** (`/coins/categories`): curated by humans, tied to CEX-listed coins (DeFi, Meme Coins, AI Agents, Layer 1, etc.)
- **GeckoTerminal categories** (`/categories`): auto-assigned from pool/launchpad context (pump.fun, Raydium, specific ecosystem tags)

This reflects the fundamental split between the CEX and DEX product lines. They intentionally don't merge them because the audiences and use cases differ. A product building on both needs to handle two taxonomies.

### 5.3 Top Pool as Canonical Price
For DEX tokens, price is always sourced from the "top pool" (highest liquidity × volume). This is a UX-first decision — giving users one canonical price rather than a confusing list. But it creates edge cases:
- A token's price can appear to jump if its "top pool" changes (e.g. new pool gets more volume)
- Tokens with no active pools in the last 7 days return null price by default; `include_inactive_source=true` extends the lookback to 1 year
- Enterprise users can pin a specific pool address for full price control

### 5.4 FDV vs Market Cap as a Trust Signal
The docs note: "If the token's market cap is not verified by the team, the API response will return `null` for its market cap value." This means:
- Verified market cap = CoinGecko team has confirmed the circulating supply figure
- Unverified = the API returns FDV as a fallback (or null), which the frontend displays but marks as approximate
- This is a deliberate data quality gate — the API actively distinguishes verified from unverified data

### 5.5 GT Score as the DEX Trust Score
The GT Score (0–100) is CoinGecko's answer to the now-deprecated `trust_score` for exchange tickers. It's a composite signal for DEX token quality:
- User engagement on GeckoTerminal
- Trading activity (volume + transaction count)
- Liquidity depth
- Security checks (honeypot, locked liquidity)
- GT Verified badge = manually reviewed by CoinGecko team
- Score ≥ 75 = eligible for `good_gt_score` filter in megafilter

---

## 6. Update Frequency Reference Table

| Data type | Endpoint | Cache / update |
|---|---|---|
| Simple price (CEX) | `/simple/price` | 20s |
| Coins markets | `/coins/markets` | 30s |
| Coin detail | `/coins/{id}` | 60s |
| Exchange tickers | `/exchanges/{id}/tickers` | 60s |
| Global market data | `/global` | ~60s |
| Trending search | `/search/trending` | 10min |
| Categories | `/coins/categories` | 5min |
| On-chain token price (REST) | `/onchain/simple/.../token_price/...` | Real-time (cacheless, Pro) |
| On-chain pool detail | `/networks/{network}/pools/{address}` | 10s |
| On-chain token detail | `/networks/{network}/tokens/{address}` | 10s |
| On-chain OHLCV | `/tokens/{address}/ohlcv/{timeframe}` | 10s |
| On-chain trades feed (REST) | `/pools/{address}/trades` | Real-time (cacheless) |
| On-chain trending pools | `/networks/trending_pools` | 30s |
| On-chain new pools | `/networks/{network}/new_pools` | 30s |
| Token/pool info (metadata) | `/.../info` | 60s |
| Holder chart | `/.../holders_chart` | 60s |
| Treasury data | `/public_treasury/...` | 5min |
| WebSocket: token price | OnchainSimpleTokenPrice (G1) | Up to 1s |
| WebSocket: trades | OnchainTrade (G2) | Up to 0.1s |
| WebSocket: OHLCV | OnchainOHLCV (G3) | Up to 1s |

---

## 7. Implications for OKX DEX (Interview Angles)

### What CoinGecko proves is possible
CoinGecko demonstrates that a single unified API can cover 100+ chains, 1,400+ exchanges, and real-time DEX data at scale. This is the blueprint for what OKX's internal data infrastructure needs to replicate — but owned, lower latency, and enriched with OKX's proprietary order flow data.

### Where OKX needs to go further

| CoinGecko approach | OKX requirement |
|---|---|
| Price sourced from top pool (30s cache for free tier) | Sub-100ms price feeds for DEX order routing |
| Trending = search volume + price momentum | Trending = same + OKX user behavior (watchlists, trades, searches) |
| GT Score is a vendor-assigned score | OKX needs its own trust/risk score tuned to its user base and regulatory requirements |
| Two separate category systems (CoinGecko vs GeckoTerminal) | OKX needs one unified taxonomy across CEX + DEX assets |
| Smart money = top PnL traders, public data only | OKX can combine on-chain smart money with CEX order flow for stronger signals |
| Megafilter for pool discovery | OKX DEX needs same with additional filter: "tradeable on OKX DEX" |
| No AI integration in data product itself | OKX JD explicitly requires AI trading products (LLM-driven trading rationale, smart money signal summarization) |

### The central design challenge for OKX Infra PM
CoinGecko serves external developers. OKX's central data service serves internal products (Wallet, DEX, CEX display, risk control, compliance) with different SLAs, latency requirements, and data needs. The infra PM must design a system that:
1. Indexes the same raw on-chain data as GeckoTerminal
2. Serves it to multiple internal consumers with different latency tiers (real-time for trading, 30s for display, daily for compliance)
3. Adds OKX-specific enrichment: user behavior signals, proprietary smart money labels, exchange-specific category tags
4. Supports the AI/LLM layer that interprets signals into trading rationale

---

*This document should be updated as new CoinGecko product updates are released or as new OKX-specific context is discovered.*
