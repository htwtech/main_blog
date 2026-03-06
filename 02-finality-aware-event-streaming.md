---
title: "Finality-Aware Event Streaming: How min_stage Works"
slug: "finality-aware-event-streaming-min-stage"
date: "2026-03-01"
description: "Stop processing events that might get rolled back. Use consensus stage gating to receive only finalized events."
author: "HighTower"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://pbs.twimg.com/media/HCwJicRWkAABAI8.jpg"
coverAlt: "Monad Finality-Aware Streaming"
tags: ["monad", "streaming", "finality", "infrastructure"]
---

# Finality-Aware Event Streaming: How `min_stage` Works

*Stop processing events that might get rolled back. Use consensus stage gating to receive only finalized вЂ” or even speculative вЂ” events, depending on your use case.*

---

## The Finality Problem

On most EVM chains, events arrive in one of two states: confirmed or unconfirmed. You subscribe to new logs, process them, and hope the block doesn't get reorganized.

When a reorg happens, your application has already acted on invalid data. A bridge may have minted tokens for a transfer that no longer exists. An indexer may have stored transaction data that was rolled back. A dashboard may have shown a block that was replaced.

The typical workaround is to wait for N confirmations before processing events. But this introduces latency and still doesn't give you a guarantee вЂ” it just makes reorgs less likely.

Monad approaches this differently. MonadBFT is a single-slot finality consensus protocol вЂ” once a block is finalized, it's irreversible. There are no reorgs after finalization. And the Monad Execution Events Gateway lets you tap into this directly.

---

## MonadBFT Consensus Stages

Every block on Monad passes through a sequence of consensus stages. The Gateway tracks these stages and exposes them on every event via the `commit_stage` field.

### Proposed

```
Latency from execution: ~0ms
```

The block has been executed by the proposer. Transactions have run, logs have been emitted, state has been updated вЂ” but no other validators have weighed in yet.

This is the earliest possible point at which events are available. Applications that need maximum speed (and can handle the risk of a block not being finalized) can consume events at this stage.

**Risk:** The block may not achieve consensus. In practice this is rare, but it can happen if the proposer is faulty or the network is partitioned.

### Voted (QC)

```
Latency from execution: ~400ms
```

The block has received a Quorum Certificate (QC) вЂ” signatures from 2/3+ of validators confirming they've seen and validated the block. This is a strong signal that the block will be finalized, but it's not yet irreversible.

The `BlockQC` event fires at this stage:

```json
{
  "event_name": "BlockQC",
  "block_number": 59419670,
  "block_id": "0xabc...",
  "commit_stage": "Voted"
}
```

### Finalized

```
Latency from execution: ~800ms
```

The block is **irreversibly committed** to the chain. No future consensus round can undo it. This is the safety boundary вЂ” once a block reaches `Finalized`, its events are guaranteed to be permanent.

The `BlockFinalized` event fires at this stage:

```json
{
  "event_name": "BlockFinalized",
  "block_number": 59419670,
  "block_id": "0xabc...",
  "commit_stage": "Finalized"
}
```

For most applications that need correctness guarantees, this is the right stage to use.

### Verified

```
Terminal stage
```

The state root has been verified вЂ” the block's execution output matches the committed state. This is the terminal stage in the block's lifecycle.

---

## How `min_stage` Works

The `min_stage` parameter in your subscription controls when events are delivered to you. It acts as a gate: events from blocks that haven't reached your specified stage are silently held back.

### Without `min_stage` (default)

By default, events are delivered as soon as they're available вЂ” at the `Proposed` stage. You get maximum speed but no finality guarantee:

```json
{"subscribe": ["TxnLog", "BlockStart", "BlockEnd"]}
```

Events arrive within milliseconds of execution.

### With `min_stage: "Finalized"`

Events are only delivered after the block has been irreversibly committed:

```json
{
  "subscribe": {
    "events": ["TxnLog", "BlockStart", "BlockEnd"],
    "min_stage": "Finalized"
  }
}
```

This adds ~800ms of latency compared to `Proposed`, but guarantees that every event you receive is permanent. The events themselves are identical вЂ” same content, same ordering вЂ” they're just delivered later.

### With `min_stage: "Voted"`

A middle ground вЂ” events are delivered after the block receives its QC:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "min_stage": "Voted"
  }
}
```

~400ms latency, with high confidence (but not absolute guarantee) that the block will be finalized.

---

## Choosing the Right Stage

Different applications have different requirements. Here's a practical guide:

### Bridge / Relayer: `Finalized`

Cross-chain bridges must never relay a transaction that gets rolled back. A false transfer could lead to double-spending across chains.

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0xBRIDGE_CONTRACT"]}},
        {"field": "topics", "filter": {"values": ["0xBRIDGE_DEPOSIT_TOPIC"]}}
      ]
    }],
    "min_stage": "Finalized"
  }
}
```

With MonadBFT's ~800ms finality, this is dramatically faster than waiting for 12+ confirmations on other chains.

### Exchange Deposit Tracking: `Finalized`

Exchanges need certainty before crediting user accounts:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0xTOKEN_CONTRACT"]}},
        {"field": "topics", "filter": {"values": ["0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef"]}}
      ]
    }],
    "min_stage": "Finalized"
  }
}
```

This filters for ERC-20 Transfer events on a specific token, delivered only after finalization.

### Analytics Dashboard: `Voted`

Real-time dashboards benefit from lower latency. A block that has a QC is very likely to be finalized, and the small risk of showing a non-finalized block is acceptable for analytics:

```json
{
  "subscribe": {
    "events": ["BlockStart", "BlockEnd", "TPS"],
    "min_stage": "Voted"
  }
}
```

### Speculative Processing: No `min_stage`

Applications that want to act as early as possible вЂ” and can handle rollbacks вЂ” should omit `min_stage` entirely:

```json
{"subscribe": ["TxnLog", "TxnHeaderStart", "TxnEnd"]}
```

This is useful for:
- Pre-confirmation UIs ("your transaction is being processed")
- Speculative indexing (index immediately, mark as unconfirmed)
- Monitoring and alerting (detect anomalies as early as possible)

---

## Tracking Block Lifecycle with `Lifecycle`

If you need to know exactly when blocks transition between stages, subscribe to the `Lifecycle` metric:

```json
{"subscribe": ["Lifecycle"]}
```

You'll receive a message every time a block changes stage:

```json
{
  "server_seqno": 42,
  "Lifecycle": {
    "block_number": 59419670,
    "block_id": "0xabc...",
    "stage": "Finalized",
    "previous_stage": "Voted",
    "time_in_previous_stage_ms": 420,
    "block_age_ms": 850
  }
}
```

This tells you:
- **Which block** changed stage
- **From which stage** and **to which stage**
- **How long** it spent in the previous stage
- **Total age** of the block since it was first proposed

### Building a Finality Monitor

Combine `Lifecycle` with `BlockStart` to build a real-time finality monitor:

```typescript
import { MonadEventsClient } from 'monad-execution-events';

const client = new MonadEventsClient({
  url: 'wss://gateway.example.com/ws',
});

client.subscribe(['BlockStart', 'Lifecycle']);

const blocks = new Map<number, { proposed_at: number }>();

client.on('event', (event) => {
  if (event.event_name === 'BlockStart') {
    blocks.set(event.block_number, { proposed_at: Date.now() });
  }
});

client.on('lifecycle', (lc) => {
  if (lc.stage === 'Finalized') {
    const block = blocks.get(lc.block_number);
    if (block) {
      const finality_ms = Date.now() - block.proposed_at;
      console.log(`Block ${lc.block_number} finalized in ${finality_ms}ms`);
      blocks.delete(lc.block_number);
    }
  }
});
```

### Combining `Lifecycle` with `min_stage`

You can subscribe to both вЂ” use `min_stage` to gate your business-critical events, and `Lifecycle` to observe the consensus pipeline:

```json
{
  "subscribe": {
    "events": ["TxnLog", "Lifecycle"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0xYOUR_CONTRACT"]}}
      ]
    }],
    "min_stage": "Finalized"
  }
}
```

Note: `Lifecycle` events are metric-type items and are not subject to `min_stage` gating вЂ” they always arrive in real time.

---

## Comparison with Other Chains

| | Monad (Gateway) | Ethereum | Optimism / Arbitrum |
|--|-----------------|----------|---------------------|
| **Finality type** | Single-slot (MonadBFT) | Probabilistic в†’ Casper FFG | L2 soft confirm в†’ L1 finality |
| **Time to finality** | ~800ms | ~15 minutes | 7+ days (challenge period) |
| **Reorg risk after finalization** | None | None (after FFG) | None (after L1) |
| **Finality-aware streaming** | `min_stage` parameter | Not available | Not available |
| **Stage granularity** | 4 stages | 2 states | 2-3 states |

The key advantage is not just speed вЂ” it's granularity. You can choose your risk/latency tradeoff per subscription, and the Gateway enforces it server-side.

---

## What's Next?

- **[What Is Monad Execution Events Gateway?](01-intro-what-is-monad-execution-gateway.md)** вЂ” Overview of the Gateway, its capabilities, and quick start guide.
- **[Real-Time Staking Analytics on Monad](03-real-time-staking-analytics.md)** вЂ” Practical guide to tracking staking events with field filters and function selectors.

---

*Learn more in the [full API reference](../API.md) or the [Subscriptions & Filters Reference](../SUBSCRIPTIONS.md). Get started with the [TypeScript SDK](https://www.npmjs.com/package/monad-execution-events) or [Python SDK](https://pypi.org/project/monad-execution-events/).*
