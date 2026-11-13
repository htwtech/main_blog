---
title: "Hyperliquid API: REST, WebSocket, SDK & Rate Limits"
slug: "hyperliquid-api"
canonical_url: "https://supanode.xyz/blog/hyperliquid-api"
date: "2026-10-11"
description: "A practical guide to Hyperliquid's Info API, Exchange API, WebSocket, Python SDK, authentication, rate limits, and production integration."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/htw-cover-hyperliquid-api-developers-guide.png"
coverAlt: "Hyperliquid API: REST, WebSocket, SDK & Rate Limits"
tags: ["Hyperliquid","API","WebSocket","Python SDK"]
---

Hyperliquid exposes several developer interfaces under one name. Reads and signed trading actions share the same HTTP transport but use different request and state semantics; live subscriptions run over WebSocket; HyperEVM has its own Ethereum-compatible JSON-RPC.

That separation shapes the early decisions — which endpoint answers a given question, whether a call needs a signature, and how wide the gap is between an accepted action and a confirmed one. Rate limits also bind at different layers, so capacity depends on both the interface and the workload.

The limits, asset-ID encodings and SDK methods below reflect official documentation as of early October 2026\. The trading example targets testnet.

## What Is the Hyperliquid API?

Hyperliquid exposes developer access to HyperCore, the L1 holding the native perpetual and spot order books, and to HyperEVM, the EVM execution environment. The two share consensus but expose different developer interfaces and different state.

| Interface | What it does | Protocol | Signing | Typical use |
| :---- | :---- | :---- | :---- | :---- |
| Info API | Reads market, metadata, order and account state | HTTP `POST /info` | None | Snapshots, discovery, reconciliation |
| Exchange API | Orders, cancels, modifies, leverage, other L1 actions | HTTP `POST /exchange` | Yes, wallet signature | Trading and account changes |
| WebSocket API | Streams state, and also transports Info/action requests | WebSocket | Only for signed actions | Live market and order/fill state |
| HyperEVM JSON-RPC | EVM state, contracts, logs | Ethereum JSON-RPC | Normal EVM tx signing | Smart contracts, HyperEVM apps |

Verified endpoints:

* Mainnet HTTP `https://api.hyperliquid.xyz`, testnet `https://api.hyperliquid-testnet.xyz`  
* WebSocket `wss://api.hyperliquid.xyz/ws` and `wss://api.hyperliquid-testnet.xyz/ws`  
* HyperEVM RPC `https://rpc.hyperliquid.xyz/evm` on chain ID `999`, testnet `https://rpc.hyperliquid-testnet.xyz/evm` on chain ID `998`

The two core HyperCore endpoints use action-style request bodies. Everything readable arrives through `POST /info` with a `type` discriminator, everything writable through `POST /exchange` with an `action` object. In typed codebases both request families map cleanly onto discriminated unions, which survives schema growth better than URL-based routing. The signing column in that table is the real divide: unsigned reads on one side, wallet-signed actions on the other, with a separate question of whether the action's result is also the order's final state.

## Hyperliquid Info API: Reading Market and Account Data

The Info API covers snapshots: market data, metadata, order state, account state, plus a bounded window of recent history.

### Main Info API Queries

| Query | Minimal body | Returns | Needs address? |
| :---- | :---- | :---- | :---- |
| `allMids` | `{"type":"allMids"}` | symbol → mid price map | No |
| `meta` | `{"type":"meta"}` | perp universe, `szDecimals`, margin data | No |
| `spotMeta` | `{"type":"spotMeta"}` | spot tokens and pairs | No |
| `l2Book` | `{"type":"l2Book","coin":"BTC"}` | up to 20 levels per side | No |
| `clearinghouseState` | `{"type":"clearinghouseState","user":"0x..."}` | perp positions, margin | Yes |
| `openOrders` | `{"type":"openOrders","user":"0x..."}` | current open orders | Yes |
| `orderStatus` | `{"type":"orderStatus","user":"0x...","oid":123}` | order lifecycle status | Yes |

The API-wallet signer address handles authorization; account-state queries still use the master or subaccount address.

Prices and sizes arrive as JSON strings. Preserving the exact representation matters once those same values enter a signed payload, since trailing zeroes and number formatting are both named in the signing docs as failure sources.

### Example: Querying Hyperliquid Market Data

```
curl --request POST \
 --url https://api.hyperliquid.xyz/info \
 --header 'Content-Type: application/json' \
 --data '{"type":"allMids"}'
```

The response is an object keyed by market, with price strings as values. When a book is empty, documentation specifies the last trade price as the fallback mid — relevant for any liveness check built on top of mids.

`l2Book` returns a point-in-time snapshot of up to 20 levels per side. A maintained local book needs a streaming update path, since repeated snapshot calls produce independent point-in-time states with no incremental updates between them.

Recent history is available but bounded. `userFillsByTime` returns at most 2,000 fills per response across the 10,000 most recent, and `candleSnapshot` exposes the latest 5,000 candles. Workloads that repeatedly scan months of activity need [persistent historical storage](/blog/hyperliquid-historical-data).

## Hyperliquid Exchange API: Trading and Order Management

[`POST /exchange`](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/exchange-endpoint) accepts signed actions: placing orders, cancels, modifies, leverage updates, and other authorized account operations. A request carries an `action`, a `nonce`, a `signature`, and optionally `vaultAddress` and `expiresAfter`.

### Supported Trading Actions

The wire format is compact to the point of being unfriendly — `a` asset, `b` buy/sell, `p` price, `s` size, `r` reduce-only, `t` order type:

```json
{
  "action": {
    "type": "order",
    "orders": [
      {"a": 0, "b": true, "p": "100000", "s": "0.001", "r": false,
       "t": {"limit": {"tif": "Gtc"}}}
    ],
    "grouping": "na"
  },
  "nonce": 0,
  "signature": {"r": "...", "s": "...", "v": "..."},
  "vaultAddress": null
}
```

That object illustrates the schema only. Asset, nonce and signature are account- and environment-specific, so it will not execute as pasted.

Limit orders come in three time-in-force variants. `Gtc` rests until canceled, `Alo` is post-only and gets canceled instead of crossing, `Ioc` matches what it can immediately and cancels the remainder. That last behavior decides which statuses an order can return: `resting` is reachable for `Gtc` and `Alo`, while an `Ioc` order resolves within the same action. Trigger orders carry TP/SL semantics in their own structure. 

The Python SDK's `market_open()` helper computes an aggressive price and submits an IOC limit order through the same order action. There is no separate protocol-level market type, and the helper's default slippage constant is 5%, so applications with their own risk limits should set that bound explicitly.

Precision belongs to the protocol too. Prices allow up to five significant figures, and separately no more than `MAX_DECIMALS - szDecimals` decimal places, where `MAX_DECIMALS` is 6 for perps and 8 for spot. Both rules apply at once, integer prices are always valid, and sizes follow `szDecimals` directly. Values formatted for display can produce `Tick` rejections when reused in a signed action.

### Request Structure and Signing

Hyperliquid recommends using an existing SDK for signing. A custom signer has to reproduce MessagePack field ordering, number formatting, address normalization and signature construction exactly, and a difference in any one of them recovers a different signing address — an error that typically surfaces as a nonexistent wallet. Lowercase addresses before signing and sending.

A testnet order, following the shape of the official `basic_order.py` example — the agent key signs, the real account address is passed separately, and a client order ID goes in with the request:

```py
import os
import uuid

import eth_account
from hyperliquid.exchange import Exchange
from hyperliquid.info import Info
from hyperliquid.utils import constants
from hyperliquid.utils.types import Cloid

TESTNET = constants.TESTNET_API_URL

agent = eth_account.Account.from_key(os.environ["HL_TESTNET_AGENT_PRIVATE_KEY"])
account_address = os.environ["HL_TESTNET_ACCOUNT_ADDRESS"].lower()

exchange = Exchange(agent, TESTNET, account_address=account_address)
info = Info(TESTNET, skip_ws=True)

cloid = Cloid.from_int(uuid.uuid4().int >> 128)

result = exchange.order(
    "ETH", True, 0.2, 1100, {"limit": {"tif": "Gtc"}}, cloid=cloid
)
print(result)

if result.get("status") == "ok":
    status = result["response"]["data"]["statuses"][0]
    if "resting" in status:
        oid = status["resting"]["oid"]
        print(info.query_order_by_oid(account_address, oid))
        print(exchange.cancel("ETH", oid))
```

The low bid is deliberate, and the `Gtc` time-in-force is what makes a resting status reachable at all. Whether this particular order rests depends on tick rules, oracle price bands and testnet book state, and outright rejection is also a possible outcome. Read the returned status instead of assuming one. 

A successful Exchange response means the action was processed and included in a committed block. The order lifecycle continues from there: a `resting` status with an `oid` is still an open order requiring subsequent status tracking.

Errors arrive in two layers. Transport first — the Python SDK raises `ClientError` on 4xx and `ServerError` on 5xx. Then trading-level outcomes inside perfectly valid HTTP responses:

1. Deterministic rejects. `Tick`, `MinTradeNtl`, insufficient margin, reduce-only violations, `BadAloPx`, insufficient spot balance. Resubmitting them unchanged consumes additional address-action allowance without changing the outcome.  
2. Lifecycle outcomes. An IOC canceled with no fill, trigger errors, `MissingOrder` on a cancel. These are trading-level results returned through a successful transport response.  
3. Batch-level validation. Per-request results usually come back as a vector, though some deterministic pre-validation failures return a single error for the entire batch, so parsers need to handle both shapes.

An optional 128-bit client order ID (`cloid`) gives application-level reconciliation a stable key independent of the exchange-assigned `oid`.

## API Wallets, Authentication and Python SDK

### API Wallets vs Main Wallets

What Hyperliquid calls an [API wallet](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/nonces-and-api-wallets), or agent wallet, is a signing wallet authorized by a master account. It is not a server-issued credential, which is why the common `hyperliquid api key` phrasing points at something that does not exist in this model. Approval is a signed action from the master account. Removal has more than one shape: an agent can be deregistered explicitly, displaced when a new agent is approved under the same name, or simply reach the end of its validity window with no further action from anyone. 

Its role is signing. Passing the agent address into account-state queries returns an empty result, because those queries expect the master or subaccount address. Model `signer_address` and `account_address` as separate fields even while they hold the same value.

Getting from zero to a signed request takes four steps:

1. Generate a keypair locally for the agent.  
2. Approve it from the master account with an `approveAgent` action, optionally naming it. Named and unnamed agents have different validity windows.  
3. Configure both addresses in the client: the agent private key as signer, the master or subaccount address as the queried account (`Exchange(agent, url, account_address=...)`).  
4. Submit a signed action, then read its action-level status after the transport response.

Nonces are tracked per signer, agents included. The chain retains the 100 highest nonces per signer, a new nonce must be unused and larger than the smallest in that set, and it must fall inside `(T − 2 days, T + 1 day)` relative to block time. Two consequences follow:

* Separate trading processes should use separate API wallets, since parallel workers on one signer can collide on nonce allocation.  
* Rotation means generating a fresh address. Deregistered agents can have their nonce state pruned, which makes previously signed actions replayable, and documentation advises against reusing old agent addresses.

Agent keys belong in a secret manager, outside source control and application logs. The master key can stay offline entirely when an agent covers the required actions.

### Using the Official Python SDK

[`hyperliquid-python-sdk`](https://github.com/hyperliquid-dex/hyperliquid-python-sdk) is the official library, with 0.24.0 (June 4, 2026\) as the latest release at the time of writing. The Rust and TypeScript packages linked from the docs are community-maintained, and a CCXT integration exists for systems where cross-venue portability outweighs Hyperliquid-specific surface.

```
pip install hyperliquid-python-sdk
```

```py
from hyperliquid.info import Info
from hyperliquid.utils import constants

info = Info(constants.MAINNET_API_URL, skip_ws=True)
print(info.all_mids().get("BTC"))
```

`skip_ws=True` matters for short-lived REST clients: without it, `Info` initializes a `WebsocketManager` during construction and consumes one of the ten WebSocket connections available per IP.

Pin the SDK version and capability-test the methods production depends on, since convenience wrappers occasionally lag the raw API surface. The SDK is the canonical signing reference, while the raw API documentation defines the full parameter surface. Documentation also describes the current surface as v0, with a breaking v1 normalization mentioned but undated, so keeping abbreviated wire fields behind a thin adapter reduces the migration work if that normalization lands.

## REST vs WebSocket: Choosing the Right API

| Workload | Interface | Why |
| :---- | :---- | :---- |
| Market and asset discovery | Info API | snapshot-shaped by nature |
| Occasional account summary | Info API | no stream to maintain |
| One-off order status check | Info API | single reconciliation call |
| Continuous market updates | WebSocket | officially recommended for low-latency data |
| Continuous fills and order updates | WebSocket | avoids repeated polling |
| Place / cancel / modify | Exchange over HTTP or WS POST | identical signed-action semantics |
| Reconnect recovery | Info API \+ WS snapshot | re-establishes known state |
| Long-range analytics | persistent historical storage | beyond the Info API's bounded retention |
| EVM contract state | HyperEVM JSON-RPC | different execution plane |

WebSocket also supports request/response calls in addition to subscriptions:

```json
{"method": "post", "id": 123,
"request": {"type": "info", "payload": {"type": "l2Book", "coin": "ETH"}}}
```

Responses return on the `post` channel with the same `id`, and the inner request may be `info` or a signed `action` (`explorer` requests are not supported over this path). The pattern suits applications that already maintain one long-lived transport, while HTTP can simplify retries, isolation and observability for request/response traffic.

Subscription catalogs, heartbeats, quotas and reconnect handling are covered in the [Hyperliquid WebSocket API](/blog/hyperliquid-websocket-api) guide; the matching-engine side of the same picture sits in [Hyperliquid CLOB](/blog/hyperliquid-on-chain-order-book).

## Hyperliquid API Market Types and Asset Identifiers

Asset IDs should be resolved from metadata, because the encoding differs across market classes:

* Standard perps — the asset ID is the coin's index in `meta.universe` (BTC is `0` on mainnet).  
* Spot — `asset = 10000 + spotInfo["index"]`. The spot pair index differs from the token index, and mainnet and testnet IDs diverge; HYPE is the example used in current documentation.  
* HIP-3 builder perps — `asset = 100000 + perp_dex_index * 10000 + index_in_meta`, with names in `{dex}:{coin}` form. Core trading actions are unchanged once the ID resolves correctly.  
* HIP-4 outcomes — encoded above the 100,000,000 offset. The Spot Info reference still carries a testnet-only note on `outcomeMeta` as of October 2026, while outcome markets have been announced for mainnet and mainnet trading has already been observed. The annotation describes the state of the documentation, and network availability should be established by querying each network directly.

In practice, asset resolution can be built directly from metadata:

```py
meta = info.meta()
perp_ids = {asset["name"]: i for i, asset in enumerate(meta["universe"])}

asset_id = perp_ids["BTC"]          # index in meta.universe
sz_decimals = meta["universe"][asset_id]["szDecimals"]
```

`asset_id` is what goes into the `a` field of a raw `/exchange` order, and `sz_decimals` validates the size — and caps the price decimals — before signing. The SDK's `order()` resolves standard coin names internally. Raw signed requests, builder-deployed markets and network changes still require explicit asset-ID handling.

Several perpetual Info queries accept a `dex` parameter for builder-deployed markets, including `meta`, `metaAndAssetCtxs` and `clearinghouseState`. Omitting it returns the default universe silently.

Market naming also varies by layer. PURR appears as `PURR/USDC`, other spot markets surface as `@{index}` at the HyperCore level, and a UI ticker such as BTC/USDC can map to a different L1 token name. Mainnet IDs, testnet IDs and UI tickers should stay separate identifiers in the application model — cached for efficiency, revalidated per environment.

## Hyperliquid API Rate Limits and Production Constraints

HyperCore is documented at 200,000 orders per second. A public-API client lives inside much smaller budgets, and several of them apply in parallel.

| Dimension | Current official rule |
| :---- | :---- |
| REST aggregate | 1,200 weight/min per IP |
| `allMids`, `l2Book`, `clearinghouseState`, `orderStatus`, `spotClearinghouseState`, `exchangeStatus` | weight 2 |
| `userRole` | weight 60 |
| Other documented Info calls | weight 20 |
| Exchange action | `1 + floor(batch_length / 40)` |
| Address action allowance | 1 request per 1 USDC traded since inception, plus a 10,000-request initial buffer |
| Rate-limited address | 1 request / 10 sec, cancels get `min(limit + 100000, limit * 2)` |
| Open orders | 1,000 default, \+1 per 5M USDC volume, capped at 5,000 |
| WebSocket | 10 connections, 1,000 subscriptions, 2,000 outbound messages/min per IP — [detailed quotas here](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits) |
| HyperEVM default RPC | 100 requests/min |

The 1,200-weight budget can disappear quickly depending on the method mix:

```
allMids at 10 Hz:  10 × 60 × weight 2  = 1,200 weight/min
meta at 1 Hz:       1 × 60 × weight 20 = 1,200 weight/min
```

Either pattern consumes the full per-IP minute from one process on one endpoint. Metadata fits a cached refresh path; continuous prices fit a WebSocket subscription.

Batching reduces IP weight while address allowance stays proportional to order count. A batch of `n` orders counts as one request for IP accounting — weight 2 at length 79 — and as `n` requests against the address budget.

Two additional controls affect production senders. `expiresAfter` rejects an action past a millisecond deadline, and expired actions consume 5× the normal address-limit budget, so tight deadlines demand synchronized clocks. `scheduleCancel` acts as a dead man's switch, canceling all open orders at a scheduled time at least five seconds out.

Retry behavior splits along the same line as the error layers. Reads retry freely within budget; writes retry only once the first attempt is known not to have landed, or after reconciling through `orderStatus` or a `cloid`. Transport and transient failures suit exponential backoff with jitter. Deterministic rejects such as `Tick` need a corrected payload.

## Example: A Hyperliquid API Integration Workflow

One concrete pass, buying 0.1 ETH perp on testnet, with the branch points marked.

1\. Discover the market. `info.meta()` → build the name-to-index map shown earlier → `asset_id = perp_ids["ETH"]`, with `szDecimals` alongside it. This also validates the market name before signing.

2\. Read the book. `info.l2_snapshot("ETH")` returns up to 20 levels per side, which is enough to see available depth near the touch. Two paths diverge here. For aggressive entry, set a limit price at or above the best ask and send `Ioc` — it matches against resting liquidity up to that bound, and anything left over is canceled rather than queued, so a thin level can leave the order partially filled. For a maker order, price at or below the best bid and send `Alo`, since a post-only order that would cross gets canceled. Either way, round the price to five significant figures and within `MAX_DECIMALS - szDecimals` decimal places, and the size to `szDecimals`. 

3\. Submit the signed order. `exchange.order("ETH", True, 0.1, price, {"limit": {"tif": tif}}, cloid=cloid)`, with `tif` set to `"Ioc"` or `"Alo"` according to step 2, and the `cloid` generated before the request so it exists independently of the response. The HTTP response confirms that the action reached a committed block; the action-level status carries the order outcome, and the chosen TIF determines which outcomes are possible. 

4\. Branch on the action-level status. Which rows are reachable depends on the TIF chosen in step 2:

| Status | Reachable with | Meaning | Next step |
| :---- | :---- | :---- | :---- |
| `filled` with `avgPx`, `totalSz`, `oid` | `Ioc`, `Gtc`, `Alo` | Matched against resting liquidity; `totalSz` can be smaller than the requested size | Record `totalSz` and `avgPx` as the actual position change, then decide what to do with the shortfall |
| `resting` with `oid` | `Gtc`, `Alo` | Order sits on the book | Track lifecycle until fill or cancel |
| `error` string, e.g. `Tick`, `MinTradeNtl`, `BadAloPx`, or an immediate-match failure on `Ioc` | any TIF | Nothing was placed | Correct price, size or TIF before resubmitting |

The partial-fill case is the one that quietly breaks position accounting. An `Ioc` order for 0.1 ETH that finds 0.06 at or better than the limit price returns `filled` with `totalSz` of 0.06, and the remaining 0.04 never existed as an open order. Requested size and executed size are separate numbers, and only `totalSz` belongs in position state.

5\. Reconcile. `info.query_order_by_oid(account_address, oid)` with the master address reports whether the order remains open. After a timeout or a lost response, where no `oid` ever reached your records, `info.query_order_by_cloid(account_address, cloid)` resolves the same question from the key you generated in step 3\. For the `Ioc` path there is no lifecycle left to poll, so recovery means checking recent fills for that `cloid` to establish whether anything executed. 

6\. Move steady state onto WebSocket. Once more than one order is open at a time, polling `orderStatus` per order starts consuming the weight budget. Subscribe to user fills and order updates, keep Info as the snapshot and recovery path.

Steps 1–2 return snapshots. Step 3 changes state. Steps 4–5 confirm it independently, and keeping confirmation separate preserves the distinction between accepted, resting, filled and canceled states.

## HyperCore API vs HyperEVM JSON-RPC

| Dimension | HyperCore Info/Exchange | HyperEVM JSON-RPC |
| :---- | :---- | :---- |
| State | native trading and account state | EVM accounts, contracts, logs |
| API style | Hyperliquid-specific JSON | Ethereum JSON-RPC |
| Mainnet | `api.hyperliquid.xyz` | `rpc.hyperliquid.xyz/evm`, chain `999` |
| Testnet | `api.hyperliquid-testnet.xyz` | `rpc.hyperliquid-testnet.xyz/evm`, chain `998` |
| Historical state, default endpoint | n/a | restricted |
| WebSocket | native HyperCore WS | none on the official default RPC |

HyperEVM blocks are built inside Hyperliquid execution and inherit HyperBFT security. The default RPC restricts several forms of historical access: a number of state queries serve only the latest block, `eth_getLogs` is capped at a 50-block range and four topics, and requests needing historical state are unsupported there. Archive-grade implementations exist outside the default endpoint.

HyperCore and HyperEVM expose separate state surfaces. Account state such as `clearinghouseState` belongs to the Info API; HyperEVM JSON-RPC serves EVM account, contract and log state. Full method coverage sits in the [official HyperEVM documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm).

## When Do You Need Infrastructure Beyond the Public API?

For a service reading prices occasionally and sending a modest flow of orders, the public endpoints are sufficient on their own.

The main infrastructure requirements beyond the public endpoints are sustained ingestion, isolated capacity, and repeated long-range queries.

Sustained live ingestion. WebSocket quotas are shared across every service and monitored account behind the same IP, so multi-service deployments draw on the same connection and message budgets. [Hyperliquid WebSocket Streaming](https://supanode.xyz/services/hyperliquid/websocket) provides a provisioned endpoint carrying live orderbook, executed-trade and account events over Hyperliquid's native pub/sub. Delivery starts at subscription time, and earlier history comes from a historical-data source.

Isolated capacity. Workloads that require their own hardware, region placement and reserved endpoint capacity map to a [Hyperliquid Dedicated Node](https://supanode.xyz/services/hyperliquid/dedicated-node). Dedicated infrastructure changes connection and query capacity while protocol-level limits remain unchanged: address-based action allowances, nonce windows and open-order caps apply the same way. Shared streaming stays the simpler option while its capacity covers the workload.

Repeated historical queries. Long-range analytics needs persistent storage beyond the Info API's bounded retention. [Hyperliquid Indexer](https://supanode.xyz/services/hyperliquid/indexer) holds perpetual fill history from 2025-07-27 onward with nanosecond block timing, liquidation and TWAP context, builder codes and maker/taker flags, queryable as ClickHouse SQL. Funding rates sit outside that dataset.

Streaming, dedicated nodes and indexing address transport and query cost. Trading actions still go through `POST /exchange` with the application's own signer, nonce, asset-ID and account semantics.

Keeping the interfaces separate gives each state path a clear role: Info for snapshots and reconciliation, Exchange for signed state changes, WebSocket for continuous updates, HyperEVM JSON-RPC for EVM state. Application logic handles the transitions between them.

Building on Hyperliquid? [Supanode's streaming, indexing and dedicated infrastructure](https://supanode.xyz/services/hyperliquid) covers the workloads that outgrow the public endpoints.

