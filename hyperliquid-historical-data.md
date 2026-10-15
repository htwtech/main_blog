---
title: 'Hyperliquid Historical Data: API, S3, and Backtesting'
slug: 'hyperliquid-historical-data'
canonical_url: 'https://supanode.xyz/blog/hyperliquid-historical-data'
date: "2026-10-04"
description: 'A practical guide to Hyperliquid historical data across the Info API, public S3 archives, live capture, indexed datasets, and backtesting workflows.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Hyperliquid%20Historical%20Data_%20API,%20S3,%20and%20Backtesting.png
coverAlt: 'Hyperliquid Historical Data: API, S3, and Backtesting'
tags: ["Hyperliquid", "API", "S3", "Backtesting"]
---

Searches for Hyperliquid historical data often return HYPE price-history pages from CoinMarketCap or Investing.com. Here, the term means exchange-level records: fills, funding payments, candles, order-book snapshots, and account events.

Hyperliquid exposes that history through several separate interfaces: the [Info API](/blog/hyperliquid-api), two public S3 buckets, and data captured directly from WebSocket or node output. The right historical-data source depends on retention, completeness, granularity, and queryability.

## Where Hyperliquid Historical Data Comes From

![Hyperliquid historical data sources compared by freshness, query readiness, limitations, and use case.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Hyperliquid%20Historical%20Data%20Sources.png)

<sub>Historical data sources by freshness and query readiness.</sub>


| Need | Best source | Main limitation |
| :---- | :---- | :---- |
| Recent candles | Info API (`candleSnapshot`) | Only the latest 5,000 candles |
| Recent user fills | Info API (`userFillsByTime`) | Only \~10,000 most recent fills reachable |
| Historical funding rates | Info API (`fundingHistory`) | Paginated, no stated retention horizon |
| Historical L2 snapshots | `hyperliquid-archive` | Roughly monthly uploads, gaps possible |
| Bulk fills | `hl-mainnet-node-data` | Raw files, ingestion is on you |
| Future/ongoing capture | WebSocket or node | Designed for ongoing capture; backfill beyond available snapshots/API retention still needs another source, and persistence/gap recovery remain your responsibility  |
| Repeated analytical queries | Indexed database | Coverage depends on the indexed dataset |

## What the Hyperliquid Info API Can Retrieve Historically

The [Info API](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) is designed for bounded historical retrieval. Every relevant call is a `POST` to `https://api.hyperliquid.xyz/info` — no key required for these read-only queries, just a JSON body and the account address where relevant.

| Endpoint | Returns | Historical cap |
| :---- | :---- | :---- |
| `userFills` | Fills for one address | 2,000 most recent |
| `userFillsByTime` | Address fills in a time range | 2,000/response, \~10,000 reachable |
| `historicalOrders` | Orders for one address | 2,000 most recent |
| `candleSnapshot` | OHLCV candles | 5,000 most recent |
| `fundingHistory` | Market funding rate series | Paginated, no absolute cap stated |

Time-range endpoints paginate after 500 elements or distinct blocks. Larger windows therefore require pagination logic that advances `startTime` from the last timestamp received.

For small accounts or a short lookback window, the Info API is usually the simplest native path — a handful of paginated calls covers the question without any S3 setup. That calculus changes once an account has more fills than the \~10,000-record ceiling exposes. At that point, repeatedly paginating the same account's fill history through a rate-limited endpoint becomes slower and more fragile than pulling the equivalent fill history from S3 once. Candle and funding history don't have that same S3 fallback — neither is published as a bulk dataset in `hyperliquid-archive`, so older candles need a separate capture source or reconstruction from execution data, and funding history stays inside the API's own pagination limits. 

The 5,000-candle limit translates into very different time horizons depending on interval. One-minute bars hit that ceiling after a little over three and a half days. Step up to 15-minute candles and the same cap stretches to roughly 52 days; at the 1-hour interval, it reaches around 208 days.

```
curl -sS -X POST 'https://api.hyperliquid.xyz/info' \
 -H 'Content-Type: application/json' \
 -d '{
   "type": "candleSnapshot",
   "req": {
     "coin": "BTC",
     "interval": "15m",
     "startTime": 1790812800000,
     "endTime": 1790816400000
   }
 }'
```

REST requests from one IP share a [1,200-weight-per-minute budget](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits), and endpoints returning more rows — `historicalOrders`, `userFillsByTime`, `fundingHistory` — cost more weight as the result set grows. Hyperliquid explicitly recommends S3 for large Explorer API batch requests, and its historical-data docs expose separate bulk S3 sources for fills and L2 snapshots. 

## What Is Available in Hyperliquid's Public S3 Archives

Hyperliquid publishes two separate S3 buckets, each covering a different slice of history.

`hyperliquid-archive` holds historical L2 book snapshots and asset contexts:

```
s3://hyperliquid-archive/market_data/[date]/[hour]/[datatype]/[coin].lz4
s3://hyperliquid-archive/asset_ctxs/[date].csv.lz4
```

The asset-context files carry per-asset market state at each snapshot — fields in the current API schema include oracle and mark prices, open interest, funding, and premium figures — useful for reconstructing the conditions a trade or fill happened under, not just the trade itself. Hyperliquid's archive page doesn't publish a separate versioned schema for the archived CSVs, so column names should be validated during ingestion rather than assumed from the live API. 

[Hyperliquid's own documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/historical-data) says uploads happen roughly once a month, offers no guarantee of timely updates, and warns that data may be missing — candles and spot asset data aren't part of this bucket at all.

This fits research workflows that can tolerate delayed backfills and verify coverage independently. Workloads that require complete next-day book history need a separate capture path.

```
aws s3 cp \
 s3://hyperliquid-archive/market_data/20230916/9/l2Book/SOL.lz4 \
 /tmp/SOL.lz4 \
 --request-payer requester

unlz4 --rm /tmp/SOL.lz4
```

The bucket uses AWS Requester Pays, so the downloading account covers request and transfer costs even though access itself has no subscription fee.

The second bucket covers a different layer of history. `hl-mainnet-node-data` captures the executed activity underlying those snapshots: `node_fills_by_block` for current-format bulk fills, with older records sitting under `node_fills` and `node_trades` in an earlier schema. A backfill spanning both eras therefore needs schema-aware parsing for each format.

Future events need to be captured as they happen through the [Hyperliquid WebSocket API](/blog/hyperliquid-websocket-api), with reconnect and gap-recovery logic around the live stream.

## Historical Trades, Funding, Candles, and Order-Book Data

| Data type | Best source | Scope | Key caveat |
| :---- | :---- | :---- | :---- |
| Trades | Node/bulk trade feed | Market-wide | Older format differs from the current fill schema |
| Fills | `hl-mainnet-node-data`, or Info API for recent | Account or bulk | API caps make it unsuitable for a wallet's full lifetime |
| Funding rate | `fundingHistory` | Market-wide | No documented retention ceiling, but paginated |
| User funding paid | `userFunding` | Account | Needs address plus time range |
| Candles | `candleSnapshot` | Market-wide | Archive provides no candles at all |
| L2 snapshots | `hyperliquid-archive` | Market-wide | Monthly-ish cadence, gaps possible |

A trade is a match between two sides; a fill is one account's side of that match, carrying context such as starting position, realized PnL and fees. For wallet-level analysis, fills provide the account-specific context needed for position, fee, and realized-PnL queries.

Funding operates on two separate series. `fundingHistory` is the market-wide rate for a coin, settled every hour regardless of who's holding a position — the series a funding-aware strategy backtest typically simulates against. `userFunding` is what a specific address actually paid or received, useful when analyzing or reproducing that account's realized funding cash flows. 

Candles through `candleSnapshot` are the simplest path to OHLCV, but the 5,000-row cap defines how far back any given interval actually reaches. Candles outside that window aren't retrievable through the API; filling that gap means sourcing them from a separate capture process or deriving bars from execution data, deduplicating across each trade's fill records to avoid double-counting volume.

Hyperliquid's official public historical L2 snapshots are available through `hyperliquid-archive`, with the same monthly-ish cadence and documented gap risk. The live `l2Book` query returns current depth, capped at 20 levels per side, while historical book reconstruction requires archived state. 

Individual order lifetimes, queue position, and full L4 reconstruction depend on [Hyperliquid L2 and L4 order-book data](/blog/hyperliquid-on-chain-order-book), a separate data path from these aggregated L2 snapshots.

## Choosing Hyperliquid Data for Backtesting

The required dataset follows from what the backtest is trying to simulate:

* A candle-level signal study often runs on OHLCV alone, with the 5,000-row cap defining the available historical window for each interval.  
* Execution and wallet research usually needs fill-level data, since fills preserve order identity, fees, and realized PnL.  
* A funding-sensitive perp strategy needs `fundingHistory` for the market series and `userFunding` for what a specific account actually paid or received. Funding settles hourly and needs to be included in holding-period return calculations.  
* Liquidity and slippage research draws on archived L2 state from `hyperliquid-archive`, since the live `l2Book` endpoint only returns current depth.  
* Queue-sensitive research needs order-level history, which aggregated L2 snapshots don't preserve.

Hyperliquid documents that `hyperliquid-archive` may be missing data, so backtests using it should verify coverage for the required date range.

Backtest fidelity depends on matching data granularity to the behavior being simulated. Candle data can support bar-level signals, while execution, funding, liquidity, and queue-sensitive models require progressively more detailed historical records.

## Raw Archives vs a Historical Data Indexer

Using the raw archives requires an ingestion layer for decompression, schema handling, validation, deduplication, storage, and querying.  An indexer moves that ingestion and normalization work into a persistent query layer.

Fill-level backtesting, wallet monitoring, and dashboard queries built on that same schema can all reuse one normalized dataset. An indexed database keeps the ingestion and query layer persistent instead of rebuilding it around each new analysis. 

Supanode, the indexer we run against this dataset, maintains perpetual-fill history covering all perpetual markets from July 27, 2025 onward. It includes liquidation, TWAP, builder, and position context and is queryable through ClickHouse SQL with real-time refresh.

Coverage starts on July 27, 2025, and [funding currently sits outside the indexed dataset](https://supanode.xyz/docs/hyperliquid/indexer) — it's still available through Hyperliquid's native `fundingHistory` and `userFunding` endpoints. Order-level book archive access is provisioned separately and isn't part of the standard fill dataset.

This setup fits workloads built around repeated SQL queries across full-market history. Native Hyperliquid endpoints remain useful for funding, bounded recent lookups, and other data outside the indexed schema.

## Which Hyperliquid Historical Data Source Should You Use?

| Scenario | Use | Why |
| :---- | :---- | :---- |
| Small, recent, one-off query | Info API | Simplest native path, no infrastructure to maintain  |
| Bulk historical L2 or fills | Public S3 (`hyperliquid-archive` / `hl-mainnet-node-data`) | First-party bulk source; requires your own ingestion and gap checking |
| Ongoing live capture | WebSocket or a node | Captures events as they happen; backfill beyond available snapshots/API retention still needs another source, with persistence and gap recovery left to you |
| Repeated full-market analytics | Indexed database | Query layer already built, schema and coverage documented |

For repeated SQL queries across full-market perpetual fills, the [Hyperliquid historical data indexer](https://supanode.xyz/services/hyperliquid/indexer) provides a pre-indexed ClickHouse dataset. The [Hyperliquid indexer documentation](https://supanode.xyz/docs/hyperliquid/indexer) has the current schema and coverage details.
