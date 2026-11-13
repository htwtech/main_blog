---
title: "Best Solana gRPC Providers: Pricing, Limits & Replay"
slug: "best-solana-grpc-providers"
canonical_url: "https://supanode.xyz/blog/best-solana-grpc-providers"
date: "2026-10-29"
description: "Compare seven Solana Yellowstone gRPC providers by pricing, subscription limits, replay, and regions to find the right service for your workload."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/htw-cover-solana-grpc-providers.png"
coverAlt: "Best Solana gRPC Providers: Pricing, Limits & Replay"
tags: ["Solana","gRPC","Yellowstone gRPC","Providers"]
---

# Best Solana Yellowstone gRPC Providers: Pricing, Limits, and Replay

## Quick answer: which provider should you choose?

The best Solana gRPC provider depends on what your application streams and how it recovers missed data. Supanode offers flat-rate plans from $99 per 30 days for Frankfurt-based workloads. Shyft is another unmetered option, while Helius and Alchemy offer longer historical replay.

This guide compares seven providers by pricing, subscription limits, regions and recovery options. Supanode publishes this comparison and is one of the providers included.

## Solana Yellowstone gRPC providers at a glance

| Provider | Starting price | Key limits | Recovery | Regions |
| --- | --- | --- | --- | --- |
| [Supanode](https://supanode.xyz/docs/solana/grpc/limits) | $99 / 30 days; unmetered gRPC | 1–50 subscriptions; 100–100,000 watched addresses | 100-slot shared replay; configurable dedicated buffers; 2-day RPC history | Frankfurt shared; dedicated locations on request |
| [Helius LaserStream](https://www.helius.dev/docs/laserstream) | $499/mo + metered usage | 10 Business / 100 Professional connections | Up to about 48h replay | Nine locations, including Frankfurt |
| [QuickNode](https://www.quicknode.com/docs/solana/solana-grpc/overview) | $499/mo + metered usage | 10 Scale / 25 Business streams | About 11-minute replay | US, Europe, Japan |
| [Triton One](https://docs.triton.one/project-yellowstone/dragons-mouth-grpc-subscriptions) | $0.08/GB; $125 prepaid balance | Connection limits not publicly specified | Dragon's Mouth buffer; Fumarole up to 4 days | GeoDNS; Fumarole in Amsterdam and New York |
| [Chainstack](https://docs.chainstack.com/docs/yellowstone-grpc-geyser-plugin) | $98/mo including base plan and add-on | Up to 25 streams; 200 keys per filter list | About 100-slot shared replay | Geo-balanced Global Nodes |
| [Alchemy](https://www.alchemy.com/solana-grpc) | $75/TB; no minimum | Connection and filter caps not publicly listed | About 48h replay | US, Europe, Asia-Pacific |
| [Shyft](https://shyft.to/solana-yellowstone-grpc) | $199/mo; unmetered | 10–50 connections; large address filters | 150-slot replay | Six live locations, including Frankfurt |

## The providers: pricing, strengths, and limits

### Supanode

Supanode uses prepaid 30-day bundles, from FOCUS at $99 to PROFESSIONAL at $459. Each includes unmetered gRPC throughput and RPC access. FOCUS supports one subscription and 100 watched addresses; higher plans increase both limits.

Shared nodes run in Frankfurt and allow up to 2,000 addresses per filter and 10 filters per subscription. Owner-program filters start at BUILD. Some broad program filters are restricted on shared nodes, and unfiltered full-block streaming requires PROFESSIONAL with support activation.

Shared replay covers 100 slots. A [Supanode Dedicated Node](https://supanode.xyz/services/solana/dedicated-node) can be configured with a 120- or 240-slot buffer, a chosen region and custom filter limits. Dedicated pricing is quoted per configuration.

For longer gaps, the included RPC service provides two days of history for retrieving missed finalized blocks through `getBlock`. This gives full-block indexers a recovery path beyond the live stream's buffer.

Shared and dedicated setups can be trialed for up to 24 hours. See [Supanode gRPC plans and limits](https://supanode.xyz/services/solana/grpc).

### Helius LaserStream

LaserStream starts at $499/mo on Business or $999/mo on Professional. Streaming costs 2 credits per 0.1 MB of uncompressed data. Fixed-price data add-ons are available on Professional.

Business supports 10 active connections and Professional supports 100. Frankfurt is among nine mainnet locations.

Historical replay extends to about 48 hours. Data older than roughly 20 minutes is recovered as finalized blocks, which suits historical backfill rather than reconstruction of every intermediate account update.

See [Supanode vs Helius](https://supanode.xyz/compare/helius) for a direct comparison of the two services.

### QuickNode Solana gRPC

Solana gRPC is included in Scale at $499/mo and Business at $999/mo, on Mainnet and Devnet.

Scale allows 10 streams with 10 accounts per stream; Business allows 25 streams with 50 accounts per stream. Larger address sets require compressed filters and client-side filtering.

Replay covers roughly the last 11 minutes. Streaming costs 15 credits per 0.1 MB, with overage rates depending on the plan. Compression can reduce billed traffic.

The credit model is compared with Supanode's flat bundles in [Supanode vs QuickNode](https://supanode.xyz/compare/quicknode).

### Triton One: Dragon's Mouth and Fumarole

Triton maintains the open-source [Yellowstone repository](https://github.com/rpcpool/yellowstone-grpc) and offers two hosted streaming products.

Triton One Yellowstone gRPC pricing is usage-based: $0.08/GB, with a $125 prepaid balance at signup.

Dragon's Mouth supports live streaming, including blocks; its replay depth is not publicly specified. Fumarole offers up to four days of recovery for accounts and transactions, but does not stream full blocks. Fumarole runs in Amsterdam and New York.

See [Supanode vs Triton](https://supanode.xyz/compare/triton) for a direct comparison.

### Chainstack

Chainstack starts at $98/mo: a $49 Growth plan plus the entry-level $49 gRPC add-on. Larger add-ons support up to 25 streams.

Filters allow up to 200 keys per list. Shared nodes restrict broad Token and System Program filters, so these plans suit targeted subscriptions rather than unrestricted program-wide feeds.

Shared replay covers about 100 slots. Dedicated Nodes offer configurable buffers. Yellowstone is available on Mainnet.

### Alchemy

Alchemy charges $75/TB with no base plan or minimum monthly commitment. It supports account, transaction and block streams, with about 48 hours of replay.

Stream monitoring is included, and regional options cover the US, Europe and Asia-Pacific.

Its usage model makes the bill easy to estimate from billed traffic, particularly for smaller filtered streams.

### Shyft

Shyft offers unmetered plans at $199, $349 and $649 per month, with 10, 20 and 50 connections respectively.

Large filters are a key feature: up to 150,000 transaction addresses or 400,000 account addresses per filter. Build's 10 connections can accommodate a backend with several parallel consumers without moving to a higher tier for connection count alone.

Replay covers up to 150 slots. Six live regions include Frankfurt. Shyft also offers dedicated gRPC nodes and a public lag dashboard.

## Filters and subscriptions: will your workload fit the plan?

Connection limits and subscription limits are not interchangeable. Check how many simultaneous streams and watched addresses a plan supports, as well as any restrictions on broad program filters.

For filtering details, see the [Yellowstone gRPC guide](/blog/yellowstone-grpc-solana-guide). If you are choosing between gRPC, WebSocket and shreds, start with [Solana data streaming options](/blog/solana-data-streaming-guide).

## Replay and reconnects: what can you recover?

Replay re-delivers missed data through the stream. RPC backfill retrieves historical blocks or transactions through separate requests.

Short replay windows cover brief disconnects. Longer outages may need historical replay or RPC backfill, depending on the data your application needs.

For implementation details, see how to [recover a Solana stream gap](https://supanode.xyz/learn/recover-solana-stream-replay).

## Solana gRPC pricing by workload

The following examples compare costs for three illustrative workloads. Data volumes are per billing period.

- **A — Small monitor:** one stream, 50 addresses, a Frankfurt backend, 300 GB of billed data and 20-slot recovery.
- **B — Multi-consumer backend:** eight streams, 5,000 addresses, 2 TB of billed data and 120-slot recovery.
- **C — Full-block indexer:** 20 TB of billed data and recovery after six hours offline. Finalized blocks are sufficient.

| Provider | A: small monitor | B: multi-consumer backend | C: six-hour recovery |
| --- | --- | --- | --- |
| Supanode shared | $99 — FOCUS | Dedicated Node needed for 120-slot replay | $459 — PROFESSIONAL, RPC backfill |
| Supanode Dedicated Node | Custom quote | Custom quote | Custom quote, RPC backfill |
| Helius | $499 — Business | $499 — Business | $1,749 — Professional + 10 TB add-on |
| QuickNode | $499 — Scale with compressed filters, or $999 — Business | $499 — Scale with compressed filters | Not calculated |
| Triton One | $24 usage; $125 prepaid balance required | $160 usage | Not calculated |
| Chainstack | $98 — Growth + add-on | Shared replay limited to about 100 slots | Not calculated |
| Alchemy | $22.50 | $150 | $1,500 |
| Shyft | $199 — Build | $199 — Build | Not calculated |

Helius meters uncompressed data; QuickNode can meter compressed data, so the same traffic can produce different bills.

For C, Supanode's $459 bundle uses RPC backfill to recover missing finalized blocks. Helius and Alchemy provide historical replay through their streaming products. The recovery methods differ even though they address the same outage.

## Regions and trial checks before production

Test from the region where your application runs:

- Check latency and delivery consistency with your actual filters.
- Verify stream and address limits.
- Test recovery after a disconnect and check the resulting bill.

For a detailed method, see how to [benchmark a Solana gRPC stream](https://supanode.xyz/learn/benchmark-solana-grpc).

## Which provider should you shortlist?

- **Small Frankfurt monitor:** Supanode FOCUS starts at $99 per 30 days; Shyft Build is another flat-rate option.
- **More consumers and longer replay:** consider a Supanode Dedicated Node, Helius or Shyft.
- **Six-hour full-block recovery:** Supanode PROFESSIONAL uses RPC backfill; Helius and Alchemy offer historical stream replay.

[Check Supanode gRPC plans and limits](https://supanode.xyz/services/solana/grpc)


