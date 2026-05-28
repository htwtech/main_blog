---
title: "Solana RPC Infrastructure Guide for Production dApps"
slug: "solana-rpc-infrastructure-guide"
date: "2026-05-28"
description: "Learn how Solana RPC infrastructure works for production dApps, from RPC endpoints and nodes to public, shared, dedicated, and self-hosted setups."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Solana%20RPC%20Infrastructure%20Guide%20%20for%20Production%20dApps.png"
coverAlt: "Solana RPC infrastructure architecture for production dApps"
tags: ["solana", "solana-rpc", "rpc-infrastructure", "developer-tools", "web3"]
---

# Solana RPC Infrastructure Guide for Production dApps

On Solana, where users expect sub-second interactions, the RPC layer becomes part of the product experience whether you planned for it or not. When an endpoint returns 429s, a WebSocket subscription silently drops, or a transaction retry loop burns an expired blockhash, users blame the app. Solana's own documentation is fairly direct about this: [poor RPC performance is indistinguishable from poor cluster performance](https://solana.com/rpc) from the user's perspective.

[Solana RPC infrastructure](https://solana.com/docs/rpc) starts with the endpoint URL, but it extends into routing, auth, scaling, failover, and stream delivery. The endpoint URL is just the visible tip. This guide covers what actually sits behind that endpoint URL — how production dApps use RPC for reads, writes, subscriptions, and backend data flows, and when shared infrastructure stops being enough.

## What Is Solana RPC Infrastructure?

Behind that URL sits an RPC node that actually serves those requests: reads, transaction submission, subscriptions. The RPC provider is the managed layer on top, handling regions, auth, scaling, support, and failover.

## Solana RPC Endpoint vs RPC Node vs RPC Provider

These three things are often conflated, but they're functionally different:

| Term | What it is | Why it matters |
|---|---|---|
| RPC endpoint | URL your app sends requests to | Determines cluster, auth, limits, routing |
| RPC node | Node serving JSON-RPC/WebSocket requests | Handles reads, transaction submission, subscriptions |
| RPC provider | Managed service around nodes | Adds regions, auth, scaling, support, failover |
| Public RPC | Shared, open endpoint | Useful for testing, limited for production |
| Private/dedicated RPC | Controlled endpoint for specific team/workload | Better isolation, limits, support, predictability |
| Self-hosted RPC | Your own RPC infrastructure | More control, more operational burden |

One distinction matters here: an RPC node is a different thing from a validator. Solana's own infrastructure page describes RPC nodes as typically dedicated to servicing requests rather than participating in consensus. The software stack overlaps, both run Agave, but the operational goal diverges. Validators optimize for consensus participation. RPC nodes optimize for request latency, indexing, stream delivery, and reliability under external traffic.

Client diversity is also changing the validator layer. Frankendancer is available on mainnet-beta, while full Firedancer remains a separate from-scratch client track. For most RPC architecture decisions, production teams still need to think in terms of RPC API coverage, operational tooling, indexing, failover, and provider support — a new validator client doesn't remove the need for RPC infrastructure planning.

## Why Solana RPC Infrastructure Matters for dApps

The production failure pattern is simple: when an RPC endpoint returns a 429, or a WebSocket subscription silently drops, or a transaction retry loop burns an expired blockhash, users attribute that to the app.

The specific failure patterns worth understanding:

- **Balances stay stale** — slow or lagging node, account reads return outdated state
- **Transactions hang or silently fail** — write path issues, expired blockhashes, missing retry logic
- **Confirmation tracking breaks down** — signature status polling returns stale data
- **WebSocket subscriptions drop** — reconnect storms, missed updates, stale UI
- **Users see inconsistent state** — UI and chain diverge, trust erodes

None of these surface as "infrastructure problems" to an end user. Slow balances, failed swaps, a UI stuck on stale state: that's what RPC failure looks like from the other side of the screen.

## How Production dApps Use Solana RPC Endpoints

The range of what production dApps push through an RPC layer is wider than most teams initially plan for:

- Reading account state and fetching balances
- Fetching token accounts and program-owned accounts
- Simulating transactions before submission
- Sending signed transactions and tracking confirmation status
- Subscribing to account, program, and log updates
- Monitoring slots and blocks
- Supporting indexers and backend jobs

[HTTP JSON-RPC](https://solana.com/docs/rpc/http) covers request-response calls — `getLatestBlockhash`, `getMultipleAccounts`, `simulateTransaction`, `sendTransaction`. [WebSocket covers persistent subscriptions](https://solana.com/docs/rpc/websocket) — `accountSubscribe`, `programSubscribe`, `logsSubscribe`, `signatureSubscribe`. These solve different problems, and running them through the same endpoint without thinking about load separation is a common early mistake.

![Solana RPC traffic split across read, write, WebSocket, and indexing paths](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Solana%20RPC%20Infrastructure%20Guide%20for%20Production%20dApps%20ILL1.png)

<sub>Most production Solana apps end up splitting RPC traffic across separate read, write, WebSocket, and indexing paths — usually after hitting the first real capacity or latency issue on a shared endpoint.</sub>

### Encoding options also affect RPC performance

`base64` is generally the better default for binary account data. `base64+zstd` reduces response size when bandwidth matters and the client can decompress. `jsonParsed` is useful for development and readability but adds parsing overhead and depends on provider support for the specific method. `base58` is not a performance default for binary data. For methods like `getProgramAccounts`, `filters` and `dataSlice` can significantly reduce response size before encoding becomes a concern.

### Commitment Levels: processed, confirmed, finalized

Commitment level controls the trade-off between read freshness and rollback protection. Using one default everywhere is a common source of subtle bugs — a balance read that's fine at `confirmed` can cause problems if used for settlement logic that needs finality.

| Commitment | When to use |
|---|---|
| `processed` | Fastest reads, debugging, highest rollback risk — not suitable for user-facing state |
| `confirmed` | Default for most app UX and transaction flows; balances speed and safety |
| `finalized` | Settlement, accounting, compliance flows where rollback risk is unacceptable |

## Public Solana RPC Endpoints and Production Reality

Solana's documentation says this directly: [public endpoints are shared infrastructure](https://solana.com/docs/references/clusters), rate-limited, and intended for testing rather than production applications. They can return `429` when rate limits are hit, and `403` when an IP address or website gets blocked. Limits can change without notice. High-traffic applications may get blocked without warning.

Public endpoints are genuinely useful for testing, demos, and early experiments. The problem is carrying them into production workloads.

Code examples use `@solana/web3.js` v1 for compatibility with existing Solana codebases. New projects can also evaluate `@solana/kit`, the renamed 2.x line of `@solana/web3.js`.

A minimal retry wrapper for 429 and 403 handling looks like this:

```typescript
type RpcPayload = {
  jsonrpc: "2.0";
  id: string | number;
  method: string;
  params?: unknown[];
};

type RpcSuccess<T> = {
  jsonrpc: "2.0";
  id: string | number;
  result: T;
};

type RpcFailure = {
  jsonrpc: "2.0";
  id: string | number | null;
  error: {
    code: number;
    message: string;
    data?: unknown;
  };
};

const sleep = (ms: number) =>
  new Promise((resolve) => setTimeout(resolve, ms));

export async function rpcCallWithRetry<T>(
  endpoint: string,
  payload: RpcPayload,
  maxRetries = 3
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const response = await fetch(endpoint, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(payload),
    });

    if (response.status === 403) {
      throw new Error(
        "RPC endpoint returned 403. The client, IP, or origin may be blocked."
      );
    }

    if (response.status === 429) {
      if (attempt === maxRetries) {
        throw new Error("RPC endpoint returned 429 after all retries.");
      }

      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number(retryAfter) * 1000
        : 500 * 2 ** attempt;

      await sleep(delayMs);
      continue;
    }

    if (response.status >= 500) {
      if (attempt === maxRetries) {
        throw new Error(`RPC endpoint returned ${response.status}.`);
      }

      await sleep(500 * 2 ** attempt);
      continue;
    }

    if (!response.ok) {
      throw new Error(`RPC request failed with HTTP ${response.status}.`);
    }

    const data = (await response.json()) as RpcSuccess<T> | RpcFailure;

    if ("error" in data) {
      throw new Error(
        `RPC error ${data.error.code}: ${data.error.message}`
      );
    }

    return data.result;
  }

  throw new Error("RPC call failed after all retries.");
}
```

This is still a defensive client-side pattern, not a replacement for private or dedicated RPC. If production traffic regularly hits 429, the infrastructure tier is already wrong.

On shared or public endpoints, this kind of wrapper buys some resilience. On dedicated infrastructure, 429s should be rare in the first place — and when 403 appears, it usually signals a misconfiguration rather than a quota problem.

## Main Solana RPC Infrastructure Options

There are six common infrastructure models, each with real trade-offs:

| Setup | Best for | Pros | Limits / risks |
|---|---|---|---|
| Local validator / local cluster | Development and local testing | Full local control, no external dependency | Development and local testing only |
| Public RPC | Testing, demos, small experiments | Free, easy to start | Rate limits, no SLA, blocking risk |
| Shared provider RPC | Early production, moderate traffic | Managed, easy, usually enough to start | Shared capacity, tier limits, possible noisy-neighbor effects |
| Private/dedicated RPC | Production apps, trading, DeFi, high load | Isolation, stronger limits, support, predictable performance | Higher cost |
| Self-hosted RPC | Teams with infra capability and specific control needs | Full control, custom tuning | High ops burden: hardware, monitoring, upgrades, failover |
| Hybrid/failover | Critical production workloads | Resilience, backup providers, routing control | More complexity, testing required |

For teams with real traffic, hybrid is often the most practical default. A working split: managed shared or dedicated RPC for fast deployment and regional elasticity, a private write gateway for transaction-critical paths, a gRPC stream for backend ingestion, and self-hosted nodes only when account indexing, historical retention, or compliance actually requires it.

A minimal read-path failover pattern can look like this:

```typescript
type RpcEndpoint = {
  name: string;
  url: string;
};

const readEndpoints: RpcEndpoint[] = [
  { name: "primary", url: process.env.RPC_PRIMARY! },
  { name: "secondary", url: process.env.RPC_SECONDARY! },
  { name: "fallback", url: process.env.RPC_FALLBACK! },
];

let nextReadEndpoint = 0;

export async function callReadRpc<T>(payload: RpcPayload): Promise<T> {
  let lastError: unknown;

  for (let i = 0; i < readEndpoints.length; i++) {
    const endpoint = readEndpoints[nextReadEndpoint];
    nextReadEndpoint = (nextReadEndpoint + 1) % readEndpoints.length;

    try {
      return await rpcCallWithRetry<T>(endpoint.url, payload, 1);
    } catch (error) {
      lastError = error;
      console.warn(`Read RPC failed on ${endpoint.name}`, error);
    }
  }

  if (lastError instanceof Error) {
    throw lastError;
  }

  throw new Error("All read RPC endpoints failed.");
}
```

This pattern is acceptable for read-heavy calls, where failing over to another endpoint is usually safe. Do not blindly round-robin transaction submission. Write paths need stricter routing because blockhash freshness, retries, and confirmation tracking are stateful.

Round-robin works for reads. For write paths — transaction submission specifically — a policy-based router is safer, because in-flight blockhash and landing assumptions may differ across providers.

## HTTP RPC, WebSocket, gRPC, and Indexing in Solana Infrastructure

These solve different problems at different layers. Conflating them is where architecture decisions start going wrong.

**HTTP JSON-RPC** is the baseline for deterministic request-response work. It uses JSON-RPC 2.0 over HTTP POST. Good for reads, simulation, and transaction submission.

**WebSocket** handles persistent subscriptions for real-time updates — account changes, logs, signature status, slots, blocks. Still the right choice for browser-facing and wallet UX. Becomes fragile under backend-scale fanout. Subscription and connection limits vary by provider and tier. After a reconnect, apps need to resubscribe and explicitly refresh account state — missed events during the gap won't be replayed. If WebSocket fanout becomes critical ingestion infrastructure rather than a UX layer, it should eventually be separated and replaced or supplemented with gRPC or a dedicated streaming pipeline.

**gRPC / Yellowstone-compatible streams** are built around Geyser-based streaming rather than client-side polling or PubSub fanout. For backend pipelines — indexers, bots, analytics — this is usually the better fit once scale becomes a real concern. We'll cover this separately in a dedicated data streaming guide.

**Indexed and historical data** often requires a separate data layer entirely. Archive access, instruction-level parsing, and analytics use cases usually need something beyond basic RPC. Standard RPC handles low-volume verification fine; production systems doing heavy historical queries need something else.

In practice: WebSocket handles client-facing subscriptions, gRPC covers backend ingestion pipelines, and indexed APIs carry the historical and analytics load that basic RPC wasn't built for.

## Solana RPC Performance Metrics That Actually Matter

"Performance" is one of those words that gets used to mean everything at once. For Solana RPC specifically, these dimensions need to be tracked separately:

1. **Read latency** — how fast account and state queries return (p50/p95/p99, not just median)
2. **Write path / transaction submission** — how quickly signed transactions are forwarded and accepted
3. **Transaction landing and confirmation tracking** — RPC is part of the path, but send-latency and landing probability are different things
4. **WebSocket stability** — reconnect frequency, missed updates, stale UI propagation
5. **Rate limits** — RPS, TPS, method-level limits, credit systems; these affect reliability as much as raw speed
6. **Geographic routing** — region proximity to users, validators, trading systems, or backend infra
7. **Failover behavior** — what actually happens when an endpoint, provider, or node fails
8. **Data availability** — archive access, heavy methods like `getProgramAccounts`, indexed data
9. **Slot lag / freshness** — whether the node is serving state close to the current cluster head. Monitor by comparing `getSlot` against a trusted reference endpoint. A node that's several slots behind during normal operation can return fast responses that are still stale.

A useful RPC benchmark should test these dimensions separately. Median latency from one method in one region does not tell you much; the test has to cover method mix, burst behavior, tail latency, freshness, and stream gaps.

![Solana RPC benchmark workflow](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Solana%20RPC%20Infrastructure%20Guide%20for%20Production%20dApps%20ILL2.png)

<sub>A useful RPC benchmark separates regions, method mix, steady-state load, burst behavior, tail latency, freshness, and stream gaps. Testing `getSlot` at 50 RPS from one region tells you almost nothing about how the endpoint holds up under real application traffic.</sub>

[`getMultipleAccounts` can fetch up to 100 pubkeys](https://solana.com/docs/rpc/http/getmultipleaccounts) in a single request — worth using deliberately, since request collapsing reduces overhead even when providers meter method calls individually.

On the write path, the fundamentals: fetch blockhashes at `confirmed`, track [`lastValidBlockHeight`](https://solana.com/docs/rpc/http/getlatestblockhash), understand that [blockhashes expire after roughly 150 blocks](https://solana.com/developers/guides/advanced/confirmation), usually around 60–90 seconds, disable blind retries with `maxRetries: 0`, and use `skipPreflight: true` only when the transaction has already been validated and the latency trade-off is intentional. For congestion-sensitive flows, priority fees should be an explicit part of the send path: they're set through compute budget instructions such as `ComputeBudgetProgram.setComputeUnitPrice`, and the effective fee depends on both the compute unit price and the compute unit limit.

A blockhash-aware send workflow in TypeScript, following Solana's production guidance:

```typescript
import {
  Commitment,
  Connection,
  Keypair,
  SendOptions,
  Transaction,
} from "@solana/web3.js";

export async function sendWithFreshBlockhash(
  connection: Connection,
  transaction: Transaction,
  signers: Keypair[],
  options: SendOptions = {
    maxRetries: 0,
    skipPreflight: false,
  }
): Promise<string> {
  const commitment: Commitment = "confirmed";

  const { blockhash, lastValidBlockHeight } =
    await connection.getLatestBlockhash(commitment);

  transaction.recentBlockhash = blockhash;
  transaction.sign(...signers);

  const rawTransaction = transaction.serialize();

  const signature = await connection.sendRawTransaction(
    rawTransaction,
    options
  );

  try {
    const result = await connection.confirmTransaction(
      {
        signature,
        blockhash,
        lastValidBlockHeight,
      },
      commitment
    );

    if (result.value.err) {
      throw new Error(
        `Transaction failed: ${JSON.stringify(result.value.err)}`
      );
    }
  } catch (err) {
    // If the blockhash has expired or the transaction is no longer valid,
    // do not retry with the same blockhash.
    // Fetch a fresh blockhash, rebuild the transaction, re-sign, and resend.
    // The transaction cannot be reused after blockhash expiry:
    // recentBlockhash must be updated and signers must sign again.
    throw err;
  }

  return signature;
}
```

After a blockhash expires, the transaction cannot be retried as-is. The send path must fetch a new blockhash, reconstruct the transaction with the updated `recentBlockhash`, and re-sign before attempting another submission.

This example uses a legacy `Transaction` for readability. In production, complex Solana flows often use `VersionedTransaction` and Address Lookup Tables when account lists approach the 1,232-byte transaction size limit; V0 transactions are required to use ALTs.

The "get blockhash and send later" pattern is where a lot of production bugs originate. Tracking expiry explicitly and refreshing on [retry cycles](https://solana.com/developers/guides/advanced/retry) fixes the most common class of silent transaction failures.

At minimum, track basic node health and identity using `getHealth`, `getVersion`, and `getClusterNodes`, then layer in app-level metrics: error rate, slot lag, confirmation latency, WebSocket reconnect frequency, and provider incident data. Neither side alone gives a complete picture.

## Heavy RPC Methods and Their Caveats

Some RPC methods behave differently from standard reads under production load.

`getProgramAccounts` scans all accounts owned by a program. Without `filters` and `dataSlice`, responses can be large enough to cause timeouts or hit provider limits. Many shared RPC tiers restrict, rate-limit, or price heavy scan methods differently from standard reads. Treat raw account scans as an infrastructure risk, not a free primitive.

`getSignaturesForAddress` returns up to 1000 signatures per request and requires cursor pagination using `before` and `until` parameters for anything beyond the first page.

`getBlock` with `transactionDetails: "full"` returns block metadata plus full transaction data. Use it deliberately — it's one of the heavier responses the RPC layer can return.

All three affect latency, cost, and provider limits in ways that don't show up in basic benchmarks. For high-volume indexing or historical queries, the right path is usually a Geyser/gRPC stream or a dedicated indexing layer, not repeated heavy RPC calls.

## Solana RPC Security Risks

Exposed endpoints create a surface area that teams underestimate until something goes wrong. The common failure modes:

- **Exposed API keys in frontend code** — quota draining, abuse, cost spikes
- **No IP or domain allowlisting** — open endpoints get scraped, hammered, and abused
- **No rate limiting on the application layer** — one bad actor can saturate shared quota
- **Bot traffic** — automated crawlers and MEV bots treat public-looking endpoints as fair game
- **Stale frontend assumptions** — hardcoded endpoints that bypass security controls after a provider change

Most managed providers now offer JWT auth, referrer whitelisting, per-method rate limiting, and dedicated endpoint isolation. A "secure RPC endpoint" complements an application security audit rather than replacing it, but it does contain the blast radius when something goes wrong.

## When Shared Solana RPC Is Enough

Shared managed RPC is a reasonable starting point for many teams. It makes sense when:

- The app is in early production with predictable, moderate traffic
- Sub-second updates aren't critical for the core user experience
- The app has limited reliance on WebSocket streams
- The team wants a managed setup without ops burden
- Budget constraints make dedicated infrastructure premature

Before committing to a shared tier, check the actual limits: RPS and TPS caps, method-level restrictions, WebSocket connection limits, region availability, support tier, upgrade path, and pricing model. The headline plan number rarely captures the real production fit.

## When Dedicated Solana RPC Becomes Necessary

Shared infrastructure has a ceiling. Dedicated or private RPC becomes the right call when:

- Production users depend on reliable state reads
- The app is DeFi or trading and sensitive to latency
- There's frequent high-volume transaction submission
- WebSocket subscription load is heavy and stability matters
- Workload has grown beyond shared tier capacity
- Custom region or location requirements exist
- The team needs a real support and SLA relationship

If a workload is already hitting shared endpoint limits, moving to dedicated RPC is an infrastructure decision. The pricing difference is usually the smaller part of the calculation. The question is whether the cost of unreliability (user churn, failed transactions, support load) exceeds the cost of dedicated infrastructure.

### Stake-weighted QoS and dedicated RPC

For write-heavy and latency-sensitive workloads, SWQoS is worth understanding. Solana's QUIC-based transaction forwarding gives stake-weighted priority to a portion of the leader's TPU capacity. In the default model, 80% of leader TPU capacity is reserved for staked connections and 20% for unstaked traffic. A validator can virtually assign stake weight to trusted RPC peers through staked node overrides, which means transactions forwarded from those RPC nodes get prioritized under congestion rather than competing in the general queue.

This requires a trusted relationship between the RPC node and a staked validator — it's not a configuration checkbox on any managed plan, and it doesn't automatically apply to shared endpoints. For latency-sensitive production flows where transaction landing probability under congestion matters, this is worth evaluating as part of the infrastructure decision. We'll cover staked RPC and SWQoS in a separate guide.

## When Self-Hosted Solana RPC Makes Sense

Self-hosting gives the most control — ledger retention, account indexes, private networking, custom stream pipelines. It also inherits the hardest parts of Solana ops.

Self-hosting is worth considering when:

- The team has real infra and DevOps expertise in-house
- Full control over ledger and account data is required
- The workload justifies fixed infrastructure cost
- Compliance or private networking requirements go beyond what managed providers offer

For most dApp teams, the honest answer is that managed private or dedicated RPC is more practical. The operational overhead of self-hosting — hardware, bandwidth, upgrades, snapshots, monitoring, security, failover — is genuinely a different job from building a product.

A typical [Agave RPC node startup](https://solana.com/vi/developers/guides/advanced/exchange), derived from Anza's documented flags and account-index guidance:

```bash
#!/usr/bin/env bash
set -euo pipefail

exec agave-validator \
  --identity /home/sol/validator-keypair.json \
  --ledger /mnt/ledger \
  --accounts /mnt/accounts \
  --full-rpc-api \
  --no-voting \
  --private-rpc \
  --rpc-port 8899 \
  --rpc-bind-address 0.0.0.0 \
  --dynamic-port-range 8000-8020 \
  --entrypoint entrypoint.mainnet-beta.solana.com:8001 \
  --entrypoint entrypoint2.mainnet-beta.solana.com:8001 \
  --entrypoint entrypoint3.mainnet-beta.solana.com:8001 \
  --expected-genesis-hash <MAINNET_GENESIS_HASH> \
  --limit-ledger-size \
  --account-index program-id \
  --account-index spl-token-owner \
  --account-index spl-token-mint
```

`--private-rpc` keeps the node off the public gossip-visible endpoint list. The `--account-index` flags matter for scan-heavy methods: `getProgramAccounts` and SPL token lookups perform poorly without in-memory indexes. The `spl-token-owner` and `spl-token-mint` indexes cover SPL token queries including Token-2022 / Token Extensions accounts. And `--limit-ledger-size` is a practical default because retaining the entire chain history locally is rarely viable for standard RPC use cases.

Account indexes also change the hardware profile. For token-heavy workloads, enabling broad account indexes can push RAM requirements toward the 1 TB range, depending on indexing scope and Agave version. The operational and hardware burden grows quickly once production-grade queries depend on indexed account access — which is one reason self-hosted RPC tends to cost more in practice than the managed alternative it's replacing.

Anza explicitly warns that Docker is generally not recommended for live clusters due to performance overhead. Container setups are reasonable for dev, staging, or carefully controlled internal experiments — not as a default mainnet recommendation.

## Solana RPC Infrastructure Production Checklist

Before choosing between shared, dedicated, or self-hosted infrastructure, work through these questions:

| Question | Why it matters |
|---|---|
| What is our expected RPS/TPS? | Defines shared vs dedicated needs |
| Which RPC methods do we call most? | Heavy methods create cost and latency pressure |
| Do we need WebSocket subscriptions? | Persistent streams require stability and reconnect logic |
| Do we send transactions through this endpoint? | Write path has different reliability needs than reads |
| What commitment level do we use? | Affects freshness vs finality trade-off |
| What happens if the endpoint fails? | Defines failover requirements |
| Do we expose the endpoint in the frontend? | Security and quota abuse risk |
| Do we need archive or indexed data? | Basic RPC may not be enough |
| Which regions matter? | Backend, user, and validator proximity affects latency |
| What support and SLA do we need? | Production incidents need a response path |
| What is our upgrade path? | Public → shared → dedicated → cluster/hybrid |

## Recommended Solana RPC Setup by Workload

| Workload | Recommended setup |
|---|---|
| Local development | Local validator / devnet |
| Demo / hackathon / small test | Public RPC or free tier |
| Early dApp production | Shared managed RPC |
| Consumer app with growing users | Shared RPC with a clear upgrade path, then dedicated |
| DeFi app | Private/dedicated RPC, WebSocket monitoring, failover |
| Trading / latency-sensitive app | Low-latency dedicated RPC, region-aware routing, possibly gRPC/ShredStream |
| Explorer / analytics / indexer | RPC + indexing/historical data layer |
| Large production team | Dedicated cluster or hybrid/multi-provider setup |

## How Solana RPC Pricing Affects Infrastructure Choices

RPC pricing is a capacity-planning problem. The monthly subscription number is just where most teams start looking.

Two Solana RPC plans can have the same headline price and behave very differently in production. One provider may meter usage through credits. Another may count request units. Another prices by compute units, flat RPS, transaction throughput, WebSocket usage, gRPC streams, archive access, or dedicated infrastructure. These units don't map cleanly to each other unless you know what your application actually sends through the RPC layer.

Start with the workload, not the price page.

A wallet, a DeFi app, a trading backend, an analytics dashboard, and an indexer don't consume RPC in the same way. A wallet may care about fast account reads and stable portfolio refreshes. A DeFi app may care more about transaction submission, confirmation tracking, and WebSocket updates. A backend indexer may care less about browser latency and more about stream throughput, historical coverage, and predictable ingestion.

The practical question is: what does one real user action cost?

Map the RPC calls behind a few common flows:

- Initial app load, wallet connect, balance refresh
- Swap or trade execution
- Deposit and withdrawal flows
- Transaction simulation, submission, confirmation tracking
- Live UI updates
- Backend indexing jobs

Then separate the traffic by path.

**Read traffic** includes account reads, token account fetches, program-owned accounts, balances, slots, and block data. This is where batching, caching, and method choice matter. `getMultipleAccounts` can reduce request overhead by fetching many accounts at once, but heavy account scans and repeated polling can still dominate cost.

**Transaction submission** is a different problem entirely. It depends on fresh blockhashes, retry policy, preflight behavior, priority fees, and confirmation tracking. A plan that looks fine for reads can still be weak for a transaction-heavy app if `sendTransaction` throughput, latency, or support during incidents is limited.

**Stream traffic** — WebSocket subscriptions, reconnect behavior, account updates, logs, signatures, slot notifications — can become a separate scaling problem. Backend pipelines may eventually need gRPC instead of polling or WebSocket fanout, and that cost should be evaluated separately from basic HTTP RPC.

The main mistake is comparing average monthly usage only. Production failures usually show up at the edges: traffic spikes, campaign launches, NFT mints, token listings, market volatility, backend job bursts, reconnect storms. A plan can fit the monthly quota and still fail during peak load if the RPS, transaction throughput, WebSocket, or support limits are too low.

A useful pricing check should answer:

- What is our normal RPS, and what is our peak RPS?
- Which RPC methods dominate usage?
- How many requests does one user flow generate?
- How much traffic is backend-driven rather than user-driven?
- Do we rely on WebSocket subscriptions or gRPC streams?
- Do we submit transactions through the same endpoint?
- Do we need archive access or indexed data?
- What happens if traffic doubles during a launch or market event?
- How quickly can we upgrade without changing application code?

Shared RPC is often the right starting point when traffic is predictable and the app can tolerate provider-level limits. Dedicated RPC becomes easier to justify when rate limits, unstable latency, dropped subscriptions, or support delays start costing more than the price difference. Self-hosting is a separate category — it only makes sense when the team is genuinely ready to own hardware, bandwidth, monitoring, upgrades, security, failover, and incident response as a ongoing operational commitment.

The lowest-cost setup is usually the one that fits the workload well enough that the team stops thinking about the infrastructure layer.

For HighTower shared RPC, dedicated node, and dedicated cluster options, see [HighTower RPC pricing](https://www.htw.tech/market/rpc/pricing).

## How to Choose the Right Solana RPC Infrastructure Setup

There is no universal setup here, because workload shape matters more than provider category.

Solana's own docs confirm that public RPC endpoints are shared, rate-limited infrastructure intended for testing — for production workloads, private or dedicated infrastructure is the documented recommendation. Shared RPC stays appropriate longer than teams expect, as long as the workload fits the tier limits and the failure tolerance is real. Dedicated RPC is justified when those thresholds are crossed — rising error rates, rate limit hits, WebSocket instability are usually visible before they become a crisis. Self-hosting is a legitimate path for teams with the operational capacity, but it's an infrastructure decision with real costs, not a shortcut to control.

The right setup follows workload shape, failure tolerance, and how much operational overhead the team can actually absorb.

Use the production checklist above to compare shared, dedicated, and self-hosted setups against your actual workload.

If your team is comparing shared RPC, dedicated nodes, or private Solana RPC clusters, review [HighTower's RPC infrastructure](https://www.htw.tech/market/rpc) and [pricing](https://www.htw.tech/market/rpc/pricing) pages, or reach out with your workload, region, WebSocket, gRPC, and failover requirements.
