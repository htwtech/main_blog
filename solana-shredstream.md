---
title: 'Solana ShredStream: Raw Shreds, Decoded Shreds, and Yellowstone gRPC'
slug: 'solana-shredstream'
canonical_url: 'https://supanode.xyz/blog/solana-shredstream'
date: "2026-07-17"
description: 'Solana ShredStream delivers raw shreds over UDP before RPC or gRPC, but the early signal only matters if decoding and recovery costs are justified.'
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Solana%20ShredStream.png"
coverAlt: "Solana ShredStream: Raw Shreds, Decoded Shreds, and Yellowstone gRPC"
tags: ["Solana ShredStream", "Raw Shreds", "Solana Shreds"]
---

Solana ShredStream gives you raw Solana shreds over UDP before that same slot data appears through processed RPC or gRPC surfaces.

That timing advantage is real, but it only holds if you can turn raw packets into a usable signal fast enough — ordering, deduplication, FEC recovery, reconstruction, parsing, filtering, and downstream validation all sit on the critical path before anything reaches your strategy or your index.

Raw shreds are earlier in the pipeline by design. The decision is whether that timing advantage is worth the decoding, recovery, and operational burden, or whether [Yellowstone gRPC](/blog/yellowstone-grpc-solana-guide) already gives you the execution-aware stream you need.

## What Solana ShredStream Actually Delivers

Solana produces block data in pieces before validators reconstruct and process it. A Solana leader assembles transactions into entries for its slot, then breaks that entry data into shreds — small, erasure-coded fragments that can be verified and reassembled independently of each other. Those shreds move across the validator network through [Turbine propagation](https://docs.anza.xyz/consensus/turbine-block-propagation), a fan-out broadcast tree built so that no single validator has to talk to every other validator directly.

A ShredStream provider positions itself close to that propagation layer and forwards the native shred packets to subscribers over raw UDP, ahead of the point where that same data gets reconstructed into transactions and served through an RPC call or a gRPC stream.

The data path looks like this:

![Diagram showing how Solana ShredStream reaches the client from Solana leader, Turbine propagation, ShredStream provider receiver, raw UDP multicast, client receiver and decoder, and strategy or infrastructure process](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/How%20Solana%20ShredStream%20reaches%20the%20client.png)

<sub>From Solana leader to client-side decoder.</sub>

None of this touches consensus. Turbine still propagates the same shreds to the same validators regardless of who else is listening on a parallel raw feed, and the leader schedule, voting, and finality proceed exactly as they would without any raw feed subscription anywhere near the network. What changes is who gets to look at the fragments first, and how much decoding work the consuming system is built to handle.

## What Raw Shreds Do Not Provide

The important limitation is that raw shreds are pre-execution data. A raw shred contains block data as the leader packaged it, not the result of executing that data.

Raw shreds do not contain:

* Execution result — whether a transaction succeeded, failed, or never landed at all is only known after processing.  
* Logs — program logs are generated during execution.  
* Balance changes — [pre- and post-balances](https://solana.com/docs/rpc/http/gettransaction) come from executed transactions.  
* Inner instructions — CPI details only exist after runtime execution.  
* Confirmed state — a shred carries proposed block data, not consensus confirmation.  
* Finality — finality comes later in the commitment process.

A raw feed can tell a client that a transaction was submitted with a particular shape and intent, but confirming what actually happened to it depends on execution further downstream, and that gap sits at the protocol level rather than in any single provider's implementation.

## Raw Packet Latency Is Not Actionable Latency

Getting the first UDP packet is only the opening move. A production client still has to work through a chain of steps before a raw shred becomes something a strategy can act on:

1. Packet ordering and deduplication across the multicast stream.  
2. Missing-shred recovery using erasure coding, since [UDP doesn't guarantee delivery](https://www.rfc-editor.org/info/rfc1122/).  
3. Shred and entry reconstruction back into something resembling a block.  
4. [Transaction deserialization](https://solana.com/docs/core/transactions/transaction-structure).  
5. Parsing and program-specific filtering, so the pipeline only surfaces what the strategy cares about.  
6. Handoff to the strategy or downstream service that actually does something with the signal.

The time from feed arrival to a usable, actionable application event is the metric worth tracking — packet arrival alone says little about how fast a strategy can actually react.

Any headline claim that raw shreds arrive a fixed number of milliseconds ahead of gRPC should be treated as incomplete without a stated methodology behind it. That number depends on too many variables to generalize: provider and validator placement, where the receiving host sits, the network path between the two, the decoder implementation, transaction complexity, system load at the moment of measurement, and where exactly the clock starts and stops. Two teams running the same underlying feed can report very different numbers depending on how carefully they built the decoding layer on top of it.

Latency comparisons need a controlled benchmark, using the same discipline as [low-latency Solana RPC testing](/blog/low-latency-solana-rpc): same host, same region, same timestamp source, same event definition, and the same workload.

## Running a Raw Shred Pipeline in Production

Everything in the previous section — ordering, recovery, reconstruction — reads like a decoding problem on paper. In production it turns into an operations problem. The decoder is not a one-time build; it becomes part of the production system, and it keeps demanding attention long after the pipeline first goes live.

### Ordering and deduplication

Ordering is usually the first operational problem. Shreds don't arrive in the sequence they were generated — multicast fan-out, jitter, and route differences mean a client has to buffer and reorder packets before reconstruction can even start, and that buffer needs sizing against real-world jitter rather than a best-case number from a spec sheet. The same shred can also arrive more than once, particularly over redundant delivery paths. Skip deduplication and a decoder does redundant work at best; at worst it produces inconsistent state that's hard to trace back to its source later.

### FEC recovery

Shreds are erasure-coded because UDP drops packets. Once enough of a [FEC set](https://github.com/anza-xyz/agave/blob/master/ledger/src/shred.rs) has arrived, the missing pieces can be reconstructed mathematically, but that recovery logic has to be both correct and fast. Get it wrong, and the client doesn't crash or throw an error — it quietly falls behind, with nothing obvious pointing to why.

### Bursts and backpressure

Shred volume isn't steady. Busy slots, network congestion, and leader-specific behavior can all produce a burst that a fixed-size buffer was never sized for, and a pipeline tuned only against average load can fail during the bursts it was supposed to handle. When the decoder can't keep up, something has to give — packets get dropped, queued, or the pipeline blocks further upstream. Each option fails differently, and a pipeline without an explicit backpressure policy ends up choosing one by accident, often in a way that does not match the workload.

### Monitoring

None of this is visible without instrumentation. Packet loss rate, decode latency, queue depth, and reconstruction failures all need their own visibility, because a degraded feed can look healthy until downstream results start diverging.

### Restarts

Restarts need explicit handling because they are normal production events, not edge cases. A decoder that restarts mid-slot needs defined behavior for whatever it missed during the gap, and that gap tends to surface at the worst possible moment — usually during the same burst the pipeline was least prepared for.

None of this is exotic engineering. It's ordinary work in low-latency systems generally, but it's continuous work — a raw shred pipeline isn't something assembled once and left alone.

## Solana ShredStream vs Yellowstone gRPC vs Decoded Shreds

The architecture decision becomes clearer when raw shreds, decoded shreds, and Yellowstone gRPC are compared by delivery stage, completeness, and operational cost.

| Criterion | Raw Shreds | Decoded Shreds | Yellowstone gRPC |
| :---- | :---- | :---- | :---- |
| Delivery stage | Pre-execution raw fragments | Pre-execution reconstructed transactions | Post-execution validator events |
| Interface | UDP | Yellowstone-compatible gRPC | Yellowstone gRPC |
| Client decoding | Required | Managed by provider | Not required |
| Server-side filters | No | Product-dependent | Yes |
| Execution result | No | No | Yes |
| Logs and balance changes | No | No | Yes |
| Data completeness | Raw fragments only; no execution context | Reconstructed transactions; still no execution context | Full execution metadata and state |
| Replay / recovery support | Client-built, not guaranteed | Provider-managed, product-dependent | Standard gRPC reconnect and catch-up patterns |
| Bandwidth footprint | High — full unfiltered stream | Moderate — reconstructed but typically still broad | Lower — filterable to relevant accounts/programs |
| Operational complexity | High: decoder, recovery, monitoring, restart handling | Lower: provider absorbs decoding | Low: standard client libraries |
| Confirmation source | None — intent only | None — intent only | Validator-confirmed execution state |
| Best fit | Earliest signal, custom pipelines | Early signal with simpler integration | Most production streaming workloads |

Decoded Shreds sit between the two extremes: they remove the client-side decoding burden while keeping the pre-execution timing advantage. The exact trade-off depends on the Decoded Shreds implementation — reconstruction guarantees, filtering, replay behavior, and pricing — detailed in the [Decoded Shreds documentation](https://supanode.xyz/docs/solana/shreds/decoded).

The trade-off is signal timing versus data completeness, integration cost, and operational risk. Raw shreds provide the earliest feed, but leave decoding and recovery to the client. Yellowstone gRPC provides a filtered, execution-aware stream with lower integration overhead, built on the standard Yellowstone gRPC client model. Neither one is a strict upgrade over the other; they answer different questions.

## When Raw Shreds Are the Right Choice

Raw shreds are justified under a narrow set of conditions:

* Earlier visibility can meaningfully change execution priority or expected profit.  
* The strategy is built to act on transaction intent rather than confirmed state.  
* The team can build and maintain a production-grade decoder, including the ordering, recovery, and monitoring work described above.  
* There's a second data source available to validate early signals against, since intent isn't outcome.

In practice that tends to map onto market making, latency-sensitive arbitrage, liquidation infrastructure, MEV or searcher systems, [custom RPC or validator infrastructure](/blog/solana-rpc-infrastructure-guide), and specialized indexers built around raw, unprocessed input. For trading systems specifically, the benchmark should measure whether earlier intent actually improves execution outcome, not just whether the packet arrived first — the same distinction covered in the [trading bot infrastructure](/blog/solana-trading-bot-infrastructure) notes.

If the workload does not depend on the earliest intent signal, Yellowstone gRPC or decoded shreds usually create less operational drag.

## When Yellowstone gRPC Is the Better Choice

For most production workloads, Yellowstone gRPC is the right starting point. It's the better fit when an application needs:

* Account, transaction, block, or log filters.  
* Execution status.  
* Balance changes.  
* Inner instructions.  
* Confirmed or processed state.  
* Standard tooling and a lower operational footprint.  
* A reliable application-level stream rather than a race to the first possible packet.

Wallets, dashboards, explorers, monitoring and alerting systems, general-purpose indexers, and analytics platforms usually fit this model better than a raw shred pipeline.

## A Production Architecture Usually Uses More Than One Feed

Production trading and infrastructure workloads often use a hybrid setup rather than treating one feed as the entire data path.

![Diagram showing a hybrid Solana ShredStream architecture with raw shred feed for early transaction intent detection and Yellowstone gRPC or RPC for execution result, state, confirmation, reconciliation, recovery, and storage](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/The%20hybrid%20architecture.png)

<sub>Raw shreds for speed, gRPC for confirmation.</sub>

There are practical reasons this pattern shows up so consistently:

* Early data doesn't guarantee successful executed state, so outcome and confirmation still have to come from a processed source.  
* UDP delivery brings packet loss and gaps that need explicit handling, not assumptions.  
* Production systems need confirmation somewhere downstream of the raw feed, along with the replay and reconciliation logic to catch whatever the raw path missed.

A raw feed on its own covers roughly the front half of an indexing or state pipeline; the back half — confirmation, replay, reconciliation — still has to come from a processed source.

## Total Cost of Ownership Beyond the Feed Price

For teams building the decoder in-house, the feed subscription may be only one part of the [total cost](/blog/managed-rpc-vs-self-hosted-rpc). Before committing to raw shreds, it's worth pricing out the rest of the stack:

* Engineering time. Building and maintaining a decoder — ordering, deduplication, FEC recovery, restart handling — is ongoing work, not a one-time build. Someone owns that code indefinitely.  
* Monitoring and alerting infrastructure. Packet loss, decode latency, and queue depth all need dashboards and alert thresholds, separate from whatever tooling the strategy itself uses.  
* Redundancy. A single feed and a single host is a single point of failure. Serious deployments typically run redundant receivers, which multiplies infrastructure cost accordingly.  
* Network proximity costs. Getting meaningful latency out of a raw feed usually means colocating or near-colocating infrastructure close to the provider's region, which carries its own hosting premium.  
* Bandwidth. An unfiltered raw stream is heavier than a filtered gRPC subscription; bandwidth costs scale with that difference.  
* On-call and incident response. A decoding pipeline that silently falls behind during a burst doesn't announce itself — someone has to notice, diagnose, and fix it, often under time pressure.  
* Opportunity cost. Engineering time spent maintaining a shred decoder is time not spent on the strategy itself. For some teams that trade-off is obviously worth it; for others it quietly isn't.

None of this makes raw shreds a bad choice; it just means the operational cost lives beyond the invoice — in decoder maintenance, redundancy, monitoring, proximity, bandwidth, and incident response.

## How to Evaluate a ShredStream Feed

The reliable way to evaluate a feed like this is to benchmark it in the environment where it's actually going to run, not in a vendor's demo environment.

Measure:

* p50, p95, and p99 delivery latency.  
* Time to first usable decoded event, not just first packet.  
* Packet loss and recovery rate.  
* Duplicate rate.  
* Decoder queue depth and processing time.  
* CPU, memory, and bandwidth requirements under load.  
* Gap behavior during traffic spikes, since that's when it matters most.  
* The actual gap between shred-based detection and gRPC-based confirmation.  
* End-to-end effect on the real strategy outcome.

When comparing feeds, keep the receiving host, location, timestamp source, event definition, observation period, and workload identical across both tests. Two feeds measured under different conditions will produce numbers that look comparable and aren't. The safer approach is to test against the actual workload and the actual decoder rather than choosing based on an advertised headline number.

## Supanode Raw Shreds

For teams that want to test this path directly, Supanode by HighTower operates a live raw shred feed with the following parameters:

| Attribute | Supanode Raw Shreds |
| :---- | :---- |
| Data | Native Solana shreds |
| Protocol | Raw UDP multicast |
| Region | Frankfurt |
| Authentication | IP allowlist |
| Server-side filters | None |
| Published delivery latency | 0.4 ms p99 from the Supanode receiver |
| Price | $200/month per IP |
| Subscription window | 7-day minimum, 90-day maximum |
| Trial | Up to 24 hours |
| Client responsibility | Recovery, reconstruction, decoding, parsing, and filtering |

The 0.4 ms p99 figure describes delivery from the receiver to the allow-listed IP, not full application latency — the client-side steps covered earlier in this guide still apply.

Decoded Shreds, which handle reconstruction on the provider side, are on the roadmap for v1.1 and not yet available; pricing for that product hasn't been published.

Full specification is available in the [Raw Shreds documentation](https://supanode.xyz/docs/solana/shreds/raw) and on the [ShredStream service page](https://supanode.xyz/services/solana/shredstream); current terms are listed on the [Solana pricing page](https://supanode.xyz/pricing/solana).

Test Supanode Raw Shreds against your own decoder and workload.
