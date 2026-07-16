---
title: 'Hyperliquid WebSocket API: Subscriptions, Limits, and Production Setup'
slug: 'hyperliquid-websocket-api'
canonical_url: 'https://supanode.xyz/blog/hyperliquid-websocket-api'
date: "2026-07-16"
description: 'A practical breakdown of Hyperliquid WebSocket subscriptions, public endpoint limits, reconnect behavior, recovery patterns, and production infrastructure choices.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Hyperliquid%20WebSocket%20API_%20Subscriptions,%20limits,%20and%20production%20setup.png"
coverAlt: "Hyperliquid WebSocket API: Subscriptions, Limits, and Production Setup"
tags: ["Hyperliquid API", "Hyperliquid WebSocket", "Hyperliquid setup"]
---

The native Hyperliquid WebSocket API handles light, real-time workloads well: basic dashboards, single-market views, threshold alerts, and small multi-wallet balance tracking.

Architecture constraints start to matter once a product needs broader market coverage, more user-specific streams, stronger reconnect handling, or reliable recovery after disconnects. At that point, WebSocket limits move from documentation footnotes to real architecture decisions.

In this guide, you'll find the main Hyperliquid WebSocket subscriptions, the official limits, reconnect behavior, recovery patterns, and the point where a managed WebSocket, gRPC stream, or dedicated data endpoint is worth evaluating. 

## What Hyperliquid WebSocket is used for

The WebSocket API is the main path for products that need to react to Hyperliquid state in near real time rather than poll for it. In practice, that covers a fairly consistent set of workloads:

* price boards and market overviews pulling mid prices across many assets  
* order book viewers and depth visualizations  
* trade tapes and execution monitoring tools  
* candle-based charting and threshold alerts  
* account dashboards tracking real-time balances, open orders, leverage, and clearinghouse margins  
* portfolio and multi-wallet monitoring tools  
* market data pipelines feeding a warehouse or analytics layer

Most products in this category eventually need one of the native WebSocket subscriptions. Picking the right feed for each workload, and knowing where the public endpoint limits start pushing back, is where most of the actual engineering work happens. 

## Hyperliquid WebSocket endpoint and connection format

The [mainnet endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket) is `wss://api.hyperliquid.xyz/ws`. Testnet is `wss://api.hyperliquid-testnet.xyz/ws`. Both accept the same message shapes, which is convenient for testing subscription logic before pointing it at production data.

The connection lifecycle is simple. Open the socket, send a `subscribe` message with a `subscription` object describing what you want, and get back a `subscriptionResponse` acknowledgement. From there, channel-specific messages start arriving. Unsubscribing mirrors this exactly — same subscription object, `unsubscribe` method instead.

A minimal client skeleton that subscribes, sends heartbeats, reconnects, and detects initial snapshot messages looks like this:

```ts
import WebSocket from "ws";

const WS_URL = "wss://api.hyperliquid.xyz/ws";

const subscriptions = [
  { type: "allMids" },
  { type: "trades", coin: "BTC" },
];

let ws: WebSocket;
let lastInboundAt = Date.now();
let heartbeat: NodeJS.Timeout | undefined;
let reconnectAttempt = 0;

function send(payload: unknown) {
  if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify(payload));
  }
}

function subscribeAll() {
  for (const subscription of subscriptions) {
    send({ method: "subscribe", subscription });
  }
}

function connect() {
  ws = new WebSocket(WS_URL);

  ws.on("open", () => {
    lastInboundAt = Date.now();
    subscribeAll();

    // Only reset the attempt counter after the connection has held for a while.
    // Resetting immediately on open would let a connect-then-drop loop reconnect every second.
    setTimeout(() => {
      if (ws.readyState === WebSocket.OPEN) {
        reconnectAttempt = 0;
      }
    }, 10_000);
  });

  ws.on("message", (raw) => {
    lastInboundAt = Date.now();

    let msg: any;

    try {
      msg = JSON.parse(raw.toString());
    } catch (error) {
      console.error("Invalid WebSocket payload", error);
      return;
    }

    if (msg.channel === "subscriptionResponse") {
      console.log("subscribed:", msg.data.subscription);
      return;
    }

    if (msg.channel === "pong") return;

    if (msg.data?.isSnapshot) {
      console.log("snapshot:", msg.channel);
      // Rebuild local state or dedupe against already processed data.
      return;
    }

    console.log("live update:", msg.channel, msg.data);
  });

  ws.on("close", reconnect);

  ws.on("error", () => {
    ws.close();
  });

  if (heartbeat) clearInterval(heartbeat);

  heartbeat = setInterval(() => {
    const idleMs = Date.now() - lastInboundAt;

    if (idleMs > 45_000) {
      send({ method: "ping" });
    }
  }, 5_000);
}

function reconnect() {
  if (heartbeat) clearInterval(heartbeat);

  const baseDelay = Math.min(30_000, 1_000 * 2 ** reconnectAttempt);
  const jitter = Math.floor(Math.random() * 500);

  reconnectAttempt += 1;

  setTimeout(connect, baseDelay + jitter);
}

connect();
```

Time-series feeds such as userFills return a snapshot of prior data on the initial subscription ack, tagged with `isSnapshot: true`. If that data was already processed before a disconnect, the snapshot can be skipped instead of replayed into local state. Checking that flag matters most for account and fill views, where re-applying an old snapshot on top of current state is exactly what produces duplicated or inflated numbers. 

The [heartbeat behavior](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket/timeouts-and-heartbeats) matters for quiet channels too: Hyperliquid closes a connection if it has not sent anything for 60 seconds. Clients subscribed to low-activity feeds should send `{"method": "ping"}` and expect `{"channel": "pong"}` back. Reconnect logic should key off inbound silence rather than assumptions about server-side timing.

## Main Hyperliquid WebSocket subscription types

Hyperliquid documents over twenty subscription types, but most production workloads only ever touch a handful of them. A practical mapping:

| Use case | Subscription type |
| :---- | :---- |
| Mid prices across markets | `allMids` |
| User notifications | `notification` |
| User web/account data (aggregate) | `webData3` |
| TWAP order state | `twapStates` |
| Account clearinghouse state | `clearinghouseState` |
| Open orders | `openOrders` |
| Candles | `candle` |
| Order book | `l2Book` |
| Trades | `trades` |
| Order lifecycle events | `orderUpdates` |
| User fills (snapshot \+ streaming) | `userFills` |
| Non-order user events (funding, liquidations) | `userEvents` |

Use the [official subscriptions reference](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket/subscriptions) as the source of truth for exact type names and payload shapes, because schema drift breaks downstream parsing quickly.

## Choosing the right subscription for your workload

Subscription names are easy to copy. Choosing the right feed for the job is the harder part.

* **Price board** — `allMids` covers this cheaply across many assets with a single subscription object. No reason to pull `l2Book` just to show a number ticking.  
* **Trading dashboard** — combine `webData3` for the aggregate view with `clearinghouseState` when sharper account detail is needed.  
* **Order book viewer —** `l2Book`. Native `l2Book` behaves as a snapshot-style feed rather than a compact incremental diff stream, which works fine for UI rendering but scales poorly as an all-market depth-tracking backbone. Each update carries a fresh published book snapshot rather than an incremental delta.   
* **Alert system** — `candle` for threshold-based alerts, `trades` for tick-level triggers.  
* **Account monitor** — `openOrders` plus `userEvents`; add `orderUpdates` when tracking order lifecycle rather than just state.  
* **Portfolio dashboard** — `clearinghouseState` and `webData3` together; avoid subscribing to every user feed unless the product actually needs it.  
* **Analytics or indexing pipeline** — feeds high-volume streams, including `trades`, `l2Book` snapshots, `orderUpdates`, and `candle` data, where subscription math directly impacts architecture.  
* **Full-market data pipeline** — use `allMids` or `bbo` for broad pricing, then add per-market `trades`, `l2Book`, or `candle` subscriptions only where the pipeline needs them.  
* **User execution monitoring** — combine `orderUpdates`, `userFills`, and selected `userEvents`, with deduplication where the feeds overlap.


A common failure mode is subscribing broadly "just in case" instead of mapping feeds to product requirements. It works fine for a week, then the subscription count creeps past what the endpoint was designed to comfortably serve.

## Current Hyperliquid WebSocket limits

These are the [official per-IP limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits) as documented by Hyperliquid:

1. Maximum 10 WebSocket connections  
2. Maximum 30 new WebSocket connections per minute  
3. Maximum 1000 WebSocket subscriptions  
4. Maximum 10 unique users across user-specific WebSocket subscriptions  
5. Maximum 2000 messages sent to Hyperliquid per minute across all WebSocket connections  
6. Maximum 100 simultaneous inflight post messages across all WebSocket connections

These numbers define the actual shape of what can be built on the public endpoint, not minor implementation details. A dashboard tracking a small number of markets and users is unlikely to approach these limits under normal operation. A system trying to cover the full market or monitor many wallets will hit them early, usually earlier than the roadmap assumed. 

## When market coverage starts hitting WebSocket limits 

For market-specific feeds such as trades, `l2Book`, and candle, subscription count grows by market × feed type. Global feeds such as allMids are different, which is why they are better suited for broad price boards. 

Track 150 markets across `trades`, `l2Book`, and `candle`, and that's already 450 subscriptions before a single user-specific stream enters the picture. Add per-user monitoring on top and the math gets worse quickly. The 10-user cap can become the binding constraint before the 1000-subscription ceiling does — a dashboard watching fifteen or twenty wallets is already over budget on users, regardless of how much subscription headroom remains.

Ten connections per IP is a hard ceiling on independent upstream sockets, and opening one per internal consumer runs into it fast. Teams with many internal consumers are better off consolidating subscriptions and fanning the data out behind their own ingestion layer. Dashboards, analytics systems, and anything with multi-user account monitoring need subscription planning as a design step, not a post-incident cleanup task. 

## Reconnects, snapshots, and missed data recovery

Reconnect handling is where most production WebSocket clients either become reliable or start losing data silently.

Hyperliquid sets a clear expectation here: automated clients should expect periodic server-side disconnects, without warning, and are expected to reconnect gracefully. Missed data during a reconnect gap can show up in the snapshot acknowledgement received on resubscribe, or it can be recovered through the corresponding info request. Neither of those is optional when correctness matters to the product.

A production-safe recovery pattern generally looks like this:

* persist last processed state in external storage, not memory tied to the connection lifetime  
* handle reconnects as a normal runtime event, not an exception path  
* resubscribe in a deterministic, repeatable order  
* reconcile the initial snapshot against the last persisted state after reconnect  
* backfill missing history through the corresponding info request, or through a [Hyperliquid historical data](https://hyperliquid.gitbook.io/hyperliquid-docs/historical-data) API when deeper backfill is needed  
* dedupe aggressively — replayed or overlapping state during recovery is expected, not a bug  
* resume live processing only after backfill and dedupe are complete

### Concrete Info API recovery examples

The recovery pattern above is generic. In practice, what to query depends on what the product tracks.

Account view recovery

For account dashboards, use the snapshot ack first, then refresh current account state through the Info API before trusting the local view again. After reconnecting [`clearinghouseState`](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals) or `openOrders`, query the current account state:

```
{
 "type": "clearinghouseState",
 "user": "0x...",
 "dex": ""
}
```

and, if the product tracks open orders separately:

```
{
 "type": "openOrders",
 "user": "0x...",
 "dex": ""
}
```

Use the Info response as the new baseline, then apply live WebSocket updates on top of it.

Fills recovery

For fills, use the snapshot from userFills to catch recent activity after resubscribe. If the reconnect gap could be larger than that snapshot covers, query fills through the Info API and dedupe against the last stored fill:

```
{
 "type": "userFills",
 "user": "0x...",
 "aggregateByTime": false
}
```

For a bounded gap, query fills from the last processed timestamp through the [time-based fills endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) rather than replaying unbounded history.

Order book recovery

For order book views, do not try to repair an uncertain local book from partial memory after reconnect. Query a fresh L2 book snapshot first:

```
{
 "type": "l2Book",
 "coin": "BTC"
}
```

Then replace the local book baseline and resume consuming live `l2Book` messages. This is safer than applying new updates on top of a stale or partially corrupted book.

Without this recovery path, the client can look correct in testing while still dropping or duplicating events during volatility spikes, which is exactly when correctness matters most.

## WebSocket vs REST vs gRPC vs managed streams

| Option | Best for | Weakness |
| :---- | :---- | :---- |
| REST / Info API | Snapshots, backfill, occasional queries | Not built for continuous low-latency data |
| Native WebSocket | Focused, real-time feeds on specific markets | Connection, subscription, user-stream, and message limits |
| Hyperliquid gRPC streaming | High-volume real-time pipelines | Provider-dependent access, not part of the public Hyperliquid endpoint |
| Managed data streams | Production delivery, broad coverage, filtering, replay/backfill | Paid infrastructure dependency |

Each option covers a different part of the data path. REST fills the gaps native WebSocket leaves — it's how a client backfills after a reconnect, or seeds initial state before subscribing live. gRPC and managed streams enter the picture once the workload outgrows what a public, rate-limited endpoint was built to serve. The decision comes down to how many markets, how many users, and how much replay/backfill the product actually needs.

## Native Hyperliquid WebSocket vs provider endpoint

A provider endpoint becomes worth evaluating when public endpoint limits start affecting product decisions. Criteria worth checking against a [managed Hyperliquid WebSocket endpoint](https://supanode.xyz/docs/hyperliquid/websocket):

* higher or custom connection and subscription limits  
* private endpoint access instead of shared per-IP caps  
* regional latency and routing options  
* uptime guarantees and failover behavior  
* replay and backfill support after disconnect  
* [historical data access](https://supanode.xyz/docs/hyperliquid/indexer) alongside the live feed  
* WebSocket and gRPC both available, not just one  
* compatibility with official Hyperliquid schemas (subscription names, message shapes) — drift here breaks downstream parsing silently  
* monitoring, alerting, and actual support channels  
* documented behavior during high-volatility periods, when both connection stability and message volume spike at once

A managed endpoint that fails on schema compatibility can be worse than no managed endpoint at all, since it looks like a drop-in replacement until it isn't.

## How to test Hyperliquid WebSocket reliability

Before committing to either native WebSocket or a provider, it's worth instrumenting the connection and measuring it directly rather than trusting a claims page. Metrics worth tracking:

* connection drops per hour or per day  
* reconnect duration, from disconnect to reopened socket  
* resubscribe time after reconnect  
* event delay from origin to consumer  
* message backlog under load  
* missed events after reconnect  
* duplicate events after reconnect  
* subscription count relative to the 1000 ceiling  
* message rate relative to the 2000/minute cap  
* recovery time specifically during volatility spikes, since that's when connection stability and data volume are both stressed at once

Vendor benchmark pages rarely measure the exact route a product actually uses, so specific latency numbers are worth confirming through this kind of instrumentation before they get repeated internally as fact.

## Use native WebSocket for focused real-time workloads 

Native WebSocket handles a fairly wide range of real products without any provider involved:

* validating demand with an early MVP  
* a dashboard covering a limited, fixed set of markets  
* alerts for a small, defined asset list  
* moderate account monitoring across a handful of wallets  
* internal tools where occasional reconnects do not create production incidents

None of these need broad coverage or heavy replay guarantees, and moving to a managed provider too early is usually just added cost without added value.

## When to upgrade to managed Hyperliquid infrastructure

The signals tend to show up together rather than one at a time:

* broad market coverage across most or all listed assets  
* many user-specific streams pushing past the 10-user cap  
* a production dashboard with real uptime expectations  
* execution monitoring where missed events have a cost  
* an analytics or indexing pipeline running continuously  
* full-market order book coverage rather than a handful of pairs  
* a genuine need for gRPC-level throughput  
* replay or backfill requirements beyond what info requests conveniently cover  
* consistent latency requirements under load, not just on a good day  
* public endpoint limits actively shaping product roadmap decisions

The clearest signal is when endpoint limits start shaping the roadmap itself — when a feature gets scoped down or delayed because of the subscription cap rather than because of actual product priorities. At that point, it is cleaner to compare managed WebSocket, gRPC streaming, or a [dedicated Hyperliquid node](https://supanode.xyz/docs/hyperliquid/dedicated) against what the public endpoint can realistically sustain.

## Provider checklist for Hyperliquid WebSocket infrastructure

Get clear answers to these questions before committing to any Hyperliquid data infrastructure provider:

1. What are the WebSocket connection limits, and are they per IP, per workspace, per token, or negotiated?  
2. What are the subscription limits, and how do they compare to the public 1000-subscription cap?  
3. Are user-specific streams supported at scale, beyond the public 10-user ceiling?  
4. Does the endpoint stay compatible with official Hyperliquid schemas as those schemas evolve?  
5. Is gRPC available alongside WebSocket, or is it WebSocket-only?  
6. Is historical data available, or is this a live-only feed?  
7. Is replay/backfill available after a disconnect, and how far back does it go?  
8. Which regions are supported, and does that match where consumers actually run?  
9. How is failover handled — automatic, manual, or not addressed at all?  
10. What happens to delivery and latency during high-volatility periods specifically?  
11. What monitoring, alerting, and support are actually included, versus sold as an add-on?  
12. Is pricing based on requests, subscriptions, streams, bandwidth, or some custom usage metric — and does that match the actual traffic pattern?

---

Native Hyperliquid WebSocket does its job well for focused, real-time workloads: a limited set of markets, a manageable number of users, and products where occasional reconnects do not create product-critical data loss. Production systems that go beyond that need to design around connection limits, subscription budgets, the user-stream cap, and a reconnect path that treats missed data as something to recover, not something to ignore.

Need higher WebSocket limits or managed Hyperliquid data access? If connection limits, subscription caps, user-stream limits, or replay requirements are starting to shape the product roadmap, [compare managed Hyperliquid WebSocket, gRPC, and API endpoint options](https://supanode.xyz/pricing/hyperliquid) against what the public endpoint can actually sustain   
