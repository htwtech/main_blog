---
title: "RPC Security: Common Attack Vectors and How to Protect Your Solana dApp"
slug: "rpc-security-common-attack-vectors-and-how-to-protect-your-solana-dapp"
date: "2026-05-15"
description: "A practical guide to securing the Solana RPC layer, covering exposed endpoints, method abuse, stale reads, unsafe transaction submission, fragile streams, and self-hosted misconfiguration."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/RPC%20Security_%20Common%20Attack%20Vectors%20and%20%20How%20to%20Protect%20Your%20Solana%20dApp.png"
coverAlt: "RPC Security: Common Attack Vectors and How to Protect Your Solana dApp"
tags: ["solana", "rpc", "security", "dapps", "infrastructure", "typescript"]
---

# RPC Security: Common Attack Vectors and How to Protect Your Solana dApp

Your Solana dApp's RPC layer controls reads, writes, stream subscriptions, and confirmation tracking. If it's misconfigured, attackers don't need to touch your program at all.

RPC security gets treated as a deployment checkbox. Set the endpoint, add the key, ship. But the RPC layer on Solana is the surface through which your app reads balances, fetches account state, submits transactions, tracks confirmations, and streams real-time updates. If that surface is exposed or poorly scoped, attackers can abuse your endpoint, exhaust your quota, push stale state into your UI, or slow transaction throughput until users stop trusting the product. The logs will show nothing obviously wrong.

Solana's current throughput makes this more consequential than it used to be. Sustained high non-vote TPS, blocks running close to the [60M compute unit ceiling](https://blog.syndica.io/deep-dive-solana-onchain-activity-january-2026/) on heavy days. At that scale, weak RPC architecture produces user-facing failures. Slow confirmations, dropped transactions, balances that look fine on the backend and wrong on screen.

## Attack Vector 1: Exposed Endpoints and Leaked Credentials

A cross-ecosystem study of over 8,000 dApps found 422 fully exposed RPC endpoints — the full URL, including the API key, directly accessible from the browser. Only 22% of those had any kind of allowlist in place.

If your RPC URL lives in the frontend bundle, anyone who opens DevTools can find it. Frontend environment variables are not secret once bundled — source maps, client-side error reporting tools, and browser extensions can surface endpoint URLs just as easily. From there they can hammer expensive methods, run broad account scans, drain your monthly quota, or use your endpoint as a free relay. Solana's own documentation is explicit: [public shared endpoints are rate-limited and not intended for production use](https://solana.com/docs/references/clusters), and a private endpoint URL left in client-side code carries the same exposure.

Four things to lock down:

1. Keep your RPC URL and API key server-side; the browser reaches a proxy, not the endpoint itself
2. Run a backend proxy that exposes only what the frontend actually needs
3. Add IP or domain allowlisting at the provider level where possible
4. Rotate credentials on a schedule, not reactively after a suspected breach

Treat an RPC URL like a database connection string.

## Attack Vector 2: Method Abuse and Unconstrained Read Access

Running expensive read methods repeatedly, scanning program accounts broadly, hitting methods the frontend never uses — all of this raises cost and slows responses for real users, often long before anything in monitoring looks unusual.

[`getProgramAccounts`](https://solana.com/docs/rpc/http/getprogramaccounts) is the obvious pressure point. Without filters, and often without `dataSlice`, it can return enormous payloads and runs slowly on shared infrastructure. `dataSlice` reduces the size of returned account data, but a bare scan without filters still forces the node to walk the full account set. Both matter, and neither substitutes for the other.

Yellowstone gRPC streams without tight scoping have a similar cost profile. Large or full-chain subscriptions can push several Gbps during busy periods, and if the client can't keep up, the feed drifts, buffers, or disconnects. Narrow filters aren't just an optimization at that point — they're what keeps the stream usable under load.

OWASP calls this "[Unrestricted Resource Consumption](https://owasp.org/API-Security/editions/2023/en/0xa4-unrestricted-resource-consumption/)". On Solana, the read path carries as much exposure as the write path.

Fixes:

- Allowlist the exact methods the frontend needs, block everything else at the proxy
- Require filters on `getProgramAccounts`; use `dataSlice` to limit returned account data size when the frontend only needs a slice of each account
- Scope WebSocket subscriptions to specific accounts
- Rate-limit per method, since cost varies by orders of magnitude across the available surface

The proxy does not need to be complicated. The important part is that the request is validated before it reaches the upstream RPC provider.

```typescript
const ALLOWED_METHODS = new Set([
  "getAccountInfo",
  "getBalance",
  "getBlockHeight",
  "getLatestBlockhash",
  "getMultipleAccounts",
  "getProgramAccounts",
  "getSignatureStatuses",
  "getSlot",
  "getTokenAccountsByOwner",
  "simulateTransaction",
]);

const MAX_BATCH_SIZE = 10;

function validateRpcPayload(payload: unknown): string | null {
  const calls = Array.isArray(payload) ? payload : [payload];

  if (Array.isArray(payload) && payload.length > MAX_BATCH_SIZE) {
    return "JSON-RPC batch too large";
  }

  for (const call of calls) {
    if (!call || typeof call !== "object" || Array.isArray(call)) {
      return "Invalid JSON-RPC payload";
    }

    const c = call as Record<string, unknown>;

    if (c.jsonrpc !== "2.0") return "jsonrpc must be '2.0'";
    if (typeof c.method !== "string") return "method must be a string";

    if (!ALLOWED_METHODS.has(c.method)) {
      return `method not allowed: ${c.method}`;
    }

    // Solana RPC methods use positional params arrays.
    if (c.params !== undefined && !Array.isArray(c.params)) {
      return "params must be an array";
    }

    if (c.method === "getProgramAccounts") {
      const error = validateGetProgramAccounts(
        c.params as unknown[] | undefined
      );
      if (error) return error;
    }
  }

  return null;
}
```

For `getProgramAccounts`, do not treat `dataSlice` as a replacement for filters. `dataSlice` reduces the returned payload, but a bare scan can still force the node to walk a large account set.

```typescript
function validateGetProgramAccounts(params: unknown[] | undefined): string | null {
  if (!params || typeof params[0] !== "string") {
    return "getProgramAccounts requires a program id";
  }

  const config = params[1];
  if (!config || typeof config !== "object" || Array.isArray(config)) {
    return "getProgramAccounts requires a config object with filters";
  }

  const filters = (config as { filters?: unknown }).filters;
  if (!Array.isArray(filters) || filters.length === 0) {
    return "getProgramAccounts requires filters";
  }

  return null;
}
```

## Attack Vector 3: Resource Exhaustion and Quota Burning

Someone discovers an open endpoint and starts using it as infrastructure, or deliberately spikes costs. Degraded response times, throttled connections, quota overages that force emergency upgrades — and because it builds gradually, it surfaces as a product complaint before anyone identifies it as a security event.

Useful controls:

- Per-IP and per-session rate limits at the proxy layer
- Response caching for state that tolerates staleness — block height, token supply, static account data
- Request batching to cut round-trips without expanding the callable surface
- Per-method traffic monitoring, so a spike in `getProgramAccounts` shows up independently of aggregate counts
- [WAF rules](https://developers.cloudflare.com/waf/rate-limiting-rules/) to intercept automated patterns before they reach the RPC layer

If the app makes heavy historical reads — backfilling transaction history, scanning past program state — those belong on a [dedicated indexing layer](https://www.htw.tech/market/indexing/pricing), with the hot-path transaction endpoint kept separate.

## Attack Vector 4: Stale Reads and Inconsistent Commitment

Stale reads produce real security failures: stale balance checks, duplicate action submissions, UI states showing "confirmed" when the chain hasn't settled. The failure often registers as a UX bug before anyone traces it back to a state consistency problem.

Solana's [three commitment levels](https://solana.com/docs/rpc#configuring-state-commitment) — `processed`, `confirmed`, `finalized` — carry meaningfully different finality guarantees. Apply a single level everywhere and leave it unexamined, and you end up with a backend reading at one commitment, a stream consuming another, and transaction logic trusting a third. The state the app sees becomes a composite of three different chain views.

There's also a [pooled backend failure mode](https://solana.com/developers/guides/advanced/retry) worth knowing. Blockhashes fetched from an advanced node and submitted through a lagging one look valid but get dropped silently — the submitting node hasn't seen the blockhash yet. Solana's retry guide documents this explicitly.

Commitment in practice:

- Define commitment per flow. Balance checks, transaction submission, and confirmation display each carry different freshness requirements
- Use `minContextSlot` on reads where ordering matters
- Check `context.slot` in responses when application logic depends on recency
- Avoid mixing backend views across commitment levels in the same flow without understanding the tradeoffs

A related assumption worth examining: an RPC response being well-formed does not mean it reflects current chain state. A lagging, misconfigured, or compromised endpoint can omit recent state, return a stale slot context, fail to surface a signature status that already exists, or cause two parts of the same app to disagree about the same account. For flows where that disagreement has consequences — balance gates, settlement logic, conditional actions — compare `context.slot` across responses, enforce `minContextSlot`, and avoid mixing providers within a single transaction lifecycle unless the app has explicit consistency checks between them.

## Attack Vector 5: Unsafe Transaction Submission

[`sendTransaction`](https://solana.com/docs/rpc/http/sendtransaction) returning a signature means the RPC node accepted the submission. Accepted means the RPC service received the transaction for submission. It does not mean a leader saw it, processed it, confirmed it, or finalized it — those are separate outcomes that require explicit tracking.

If an app updates internal state on acceptance and skips polling entirely, it creates exploitable trust gaps: duplicate intents, false success states, mis-settlement that's hard to untangle after the fact.

Blockhash expiry compounds this. [Blockhashes expire after roughly 150 slots](https://solana.com/developers/guides/advanced/confirmation). Without tracking that window explicitly, an app can keep resubmitting an already-expired transaction while showing "pending" to the user.

Solana's production readiness guidance recommends [`maxRetries: 0`](https://solana.com/docs/payments/production-readiness) when the application is prepared to manage retry behavior itself. At minimum, the app should track confirmation and blockhash expiry rather than treating RPC acceptance as success.

The implementation is straightforward: send with `maxRetries: 0`, poll [`getSignatureStatuses`](https://solana.com/docs/rpc/http/getsignaturestatuses), and stop once [`getBlockHeight`](https://solana.com/docs/rpc/http/getblockheight) passes `lastValidBlockHeight`.

```typescript
// send-and-track.ts
// Node 20+

type Commitment = "processed" | "confirmed" | "finalized";

type JsonRpcResponse<T> =
  | { jsonrpc: "2.0"; id: number; result: T }
  | { jsonrpc: "2.0"; id: number; error: { code: number; message: string } };

type SignatureStatus = {
  confirmationStatus?: Commitment;
  err: unknown | null;
} | null;

let nextId = 1;

async function rpc<T>(url: string, method: string, params: unknown[] = []): Promise<T> {
  const id = nextId++;

  const resp = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ jsonrpc: "2.0", id, method, params }),
  });

  if (!resp.ok) throw new Error(`HTTP ${resp.status} from RPC`);

  const body = (await resp.json()) as JsonRpcResponse<T>;
  if ("error" in body) throw new Error(`${body.error.code}: ${body.error.message}`);

  return body.result;
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

export async function sendAndTrack(params: {
  rpcUrl: string;
  signedBase64Tx: string;
  lastValidBlockHeight: number;
  abortAfterMs?: number;
}): Promise<string> {
  const {
    rpcUrl,
    signedBase64Tx,
    lastValidBlockHeight,
    abortAfterMs = 75_000,
  } = params;

  const signature = await rpc<string>(rpcUrl, "sendTransaction", [
    signedBase64Tx,
    {
      encoding: "base64",
      maxRetries: 0,
      skipPreflight: false,
      preflightCommitment: "confirmed",
    },
  ]);

  const startedAt = Date.now();

  while (Date.now() - startedAt < abortAfterMs) {
    const statuses = await rpc<{ value: SignatureStatus[] }>(
      rpcUrl,
      "getSignatureStatuses",
      [[signature], { searchTransactionHistory: false }],
    );

    const status = statuses.value[0];

    if (status?.err) throw new Error(`Transaction failed: ${JSON.stringify(status.err)}`);

    if (
      status?.confirmationStatus === "confirmed" ||
      status?.confirmationStatus === "finalized"
    ) {
      return signature;
    }

    const currentBlockHeight = await rpc<number>(rpcUrl, "getBlockHeight", [
      { commitment: "confirmed" },
    ]);

    if (currentBlockHeight > lastValidBlockHeight) {
      throw new Error("Blockhash expired. Rebuild, re-sign, and retry with a fresh blockhash.");
    }

    await sleep(400);
  }

  throw new Error("Timed out waiting for confirmation.");
}
```

This helper does not rebuild or retry the transaction. It shows the minimum safe tracking pattern after submission: poll status, check expiry, and stop showing "pending" once the blockhash window has passed. Capture `lastValidBlockHeight` at the build step and pass it through.

For high-value flows, [stake-weighted QoS](https://solana.com/developers/guides/advanced/stake-weighted-qos) and direct validator routing are worth examining separately from [endpoint selection](https://www.htw.tech/market/rpc/pricing). Both affect whether a transaction lands under congestion, but neither is something a frontend toggles with a normal JSON-RPC parameter. They belong to provider selection, endpoint topology, and routing strategy.

## Attack Vector 6: Fragile Stream Handling

A dropped WebSocket or gRPC stream that goes unhandled opens a state gap. In a trading app, a wallet, or an indexing backend, decisions get made on whatever the app last received before the connection went quiet.

[WebSocket subscriptions](https://solana.com/docs/rpc/websocket/accountsubscribe) are connection-scoped in practice. After a reconnect, the subscription ID from the previous session is gone. The app needs to re-subscribe explicitly and backfill any state that may have changed while the connection was down, typically with a [`getAccountInfo`](https://solana.com/docs/rpc/http/getaccountinfo) call before resuming stream processing.

Yellowstone gRPC subscriptions carry the same requirement. Large unfiltered streams can be throttled, which creates gaps without a clean reconnect signal.

For high-throughput feeds, a few operational details matter: RTT under roughly 50ms where possible, adaptive HTTP/2 flow control enabled, and message processing kept off the receive loop. Doing heavy work synchronously in the handler drops events under load before anything else signals a problem.

Serverless functions are a poor fit for long-running stream consumers. For Lambda or GCF-style execution environments, the mismatch is structural rather than a simple tuning issue. Execution time limits and unpredictable cold-start behavior will produce missed events regardless of how the subscription is configured.

Stream hardening:

- Heartbeat and reconnect logic with explicit re-subscription on reconnect
- State backfill after any gap before resuming processing
- Filters scoped to specific accounts
- Stream ingestion on separate infrastructure from request-response RPC
- Long-lived persistent processes for anything consuming a stream

## Attack Vector 7: Self-Hosted RPC Misconfiguration

Running a Solana validator or RPC node extends the attack surface below the application layer. Exposed admin ports, processes running as root, SSH with default credentials, no firewall rules, no intrusion detection — these are infrastructure-level exposures that sit outside what application security covers.

Running your own validator or RPC node makes host-level hardening part of the security model. The items that tend to slip:

- Admin and metrics ports closed to the public internet — accessible over private networking or VPN only
- RPC and validator services running under dedicated non-root users
- Key-based SSH authentication, password auth disabled
- `fail2ban` or equivalent for brute-force protection
- Alerting on auth failures, unexpected inbound connections, and resource anomalies
- Private networking between the validator and any backend services communicating with it

On a managed or shared RPC tier, most of this is handled at the infrastructure level. On dedicated or self-managed nodes, go through this list explicitly.

## What a Hardened Solana dApp RPC Stack Looks Like

The browser reaches a backend proxy, scoped to the methods the frontend actually needs. Origin checks, per-IP rate limits, request timeouts — enforced at the proxy. The full endpoint URL stays server-side.

Hot-path reads go to one endpoint. Historical reads and backfill queries go to a dedicated indexing layer, keeping the transaction path free from competing load.

Streams run in persistent processes on separate infrastructure. The send path carries its own confirmation loop and blockhash expiry handling.

Commitment levels are set per flow. `context.slot` and `minContextSlot` are in use wherever ordering matters. Credentials rotate on a schedule.

Per-method traffic monitoring should be active — not just aggregate request counts. The signals worth tracking:

- 401, 403, and 429 response rates, per endpoint
- Requests per second broken out by method
- `getProgramAccounts` call count and response latency independently
- WebSocket reconnect frequency
- Missed slots or signatures in stream consumers
- `sendTransaction` accepted versus confirmed ratio over time
- Blockhash expiry failures from the retry loop
- `simulateTransaction` error categories
- Slot lag relative to a reference endpoint

A spike in any of these in isolation is usually noise. A spike in two or more at once is worth investigating before it becomes a billing or availability event.

```typescript
async function forwardRpc(payload: unknown, rpcUrl: string): Promise<Response> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 2_500);

  try {
    return await fetch(rpcUrl, {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(payload),
      signal: controller.signal,
    });
  } finally {
    clearTimeout(timeout);
  }
}
```

Origin checks are useful for browser traffic, but they are not authentication. Non-browser clients can omit or spoof `Origin` headers. In production, combine this pattern with provider-level allowlists, signed sessions or JWTs, edge rate limits, and persistent rate-limit state such as Redis. Behind a reverse proxy, derive client IP only from trusted forwarded headers, or enforce rate limits at the edge.

In production, rate-limit state wants persistent storage — Redis rather than in-process. Structured per-method logging makes monitoring actionable. If read and write traffic need separate upstream endpoints, split the proxy routes rather than routing everything through one handler.

When reads, writes, history, and streams share one endpoint, any workload spike affects the others. A send path that degrades under congestion produces the same user-facing outcome as one that's been tampered with.
