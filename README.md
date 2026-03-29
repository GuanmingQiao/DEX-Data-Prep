# DEX Data Infrastructure — Interview Prep

Study notes on DEX and on-chain data infrastructure, prepared for an Infra Product Manager interview at a leading crypto exchange.

## What's in here

| File | Contents |
|---|---|
| `CoinGecko Product Analysis.md` | Structured product breakdown of CoinGecko's API surface, data collection methods, business model, and implications for a DEX platform |
| `Session 1 - AMM & On-Chain Data Fundamentals.md` | AMM mechanics, constant product formula, slippage, trading fees, LP token economics, impermanent loss, on-chain data trust |
| `Session 2 - Cross-Chain & MEV Protection.md` | Bridge mechanisms, cross-chain data products, sandwich attacks, MEV prevention, and how data infrastructure defends against MEV |
| `Session 3 - DEX Architecture & Data Infrastructure.md` | DEX types (AMM, order book, RFQ/intent), smart contract architecture, why DEXes need data infra, and the eight data infrastructure layers |

## How this was built

These notes were generated through an interactive study session using:

- **CoinGecko's official LLM documentation** (`llms-full.txt`) — the full API reference and changelog used as the primary knowledge source on how production crypto data infrastructure works
- **CoinGecko MCP server** — connected directly to live CoinGecko data to ground concepts in real market data
- **An Infra PM job description** — used to map every concept back to the specific responsibilities of the role

The goal was to use CoinGecko — the industry reference for crypto market data — as a concrete lens for understanding the data systems a DEX platform builds and operates internally.
