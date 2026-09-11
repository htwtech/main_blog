---
title: 'Hyperliquid CLOB: How HyperCore Matches Orders'
slug: 'hyperliquid-on-chain-order-book'
canonical_url: 'https://supanode.xyz/blog/hyperliquid-on-chain-order-book'
date: "2026-09-11"
description: 'How the HyperCore CLOB works across consensus batch ordering, order matching, L2 versus L4 data, latency, and production data access.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Hyperliquid%20CLOB_%20How%20HyperCore%20Matches%20Orders.png
coverAlt: 'Hyperliquid CLOB: How HyperCore Matches Orders'
tags: ["Hyperliquid", "CLOB", "L2 versus L4 data"]
---

Where a matching decision happens matters as much as whether a trade eventually settles on-chain. An order book run by an operator or a validator subset outside consensus still requires trusting that party's internal execution, even once the resulting trade is published to a public ledger. Hyperliquid's HyperCore handles that question differently: it holds the perpetual and spot order books, together with margin and clearinghouse state, as native protocol state, so matching itself becomes part of what validators agree on. Every order, cancel, trade, and liquidation executes inside that state and settles with one-block finality under HyperBFT consensus.

Off-chain and hybrid designs typically keep the matcher outside that consensus boundary: an operator or a subset of participants runs matching logic under its own rules, separate from the base protocol, and the chain records the trade only once matching has already happened elsewhere. Hyperliquid folds that decision inside consensus.

## HyperCore, HyperBFT, and the CLOB

HyperBFT produces an ordered stream of consensus batches — a block typically carries one or two of these. Hyperliquid's mempool and consensus logic are semantically aware of HyperCore order-book actions: an action is tagged as a cancel, a GTC order, an IOC order, or another action type, and that tag affects how it is sorted inside a batch.

Within each consensus batch, Hyperliquid's semantically aware mempool and consensus logic apply the [documented action ordering](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/order-book): non-GTC/IOC actions first, cancels second, and actions containing GTC/IOC orders third. HyperCore then executes those actions against native order-book and margin state. 

Price-time priority depends on a deterministic order of actions reaching the matcher. Ordinary application-level smart-contract execution has no comparable mechanism for this: a contract cannot reach into the mempool and reorder pending calls by category before its own logic runs, though a chain could in principle build custom sequencing rules to approximate it. HyperCore's tick/lot validation and margin checks are native protocol logic, while Hyperliquid's mempool and consensus logic provide the semantic action ordering that precedes matching. 

This design makes matching deterministic and consensus-verifiable, while putting more of the performance load on execution itself.

## Three Models Behind an On-Chain Order Book

* Native on-chain matching. The matcher is part of the consensus-agreed state machine itself. Matching functions as a native state transition governed by the protocol's own execution rules. This is HyperCore's model.  
* Off-chain matching, on-chain settlement. An operator or a subset of participants runs the matching logic under its own rules, separate from the base protocol's consensus, and the chain records the trade only once that matching has already happened.  
* Application-level on-chain CLOB. Matching logic runs as a smart contract or program on a general-purpose chain. The book and the matcher are both public state, but the CLOB operates within that host chain's generic account, compute, and transaction model rather than as a native execution component of the chain itself.

HyperCore falls into the first category. An application-level CLOB, by contrast, inherits whatever generic transaction-ordering rules its host chain applies to every other contract call, with no category-aware sort available to it. HyperCore's cancel-before-IOC/GTC rule exists because the sort happens at the execution-layer level, before any contract-style logic runs at all.

## The Order Lifecycle, Start to Finish

Getting into consensus. The order is signed by the trader or their API wallet, submitted through an API server or directly to a node, and propagated across the HyperBFT validator network before landing in a committed consensus batch — a block typically carries one or two of these.

Inside the batch. Actions are sorted by category — non-GTC/IOC actions first, cancels second, GTC/IOC-containing actions third — following the semantic awareness built into Hyperliquid's mempool and consensus logic. Tick size, lot size, and, for a new order, an opening margin check are validated at this stage.

Execution and clearing. Depending on whether it crosses the book, the order matches, rests, or is canceled under price-time priority. The clearinghouse updates positions, margin, and balances in the same state transition as the book.

Downstream. The result is committed as new HyperCore state with one-block finality, then exposed through the Info API, WebSocket feeds, and node-level outputs.

## A Worked Example: One Batch, Three Actions 

![HyperCore consensus batch ordering cancel A before IOC B and GTC C.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HOW%20HYPERCORE%20ORDERS%20ACTIONS%20IN%20A%20CONSENSUS%20BATCH.png)

<sub>Semantic ordering inside one HyperCore consensus batch.</sub>


Say BTC's book currently rests like this:

* Asks: order `#1001`, 2 BTC at $60,010 (oldest at that level); order `#1002`, 3 BTC at $60,015.  
* Bids: nothing near the top of book worth matching against here.

Three actions land in the same consensus batch:

* Action A — a maker cancel of order `#1001`.  
* Action B — an IOC buy for 4 BTC, limit $60,020.  
* Action C — a GTC buy for 1 BTC at $60,008.

Within the batch, Hyperliquid's mempool and consensus logic sort actions by category before matching executes against the book: Action A, a cancel, falls into category two; Actions B and C, both containing a GTC/IOC order, fall into category three and are ordered between themselves by proposer order. Category one has nothing in it this round. Assume the block proposer places Action B before Action C within category three — that gives a processing order of A, then B, then C, regardless of which action actually reached the network first. 

Processing A first removes order `#1001` entirely. The ask side now shows only `#1002`: 3 BTC at $60,015.

Action B executes next. The IOC wants 4 BTC up to $60,020, and the only resting ask left is `#1002` at $60,015. It matches all 3 BTC of `#1002` — a full fill for the maker, a partial fill for the 4-BTC taker. The remaining 1 BTC of the IOC has no more asks at or below $60,020 to match against, and since it's IOC, that 1 BTC doesn't rest — it's canceled outright as part of the same action.

Action C runs last. With the ask side now empty, the GTC buy at $60,008 has nothing to cross, so it rests as a new bid: 1 BTC at $60,008.

![BTC perpetual book changes after cancel, IOC buy, and GTC buy.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HOW%20THE%20BATCH%20CHANGES%20THE%20ORDER%20BOOK.png)

<sub>How three actions change the BTC perpetual book.</sub>


The clearinghouse update happens in the same transition as the fill, not as a follow-up step. The fill changes the taker's signed BTC position by \+3 BTC and the maker's by \-3 BTC; HyperCore updates the corresponding clearinghouse state, including positions and margin requirements, as part of execution. Whether that means "more long" or "less short" for either side depends on the position each account already held before the fill. Canceling order \#1001 produces no fill or position change, but perp order-book operations reference the clearinghouse, and removing an open order can still affect the account's available margin. The new resting GTC order at $60,008 passed its opening margin check earlier in the pipeline; it produces no position change until it fills, but it remains part of the account's margin/risk state and will be rechecked when matching occurs. 

![Final states of canceled, filled, partial, and resting HyperCore orders.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/FINAL%20ORDER-BOOK%20STATE%20AFTER%20BATCH%20EXECUTION.png)

<sub>Final BTC perpetual order-book state after execution.</sub>


That single batch produces one full fill, one partial fill with a canceled remainder, one clean cancel, and one new resting order — four different outcomes from three submitted actions, resolved deterministically by the category sort rather than by whichever action happened to arrive first over the network.

## Order Status: More Than Filled or Canceled

HyperCore's status model extends well beyond `open`, `filled`, and `canceled`. Hyperliquid's order lifecycle also includes protocol-specific order statuses created by validation, risk, oracle, and order-type rules: `triggered` for conditional orders that activate, `rejected` for actions that fail validation outright, `marginCanceled` when an order can no longer be supported by account margin, `selfTradeCanceled` when the resting side of a would-be self-cross gets pulled instead of executing, `reduceOnlyCanceled` for a reduce-only order that is canceled because it would not reduce the existing position, `liquidatedCanceled` when a resting order gets removed as part of a liquidation, `badAloPxRejected` when a post-only order's price would have crossed the book, `iocCancelRejected`, and `oracleRejected` when an order's price sits outside what current oracle-based bands allow. 

A `marginCanceled` order and an `oracleRejected` order fail for entirely different reasons, at different points in the pipeline, but both terminate the same way from a trader's perspective: the order is gone. HyperCore also enforces account margin limits, oracle boundaries, and order-type restrictions before state is committed.

On MEV: cancel priority and consensus-backed ordering reduce one stale-quote race inside a shared batch, while proposer ordering within each category and latency competition still remain.

## What the Order Book Stores

![Comparison of aggregated L2 order book and individual L4 orders.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/L2%20vs%20L4%20Order%20Book%20Data.png)

<sub>L2 aggregation versus L4 order-level market data.</sub>


An `l2Book` snapshot returns aggregated levels — up to 20 per side — and each level contains a price, total size, and order count: `px`, `sz`, and `n`. That last field gives limited information about the individual orders behind the level: it shows how many orders contribute to the size, but not their individual sizes or queue order.

Order-level data carries a different shape entirely. A resting order in L4/node output exposes fields like `oid` (order ID), `timestamp`, `origSz` (original size), `sz` (remaining size), `tif` (time-in-force), and `cloid` (client order ID) — enough to track a specific order from placement to its terminal state. Raw book diffs stream this as discrete events: `new` when an order is added, `update` when its remaining size changes after a fill, `remove` when it's fully filled or canceled. Hyperliquid documents L4 snapshots as checkpoints for maintaining an order-level book: applying the ordered order-status stream on top of a snapshot keeps it current in real time, order by order.

In the BTC example, an `l2Book` consumer watching that book would have seen the ask-side size at $60,015 simply drop from 3 to 0 between two snapshots. A consumer following the `oid`\-level event stream would see the actual sequence instead — cancel of `#1001`, then `#1002` consumed by the incoming IOC, in that order, with timestamps attached to each event.

## Reconstructing the Book from Node Output

Hyperliquid documents an [L4 snapshot](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/nodes/l1-data-schemas) as containing the entire order book with full order information at every level, with same-price orders sorted in time order. Reconstruction starts from a checkpoint L4 snapshot and then applies the ordered stream of order-status updates and book-diff events (`new`/`update`/`remove`) that follow it. The result is a continuously maintained local L4 book that stays current without re-fetching a full snapshot after every change.

An `l2Book` poll and a maintained L4 book answer different questions. The former shows that a level's total size changed. The latter shows which specific order changed it, whether that order had been resting for milliseconds or minutes, and where in the queue the next order at that price stands. For a market maker deciding whether to add size at a level or move to another price, queue position can be more useful than the aggregated size alone.

## Why the Native Design Matters for Throughput

Hyperliquid documents [current mainnet capacity](https://hyperliquid.gitbook.io/hyperliquid-docs/hypercore/overview) at roughly 200,000 orders per second, with execution identified as the present bottleneck rather than consensus or networking. This is a first-party stated current capacity figure, not a measurement of continuously observed mainnet traffic or an independent load-test result. 

A matcher running inside a generic transaction model carries contract-call and generic state-access overhead without access to batch-level semantic sorting. Hyperliquid's semantically aware mempool/consensus logic and HyperCore's native margin/clearinghouse integration exist specifically to avoid that overhead. Hyperliquid's own documentation frames further scaling as an execution-optimization problem, describing consensus and networking as having more headroom left than execution currently uses.

## Hyperliquid CLOB vs Off-Chain Order Books

HyperCore's matcher sits inside the consensus state machine. Many hybrid or off-chain CLOBs run matching outside that state machine and settle the result on-chain afterward.

|  | HyperCore | Off-chain / hybrid CLOB |
| :---- | :---- | :---- |
| Matching | Native protocol execution | External matcher or sequencer |
| Book state | Consensus-backed | Usually held off-chain |
| Settlement | Same execution step as matching | Frequently a separate on-chain step afterward |
| Verifiability | Order-level state and history exposed | Depends heavily on the specific architecture |

A well-engineered centralized matcher can still be faster in isolation, since it carries no consensus overhead. Hyperliquid's architecture trades some of that raw speed for matching state transitions that validators agree on directly, as part of consensus itself.

## Hyperliquid CLOB vs AMMs

CLOBs and AMMs differ mainly in how liquidity is represented and executed.

|  |  HyperCore (CLOB) | Concentrated-liquidity AMM |
| :---- | :---- | :---- |
| Liquidity representation | Discrete bids/asks at chosen prices, ranked by price-time priority | Liquidity allocated across selected price intervals |
| Management | Makers manage orders, prices, and sizes directly | LPs manage liquidity positions and ranges |
| Settlement | Same execution step as matching | Atomic within the swap transaction |

A HyperCore maker manages a book directly — quoting, canceling, and adjusting orders, with queue position at each level determining priority. A concentrated-liquidity LP instead manages a position across a chosen price range, with liquidity active only while price stays inside that range. 

## Hyperliquid Block Time and Trading Latency

Hyperliquid speed discussions usually involve three different measurements, and each one constrains a different part of the lifecycle above.

Block cadence sits around 0.07 seconds, per [Hyper Foundation's published figure](https://www.hyperfoundation.org/). Hyperliquid documents one or two consensus batches within a typical block, so block cadence should not be read as the exact cadence of individual consensus batches or order matching. A specific order's landing latency also depends on when it is submitted and how quickly it propagates into a batch. 

![Hyperliquid 70 ms block time versus 200 ms median and 900 ms p99 latency.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Hyperliquid%20Block%20Time%20vs%20Order%20Latency.png)

<sub>Block time and end-to-end order latency differ.</sub>

End-to-end trading latency covers that separate path — from request sent to committed response received, for a client positioned close to the network. HyperCore's own benchmark reports roughly 0.2 seconds median and 0.9 seconds at the 99th percentile, several times the block-cadence figure even in the best-case, co-located scenario.

Market-data latency adds a third layer: the time between a state change committing and a consumer's application receiving and processing it. There's no single published number here, because the route matters — public WebSocket, a third-party API, or a local node feeding a locally maintained L4 book all deliver state at different speeds.

For a bot, the operational loop is broader than any single figure here: observing the market, deciding, submitting, receiving a committed response, and observing again. End-to-end bot latency therefore has to be measured across the full client path rather than inferred from the 0.07-second block cadence.

## HyperCore vs HyperEVM Block Architecture

HyperEVM shares Hyperliquid's blockchain state and HyperBFT security with HyperCore, running as a second execution lane with its own block cadence. Validators and consensus are identical between the two; only the execution rules differ.

Current documentation splits HyperEVM into [two block types](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/hyperevm/dual-block-architecture): fast blocks at roughly one second with a 3M gas limit, and slow blocks at roughly one minute with a 30M gas limit. These block times describe HyperEVM's execution lane, while HyperCore follows its roughly 0.07-second consensus cadence. HyperEVM has no equivalent to the semantic batch sort, since its transactions follow ordinary EVM ordering rather than category-based reordering. Trading logic lives in HyperCore. HyperEVM exists for general-purpose smart contracts, increasingly built to draw on HyperCore's liquidity without running inside it.

## How Developers Access Order-Book and Trade Data

Info API — on-demand snapshots, including `l2Book`'s aggregated `px`/`sz`/`n` view, up to 20 levels per side. This fits dashboards, UI order books, and strategies that sample current state before submitting an action. Queue position and individual-order behavior require order-level data.

**[WebSocket](/blog/hyperliquid-websocket-api)** — live incremental updates pushed as book, trade, and account state changes. This is the documented route for real-time public data. Production systems built on it need reconnect and gap-recovery logic, because disconnects during fast markets can leave the local view stale during the period when updates are most important.

Node outputs — trades, order-status events (`open`, `filled`, `marginCanceled`, `oracleRejected`, and the rest), raw book diffs (`new`/`update`/`remove`), and full L4 snapshots from a non-validating node. This is the data path for local queue-level reconstruction and for market-making or research workloads that need order-by-order granularity.

**[Indexer](/services/hyperliquid/indexer) and storage** — turning that live or node-level stream into a durable, queryable historical dataset. A checkpoint-plus-incremental-update pipeline handles this: establish state from a known L4 snapshot, apply diff events in order, detect gaps, backfill. Live feeds answer "what's happening now"; backtesting and gap-free historical analysis need this separate layer on top.

## What This Means for Bots and Data Pipelines

For bots and data pipelines, latency spans the full loop: observe the market, decide, sign, submit, receive a committed response, and observe again. Tail latency matters alongside the median, especially for systems that depend on fresh book state.

Feed location and data path set part of that latency before application logic runs. A strategy reading from a distant public API sees the book through a different path from one maintaining an L4 book from a local node. Feed choice, node placement, and submission path therefore become part of the trading-system design, often alongside dedicated [Hyperliquid RPC](/services/hyperliquid/dedicated-node) infrastructure.

Need reliable real-time access to HyperCore data? Explore Supanode's [Hyperliquid infrastructure](https://supanode.xyz/services/hyperliquid) for production RPC and streaming workloads.
