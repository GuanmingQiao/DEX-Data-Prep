# DEX Data Infrastructure — Interview Prep

Study notes on DEX and on-chain data infrastructure, prepared for an Infra Product Manager interview at OKX.

## What's in here

| File | Contents |
|---|---|
| `CoinGecko Product Analysis.md` | Structured product breakdown of CoinGecko's API surface, data collection methods, business model, and implications for OKX |
| `Session 1 - AMM & On-Chain Data Fundamentals.md` | AMM mechanics, constant product formula, slippage, trading fees, LP token economics, impermanent loss, on-chain data trust |
| `Session 2 - Cross-Chain & MEV Protection.md` | Bridge mechanisms, cross-chain data products, sandwich attacks, MEV prevention, and how data infrastructure defends against MEV |

## How this was built

These notes were generated through an interactive study session using:

- **CoinGecko's official LLM documentation** (`llms-full.txt`) — the full API reference and changelog used as the primary knowledge source on how production crypto data infrastructure works
- **CoinGecko MCP server** — connected directly to live CoinGecko data to ground concepts in real market data
- **OKX job description** — used to map every concept back to the specific responsibilities of the Infra PM role

The goal was to use CoinGecko — the industry reference for crypto market data — as a concrete lens for understanding the data systems OKX DEX builds and operates internally.
