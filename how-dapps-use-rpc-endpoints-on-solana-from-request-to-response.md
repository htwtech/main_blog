---
title: "How dApps Use RPC Endpoints on Solana: From Request to Response"
slug: "how-dapps-use-rpc-endpoints-on-solana-from-request-to-response"
date: "2026-05-13"
description: "A practical walkthrough of how Solana dApps use RPC endpoints to read state, submit signed transactions, simulate execution, track confirmation, and handle production failure modes."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/How%20dApps%20Use%20RPC%20Endpoints%20on%20Solana_%20From%20Request%20to%20Response.png"
coverAlt: "How dApps use RPC endpoints on Solana"
tags: ["solana", "rpc", "dapps", "infrastructure", "transactions", "indexing"]
---

# How dApps Use RPC Endpoints on Solana: From Request to Response

Every wallet tap, every swap, every balance refresh, every dashboard update runs through an RPC endpoint.

On Solana, RPC is the path your dApp uses to read state, submit signed transactions, simulate execution, track confirmation, and subscribe to live changes. The endpoint sits between your app and the network, but it also shapes what your app sees: which slot, which commitment level, how fresh the data is, and whether the transaction reaches a leader in time.

Solana is running hot. [Syndica's March 2026 on-chain activity report](https://blog.syndica.io/deep-dive-solana-onchain-activity/) shows successful non-vote TPS averaging 990, vote success holding above 99.7%, and P90 compute demand reaching 58.1M compute units per block — close to the 60M block ceiling.

The rest follows that flow from request to state read, send path, confirmation, and production failure modes.

## RPC Is a Slot-Scoped View of the Chain

A Solana RPC endpoint is the URL your app uses to read state, fetch metadata, submit signed transactions, or subscribe to changes.

Read methods include:

```text
getAccountInfo(...)
getMultipleAccounts(...)
getTokenAccountsByOwner(...)
getProgramAccounts(...)
```

Transaction-flow methods include:

```text
getLatestBlockhash(...)
simulateTransaction(...)
sendTransaction(...)
getSignatureStatuses(...)
```

Streaming methods include:

```text
accountSubscribe
signatureSubscribe
logsSubscribe
programSubscribe
```

[Solana's official RPC](https://solana.com/docs/rpc) docs define [HTTP RPC methods](https://solana.com/docs/rpc/http) as JSON-RPC 2.0 requests over HTTP POST, with the standard `jsonrpc`, `id`, `method`, and `params` fields. For JavaScript apps, `@solana/web3.js` wraps that lower-level flow — its `Connection` class is literally defined as "a connection to a fullnode JSON RPC endpoint," constructed with the endpoint URL and an optional commitment or connection config.

The endpoint gives one routed, commitment-bound view of the chain — shaped by which node answered, which slot it was on, and what commitment level was requested. Every response inherits that view.

## Two RPC Paths: Reading State and Sending Transactions

![A production Solana dApp rarely just calls RPC](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/A%20production%20Solana%20dApp%20rarely%20just%20calls%20RPC.%20It%20usually%20combines%20point%20reads,%20subscriptions,%20transaction%20submission,%20and%20sometimes%20indexed%20or%20streamed%20data..png)
<sub>A production Solana dApp rarely “just calls RPC.” It usually combines point reads, subscriptions, transaction submission, and sometimes indexed or streamed data.</sub>

"The dApp uses RPC" usually collapses two different flows into one.

One flow is read-heavy: account data, balances, signatures, slots, and program-owned state. The other is write-heavy: blockhash fetch, signing, simulation, send, and confirmation tracking.

The HTTP response gives the app a signature. Landing is a separate outcome.

Keeping those flows separate changes endpoint choice, freshness requirements, retry logic, and failure handling. A production dApp usually needs both paths, but reads and writes often benefit from different endpoint behavior, freshness guarantees, and routing choices.

A Solana dApp has two RPC problems: seeing the right state, and getting transactions to the leader before the blockhash dies.

## Step by Step: What Happens When a User Clicks "Swap"

![sendTransaction returns a signature, not a guarantee](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/sendTransaction%20returns%20a%20signature,%20not%20a%20guarantee.%20The%20dApp%20still%20has%20to%20track%20confirmation..png)
<sub>`sendTransaction` returns a signature, not a guarantee. The dApp still has to track confirmation.</sub>

The riskiest part of the swap flow usually sits between send, forwarding, and leader admission — inside RPC and networking rather than consensus itself.

### Step 1: The UI Builds Intent

When the user clicks "Swap," the dApp is still in the intent-building phase. The app assembles the swap details: mints, amount, slippage, route, and the accounts the transaction will touch. The click is local. State changes begin only after a signed transaction reaches a leader and lands in a block.

At this stage, the application may already use RPC to fetch balances, token accounts, pool state, or route-relevant account data.

### Step 2: The SDK Constructs the Transaction

The frontend or backend uses `@solana/web3.js`, Solana Kit, or another client library to build a transaction. In the classic web3.js flow, that means creating a `Transaction` or `VersionedTransaction`, attaching instructions, specifying accounts, and preparing the message for signing.

This is the point where Solana's account model starts to matter at scale.

Transactions declare the accounts they read and write up front — that parallel execution model also creates write-lock contention when thousands of transactions target the same hot accounts.

### Step 3: The App Fetches a Recent Blockhash

Before signing, the dApp calls `getLatestBlockhash`. The RPC response returns a recent blockhash and `lastValidBlockHeight`. That blockhash acts like a short-lived clock for the transaction.

[Solana's transaction docs](https://solana.com/docs/core/transactions) state that recent blockhashes are valid for 150 slots. Since slot time fluctuates, that usually gives a transaction roughly 60–90 seconds before expiry. The retry guide frames this operationally: RPC nodes may rebroadcast the transaction until it finalizes or the blockhash becomes too old.

`confirmed` is the practical default here. `processed` gives a slightly fresher blockhash with more fork exposure; `finalized` is safer but older, which shortens the usable window before expiry.

### Step 4: The App Sets Compute Budget and Priority Fee

Solana priority fees are set through compute budget instructions:

```text
ComputeBudgetProgram.setComputeUnitLimit(...)
ComputeBudgetProgram.setComputeUnitPrice(...)
```

[The official fee docs](https://solana.com/docs/core/fees/fee-structure) give the formula:

```text
prioritization_fee = ceil(
  compute_unit_price *
  compute_unit_limit / 1,000,000
)
```

The base fee is currently 5,000 lamports per signature. The priority fee is based on the requested CU limit — overestimating means paying for unused compute.

Syndica's March 2026 data makes this concrete. P90 compute per block averaged 58.1M CU, with several days hitting the 60M ceiling. Underpricing priority fees or requesting sloppy CU limits has direct scheduling consequences in that environment.

### Step 5: The App Simulates the Transaction

Simulation does three useful things before the user pays for failure:

1. Will this transaction fail?
2. How many compute units does it actually use?
3. Are the accounts and instructions shaped the way the app expects?

[`simulateTransaction`](https://solana.com/docs/rpc/http/simulatetransaction) returns logs, error state, consumed compute units, replacement blockhash data, and loaded account data size. Skipping simulation buys a small latency win and gives up the fastest way to catch compute, account, and instruction failures before submission.

### Step 6: The Wallet Signs

The wallet signs the serialized transaction. A signed transaction sitting in the browser is still only a packet waiting for delivery.

### Step 7: The App Calls `sendTransaction`

`sendTransaction` submits the signed transaction to the cluster and returns the first signature if the RPC accepts it. [Solana's method reference is explicit](https://solana.com/docs/rpc/http/sendtransaction): submit signed transaction, return first signature if accepted.

### Step 8: The RPC Node Forwards to Leaders

Solana has no public mempool. [The official retry guide](https://solana.com/developers/guides/advanced/retry) explains that the receiving RPC node attempts to broadcast to the current and next leaders. The leader schedule is known in advance, so the RPC routes directly rather than gossiping across the entire network.

By default, RPC nodes retry forwarding every two seconds until the transaction finalizes or the blockhash expires. When the outstanding rebroadcast queue exceeds 10,000 transactions, newly submitted transactions are dropped.

That's why "my transaction disappeared" can happen before consensus ever sees it.

### Step 9: The Leader's TPU Processes the Transaction

Once the transaction reaches the leader, it enters the Transaction Processing Unit. Solana's retry guide breaks the TPU into five stages:

- Fetch Stage
- SigVerify Stage
- Banking Stage
- Proof of History Service
- Broadcast Stage

Priority fees matter in scheduling, but a transaction has to reach the pipeline first. If the transaction fails at admission, the scheduler never sees the fee.

### Step 10: The App Tracks Confirmation

After `sendTransaction`, the app tracks status using `getSignatureStatuses` or `signatureSubscribe`. The HTTP response only returned the signature — confirmation is a separate process.

Commitment level matters here:

- `processed` — fastest view, less safe
- `confirmed` — practical default for most user-facing flows
- `finalized` — canonical, better for accounting, settlement, and historical guarantees

## Reading State

![RPC gives the app account data and slot context](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/RPC%20gives%20the%20app%20account%20data%20and%20slot%20context.png)
<sub>RPC gives the app account data and slot context. The dApp still has to decode the bytes into product state.</sub>

For many consumer and dashboard-heavy apps, read traffic dominates: balances, token accounts, pool state, positions, and program-owned accounts. A basic read looks like this:

```bash
curl "$SOLANA_RPC_URL" \
-X POST \
-H "Content-Type: application/json" \
-d '{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "getAccountInfo",
  "params": [
    "ACCOUNT_PUBKEY_HERE",
    {
      "encoding": "base64",
      "commitment": "confirmed"
    }
  ]
}'
```

The RPC node returns account state and metadata for the requested address. For multiple accounts, `getMultipleAccounts` is usually better than separate `getAccountInfo` calls because it reduces round trips. For broad program scans, `getProgramAccounts` can become expensive and is often rate-limited or constrained by providers.

Scaling problems usually start when point reads turn into repeated scans, broad polling, and RPC-shaped database behavior. A practical split:

- HTTP JSON-RPC → point reads
- WebSocket → hot state subscriptions
- Geyser/gRPC → backend streams and ingestion
- Indexer → historical queries, broad scans, app-specific data models

## Choosing the Right Commitment Level

Every Solana response is tied to a view of the chain. Commitment belongs in product behavior, not in leftover SDK defaults. It changes what answer the app gives the user.

In practice:

- `processed` = fastest view
- `confirmed` = default app view
- `finalized` = canonical / accounting view

For live trading UX, `processed` can make sense if the app tolerates rollback. For most user-facing balances and transaction flows, `confirmed` is the better default. For reporting, history, reconciliation, and anything that should stay stable, `finalized` is the right choice.

Commitment changes product behavior, not just latency: `processed` favors speed, `confirmed` is usually the default app view, and `finalized` is better for accounting or historical guarantees.

## Three Bottlenecks Behind Transaction Landing

Transaction landing has three separate bottlenecks — admission, scheduling, and routing. Treating them as one fee knob usually leads to overspending and bad attribution.

### 1. Admission

The transaction has to reach the leader. SWQoS matters here. [Solana's SWQoS docs explain](https://solana.com/developers/guides/advanced/stake-weighted-qos) that leaders can identify and prioritize transactions proxied through a staked validator, and that RPC nodes gain virtual stake only through a trusted peering relationship with a staked validator.

Admission failure happens before the scheduler sees the fee.

### 2. Scheduling

Once the transaction reaches the leader, the scheduler ranks it. This is where compute budget and priority fee matter. Solana's fee docs define the priority fee as a function of CU price and CU limit, and explain that higher priority fees increase scheduling priority.

### 3. Routing

Execution-critical apps often use redundant send paths, private routing, Jito or non-Jito inclusion services, and specialized providers — particularly around liquidations, launches, bots, and high-value trades.

Syndica's March 2026 report shows non-Jito providers captured 51% of tip volume, up from 22% in early 2025, with 0slot, Temporal, Helius, BlockRazor, Flashblock, Node1, and Nextblock all appearing in the non-Jito tip landscape.

Raising priority fees can help once the transaction reaches the leader. It does not replace staked routing, and it does not behave like a Jito or non-Jito tip path.

## Where RPC Ends and Indexing Starts

HTTP JSON-RPC is a request/response model. That model fits one-off reads well.

If a wallet needs to update a balance when an account changes, `accountSubscribe` is cleaner than repeated polling. If a backend needs transaction logs, `logsSubscribe` may be sufficient. If an indexer needs high-volume account, slot, block, and transaction updates, WebSocket starts to look like the wrong architecture and Geyser/gRPC becomes the better fit.

Consumer apps rarely need shreds. Indexers, bots, trading systems, and real-time analytics tools often do — and those workloads increasingly rely on Yellowstone gRPC and ShredStream for latency-sensitive or high-volume data.

If an app is repeatedly polling `getProgramAccounts`, rebuilding historical state, or maintaining its own account mirror, the problem has moved from RPC into indexing.

Comparing lag, query model, and backfill requirements directly makes more sense than adding more RPC calls.

## What Breaks in Production

### 1. Blockhash mismatch across pooled backends

*Symptom*: `BlockhashNotFound`, failed preflight, unexpected transaction expiry.

*Cause*: the app fetches a blockhash from one backend and sends or simulates through another backend that is behind or on a different view.

*Fix*: use the same endpoint path for blockhash fetch, preflight, and send — or enforce slot freshness checks across providers.

### 2. RPC rebroadcast queue overflow

*Symptom*: transaction appears to disappear.

*Cause*: RPC rebroadcast queue is overloaded. Solana's retry guide states that newly submitted transactions are dropped when the outstanding rebroadcast queue exceeds 10,000 transactions.

*Fix*: own retry logic for critical flows, use `maxRetries` deliberately, and track `lastValidBlockHeight`.

### 3. Compute budget underestimation

*Symptom*: transaction pays fees but aborts.

*Cause*: CU limit is too low, or accounts and instructions are heavier than expected.

*Fix*: simulate first, use `unitsConsumed`, add a reasonable buffer. Priority fee follows the requested CU limit rather than actual usage.

### 4. Hot account contention

*Symptom*: transaction works in calm periods but fails or lands late during launches, mints, AMM spikes, or liquidations.

*Cause*: many transactions compete for the same writable accounts.

*Fix*: priority fee helps scheduling, but program and account design matter too.

### 5. Polling hot state

*Symptom*: high RPC bill, slow UI, rate limits, inconsistent state.

*Cause*: app repeatedly calls point-read methods for state that should be streamed or indexed.

*Fix*: batch with `getMultipleAccounts`, subscribe when state is hot, index when history or broad scans dominate.

### 6. Silent WebSocket drops

*Symptom*: UI or backend stops updating with no obvious failed request.

*Cause*: long-running WebSocket connections are not permanent.

*Fix*: heartbeat, reconnect logic, track the last seen slot or signature, and backfill after reconnect.

### 7. Measuring average latency instead of tail latency

*Symptom*: provider looks fast but users still hit slow edge cases.

*Cause*: average latency hides P95/P99 spikes.

*Fix*: measure your dominant methods from your own infrastructure.

## How to Evaluate Your RPC Path

Your benchmark starts with your own method mix, region, wallet flow, retry behavior, and read/write ratio. Provider dashboards are secondary.

**Slot freshness** — Compare returned `context.slot`, `getSlot`, and blockhash context across providers. Stale reads produce expired blockhashes, broken UX, and confusing state.

**P99 latency by method** — Benchmark the methods your app actually calls: `getAccountInfo`, `getMultipleAccounts`, `simulateTransaction`, `sendTransaction`, `getSignaturesForAddress`, or `getTokenAccountsByOwner` depending on your traffic mix.

**Landing rate** — Measure during high-load windows as well as calm ones.

**Simulation success and failure reasons** — Track compute exhaustion, account errors, instruction errors, and blockhash failures separately.

**WebSocket stability** — Count reconnects, missed slots, missed signatures, and backfill gaps.

**Indexing lag** — Real-time products need indexing lag measured against current slot. Eventual completeness is a different SLA.

**Endpoint topology** — Ask whether reads, sends, WebSockets, and gRPC streams are served through the same backend pool or separated paths.

Across the market, shared and dedicated Solana RPC tiers now typically include some combination of staked SWQoS endpoints, Yellowstone gRPC, ShredStream, and provider-managed indexing — the exact mix varies by plan and provider.

We run that stack out of Frankfurt. If that's the region that matters for your workload, the [2-day free trial on shared Solana plans](https://www.htw.tech/market/rpc/pricing) is a low-cost way to run the benchmark against real traffic before committing.

## The Stack Most Solana Apps Eventually Need

A Solana RPC endpoint is the path between your app and the network's current state. For reads, it decides how fresh the data is, how expensive the query becomes, and whether the response is something your app can decode safely. For writes, it decides how the transaction reaches leaders, how retries behave, and how quickly the app can tell the user what happened.

At a minimum:

```text
dApp → SDK → JSON-RPC request → RPC endpoint → node state / leader path → response → app decode / confirmation tracking
```

Under production load:

- point reads → HTTP RPC
- hot state → WebSocket
- backend ingestion → Geyser/gRPC
- historical queries → indexing
- transaction landing → measured send paths

Solana apps that grow beyond simple reads usually end up with some version of that stack. Measuring it against real traffic costs less than debugging it by surprise.
