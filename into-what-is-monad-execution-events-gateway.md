---
title: "What Is Monad Execution Events Gateway?"
slug: "what-is-monad-execution-events-gateway"
date: "2026-03-06"
description: "Real-time execution event streaming for the Monad blockchain — beyond what JSON-RPC can offer."
author: "HighTower"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
coverAlt: "Monad Execution Events Gateway"
tags: ["monad", "rpc", "infrastructure", "gateway"]
---

# What Is Monad Execution Events Gateway?

*Real-time execution event streaming for the Monad blockchain вЂ” beyond what JSON-RPC can offer.*

---

## The Problem with JSON-RPC

If you're building on Monad today, chances are you rely on JSON-RPC to read blockchain state. You call `eth_getBlockByNumber`, `eth_getLogs`, or `eth_getTransactionReceipt`, and you get back a snapshot of what happened вЂ” after the fact.

This works fine for simple use cases. But it breaks down fast when you need:

- **Low-latency event delivery.** Polling `eth_getLogs` every few seconds adds hundreds of milliseconds to seconds of delay. For time-sensitive applications вЂ” liquidation bots, bridge relayers, real-time dashboards вЂ” that delay is unacceptable.

- **Execution-level visibility.** JSON-RPC gives you the *result* of execution: logs, receipts, state diffs. It doesn't show you *how* execution happened вЂ” which internal calls were made, which storage slots were touched, how the EVM stepped through the transaction.

- **Finality awareness.** Standard RPC treats blocks as either confirmed or not. But Monad's MonadBFT consensus has multiple stages вЂ” a block can be proposed, voted on, finalized, and verified. Applications that care about finality (bridges, exchanges) need to know *which stage* a block has reached.

- **Lossless delivery.** If your WebSocket connection drops for 10 seconds, how many events did you miss? With standard `eth_subscribe`, the answer is: you don't know. You have to re-scan blocks to find out.

The Monad Execution Events Gateway solves all four problems.

---

## What Is the Gateway?

The **Monad Execution Events Gateway** is a real-time WebSocket streaming service that delivers execution-level events directly from a Monad full node.

Instead of polling for state changes, you open a WebSocket connection, send a subscription message, and receive a continuous push stream of events as they happen. Every event is tagged with a monotonic sequence number (`server_seqno`) and the block's current consensus stage.

The Gateway exposes event types that are simply not available through JSON-RPC:

- **Block lifecycle events** вЂ” when a block starts executing, receives a quorum certificate, gets finalized, or is rejected
- **Transaction internals** вЂ” full transaction headers, EVM output (gas used, status), internal call frames
- **Contract events (logs)** вЂ” the same `emit` events you'd get from `eth_getLogs`, but streamed in real time
- **Synthetic events** вЂ” like `NativeTransfer`, which surfaces ETH transfers from internal calls
- **Network metrics** вЂ” transactions per second (TPS), per-block contention analysis, top accessed accounts and storage slots

In total, the Gateway exposes **25+ event types** across blocks, transactions, contract logs, and network metrics.

---

## Key Capabilities

### Flexible Subscriptions

You choose exactly what you want to receive. At its simplest:

```json
{"subscribe": ["BlockStart", "BlockEnd", "TPS"]}
```

This subscribes you to block boundaries and TPS metrics вЂ” nothing else.

For more control, use the advanced format with field-level filters:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
        {"field": "topics", "filter": {"values": ["0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"]}}
      ]
    }]
  }
}
```

This subscribes to ERC-20 Transfer events from a specific contract вЂ” filtered server-side, so you only receive what you need.

**Available filters:**

| Field | Applies to | Description |
|-------|-----------|-------------|
| `address` | TxnLog | Emitting contract address |
| `topics` | TxnLog | Event signature + indexed params (prefix match) |
| `txn_index` | TxnLog | Transaction index range in block |
| `log_index` | TxnLog | Log index range within transaction |
| `sender` | TxnHeaderStart | Transaction sender address |
| `to` | TxnHeaderStart | Transaction recipient address |
| `function_selector` | TxnHeaderStart | First 4 bytes of calldata |

### Finality-Aware Streaming

Every event carries the block's consensus stage. You can gate delivery with `min_stage`:

```json
{
  "subscribe": {
    "events": ["TxnLog", "BlockFinalized"],
    "min_stage": "Finalized"
  }
}
```

This ensures you only receive events from blocks that have been irreversibly committed by MonadBFT consensus. No reorgs, no rollbacks.

Four stages are available:

| Stage | Meaning | Typical Latency |
|-------|---------|-----------------|
| `Proposed` | Block executed, not yet voted | ~0ms (default) |
| `Voted` | Quorum certificate from 2/3+ validators | ~400ms |
| `Finalized` | Irreversibly committed | ~800ms |
| `Verified` | State root verified | Terminal |

### Transaction Correlation

When you subscribe to `TxnHeaderStart` with `correlate: true`, matching transactions automatically deliver all their child events вЂ” logs, call frames, EVM output, and transaction end вЂ” without requiring separate subscriptions:

```json
{
  "subscribe": {
    "events": ["TxnHeaderStart"],
    "filters": [{
      "event_name": "TxnHeaderStart",
      "field_filters": [
        {"field": "sender", "filter": {"values": ["0xYOUR_ADDRESS"]}}
      ]
    }],
    "correlate": true
  }
}
```

### Cursor Resume

Every message from the Gateway carries a `server_seqno` вЂ” a monotonically increasing sequence number. If your connection drops, reconnect with:

```
wss://gateway.example.com/ws?resume_from=12345
```

The server replays all buffered messages with `server_seqno > 12345`, then seamlessly transitions to live streaming. No gaps, no duplicates.

### Hello Handshake

On connect, the server sends a `Hello` message with protocol metadata:

```json
{
  "server_seqno": 0,
  "Hello": {
    "wire_version": 1,
    "server_version": "0.1.0",
    "chain_id": 10143,
    "block_number": 59419670,
    "available_filters": ["txn_index", "log_index", "address", "topics", "sender", "to", "function_selector"],
    "blocked_events": ["AccountAccess", "StorageAccess"],
    "limits": {
      "backfill_events": 100000,
      "buffer_per_client": 4096,
      "slow_off_limit": 10000,
      "heartbeat_interval": 30,
      "heartbeat_timeout": 60,
      "max_subscribes": 5
    }
  }
}
```

This tells you everything you need: which chain you're connected to, what filters are available, which events are blocked, and the server's operational limits.

---

## Who Is This For?

### Analytics & Monitoring Platforms

Stream block and transaction events to build real-time dashboards вЂ” TPS charts, block timing, gas usage, contention heatmaps. The `TPS`, `ContentionData`, and `Lifecycle` subscriptions provide pre-computed metrics without any client-side aggregation.

### Bridge & Relayer Infrastructure

Subscribe with `min_stage: "Finalized"` to guarantee that cross-chain messages are only relayed for irreversibly committed blocks. The `Lifecycle` subscription lets you track exactly when each block transitions through consensus stages.

### Indexers & Data Pipelines

Replace `eth_getLogs` polling with a push-based stream. Field filters ensure you only receive events for the contracts and topics you care about. Cursor resume guarantees lossless delivery across reconnections.

### Wallets & Exchanges

Track transaction finality in real time. Know exactly when a deposit goes from "proposed" to "finalized" to "verified" вЂ” and surface that information to users.

### Protocol Developers

Debug smart contracts with full execution visibility: internal call frames, storage access patterns, and per-transaction gas breakdown. The `ContentionData` metric reveals which storage slots cause parallel execution bottlenecks.

---

## Quick Start

### Using wscat

Connect and subscribe to block events:

```bash
# Install wscat
npm install -g wscat

# Connect
wscat -c wss://gateway.example.com/ws

# Send subscription
> {"subscribe": ["BlockStart", "BlockEnd", "TPS"]}
```

You'll immediately start receiving events:

```json
{"server_seqno": 1, "Events": [{"event_name": "BlockStart", "block_number": 59419671, "block_id": "0x...", "commit_stage": "Proposed"}]}
{"server_seqno": 2, "TPS": 1234}
{"server_seqno": 3, "Events": [{"event_name": "BlockEnd", "block_number": 59419671, "block_id": "0x...", "commit_stage": "Proposed"}]}
```

### Using the TypeScript SDK

```typescript
import { MonadEventsClient } from 'monad-execution-events';

const client = new MonadEventsClient({
  url: 'wss://gateway.example.com/ws',
});

client.subscribe({
  events: ['TxnLog'],
  filters: [{
    event_name: 'TxnLog',
    field_filters: [
      { field: 'address', filter: { values: ['0x0000000000000000000000000000000000001000'] } }
    ]
  }]
});

client.on('event', (event) => {
  console.log(`${event.event_name} in block ${event.block_number}`);
});
```

The SDK handles reconnection, cursor resume, and heartbeat detection automatically.

### Using the Python SDK

```python
from monad_execution_events import MonadEventsClient

async with MonadEventsClient("wss://gateway.example.com/ws") as client:
    await client.subscribe(events=["BlockStart", "BlockEnd", "TPS"])

    async for event in client:
        print(f"{event['event_name']} вЂ” block {event.get('block_number', 'N/A')}")
```

---

## What's Next?

- **[Finality-Aware Event Streaming: How `min_stage` Works](02-finality-aware-event-streaming.md)** вЂ” Deep dive into consensus stages and how to build finality-aware applications.
- **[Real-Time Staking Analytics on Monad](03-real-time-staking-analytics.md)** вЂ” Practical guide to tracking Delegate, Undelegate, and validator rewards using the Gateway.

---

*The Monad Execution Events Gateway is open source. Check the [documentation](../API.md) for the full API reference, or explore the [TypeScript](https://www.npmjs.com/package/monad-execution-events) and [Python](https://pypi.org/project/monad-execution-events/) SDKs to get started.*
