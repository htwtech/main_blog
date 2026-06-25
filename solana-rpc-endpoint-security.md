---
title: 'DApp Endpoint Security for Solana RPC'
slug: 'solana-rpc-endpoint-security'
canonical_url: 'https://supanode.xyz/blog/solana-rpc-endpoint-security'
date: "2026-06-22"
description: "Learn how Solana dApps can secure RPC endpoints against API key exposure, endpoint abuse, quota draining, stale reads, unsafe transaction submission, and weak access controls."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/DApp%20Endpoint%20Security%20for%20Solana%20RPC.png"
coverAlt: "DApp Endpoint Security for Solana RPC"
tags: ["RPC", "security", "RPC Endpoints", "Solana rpc"]
---

Solana dApp security tends to focus on the program — audit it, lock the upgrade authority, ship. That work matters. But dApp endpoint security is a separate surface, and one we spend a lot of time on while running RPC infrastructure.

The endpoint handles reads, writes, confirmation tracking and live subscriptions. When it gets exposed, abused, or starts serving stale state, an audited program won't fix the resulting UX.

So this guide stays on the RPC and API layer. Wallet safety and contract auditing are out of scope. What's in scope is the part that mediates what users see, what they submit, and the moment the app decides something is real.

## What DApp Endpoint Security Means on Solana

Endpoint security here means the security of the [RPC and API layer](/blog/solana-api-rpc-guide) the application depends on. Solana's docs describe that surface plainly: it's where you [read network state, send transactions, simulate execution and subscribe to live updates](https://solana.com/docs/rpc).

An audited program can still sit behind fragile infrastructure. The contract verifies fine, the front end loads, and then a leaked endpoint key gets reused by a bot until the quota is gone and real users start collecting `429` errors instead of state.

[The RPC service sits between the dApp and the chain](https://www.ndss-symposium.org/ndss-paper/as-strong-as-its-weakest-link-how-to-break-blockchain-dapps-at-rpc-service/) as an intermediary that can become the weak link regardless of contract quality, which is how it's been studied directly.

## Why RPC Endpoints Become a Security Boundary

The RPC endpoint handles most user-facing operations:

* account and balance reads  
* token account queries  
* transaction simulation  
* transaction submission  
* confirmation and status tracking  
* WebSocket subscriptions for live updates  
* backend indexers and analytics jobs  
* frontend state rendering

For the wider stack, see our guide to [Solana RPC infrastructure](/blog/solana-rpc-infrastructure-guide).

Providers differ in what they return and when — a [2025 measurement of third-party RPC providers](https://dl.acm.org/doi/10.1145/3730567.3768594) recorded delays and missing data across providers on another chain. When the app trusts an endpoint's view, the provider affects correctness, not only availability.

## Public RPC Is Not a Production Security Model

Public endpoints are shared infrastructure, and Solana states they're [not intended for production applications](https://solana.com/docs/references/clusters). The mainnet public endpoint runs behind a load balancer with documented limits, and those limits can change without notice. The docs map the two codes you'll actually hit:

* **`403`** means your IP or site has been blocked  
* **`429`** means you've crossed the rate limit

Public RPC works for experimentation, debugging and low-stakes traffic. Once user-facing reads, drops, trading flows or backend jobs are involved, the shared limits become a problem, and private RPC infrastructure with controlled limits is the usual answer.

## Risk 1: Exposed RPC URLs and API Keys

API key exposure travels through frontend bundles, browser DevTools, source maps, public GitHub repos, log lines, client-side monitoring tools, shared staging keys, and environment variables baked into the frontend build. QuickNode states it directly in their endpoint security guide: client-side .env values stay visible after the build ships.

Anything that reaches the browser should be treated as readable by anyone, with key rotation and access rules doing the defending.

When that key gets reused outside your app:

* quota exhaustion  
* unexpected provider costs  
* degraded performance for real users  
* higher error rates  
* abuse of paid methods  
* a support burden that ends up on your team

Helius makes the same case in their [key protection docs](https://www.helius.dev/docs/rpc/protect-your-keys) — exposed keys lead to unauthorized usage, quota exhaustion and unexpected charges, and they recommend a proxy that keeps the key server-side.

[Browser-facing layers also leak](https://arxiv.org/pdf/2306.08170). Research on Web3 privacy measured large numbers of address leaks across dApps and wallet-probing scripts on thousands of mainstream sites, so frontend exposure should be assumed inspectable and reusable.

## Risk 2: Endpoint Abuse and Method Overuse

Some RPC methods carry far more blast radius than others, and `getProgramAccounts` is the Solana example to watch first.

The [method docs](https://solana.com/docs/rpc/http/getprogramaccounts) show it accepts `commitment`, `minContextSlot`, `encoding`, `dataSlice` and `filters`, so you can narrow result shape and context on purpose. Used broadly, it can scan a large account set, which is what makes it expensive and slow when the query has no filters behind it.

`dataSlice` helps with the payload but not with the scan. It reduces how much account data comes back to the client; it does nothing about how much the node had to walk to find those accounts. A broad, unfiltered `getProgramAccounts` reachable from browser or mobile traffic should be treated as a high-blast-radius read path regardless of slicing.

Common abuse patterns:

1. heavy reads with no filters and no `dataSlice`  
2. high-frequency polling from untrusted clients  
3. frontend and backend sharing a single key  
4. zero method-level controls

Method access should be explicit. Browser-facing traffic should only reach the RPC methods the UI needs. Heavier reads either require filters, move behind backend-only authorization, or run through a separate indexing path instead of competing with the user-facing RPC traffic. An allowed endpoint shouldn't mean every RPC method is allowed, and a shared unrestricted key across frontend and backend removes the isolation that makes any of this containable.

Provider-side controls cover part of this. QuickNode documents method whitelisting and method-specific rate limits. Chainstack documents origin and IP rules in their [access rules](https://docs.chainstack.com/docs/access-rules). Helius covers domain, IP and CIDR controls. The control surface is which methods are callable, from which path, with which parameters, and at what rate.

### A method policy is better than a blind proxy

A backend proxy that forwards every JSON-RPC request unchanged buys you very little. The proxy should validate before anything reaches the upstream provider:

* Is the payload valid JSON-RPC?  
* Is batch size limited?  
* Is the method on the allowlist?  
* Are `params` shaped as expected?  
* Is the method allowed for this caller class?  
* Does `getProgramAccounts` include a program id?  
* Does `getProgramAccounts` include a config object?  
* Does `getProgramAccounts` include non-empty filters?  
* Is the method rate-limited separately from cheaper ones?

| Control | Why it matters |
| :---- | :---- |
| Method allowlist | Blocks RPC methods the frontend does not need |
| Batch size limit | Prevents one request from carrying too many calls |
| `params` validation | Stops malformed or unexpected upstream calls |
| `getProgramAccounts` filters | Prevents broad account scans from untrusted paths |
| `dataSlice` for partial reads | Reduces returned payload when the UI needs part of account data |
| Method-specific rate limits | Keeps heavier methods from consuming the whole endpoint budget |
| Separate frontend/backend keys | Limits blast radius when browser-facing traffic is abused |

An illustrative policy layer — narrow on purpose:

```ts
const ALLOWED_METHODS = new Set([
  "getAccountInfo",
  "getBalance",
  "getLatestBlockhash",
  "getSignatureStatuses",
  "getTokenAccountsByOwner",
  "simulateTransaction",
]);

function validateRpcMethod(method: string): boolean {
  return ALLOWED_METHODS.has(method);
}
```

For heavier methods such as `getProgramAccounts`, the proxy should require application-level constraints: a known program id, non-empty filters, bounded batch size, and method-specific rate limits. 

This policy doesn't try to make every possible Solana RPC call safe. It makes the frontend-facing surface explicit, so anything outside the allowlist, too broad, malformed, or missing expected filters never reaches the upstream endpoint.

The same logic applies to streaming paths. Large or poorly filtered Yellowstone gRPC subscriptions become expensive and unstable under load, and filters there aren't only an optimization — they're what keeps a high-throughput stream usable when the app is processing live account or transaction updates.

## Risk 3: Quota Draining and Availability Failures

Endpoint problems usually show up as degraded UX before anything looks like a breach, which makes them easy to misdiagnose.

A typical sequence: an exposed endpoint gets reused by scripts and bots, real users start hitting 429s, backend retries amplify the load, reconnects pile on, and the bill rises. From the inside it reads as a slow provider, when the cause is the exposed endpoint.

[Chainstack's endpoint security research](https://chainstack.com/unmasking-dapp-endpoint-security-research-results) found exposure across a large dApp sample, with only a minority of fully exposed endpoints using whitelisting. It's cross-chain data, but enough to make exposure an architecture concern.

Rate limiting caps misuse but doesn't block access to the endpoint itself.

## Risk 4: Stale Reads and Commitment Mismatch

This is the Solana-specific part generic endpoint-security guides usually miss.

Solana defines three commitment levels, each with its own trade-off:

* **`processed`** is newest and fastest, though it can be rolled back  
* **`confirmed`** reflects supermajority voting and works as a common user-facing UX compromise  
* **`finalized`** is the strongest state, the right choice for accounting and settlement

Omit commitment and the default is typically `finalized`. Mixing explicit overrides with inherited defaults across read paths produces a UI that disagrees with itself.

In financial flows this becomes an integrity problem: wrong balances, false pending states, a user who clicks twice because the first action looked like it never landed. A delayed provider view can cause the same duplicate submission.

`minContextSlot` controls this. Both `getProgramAccounts` and `getLatestBlockhash` accept it, which makes read freshness an explicit setting. Set commitment per product flow, because different flows need different levels.

## Risk 5: Unsafe Transaction Submission

A successful sendTransaction means the node received the transaction. The cluster still has to process and confirm it, and that has to be checked separately.

getLatestBlockhash returns a lastValidBlockHeight, isBlockhashValid reports whether a blockhash still holds at the requested commitment, and getSignatureStatuses searches only the recent status cache unless searchTransactionHistory is on:

```ts
const signature = await connection.sendRawTransaction(serializedTx);

const status = await connection.getSignatureStatuses([signature]);

if (status.value[0]?.err) {
  throw new Error("transaction failed");
}

if (!status.value[0]?.confirmationStatus) {
  // Keep polling or subscribe until the target commitment is reached.
  // Do not show final success yet.
}	
```

Don't show irreversible success until the status loop confirms it. signatureSubscribe can speed up the UX, with the status loop and blockhash-expiry check as the source of truth.

## Risk 6: Weak WebSocket and Subscription Handling

WebSocket subscriptions fail in ways that are easy to miss: dropped subscriptions, stale frontend state after a reconnect, missed updates, reconnect storms, with the WSS path often monitored separately from HTTP RPC or not at all. There's also a control gap — QuickNode notes that domain whitelisting can't apply to WSS connections, so a control that covers HTTP traffic isn't there for browser subscriptions.

A dropped subscription leaves a state gap. The app no longer knows whether the account, signature, or program state changed while the connection was down. After reconnect, the safe sequence is explicit: reconnect, re-subscribe, then backfill state before resuming stream-driven logic. For account state, that backfill can be a fresh HTTP read such as `getAccountInfo`. For event or transaction consumers, it means comparing the last processed slot or signature with the current view before trusting the stream again.

Browser-side subscriptions help with responsive UI, but they're a weak source of truth for critical state. A tab can sleep, a mobile connection can flap, a subscription can drop, and the user keeps seeing the last rendered state. For flows that touch balances, settlement, transaction status, or user decisions, the app should keep an HTTP backfill or a server-side source of truth behind the subscription. A `receivedSignature` notification still falls short of final confirmation, so subscriptions support the UX without standing in for confirmation tracking.

For backend ingestion, Yellowstone gRPC carries the same basic hardening requirement: scope the stream. Large unfiltered streams become unstable when the client can't keep up, and under load the symptom looks like drift, buffering, throttling, or disconnects rather than a clean error. Scoped filters, heartbeat logic, reconnect handling, and backfill belong in the reliability and security model, not only in performance tuning.

### Stream hardening checklist

*  Add heartbeat and reconnect logic  
*  Re-subscribe explicitly after reconnect  
*  Backfill state after any gap before resuming stream-driven decisions  
*  Scope subscriptions to specific accounts, programs, or event classes  
*  Keep message processing off the receive loop for high-throughput feeds  
*  Monitor reconnect frequency and missed updates  
*  Run important stream consumers as long-lived workers  
*  Avoid serverless functions for long-running stream consumers  
*  Separate stream ingestion from normal request-response RPC traffic  
*  Don't rely on WebSocket notifications as the only confirmation source for critical transactions

Serverless functions are usually a poor fit for long-running stream consumers. Execution time limits, cold starts, and lifecycle interruptions create reliability problems that no amount of subscription tuning fixes. If the stream matters, run it as a persistent worker and build reconnect and backfill into the consumer from the start.

Build for the stream breaking. Detect the gap quickly, re-subscribe explicitly, and rebuild state before decisions continue.

## Risk and Control Matrix

With all six risks on the table, here's the compressed view:

| Risk | Production impact | Mitigation |
| :---- | :---- | :---- |
| Exposed key | quota exhaustion, cost increase | backend proxy, access rules, key rotation |
| Heavy method abuse | latency, `429`s, degraded UX | method allowlist, method rate limits |
| Stale reads | inconsistent UI / state | commitment strategy, `minContextSlot` |
| Unsafe submission | false success, duplicate actions | confirmation tracking, `getSignatureStatuses` |
| WebSocket drops | stale frontend | reconnect logic, monitoring, backend sync |

## Endpoint Control Layers for Solana dApps

Each layer covers part of the problem, so they're used together.

| Layer | What it protects | Example |
| :---- | :---- | :---- |
| Backend proxy | Keeps provider key server-side and validates requests | Browser calls your app API, not the raw private RPC |
| IP allowlist | Restricts server-side traffic | Backend workers only |
| Domain / referrer rules | Reduces casual frontend abuse | Production and staging domains |
| Method allowlist | Restricts callable RPC methods | Only app-required methods accepted |
| Method rate limits | Contains expensive calls | Stricter limits for heavy reads |
| Separate keys | Limits blast radius | frontend, backend, staging, production |
| Key rotation | Reduces leaked-key lifetime | rotate on suspicious usage |
| Monitoring | Detects abnormal usage | method spikes, `429`/`403` growth, geo anomalies |
| Dedicated / private RPC | Isolates production workload | separate limits, support, traffic path |

Domain and referrer rules help, but they fall short as a standalone defense. QuickNode notes that domain whitelisting can be spoofed by scripts and skips WSS entirely. For sensitive flows, a backend proxy with server-side credentials carries the load far better.

## Recommended Architecture for a Secure Solana RPC Setup

A secure setup separates traffic, keys, methods, limits and monitoring across the path:

```
Browser
  → app backend / RPC proxy
    → protected RPC endpoint
      → Solana RPC node / managed provider
        → monitoring / logs / alerts
```

Split the traffic across the path:

* frontend reads  
* backend reads  
* transaction submission  
* WebSocket subscriptions  
* indexer / historical data jobs

The backend facade does the work — the browser hits your app surface, the app validates the request and forwards it:

```ts
app.post("/api/solana/get-balance", async (req, res) => {
  const { pubkey } = req.body;

  const rpcBody = {
    jsonrpc: "2.0",
    id: crypto.randomUUID(),
    method: "getBalance",
    params: [pubkey, { commitment: "confirmed" }],
  };

  const upstream = await fetch(process.env.SOLANA_PRIVATE_RPC_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(rpcBody),
  });

  const data = await upstream.json();

  return res.json(data);
});
```

This is deliberately narrow: the browser calls an app endpoint instead of the raw RPC URL. A real proxy should also validate public keys, add authentication, enforce rate limits, log abnormal method usage, and return sanitized errors. 

With the split, a leaked or abused credential reaches one slice of the traffic, not the whole endpoint.

## Production Checklist

Checklist before shipping:

*  Treat frontend RPC keys as public identity, defended by rotation and access rules  
*  Use separate keys for development, staging and production  
*  Keep sensitive RPC credentials server-side  
*  Route sensitive flows through a backend proxy  
*  Restrict server-side endpoints by IP  
*  Use domain / referrer rules as one layer among several  
*  Add method allowlists where you can  
*  Apply method-specific rate limits to heavy methods  
*  Bound `getProgramAccounts` usage with filters, `dataSlice` and clear commitment  
*  Define commitment level per product flow  
*  Track transaction confirmation explicitly  
*  Verify the cluster before showing final success, regardless of the `sendTransaction` response  
*  Rotate keys regularly and keep an incident procedure ready for leaks  
*  Monitor method mix, `429`/`403` errors, latency and WebSocket reconnects  
*  Move to dedicated / private / managed RPC once shared limits touch production UX

## When Managed or Dedicated RPC Becomes Necessary

Managed or dedicated RPC becomes worth it when endpoint failure starts to affect business outcomes, more than at any particular traffic level.

That point arrives when:

* public or shared RPC limits begin affecting real users  
* a leaked endpoint key would damage production rather than just a test  
* transaction landing turns business-critical  
* WebSocket stability shapes the experience  
* you need method-level controls and workload isolation  
* backend and indexer traffic keeps climbing  
* the team wants monitoring, support and incident response without running RPC in-house

If your Solana dApp depends on RPC traffic for user-facing reads, transaction submission, WebSocket subscriptions or backend jobs, endpoint protection belongs in the infrastructure plan. Supanode runs managed [Solana RPC service setups](https://supanode.xyz/pricing) with private access, workload isolation, monitoring and controls shaped around the real traffic profile. 
