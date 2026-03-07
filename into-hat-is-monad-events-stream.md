---
title: "What Is Monad Events Stream?"
slug: "what-is-monad-events-stream"
date: "2026-03-06"
description: "Real-time execution events streaming for the Monad blockchain — beyond what JSON-RPC can offer."
author: "HighTower"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://pbs.twimg.com/media/HCwJRS-XwAAdt3J.jpg"
coverAlt: "Monad Events Stream"
tags: ["monad", "rpc", "infrastructure", "websocket"]
---
# Monad Events Stream: Real-Time WebSocket Streaming for Monad

*Stream execution events over WebSocket without running your own node. What JSON-RPC can't provide: EVM internals, finality stages, lossless delivery with cursor resume.*

---

## The Problem with JSON-RPC

If you're building on Monad today, chances are you rely on JSON-RPC to read blockchain state. You call `eth_getBlockByNumber`, `eth_getLogs`, or `eth_getTransactionReceipt`, and you get back a snapshot of what happened after the fact.

This works fine for simple use cases. But it breaks down fast when you need:

- **Low-latency event delivery.** Polling `eth_getLogs` every few seconds adds hundreds of milliseconds to seconds of delay. For time-sensitive applications (liquidation bots, bridge relayers, real-time dashboards) that delay is unacceptable.
- **Execution-level visibility.** JSON-RPC gives you the *result* of execution: logs, receipts, state diffs. It doesn't show you *how* execution happened: which internal calls were made, which storage slots were touched, how the EVM stepped through the transaction.
- **Finality awareness.** Standard RPC treats blocks as either confirmed or not. But Monad's MonadBFT consensus has multiple stages: a block can be proposed, voted on, finalized, and verified. Applications that care about finality (bridges, exchanges) need to know *which stage* a block has reached.
- **Lossless delivery.** If your WebSocket connection drops for 10 seconds, how many events did you miss? With standard `eth_subscribe`, the answer is: you don't know. You have to re-scan blocks to find out.

The Monad Events Stream solves all four problems.

---

## What Is the Stream? Real-Time WebSocket API for Monad Events

The **Monad Events Stream** is a hosted WebSocket streaming service that delivers execution-level events from the Monad blockchain. You connect to an endpoint and start receiving events in real time, no node required.

Instead of polling for state changes, you open a WebSocket connection, send a subscription message, and receive a continuous push stream of events as they happen. Every event is tagged with a monotonic sequence number (`seq`) and the block's current consensus stage.

Under the hood, the service runs alongside a Monad full node and reads events from its memory in real time. Execution events are written once at block execution; consensus stage transitions (`BLOCK_QC`, `BLOCK_FINALIZED`, `BLOCK_VERIFIED`) are written as separate records when each stage is reached. All of this is handled server-side. As a client, you just connect via WebSocket and subscribe.

The Stream exposes event types that are simply not available through JSON-RPC:

- **Block lifecycle events**: when a block starts executing, receives a quorum certificate, gets finalized, or is rejected
- **Transaction internals**: full transaction headers, EVM output (gas used, status), internal call frames
- **Contract events (logs)**: the same `emit` events you'd get from `eth_getLogs`, but streamed in real time over WebSocket
- **Synthetic events**: like `NativeTransfer`, which surfaces ETH transfers from internal calls
- **Network metrics**: transactions per second (TPS), per-block contention analysis, top accessed accounts and storage slots

The Stream exposes **25+ event types** across blocks, transactions, contract logs, and network metrics.

---

## Key Capabilities of the Monad WebSocket API

### Flexible Subscriptions

You choose exactly what you want to receive. At its simplest:

```json
{"subscribe": ["BlockStart", "BlockEnd", "TPS"]}
```

This subscribes you to block boundaries and TPS metrics, nothing else.

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

This subscribes to ERC-20 Transfer events from a specific contract — filtered server-side, so you only receive what you need.

**Available filters:**


| Field               | Applies to     | Description                                     |
| --------------------- | ---------------- | ------------------------------------------------- |
| `address`           | TxnLog         | Emitting contract address                       |
| `topics`            | TxnLog         | Event signature + indexed params (prefix match) |
| `txn_index`         | TxnLog         | Transaction index range in block                |
| `log_index`         | TxnLog         | Log index range within transaction              |
| `sender`            | TxnHeaderStart | Transaction sender address                      |
| `to`                | TxnHeaderStart | Transaction recipient address                   |
| `function_selector` | TxnHeaderStart | First 4 bytes of calldata                       |

### Finality-Aware Streaming

Execution events are delivered **once**, at the moment the block is executed (`commit_stage: "Proposed"`). They are not re-sent when the block progresses to later consensus stages.

To confirm finality, subscribe to `Lifecycle` events alongside your data events. When a `Lifecycle` event arrives with `to_stage: "Finalized"` for a given block, all events from that block are guaranteed to be permanent:

```json
{
  "subscribe": {
    "events": ["TxnLog", "Lifecycle"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0xYOUR_CONTRACT"]}}
      ]
    }]
  }
}
```

Four consensus stages are available:


| Stage       | Meaning                                 | Typical Latency |
| ------------- | ----------------------------------------- | ----------------- |
| `Proposed`  | Block executed, not yet voted           | ~0ms (default)  |
| `Voted`     | Quorum certificate from 2/3+ validators | ~400ms          |
| `Finalized` | Irreversibly committed                  | ~800ms          |
| `Verified`  | State root verified                     | Terminal        |

### Transaction Correlation

When you subscribe to `TxnHeaderStart` with `correlate: true`, matching transactions automatically deliver all their child events (logs, call frames, EVM output, and transaction end) without requiring separate subscriptions:

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

### Hello Handshake

On connect, the server sends a `Hello` message with protocol metadata:

```json
{
  "seq": 0,
  "Hello": {
    "protocol_version": 1,
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

Stream block and transaction events to build real-time dashboards: TPS charts, block timing, gas usage, contention heatmaps. The `TPS`, `ContentionData`, and `Lifecycle` subscriptions provide pre-computed metrics without any client-side aggregation.

### Bridge & Relayer Infrastructure

Subscribe to bridge events alongside `Lifecycle`. Buffer events on arrival, and only relay cross-chain messages after a `Lifecycle` event confirms the block has reached `Finalized`. The `Lifecycle` subscription lets you track exactly when each block transitions through consensus stages.

### Indexers & Data Pipelines

Replace `eth_getLogs` polling with a push-based stream. Field filters ensure you only receive events for the contracts and topics you care about. Cursor resume guarantees lossless delivery across reconnections.

### Wallets & Exchanges

Track transaction finality in real time. Know exactly when a deposit goes from "proposed" to "finalized" to "verified" and surface that information to users.

### Protocol Developers

Debug smart contracts with full execution visibility: internal call frames, storage access patterns, and per-transaction gas breakdown. The `ContentionData` metric reveals which storage slots cause parallel execution bottlenecks.

---

## Monad Events Stream vs eth_subscribe


|                       | `eth_subscribe` (JSON-RPC)          | Monad Events Stream (WebSocket)                |
| ----------------------- | ------------------------------------- | ------------------------------------------------ |
| **Protocol**          | JSON-RPC over WebSocket             | Custom binary + JSON over WebSocket            |
| **Latency**           | ~1-5s (varies by provider)          | <5ms from block execution                      |
| **Event types**       | newHeads, logs, pendingTransactions | 25+ types including EVM internals              |
| **Reconnection**      | Manual, gap detection required      | Cursor resume with`seq` (100K-entry buffer)    |
| **Finality tracking** | Not available                       | 4 consensus stages via`Lifecycle` events       |
| **Filtering**         | address + topics only               | address, topics, sender, to, function_selector |
| **Own node required** | No (works with providers)           | No (hosted service)                            |

---

## Quick Start: Connect via WebSocket

### Using wscat

Connect and subscribe to block events:

```bash
# Install wscat
npm install -g wscat

# Connect
wscat -c wss://gm.monad.at.htw.tech/

# Send subscription
> {"subscribe": ["BlockStart", "BlockEnd", "TPS"]}
```

You'll immediately start receiving events:

```json
{"seq": 1, "Events": [{"event_name": "BlockStart", "block_number": 59419671, "block_id": "0x...", "commit_stage": "Proposed"}]}
{"seq": 2, "TPS": 1234}
{"seq": 3, "Events": [{"event_name": "BlockEnd", "block_number": 59419671, "block_id": "0x...", "commit_stage": "Proposed"}]}
```

### JavaScript (browser or Node.js)

```javascript
const ws = new WebSocket("wss://gm.monad.at.htw.tech/");

ws.onopen = () => {
  ws.send(JSON.stringify({
    subscribe: {
      events: ["TxnLog"],
      filters: [{
        event_name: "TxnLog",
        field_filters: [
          { field: "address", filter: { values: ["0x0000000000000000000000000000000000001000"] } }
        ]
      }]
    }
  }));
};

ws.onmessage = (msg) => {
  const data = JSON.parse(msg.data);
  if (data.Events) {
    for (const event of data.Events) {
      console.log(`${event.event_name} in block ${event.block_number}`);
    }
  }
};
```

### Python

```python
import asyncio, websockets, json

async def main():
    async with websockets.connect("wss://gm.monad.at.htw.tech/") as ws:
        await ws.send(json.dumps({"subscribe": ["BlockStart", "BlockEnd", "TPS"]}))
        async for msg in ws:
            data = json.loads(msg)
            if "Events" in data:
                for event in data["Events"]:
                    print(f"{event['event_name']} in block {event.get('block_number', 'N/A')}")

asyncio.run(main())
```

### SDKs (optional)

For production use, TypeScript and Python SDKs add auto-reconnect with exponential backoff, heartbeat detection, and typed events. The protocol is plain WebSocket + JSON, so no SDK is required to get started.

---

## Frequently Asked Questions

### How do I connect to the Monad Events Stream via WebSocket?

Open a WebSocket connection to `wss://gm.monad.at.htw.tech/` and send a JSON subscription message specifying which events you want to receive. The server immediately starts pushing matching events. Any WebSocket client works (browser `WebSocket`, `wscat`, Python `websockets`). No SDK or node setup required.

### What is the difference between the Events Stream and JSON-RPC?

JSON-RPC methods like `eth_getLogs` and `eth_subscribe` deliver event data through polling or basic WebSocket push, with seconds of latency. The Events Stream pushes events over WebSocket in under 5ms, with execution-level visibility (internal calls, storage access), consensus stage tracking, and lossless cursor resume that JSON-RPC does not provide.

### Does the WebSocket connection support reconnection without losing events?

Yes. Every message carries a monotonic sequence number (`seq`). On reconnect, pass `?resume_from=<seq>` to replay all buffered messages from that point. The server maintains a 100K-entry replay buffer. You can implement this in a few lines of code, or use the optional SDKs which handle it automatically.

### Do I need to run my own Monad node?

No. The Monad Events Stream is a hosted service. You connect to a WebSocket endpoint and start receiving events immediately. The infrastructure is managed for you.

---

