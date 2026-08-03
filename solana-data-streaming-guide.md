---
title: 'Solana Data Streaming: WebSocket, gRPC, Shreds, and Indexers'
slug: 'solana-data-streaming-guide'
canonical_url: 'https://supanode.xyz/blog/solana-data-streaming-guide'
date: "2026-08-01"
description: 'A practical comparison of Solana WebSocket, Yellowstone gRPC, raw and decoded shreds, managed streams, and indexers by data stage, transport, and recovery needs.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Solana%20Data%20Streaming_%20WebSocket,%20gRPC,%20Shreds,%20and%20Indexers.png"
coverAlt: "Solana Data Streaming: WebSocket, gRPC, Shreds, and Indexers"
tags: ["Data Streaming", "Solana Shreds", "Solana Indexer", "WebSocket"]
---

Polling-based RPC architectures often become inefficient under sustained Solana workloads — rate limits kick in, state updates get missed between polls, and gaps can surface downstream before anyone notices. Solana data streaming is the continuous delivery of on-chain events without repeatedly polling RPC. The right architecture depends on when the data becomes available, how it is transported, and whether the application needs raw signals, execution-aware events, or indexed history.

WebSocket fits lightweight, browser-adjacent subscriptions. Backend ingestion that needs heavy filtering leans on [Yellowstone gRPC](https://supanode.xyz/docs/solana/grpc/overview). Specialized, latency-critical systems reach for shreds, available before receiving-validator replay and before standard post-execution metadata. Indexers sit further downstream, turning all of that into storage and query access once the data has already streamed through.

A protocol-first choice can hide the more important question: which data stage the workload actually needs. If that question is answered too late, replay, deduplication, checkpointing, and backfill logic often have to be added after the first version of the pipeline is already running.

Production Solana stacks commonly mix several of these pieces together. The practical decision is which combination fits the workload.

## What Is Solana Data Streaming?

At the simplest level, streaming means push-based delivery: the server sends updates as it observes and publishes them, without a client having to ask first.

Calling `getSignaturesForAddress` or `getTransaction` on a loop means hoping the interval is tight enough; a stream removes that guesswork — a client opens a connection and receives updates as they happen: accounts change, transactions land, slots advance, and blocks move toward confirmation.

That distinction matters more once volume grows. [Solana's own documentation on payment monitoring](https://solana.com/docs/payments/accept-payments/indexing) notes that low-volume verification can lean on repeated RPC calls, but production systems need something sturdier, because standard RPC carries rate limits, no built-in persistence, and coarse granularity. The same logic applies well outside payments. Anything continuous — a bot, a dashboard, an indexer, a risk engine — eventually runs into the ceiling of "ask and wait."

Streaming covers ingestion. History, query access, and properly decoded payloads come from separate layers built on top of that stream, including [RPC infrastructure](/blog/solana-rpc-infrastructure-guide), indexing, storage, and query systems.

## The Three Decisions: Data Stage, Transport, and Data Product

The choice is often framed as "WebSocket vs gRPC," but transport is only one part of the decision.

Three things are worth separating clearly: data stage — timing within the ledger lifecycle, running from pre-execution through processed, confirmed, finalized, and historical; transport — how the data moves between systems, whether UDP, WebSocket, gRPC over HTTP/2, or a database protocol; and data product — the shape of the payload once it lands, spanning raw fragments and decoded transactions at one end, execution-aware events and enriched business objects further along, and normalized database rows at the other.

These three axes are distinct, though not fully independent — data stage constrains which payloads are even possible, which is part of why protocol labels are misleading on their own. A gRPC stream can carry standard post-execution Geyser updates, or it can carry a provider's pre-execution reconstructed transactions — same transport, different data stage. A WebSocket can be the plain Solana JSON-RPC PubSub surface, or it can be a provider's rebuilt WebSocket product sitting on top of an internal gRPC or shred pipeline behind the scenes. Two services using identical protocol names can end up with different reliability guarantees, simply because they sit at different points in the lifecycle.

The practical takeaway: two products with the same protocol label can still differ in data stage, filtering depth, and recovery behavior. Before comparing transports, check which lifecycle stage each product actually exposes.

## Where Solana Data Becomes Available

The lifecycle explains why each access path exposes a different kind of signal.

1. A client sends a transaction, and the current leader schedules and executes it during Banking Stage.  
2. Execution output becomes entries, which [the Broadcast Stage splits into shreds and propagates through](https://docs.anza.xyz/validator/tpu) Turbine.  
3. Receiving validators reconstruct those shreds into entries and blocks, then replay the transactions to validate them independently.  
4. Once replay completes, a validator's own RPC service can expose account, transaction, and slot state directly from bank state; where a Geyser plugin is running, validator-side events also become available to gRPC-based consumers like Yellowstone.  
5. RPC, WebSocket, and gRPC providers expose selected slices of that validator state and event data to clients, through different internal mechanisms depending on the provider.  
6. Downstream consumers persist, normalize, reconcile, and query the resulting data.

Shreds sit at step 2 — the leader has already scheduled and executed the transactions to produce entries, but shreds still arrive before receiving validators replay the block and before ordinary client streams expose post-execution metadata like logs or confirmation status. Real-time access paths branch off at several different points along this lifecycle, which the diagram below separates directly.

![Solana data stages from transaction intake through leader execution, entries, Turbine shreds, validator replay, RPC and Geyser outputs, and persisted data, with access paths for UDP shreds, Yellowstone gRPC, managed streams, and downstream APIs.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Data%20Stage%20vs%20Access%20Path%20in%20Solana%20Streaming.png)

<sub>Data paths</sub>

Data stage flows top to bottom on the left; access paths branch off wherever that data actually exists. Shreds branch off before receiving-validator replay. Standard RPC and WebSocket typically tap validator bank state directly, while Yellowstone gRPC taps Geyser-based events — two separate mechanisms sitting after replay, neither one built on top of the other.

## Solana Streaming Options at a Glance

A short workload map before the full comparison:

* Simplest real-time UI → WebSocket  
* Filtered backend stream → Yellowstone gRPC  
* Early transaction-intent signal → shreds; some provider-specific preconfirmation feeds may expose an earlier scheduling-stage signal  
* Durable history and queries → indexer or database  
* Occasional reads and reconciliation → JSON-RPC

The comparison is easier to read in two parts: what each option is, and how it behaves in production.

### What each option is

| Option | Data stage | Typical payload | Transport | Filtering |
| :---- | :---- | :---- | :---- | :---- |
| JSON-RPC polling | Current/historical state on request | JSON responses | HTTP | Per request |
| Standard WebSocket | Post-execution PubSub notifications | Accounts, logs, signatures, slots, blocks | WebSocket | Method/provider dependent |
| Yellowstone gRPC | Execution-aware, commitment dependent | Protobuf account/tx/slot/block/entry updates | gRPC / HTTP2 | Strong server-side filters |
| Raw shreds | Before receiving-validator replay | Native shreds | UDP | None in the base protocol; some managed products add source-side filtering |
| Decoded/preprocessed shreds | Before receiving-validator replay; provider-reconstructed | Provider-specific decoded transactions | Usually gRPC, sometimes WebSocket | Product dependent |
| Webhooks / managed streams | Provider-processed events | Filtered or transformed events | HTTP, queue, provider-specific | Product dependent |
| Indexer / SQL / historical API | Persisted, normalized | Tables, REST/GraphQL/SQL results | HTTP / database protocol | Query language |

### How each behaves in production

| Option | Replay/backfill | Browser fit | Operational burden | Best fit | Not ideal for |
| :---- | :---- | :---- | :---- | :---- | :---- |
| JSON-RPC polling | Application-built | Strong | Low initially; high at scale | Occasional reads, reconciliation | Continuous high-volume observation |
| Standard WebSocket | Usually application/provider dependent | Strong | Medium | Frontends, lightweight subscriptions | Durable high-throughput ingestion without recovery design |
| Yellowstone gRPC | Provider and deployment dependent | Weak natively; gRPC-Web, proxies, or provider WebSocket layers can bridge it | Medium | Backend ingestion, indexers, monitoring, bots | Simple occasional reads or queryable history |
| Raw shreds | Client-built | No | Very high | Earliest transaction-intent pipelines | Account state, logs, confirmation, general apps |
| Decoded/preprocessed shreds | Product dependent | Usually no | Medium | Early signal without building a shred decoder | Confirmed execution state |
| Webhooks / managed streams | Product dependent | None directly — webhooks land on a backend, which can relay to the browser | Low | Notifications, managed workflows | Lowest-level control, universal portability |
| Indexer / SQL / historical API | Depends on retention and cursor design, not automatic for every product | Via application API | Medium to high | Analytics, history, reconciliation, serving APIs | Earliest real-time signal |

On the "Replay/backfill" column specifically: treat any provider claim there as a feature the provider built on top, separate from anything the underlying spec promises.

## Solana WebSocket: Best for Frontends and Lightweight Subscriptions

Standard Solana WebSocket is the JSON-RPC PubSub surface — the same interface documented in the [Solana API and RPC guide](/blog/solana-api-rpc-guide), just running over `ws://` or `wss://` instead of request/response HTTP. It supports account, program, logs, signature, slot, root, and (with caveats) block-level subscriptions.

The appeal is practical: it's browser-native, the implementation barrier is low, and most client SDKs handle it out of the box. That makes it the default for dashboards, wallet UIs, and lightweight backends that don't need to absorb a firehose.

A few functional limits should be accounted for before the subscription becomes part of a production path. `logsSubscribe` supports a `mentions` filter, but [only for one address per call](https://solana.com/docs/rpc/websocket/logssubscribe) — fan-out across many addresses means many subscriptions. [`blockSubscribe` remains an unstable method](https://solana.com/docs/rpc/websocket/blocksubscribe): it requires validator-side flags, only supports `confirmed` or `finalized` commitment, and is best kept out of anything load-bearing. And subscription methods commonly [default to `finalized` commitment](https://solana.com/docs/rpc/websocket) when none is specified, which is safer but slower than many app builders expect on first read.

A minimal subscription request looks something like this:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "accountSubscribe",
  "params": [
    "<ACCOUNT_PUBKEY>",
    {
      "encoding": "jsonParsed",
      "commitment": "confirmed"
    }
  ]
}
```

There's nothing exotic in that request, but the commitment parameter matters — skipping it lets the endpoint default decide the freshness and rollback-risk trade-off for the consumer.

WebSocket subscriptions don't inherently replay missed events or guarantee gap-free delivery. Reconnects, resubscription after a drop, heartbeat checks, and duplicate tolerance belong in the base design for any live socket.

## Yellowstone gRPC: Best for Filtered Backend Ingestion

Geyser is the validator-side interface that exposes execution-aware events as they happen. [Yellowstone is a gRPC implementation built on top of it](https://supanode.xyz/docs/solana/grpc/overview), and gRPC is the transport carrying the data — three separate layers that are easy to blur into one.

Server-side filtering means a consumer subscribes to specific accounts, transaction patterns, slots, blocks, block metadata, and entries, and the server narrows the firehose down before it ever reaches the client. For anything that needs sustained, high-volume, filtered ingestion — bots, monitoring systems, indexers feeding a database — that's a meaningfully different operating model than polling or plain WebSocket.

Protobuf doesn't render well in a browser, deployment runs as a backend service, and the consumer still owns ordering, deduplication, checkpointing, storage, and reconciliation. Filter caps, replay windows, and exact stream semantics vary by provider and deployment, and a general comparison like this one won't capture those specifics — check them directly against the [Yellowstone gRPC guide](/blog/yellowstone-grpc-solana-guide).

Yellowstone gRPC fits well when the workload needs sustained backend filtering before data reaches the client. It solves delivery — recovery, deduplication, persistence, and backfill remain downstream design decisions.

## Raw and Decoded Shreds: Earliest Signal, Highest Operational Cost

Shreds become available before receiving validators replay the block and before ordinary client streams expose post-execution metadata — timing that makes them both appealing and difficult to work with.

Raw shreds are the fragments the leader broadcasts through Turbine after Banking Stage has already scheduled and executed the transactions into entries. Consuming them directly means handling packet ordering, deduplication, forward error correction, reconstruction, and parsing — all client-side, all continuously, at UDP speeds. That's a serious engineering commitment, bought for one specific thing: visibility into the leader's already-ordered transaction intent before the rest of the network replays and confirms it.

Replay and confirmation are a separate question entirely. Account state, logs, inner instructions, balance changes, and confirmation status typically arrive later, once receiving validators replay the block and their own execution pipeline makes that metadata available. Shred signals open a window into what the leader has already scheduled, but confirming what the wider network accepts still requires that replay.

Decoded or preprocessed shreds are a middle layer some providers offer: reconstructed, partially decoded transaction intent delivered over gRPC or, on some products, a provider-specific WebSocket, so the consumer doesn't have to build a shred decoder from scratch. That lowers the operational bar somewhat, but the underlying trade-off stays the same — still ahead of replay, still missing logs and final state, with the exact schema and guarantees set at the provider level, separate from anything the protocol itself defines.

Shreds make sense when that pre-replay visibility creates measurable value — latency-sensitive trading and MEV-adjacent research being the clearest cases. For most other workloads, the operational cost outweighs what the earlier signal actually provides.

## Webhooks and Managed Streaming Pipelines

Webhooks push pre-packaged event alerts to an endpoint. Depending on the product, a provider may handle decoding, filtering, and transformation on their side before delivering matching events to an HTTP endpoint or a queue — closer to notification than to a continuous byte stream.

Webhooks hand off retry logic, routing, and transformation to the provider, which cuts down on infrastructure to run on the consuming side. That convenience comes with a provider-specific integration cost: semantics, retry behavior, and event shape differ by provider, so moving between providers usually means changing the consuming logic, well beyond swapping a connection string.

This fits notifications, automations, and application workflows well — think "alert a user when their transaction confirms" or "trigger a downstream job when a program emits a specific event." Lowest-level control and long-term portability are where this option falls short.

## Where Indexers and Historical Data Fit

A stream delivers changes as they happen. History and query access come from a separate layer built afterward — the same distinction that runs through Solana's broader RPC infrastructure.

An indexer or database sits downstream of ingestion. It takes the stream — Yellowstone gRPC, WebSocket, or a combination — and persists it into a normalized, queryable form: tables, a GraphQL layer, a SQL warehouse. That's where questions like "what happened to this account over the last 30 days" or "every swap above this size" actually get answered; a live stream alone doesn't answer them.

Production systems typically combine gRPC or WebSocket ingestion with a durable queue, a database, and periodic RPC backfill to fill gaps. For most application-facing workloads, historical APIs and SQL access sit downstream of ingestion and storage, after raw events have been collected, normalized, and made queryable. Analytics-heavy or reporting-focused products often flip that order — the database and query layer are the actual deliverable, and ingestion exists mainly to keep it current. The deeper architecture for this layer, including schema design and query patterns, is covered in the [Solana historical data and indexing guide](/blog/solana-indexing-historical-data-api).

## Production Reliability: Commitment, Reconnects, Replay, and Failover

[Commitment levels](https://solana.com/docs/rpc) describe how much rollback risk a consumer accepts at each stage — treating them as speed settings misses what they're actually doing. `processed` is the validator's newest view and can still be rolled back. `confirmed` means the block has received votes from supermajority stake. `finalized` means the block has reached maximum lockout under Solana's commitment model. Picking the wrong one for the job is a correctness issue.

Guarantees here depend on the specific product and stream type as much as on the protocol underneath it — some managed offerings advertise exactly-once delivery, ordering, and automatic reconnection for particular subscription types, while other streams from the same provider run on best-effort semantics. Absent a documented contract proving otherwise for the exact stream in use, a pipeline that cannot miss data should default to at-least-once-safe consumption; assuming stronger guarantees than what's actually documented is what tends to turn reconnects or volume spikes into silent gaps.

A practical reliability checklist:

* Reconnect with exponential backoff and jitter, rather than hammering a dropped endpoint.  
* Resubscribe explicitly after reconnect — a fresh connection doesn't inherit prior subscriptions.  
* Checkpoint by slot and signature for transaction-centric flows, or by slot plus a write-version marker where the product provides one for account streams.  
* Deduplicate at the consumer when the product's delivery semantics allow duplicates after a reconnect — exact behavior depends on the specific implementation.  
* Detect gaps using whatever continuity signal the specific stream actually provides — slot progression, a write-version marker, or provider-specific sequence numbers — since slots can be skipped or forked and no single universal sequence exists to check against, then backfill through RPC or an indexer once a gap is found.  
* Fail over across regions or upstream sources when the primary connection degrades.  
* Monitor consumer lag, queue depth, and heartbeat health continuously, because a silent stream and a healthy stream look identical from the application side until something downstream breaks.

![Solana streaming reliability loop: detect the gap by slot or sequence signal, reconnect with backoff and jitter, resubscribe, checkpoint, then backfill missing data through RPC or an indexer.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Production%20Streaming%20Reliability%20and%20Recovery.png)

For workloads that cannot silently miss data, these components belong in the base design from the start. Provider-advertised replay windows sit on top as a convenience; the reconnect-and-recover logic underneath still needs to exist independently of whatever the provider offers.

## Architecture Patterns by Workload

Frontend or dashboard  
`WebSocket → application state`, with HTTP RPC handling initial state load and periodic reconciliation. This is the workload where standard PubSub is usually enough. 

Backend monitoring or indexer  
`Yellowstone gRPC → durable queue → idempotent consumer → database → serving API`. Checkpointing and RPC/indexer backfill after gaps are core to this pattern, built in from the start.

Latency-sensitive trading  
`Raw or decoded shreds → early signal`, paired with `Yellowstone gRPC/RPC → execution result and confirmation`, plus a separate landing path for the transaction itself. Systems that trade on shred data still need a downstream confirmation path to know what actually executed — without it, they're only acting on intent.

Analytics and historical queries  
`gRPC or WebSocket ingestion → normalized storage → SQL/API`. RPC stays useful for reconciliation and targeted lookups here, but the bulk of analytical querying runs against normalized storage. Running analytics directly against a live stream tends to produce fragile, unrepeatable queries.

## How to Choose a Solana Data Streaming Provider

Advertised latency numbers are hard to compare unless host, region, event definition, commitment level, timestamp source, and workload are controlled. The useful questions are more specific:

* Which lifecycle stage does the product actually expose — pre-execution, processed, confirmed, finalized, or historical?  
* Which streams and methods are supported, and which commitment levels apply to each?  
* What are the filter caps, and do they scale with the address or program lists real workloads need?  
* What connection, subscription, throughput, and bandwidth limits apply, and at what point does the plan tier start to matter?  
* What replay or backfill semantics are included, and are they documented as guarantees or left as best-effort?  
* How are duplicates, gaps, and ordering handled — by the provider, or left to the consumer?  
* Where are the source validators, and what does the failover model look like across regions?  
* What format arrives on the wire, and how much decoding work falls on the consumer?  
* Which client libraries are maintained, and how well do they track protocol or SDK version changes?  
* How is authentication handled, and what's exposed at the endpoint level if a key leaks?  
* What observability and support are available when production debugging is needed?  
* How is usage billed — by connection, by message, by bandwidth — and does that model match the actual traffic shape?  
* Is there trial access to benchmark the option against the real workload before committing?

If the answer to any of these is vague, the latency figure needs more context before it is used for provider selection. For teams weighing managed infrastructure against running their own nodes and gRPC endpoints, that trade-off deserves its own analysis, covered in the [managed vs. self-hosted RPC guide](/blog/managed-rpc-vs-self-hosted-rpc). For benchmarking methodology specifically — how to measure latency rather than trust a vendor's number — the [low-latency Solana RPC guide](/blog/low-latency-solana-rpc) goes deeper than this piece needs to.

## Which Solana Data Stream Should You Use?

Architecture decisions here come down to matching latency needs against how complete the data needs to be.

Frontend and UI work stays lightweight on standard WebSockets. Backend infrastructure, monitoring bots, and indexers often benefit from the server-side filtering Yellowstone gRPC provides, particularly once throughput or filtering needs grow past what plain WebSocket handles comfortably. Raw or decoded shreds fit latency-critical execution paths, where seeing the leader's transaction intent before replay and standard confirmation justifies the operational complexity that comes with it.

Live streams cover what's happening now; some providers layer replay or backfill windows on top as a product feature, separate from anything the underlying stream guarantees. Historical queries and long-term analysis typically still need a dedicated database layer, and plain JSON-RPC commonly remains useful underneath all of it for point reads and reconciliation — though transaction submission increasingly runs through TPU/QUIC or [specialized sender paths](https://supanode.xyz/docs/solana/sender/overview), and backfill commonly happens through an indexer or a provider's managed replay just as much as through RPC itself.

Need to validate WebSocket, Yellowstone gRPC, or raw shreds against real filters, region, throughput, and recovery requirements? [Compare Solana data streaming options for your workload →](https://supanode.xyz/data-streaming)
