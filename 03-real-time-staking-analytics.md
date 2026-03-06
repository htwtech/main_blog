---
title: "Real-Time Staking Analytics on Monad"
slug: "real-time-staking-analytics-monad"
date: "2026-03-08"
description: "Build a real-time view of Monad staking activity — delegate flows, reward distributions, and validator performance."
author: "HighTower"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
coverAlt: "Monad Staking Analytics"
tags: ["monad", "staking", "analytics", "gateway"]
---

# Real-Time Staking Analytics on Monad: Tracking Delegate, Undelegate & Rewards

*Build a real-time view of Monad staking activity вЂ” delegate flows, reward distributions, and validator performance вЂ” using the Execution Events Gateway.*

---

## Monad Staking Overview

Monad uses a native staking precompile at address `0x0000000000000000000000000000000000001000`. All staking operations вЂ” delegating, undelegating, withdrawing, compounding rewards вЂ” go through this contract and emit standard EVM events.

These events are the foundation for staking analytics: who delegated, how much, to which validator, and when rewards were distributed.

The traditional approach is polling `eth_getLogs` to fetch these events. This works, but it's slow (seconds of latency), wasteful (constant polling even when nothing happens), and limited (no way to distinguish between `delegate()` and `compound()` calls, since both emit the same `Delegate` event).

The Execution Events Gateway solves all three problems with real-time push-based streaming, server-side filtering, and function selector awareness.

---

## Staking Event Signatures

The staking precompile emits the following events. Each event's `topic[0]` is the keccak256 hash of its signature вЂ” this is what you use to filter in your subscription.

| Event | Signature | topic[0] |
|-------|-----------|----------|
| **Delegate** | `Delegate(uint64,address,uint256,uint64)` | `0xe4d4df1e1827dd28252fd5c3cd7ebccd3da6e0aa31f74c828f3c8542af49d840` |
| **Undelegate** | `Undelegate(uint64,address,uint8,uint256,uint64)` | `0x3e53c8b91747e1b72a44894db10f2a45fa632b161fdcdd3a17bd6be5482bac62` |
| **Withdraw** | `Withdraw(uint64,address,uint8,uint256,uint64)` | `0x63030e4238e1146c63f38f4ac81b2b23c8be28882e68b03f0887e50d0e9bb18f` |
| **ClaimRewards** | `ClaimRewards(uint64,address,uint256,uint64)` | `0xcb607e6b63c89c95f6ae24ece9fe0e38a7971aa5ed956254f1df47490921727b` |
| **ValidatorRewarded** | `ValidatorRewarded(uint64,address,uint256,uint64)` | `0x3a420a01486b6b28d6ae89c51f5c3bde3e0e74eecbb646a0c481ccba3aae3754` |
| **ValidatorCreated** | `ValidatorCreated(uint64,address,uint256)` | `0x6f8045cd38e512b8f12f6f02947c632e5f25af03aad132890ecf50015d97c1b2` |
| **CommissionChanged** | `CommissionChanged(uint64,uint256,uint256)` | `0xd1698d3454c5b5384b70aaae33f1704af7c7e055f0c75503ba3146dc28995920` |
| **ValidatorStatusChanged** | `ValidatorStatusChanged(uint64,uint64)` | `0xc95966754e882e03faffaf164883d98986dda088d09471a35f9e55363daf0c53` |
| **EpochChanged** | `EpochChanged(uint64,uint64)` | `0x4fae4dbe0ed659e8ce6637e3c273cd8e4d3bf029b9379a9e8b3f3f27dbef809b` |

---

## Subscribing to Staking Events

### All events from the staking precompile

The simplest approach вЂ” get everything:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}}
      ]
    }]
  }
}
```

This delivers every log emitted by the staking contract: delegates, undelegates, rewards, epoch changes вЂ” all of it.

### Delegate + Undelegate only

Filter by both address and event signature (topic[0]):

```json
{"subscribe":{"events":["TxnLog"],"filters":[{"event_name":"TxnLog","field_filters":[{"field":"address","filter":{"values":["0x0000000000000000000000000000000000001000"]}},{"field":"topics","filter":{"values":["0xe4d4df1e1827dd28252fd5c3cd7ebccd3da6e0aa31f74c828f3c8542af49d840"]}}]},{"event_name":"TxnLog","field_filters":[{"field":"address","filter":{"values":["0x0000000000000000000000000000000000001000"]}},{"field":"topics","filter":{"values":["0x3e53c8b91747e1b72a44894db10f2a45fa632b161fdcdd3a17bd6be5482bac62"]}}]}]}}
```

The two filter specs use OR logic вЂ” you receive events that match *either* the Delegate topic or the Undelegate topic.

### Reward distribution tracking

Track `ValidatorRewarded` and `ClaimRewards`:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [
      {
        "event_name": "TxnLog",
        "field_filters": [
          {"field": "address", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
          {"field": "topics", "filter": {"values": ["0x3a420a01486b6b28d6ae89c51f5c3bde3e0e74eecbb646a0c481ccba3aae3754"]}}
        ]
      },
      {
        "event_name": "TxnLog",
        "field_filters": [
          {"field": "address", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
          {"field": "topics", "filter": {"values": ["0xcb607e6b63c89c95f6ae24ece9fe0e38a7971aa5ed956254f1df47490921727b"]}}
        ]
      }
    ]
  }
}
```

---

## The delegate() vs compound() Problem

Here's a subtlety that trips up many staking analytics implementations.

The staking precompile has two functions that both emit a `Delegate` event:

| Function | Selector | What it does |
|----------|----------|--------------|
| `delegate(uint64)` | `0x84994fec` | User explicitly delegates tokens |
| `compound(uint64)` | `0xb34fea67` | Reinvests accumulated rewards as new stake |

Both call the internal delegation logic, so both emit `Delegate(uint64, address, uint256, uint64)`. If you're just filtering by the Delegate event topic, you can't tell them apart.

**Does this matter?** It depends on what you're measuring:

- **Total delegated stake:** Both are correct. `compound()` genuinely increases the validator's stake вЂ” it takes pending rewards and re-delegates them. So counting both gives you the accurate total.

- **New capital inflows:** Here you only want `delegate()`. Compound is recycled capital, not new money entering the system.

- **Delegation activity by function:** You want to distinguish them for UX or analytics purposes.

### Solution: `function_selector` filter

The Gateway's `function_selector` filter matches the first 4 bytes of transaction calldata, letting you separate these cases:

**Only direct delegate() calls:**

```json
{
  "subscribe": {
    "events": ["TxnHeaderStart"],
    "filters": [{
      "event_name": "TxnHeaderStart",
      "field_filters": [
        {"field": "to", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
        {"field": "function_selector", "filter": {"values": ["0x84994fec"]}}
      ]
    }],
    "correlate": true
  }
}
```

With `correlate: true`, this delivers:
1. The `TxnHeaderStart` for the matching transaction
2. All `TxnLog` events emitted by that transaction (including the `Delegate` event)
3. `TxnEvmOutput` (gas used, success status)
4. `TxnEnd`

You get the full transaction context вЂ” not just the log.

**Only compound() calls:**

```json
{
  "subscribe": {
    "events": ["TxnHeaderStart"],
    "filters": [{
      "event_name": "TxnHeaderStart",
      "field_filters": [
        {"field": "to", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
        {"field": "function_selector", "filter": {"values": ["0xb34fea67"]}}
      ]
    }],
    "correlate": true
  }
}
```

**Both, but distinguishable:**

```json
{
  "subscribe": {
    "events": ["TxnHeaderStart"],
    "filters": [{
      "event_name": "TxnHeaderStart",
      "field_filters": [
        {"field": "to", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}},
        {"field": "function_selector", "filter": {"values": ["0x84994fec", "0xb34fea67"]}}
      ]
    }],
    "correlate": true
  }
}
```

This delivers both types, and you can distinguish them by checking the `data` field in `TxnHeaderStart` вЂ” the first 4 bytes tell you which function was called.

### All staking function selectors

| Function | Selector |
|----------|----------|
| `delegate(uint64)` | `0x84994fec` |
| `undelegate(uint64,uint256,uint8)` | `0x5cf41514` |
| `withdraw(uint64,uint8)` | `0xaed2ee73` |
| `compound(uint64)` | `0xb34fea67` |
| `claimRewards(uint64)` | `0xa76e2ca5` |

---

## Decoding Event Data

### Delegate event layout

`Delegate(uint64 validatorId, address delegator, uint256 amount, uint64 epoch)`

The event data comes as ABI-encoded bytes. The `amount` is a `uint256` in wei вЂ” divide by `10^18` to get MON:

```typescript
import { decodeAbiParameters } from 'viem';

// From TxnLog event
const data = event.data; // hex string
const topics = event.topics; // array of hex strings

// topic[0] is the event signature (already matched by filter)
// Remaining fields are in data (non-indexed)
const decoded = decodeAbiParameters(
  [
    { name: 'validatorId', type: 'uint64' },
    { name: 'delegator', type: 'address' },
    { name: 'amount', type: 'uint256' },
    { name: 'epoch', type: 'uint64' },
  ],
  data
);

const amountMON = Number(decoded.amount) / 1e18;
console.log(`Delegated ${amountMON} MON to validator ${decoded.validatorId} by ${decoded.delegator}`);
```

### ValidatorRewarded event layout

`ValidatorRewarded(uint64 validatorId, address validator, uint256 amount, uint64 epoch)`

Same structure as Delegate. The `amount` is the reward distributed to the validator for producing a block:

```typescript
// Typical: ~20 MON per block reward
const rewardMON = Number(decoded.amount) / 1e18;
console.log(`Validator ${decoded.validatorId} earned ${rewardMON} MON in epoch ${decoded.epoch}`);
```

---

## Building a Staking Dashboard

With real-time staking events streaming from the Gateway, you can build a live dashboard that shows:

### 1. Delegation Volume

Track net delegation flow per block or per time window:

```typescript
let totalDelegated = 0n;
let totalUndelegated = 0n;

client.on('event', (event) => {
  if (event.event_name !== 'TxnLog') return;

  const topic0 = event.topics[0];

  if (topic0 === '0xe4d4df...') { // Delegate
    const amount = BigInt(event.data.slice(130, 194)); // uint256 amount offset
    totalDelegated += amount;
  }

  if (topic0 === '0x3e53c8...') { // Undelegate
    const amount = BigInt(event.data.slice(130, 194));
    totalUndelegated += amount;
  }

  updateDashboard({
    netFlow: totalDelegated - totalUndelegated,
    delegated: totalDelegated,
    undelegated: totalUndelegated,
  });
});
```

### 2. Reward Rate

Track `ValidatorRewarded` events to compute average reward per block:

```typescript
const rewards: number[] = [];

client.on('event', (event) => {
  if (event.topics[0] === '0x3a420a...') { // ValidatorRewarded
    const amount = Number(BigInt('0x' + event.data.slice(130, 194))) / 1e18;
    rewards.push(amount);

    // Rolling average over last 100 blocks
    if (rewards.length > 100) rewards.shift();
    const avgReward = rewards.reduce((a, b) => a + b, 0) / rewards.length;

    console.log(`Avg reward: ${avgReward.toFixed(2)} MON/block`);
  }
});
```

### 3. Top Validators by Delegation

Aggregate `Delegate` events by `validatorId` to find the most popular validators:

```typescript
const validatorStake = new Map<number, bigint>();

client.on('event', (event) => {
  if (event.topics[0] === '0xe4d4df...') { // Delegate
    const decoded = decodeDelegate(event.data);
    const current = validatorStake.get(decoded.validatorId) ?? 0n;
    validatorStake.set(decoded.validatorId, current + decoded.amount);
  }
});
```

### 4. Finality-Safe Analytics

For dashboards that must be accurate (not speculative), add `min_stage`:

```json
{
  "subscribe": {
    "events": ["TxnLog"],
    "filters": [{
      "event_name": "TxnLog",
      "field_filters": [
        {"field": "address", "filter": {"values": ["0x0000000000000000000000000000000000001000"]}}
      ]
    }],
    "min_stage": "Finalized"
  }
}
```

This ensures your analytics only count events from finalized blocks вЂ” no risk of counting a delegation that gets rolled back.

---

## Comparison: Gateway vs eth_getLogs

| | `eth_getLogs` polling | Gateway streaming |
|--|----------------------|-------------------|
| **Latency** | 1-5 seconds (polling interval) | <5ms (push) |
| **Bandwidth** | Constant polling overhead | Zero when idle |
| **Function selector filter** | Not available | `function_selector` field |
| **Finality awareness** | Manual block confirmation counting | `min_stage` parameter |
| **Lossless delivery** | Manual gap detection | Cursor resume (`server_seqno`) |
| **Transaction correlation** | Separate RPC calls per tx | `correlate: true` in one subscription |

For staking analytics specifically, the `function_selector` filter is the differentiator вЂ” it's the only way to reliably distinguish `delegate()` from `compound()` without parsing calldata client-side.

---

## What's Next?

- **[What Is Monad Execution Events Gateway?](01-intro-what-is-monad-execution-gateway.md)** вЂ” Overview of the Gateway and quick start guide.
- **[Finality-Aware Event Streaming](02-finality-aware-event-streaming.md)** вЂ” Deep dive into `min_stage` and MonadBFT consensus stages.

---

*See the [Subscriptions & Filters Reference](../SUBSCRIPTIONS.md) for the complete list of event types and filter options. Get started with the [TypeScript SDK](https://www.npmjs.com/package/monad-execution-events) or [Python SDK](https://pypi.org/project/monad-execution-events/).*
