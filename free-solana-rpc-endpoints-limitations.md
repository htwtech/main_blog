---
title: 'Free Solana RPC: Public Endpoints, Limits, and When to Upgrade'
slug: 'free-solana-rpc-endpoints-limitations'
canonical_url: 'https://supanode.xyz/blog/free-solana-rpc-endpoints-limitations'
date: "2026-06-21"
description: "Learn what free Solana RPC actually means, how public endpoints and provider free tiers differ, where limits appear, and when production apps should upgrade."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Free%20Solana%20RPC_%20Public%20Endpoints,%20Limits%20and%20When%20to%20Upgrade.png"
coverAlt: "Free Solana RPC: Public Endpoints, Limits, and When to Upgrade"
tags: ["Free RPC", "Free Solana RPC", "Free endpoints", "RPC Endpoints"]
---

Free Solana RPC is, for most teams, where things start. You copy an endpoint URL, point your app at it, and it works — which is fine at that stage.

The trouble usually comes later, and it comes quietly. Traffic grows, the app gets real users, someone adds a trading feature or a live balance panel, and at some point the endpoint starts returning rate limits, stale data, or dropped connections. By then, the dependency is already embedded in the product.

The official Solana docs are direct about this: [public RPC endpoints are shared infrastructure](https://solana.com/docs/rpc), rate-limited, and not intended for production applications. The cluster documentation says so explicitly, alongside the note that rate limits can change without notice and that `403` is a real outcome if a site or IP gets flagged.

So the question worth asking is not "is free RPC bad" — it is what you are actually running behind it. Tutorials, devnet work, internal scripts, early private beta with a few dozen forgiving users: free/public RPC is a reasonable fit. Swaps, wallets, real-time dashboards, transaction flows where users notice failures: it is usually the wrong layer to anchor on, and the gap between those two cases can be easy to underestimate.

---

## What "free Solana RPC" actually means

"Free Solana RPC" usually refers to several different access models, and the differences can lead teams to plan around the wrong reliability model.

[**Official public Solana endpoints**](https://solana.com/docs/references/clusters) — `https://api.mainnet.solana.com`, `https://api.devnet.solana.com`, `https://api.testnet.solana.com` — are open, shared infrastructure. No account, no API key, no SLA. The docs explicitly warn against using them as the primary production dependency for launched applications. Cached endpoint lists and older tooling may reference different URLs, so the official documentation is the safer source. 

**Third-party public endpoints** are a separate category. Public directories may list multiple third-party free Solana endpoints. Useful for discovery, but a directory listing is not a promise of reliability or defined limit ceilings. Some free/public endpoints can become temporarily unavailable or depend on public-node capacity, so treat directory availability as a discovery signal, not a reliability guarantee.

**Provider free tiers** are not the same thing as open public endpoints, even at zero cost. They usually require an account, issue an API key or tokenized URL, and come with dashboards, usage logs, plan limits, and a paid upgrade path. A provider free tier has a defined failure budget and a known next step. A public open endpoint has neither.

**Local development** — `http://localhost:8899` for HTTP RPC, `ws://localhost:8900` for WebSocket — sits in its own category entirely. Running `solana-test-validator` locally removes the infrastructure question altogether.

For most workloads, the access model matters more than which provider has the largest headline free quota.

---

## Public RPC endpoint vs provider free tier

A public endpoint is unauthenticated shared access. A provider free tier is account-based access with quotas, logs, and an upgrade path.

| Access type | Auth required | Limits | Support | Production fit | Upgrade path |
| :---- | :---- | :---- | :---- | :---- | :---- |
| Official public endpoint | No | Documented per-IP ceilings; can change without notice | None | Not intended for production apps | Private/dedicated RPC |
| Third-party public endpoint | No | Varies; often undocumented | None | Low reliability guarantee | Depends on provider |
| Provider free tier | Yes — API key or account | Plan-specific credits, RPS, method-level caps, WS limits | Limited / community | Development and low-volume | Paid shared or dedicated plan |
| Paid shared/managed RPC | Yes | Higher throughput, defined quotas | Standard support | Most production workloads | Higher tier or dedicated |
| Dedicated RPC | Yes | Custom capacity allocation | Priority or enterprise | High-volume, latency-sensitive | Quote-based or custom |

Provider free tiers differ in meaningful ways: how they count usage (credits, compute units, request units), whether free access is permanent or a trial, which methods have separate ceilings, and what the first paid step looks like. These differences matter more than the headline free quota.

---

## What to check before committing to a free Solana RPC provider

The access-type table above shows the categories. This is the evaluation layer — what to actually verify before the endpoint becomes a real dependency.

**RPS and burst limits.** The official Solana public endpoints document 100 requests per 10 seconds per IP and 40 requests per 10 seconds per IP for a single RPC method. Provider free tiers publish their own numbers, and those numbers are not always comparable — some count by credits or compute units, some by raw requests. Check what the ceiling looks like under a realistic burst, not just steady-state traffic.

**Method-level caps, not just total RPS.** Some providers separately limit `sendTransaction`, `getProgramAccounts`, and WebSocket connections independently of the general rate limit. A plan that looks adequate on overall RPS can still block specific methods your app depends on. Check the method-level limits explicitly before assuming the headline number covers everything.

**WebSocket support and connection limits.** If the app relies on live updates — balance changes, log subscriptions, confirmation tracking — verify WebSocket availability and per-connection limits separately from HTTP RPC. Some free tiers include WebSocket access; others restrict it to paid plans or cap the number of concurrent subscriptions.

**Slot freshness and node health signals.** Check whether the provider exposes a status page or health indicators. RPC error `-32005` means a node is behind by slots; on free tiers with shared capacity, this can surface during traffic spikes. A provider that documents this behavior and its retry guidance is more useful than one that does not.

**Historical method support.** `getProgramAccounts`, `getTransaction`, `getSignaturesForAddress`, and `getBlock` behave differently from simple reads. Some free tiers time out or restrict these methods under load. If the app needs historical data or program-level queries, test those methods specifically rather than assuming general RPC availability covers them.

**Access controls and key exposure.** Even on a free tier, exposed API keys can lead to quota exhaustion from unauthorized traffic. Check whether the provider supports domain restrictions, IP allowlisting, or proxy-friendly access patterns before shipping anything that calls RPC from client-side code.

**Upgrade path.** The free tier should have a visible next step that does not require migrating to a different provider entirely. Method-level limits, WebSocket support, and credit accounting should carry over into paid tiers without re-architecture.

---

## Official Solana public RPC URLs

| Cluster | HTTP RPC URL | Intended use | Production note |
| :---- | :---- | :---- | :---- |
| Mainnet-beta | `https://api.mainnet.solana.com` | General access, testing | Shared, rate-limited, not for production apps |
| Devnet | `https://api.devnet.solana.com` | Developer testing | Suitable for development and CI |
| Testnet | `https://api.testnet.solana.com` | Validator / release testing | Not for general dApp development |
| Local validator | `http://localhost:8899` | Local development | Only valid for local tooling |

Always check the current Solana clusters documentation before relying on these URLs or their published limits — rate limits can and do change without notice.

```
curl https://api.mainnet.solana.com \
 -X POST \
 -H "Content-Type: application/json" \
 -d '{
   "jsonrpc": "2.0",
   "id": 1,
   "method": "getBalance",
   "params": [
     "Vote111111111111111111111111111111111111111",
     { "commitment": "finalized" }
   ]
 }'
```

SDK setup, endpoint URL patterns, WebSocket usage, and JSON-RPC method behavior belong to the broader [Solana API and RPC endpoint layer](/blog/solana-api-rpc-guide).

---

## What free/public Solana RPC is good for

Free/public RPC works well when failures are tolerable, volume is low, and no single user depends on the endpoint being available at a specific moment:

* Local scripts, CLI tooling, one-off queries  
* Tutorials, demos, and hackathon prototypes  
* Devnet and testnet development  
* Internal read-only tools with low, predictable traffic  
* Early private betas where users understand reliability is not guaranteed  
* Non-critical pipelines where retries are cheap and staleness is acceptable

Before real usage appears, dedicated RPC is usually unnecessary.

---

## Where free Solana RPC starts to break

**Rate limiting — 429 Too Many Requests.** Solana's public endpoints document exact per-IP ceilings: 100 requests per 10 seconds per IP, 40 requests per 10 seconds per IP per RPC method, 40 concurrent connections per IP, 40 new connections per 10 seconds per IP, and 100 MB per 30 seconds. Exceed those and you get `429`, with the docs pointing to the `Retry-After` header. Some provider free tiers also add method-level ceilings for calls like `sendTransaction`, `getProgramAccounts`, and WebSocket connections. One flat RPS number does not capture the real failure surface.

**Blocking — 403 Forbidden.** The official Solana docs say high-traffic websites may be blocked without notice, and define `403` as "your IP address or website has been blocked." An application can be behaving reasonably by its own standards and still look like abuse from the perspective of shared infrastructure operators. Public infrastructure does not give you an SLA, support path, or advance warning when this happens.

**Stale reads.** This comes from two sources. One is commitment level — `processed` is the freshest view but can be rolled back, while `confirmed` and `finalized` are progressively stronger guarantees. The other is endpoint health: RPC error code `-32005` maps to a node that is unhealthy or behind by slots. For anything user-visible, freshness is a correctness metric, not just a latency one.

**Expensive read patterns.** Some methods are structurally more costly and hit limits faster. `getProgramAccounts` is the clearest example — it can time out or be capped at the method level even when generic RPS looks fine. Calling `getTransaction` per-signature in a naive loop, running `getSignaturesForAddress` at scale, pulling `getBlock` on busy slots — all behave differently from a simple balance check. If the free plan only works when you avoid the methods your app needs, it is not a reliable fit for that workload.

**WebSocket fragility.** Solana exposes standard subscription methods like `accountSubscribe` and `logsSubscribe`, and subscriptions are usually better than polling for live UI. But plain RPC WebSockets are client-managed sessions with no built-in replay. If a connection drops, missed events are gone unless you backfill through HTTP. WebSocket clients need explicit reconnect logic, resubscription on reconnect, and a strategy for reconstructing state from the last known slot or signature.

**Browser traffic multiplying request count.** If a browser app calls RPC directly — without a backend proxy — every active user becomes independent RPC traffic. A modest app with a few hundred concurrent users can look like aggressive traffic on a free endpoint before anyone on the team notices. This is also where exposed API keys become a real problem: a key visible in client-side code can be scraped and abused, draining quota that affects legitimate users. We cover the security side in our [Solana RPC endpoint security guide](/blog/solana-rpc-endpoint-security
).

**Shared capacity with unknown neighbors.** Public endpoints are shared with traffic from applications entirely outside your control. A spike from an unrelated project can degrade your latency even when your own request patterns are reasonable.

---

## Free Solana RPC decision matrix by workload

| Use case | Free/public endpoint | Provider free tier | Managed/shared RPC | Dedicated RPC |
| :---- | :---- | :---- | :---- | :---- |
| Tutorial / local script | OK | OK | — | — |
| Devnet testing | OK | OK | — | — |
| Internal read-only tool | OK with limits | OK | Better | — |
| Private beta | Risky | OK with limits | Better | — |
| Portfolio / balance dashboard | Limited | Limited | Required for production | — |
| User-facing dApp | No | No as primary | Required | For high volume |
| Trading bot | No | No | Usually required | Often required |
| Swap / payment flow | No | No | Required | Often required |
| Analytics / indexer | No | No | Required | Often required |
| Real-time data backend | No | No | Required | For demanding loads |
| High-volume production app | No | No | Required | Required |

"OK with limits" means it can work, but monitor errors, slot freshness, and method-level failures from day one.

---

## Can you use free Solana RPC in production?

For public production apps, free/public RPC should not be the primary dependency.

There are narrower cases where it remains acceptable: a read-only internal tool with low and predictable traffic, a demo environment where stale data is explicitly labelled, a short-lived launch experiment with no financial flows, a non-critical analytics pipeline where failures are caught and retried asynchronously.

What it is not suitable for as the main endpoint:

* Swaps, payment flows, or any transaction that needs reliable submission and confirmation  
* Wallets or portfolio dashboards where stale balances affect usability  
* [Trading bots](/blog/solana-trading-bot-infrastructure
) or anything with latency-sensitive write paths  
* NFT mints or drops, where traffic spikes are predictable and confirmation timing matters  
* Analytics backends or indexers making heavy use of `getSignaturesForAddress`, `getTransaction`, or `getBlock`  
* Any application where production users encounter the failures directly

If endpoint failures are visible to real users, the cost is no longer only infrastructure spend. It becomes a product reliability issue.

---

## How to use free Solana RPC responsibly

Careful client behavior can reduce pressure on free/public RPC. The improvements are mostly about not amplifying the failure modes described above.

* **Backoff and jitter on 429\.** Respect `Retry-After` when present. If not, start around one second, double on each retry up to roughly 30 seconds, add ±25% jitter, and stop after a bounded number of attempts. The logic in sequence: read `Retry-After` → wait → double the delay → add jitter → retry → stop at the limit.  
* **Fix query shape before changing plans.** Use filters for [`getProgramAccounts`](https://solana.com/docs/rpc/http/getprogramaccounts), `dataSlice` or [`getMultipleAccounts`](https://solana.com/docs/rpc/http/getmultipleaccounts) where appropriate, paginate historical reads. Batching does not reduce total request count — a slow request inside a batch stalls everything else.  
* **Subscriptions over polling.** For balance updates, account changes, and log monitoring, [`accountSubscribe` and `logsSubscribe`](https://solana.com/docs/rpc/websocket) are usually better than repeated HTTP polling. They still need explicit reconnect logic, keepalive pings, and an HTTP backfill strategy after reconnect.  
* **Choose commitment intentionally.** `processed` can be rolled back. For balance screens or payment confirmation — anywhere backward movement creates confusion — `confirmed` or `finalized` is the right choice.  
* **Proxy RPC calls server-side.** If a browser can see your endpoint URL or API key, so can anyone else. Even on a free tier, exposed keys drain quota that affects real users.

---

## When to upgrade from free Solana RPC

Upgrade when any of the following stop being occasional and become normal patterns in your traffic:

* Regular `429` responses during ordinary application usage  
* Any `403` from legitimate, non-abusive traffic  
* Users reporting stale balances, stuck UI, or slow confirmation  
* WebSocket reconnects that are visible in the UX — dropped live updates, flickering state  
* Frequent reliance on `getProgramAccounts`, `getTransaction`, `getSignaturesForAddress`, or `getBlock` as core product dependencies  
* Backend jobs and frontend traffic competing for the same free quota  
* Free-tier credits becoming a planning constraint that forces architectural trade-offs  
* No support path when something goes wrong at an inconvenient time

Retry logic can reduce symptoms, but it does not replace adequate capacity.

---

## What to use instead of free public RPC

**Read reliability problems** — stale reads, inconsistent responses, unpredictable latency — are usually addressed by moving to a paid shared or managed RPC tier with a proper SLA and isolated capacity.

**Predictable throughput** — high-frequency reads, concurrent users, backend jobs running alongside frontend traffic — points toward a dedicated RPC node where the application is not competing with other tenants.

**Live data and event-driven UX** — reducing slot lag, building real-time feeds — works better with WebSocket subscriptions on a managed endpoint, or Yellowstone gRPC for backend ingestion that needs higher throughput and replay.

**Transaction submission quality** — [slow confirmation, failed landings](/blog/solana-sniper-bot-rpc-latency
), inconsistent forwarding — is a separate problem from read reliability. A dedicated write path or sender product gives more predictable behavior under congested network conditions.

**Historical and analytical workloads** — address history, transaction lookup at scale — RPC is usually a poor fit for bulk historical access. An indexer or historical data API is the more appropriate layer.

**Uptime and failover** for reliability-sensitive systems require load balancing across multiple endpoints and an active failover strategy. A single shared URL with no monitoring is usually not enough.

Read/write path separation, gRPC, ShredStream, SWQoS, and indexing layers are part of the broader [Solana RPC infrastructure](/blog/solana-rpc-infrastructure-guide
) stack.

---

## Final recommendation

Start free if the workload is genuinely low-risk. The cleaner upgrade decision comes from measurement: error rates, latency, slot freshness, WebSocket reconnects, and confirmation delays tracked from the beginning rather than diagnosed after the fact.

If your app is already seeing rate limits, stale reads, WebSocket drops, or inconsistent confirmations during normal usage, send us your workload. Supanode can size a managed or dedicated Solana RPC setup and run an eligible [48-hour trial for production workloads](https://supanode.xyz/solutions/shared-rpc).
