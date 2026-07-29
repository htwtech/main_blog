---
title: 'How to Land Transactions on Solana: Fees, SWQoS, and Retries'
slug: 'solana-transaction-landing-guide'
canonical_url: 'https://supanode.xyz/blog/solana-transaction-landing-guide'
date: "2026-07-25"
description: 'Understand why Solana transactions fail to land, how priority fees, SWQoS, retries, and confirmation tracking work, and how to improve landing rate in production.'
author: "Ilya Novikov"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/network_logos/Ilya_Novikov.jpg"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/How%20to%20Land%20Transactions%20on%20Solana_%20Fees,%20SWQoS,%20and%20Retries.png"
coverAlt: "How to Land Transactions on Solana: Fees, SWQoS, and Retries"
tags: ["Land Transactions", "Solana Sender", "SWQoS"]
---

A signature comes back from sendTransaction, and that's usually where people relax — the call succeeded, so the transaction must be fine. On Solana that assumption breaks constantly. We've sat with teams watching a block explorer refresh every few seconds, signature already in hand, waiting on a transaction that's already gone: the blockhash expired somewhere in that gap, and the transaction quietly stopped being valid before anyone caught it. 

Transaction landing on Solana means a signed transaction gets included and processed in a block on the active fork, not merely accepted by an RPC endpoint. A `sendTransaction` response confirms that a node accepted the payload for forwarding; it says nothing about whether a leader ever scheduled, executed, or included it. Landing has to be verified separately, by tracking the signature until it reaches the commitment level the application actually needs.

That gap between "accepted" and "landed" is where submission path, retry logic, fee policy, and confirmation tracking have to be evaluated separately.

## What transaction landing means on Solana

Solana transaction landing is easier to debug when the lifecycle is split into separate stages. Each stage answers a different question. Conflating them makes transaction failures harder to classify:

* Accepted — an RPC node or sender accepted the signed payload for forwarding. This is what a successful `sendTransaction` HTTP response actually tells you, and nothing more.  
* Forwarded — the payload was sent onward toward a validator's ingress or a relay such as a Jito Block Engine.  
* Landed / processed — the transaction was included and processed in a block on some fork. This is the first point where "it happened on-chain" becomes true.  
* Confirmed — the block carrying the transaction received stronger cluster commitment, supermajority-level confidence that the fork containing it is the one that persists.  
* Finalized — the block reached finality and is no longer subject to reorganization under normal operation.

One distinction matters before looking at the submission path: a transaction can land, consume the fee, and still fail during execution. Inclusion in a block and successful execution are separate events, and the failure section below treats them as such.

## The path from a signed transaction to block inclusion

client → RPC or sender → leader ingress → [validation and scheduling](https://solana.com/docs/core/transactions/transaction-pipeline) → execution → block inclusion → confirmation tracking

Standard RPC routes transactions through the shared ingress lane. SWQoS senders use [staked peering relationships](https://solana.com/developers/guides/advanced/stake-weighted-qos) to reach leader ingress with preferential capacity instead of relying only on the general RPC path, while Jito operates on a separate Block Engine auction with distinct bundle semantics. Not every transaction needs all three — most ordinary application traffic never touches Jito at all, and that's fine.

One current implementation detail matters for transaction submission: normal Agave transaction ingress is QUIC-based. Older guidance around direct UDP TPU submission should be checked against current validator behavior before it is used in a production sender.

![Diagram showing the Solana transaction landing path from a signed transaction through standard RPC, SWQoS-aware sender or Jito Block Engine to leader ingress, validation, execution, block inclusion, and commitment tracking.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/SOLANA%20TRANSACTION%20LANDING%20PATH.png)

<sub>Transaction submission paths and landing lifecycle.</sub>

Transaction landing sits on top of the broader [Solana RPC infrastructure](/blog/solana-rpc-infrastructure-guide) layer. A fast response from a [low-latency Solana RPC](/blog/low-latency-solana-rpc) endpoint confirms that the node answered quickly; whether a leader received, scheduled, or included the transaction is a separate question.

## Why sendTransaction success does not mean the transaction landed

A returned signature from [`sendTransaction`](https://solana.com/docs/rpc/http/sendtransaction) merely acknowledges that an RPC node accepted the payload. It leaves open whether the transaction ever reached a leader, entered the schedule, or executed.

Practically, this means the application layer has to do work the RPC call doesn't do for you: track the signature explicitly, watch for the `processed` → `confirmed` transition, and know when to give up because the blockhash has expired rather than waiting indefinitely on a transaction that isn't coming back. When `sendTransaction` returns a signature but confirmation never arrives, start by checking the confirmation loop and blockhash expiry handling.

## Why Solana transactions fail to land

Failures cluster fairly cleanly by which layer of the pipeline they happen in, and diagnosing in this order tends to rule out the cheaper causes first:

* Invalid or poorly constructed transaction. Bad or missing signatures, simulation errors, an underfunded fee payer, a compute-unit limit set too low for the actual instructions, or an instruction error inside a program. Simulation catches most of this before a fee is ever paid for it.  
* Stale blockhash or expired validity window. Every transaction carries a recent blockhash with a limited validity window, tracked through [`lastValidBlockHeight`](https://solana.com/developers/guides/advanced/confirmation) rather than a wall-clock timer. A lagging RPC node can make a technically valid blockhash look expired well before a fixed timer would say so.  
* Weak delivery path. An overloaded endpoint, a region mismatch between client and leader schedule (a client sending from a distant region into a poorly aligned leader path), a single submission path with no fallback, or no stake-backed ingress at all during a congested window.  
* Insufficient scheduler priority. A missing or underpriced priority fee, an oversized compute-unit request that hurts packing efficiency, account contention with other transactions competing for the same writable accounts, or a general spike in fee pressure on the accounts being touched.  
* Incorrect retry or confirmation logic. Stopping retries too early, treating an HTTP 200 as proof of completion, or rebuilding and re-signing before the original transaction has actually expired. [Rebuilding and re-signing before expiry](https://solana.com/developers/guides/advanced/retry) can create two distinct valid transactions that race each other, with downstream state effects that are harder to reason about if both land.  
* Included but failed during execution. An included transaction that fails during execution is a different failure class. The transaction reached a block, consumed its fee, and then failed inside the program logic — insufficient balance for the operation, a state precondition that no longer held, a program-level assertion. That needs execution logs, not a look at the submission path.

## Priority fees, SWQoS, and Jito tips solve different problems

Priority fees, SWQoS, and Jito tips are often discussed together, but they operate at different layers, and tuning the wrong one leaves spend increasing without landing rate moving at all.

Priority fees act strictly at the validator-scheduler level. They dictate execution priority once a transaction is already inside the leader's queue, but do nothing to help it cross the threshold in the first place.

SWQoS addresses that earlier boundary. By leaning on staked peering relationships, it grants preferential ingress capacity at the door without touching scheduling order downstream.

Jito tips run on a separate track entirely: an out-of-band Block Engine auction that only benefits transactions explicitly routed through its system. If the bundle is not selected, the tip does not improve inclusion through that path.

| Mechanism | Layer | What it influences | What it does not guarantee |
| :---- | :---- | :---- | :---- |
| Priority fee | Validator scheduler | Scheduling competitiveness once a transaction reaches a leader | Delivery to the leader, or successful execution |
| SWQoS | Leader ingress | Preferential ingress capacity via staked or trusted peering | Scheduler priority, or execution success |
| Jito tip | Block Engine auction | Competitiveness through the Jito submission path | Inclusion if the bid loses, or execution success |

There is no fixed split between priority-fee spend and Jito-tip spend. The right allocation depends on the workload, current network fee pressure, and whether the transaction actually benefits from bundle semantics or private routing. [Execution-sensitive trading systems](/blog/solana-trading-bot-infrastructure) should measure landing rate against spend rather than copy a fixed fee-to-tip ratio.

Compute-unit budgeting and fee pricing should be tuned together; [priority-fee tuning](/blog/solana-fees-priority-fees-transaction-cost) depends on both the [requested compute limit](https://solana.com/docs/core/fees/fee-structure) and the microLamports price.

## How to improve Solana transaction landing rate

There's no single fix for a low landing rate. It's usually a combination of small corrections across the pipeline, applied roughly in this order:

1. Build the transaction correctly and validate it against current program interfaces.  
2. Estimate compute usage realistically rather than defaulting to the maximum.  
3. Set a compute-unit limit close to actual consumption, with a modest buffer.  
4. Set a priority fee that reflects current fee pressure on the accounts involved.  
5. Fetch a fresh blockhash immediately before signing, and retain `lastValidBlockHeight`.  
6. Simulate before using any no-preflight submission path.  
7. Sign once, serialize once, and treat those bytes as final.  
8. Submit through the delivery path appropriate to the workload's urgency.  
9. Track signature status through `processed` and `confirmed`, not just the send response.  
10. Resend the same signed bytes on a backoff schedule until the blockhash actually expires, then rebuild once.

For many workloads, the largest improvements come after transaction construction: fresh blockhash handling, submission path choice, retry behavior, and confirmation tracking.

## Standard RPC, SWQoS sender, or Jito: which path fits?

There isn't a universally correct submission path. It depends on urgency, whether atomic multi-transaction bundling matters, and how much operational complexity the team can absorb.

| Submission model | Fits well when | Trade-off |
| :---- | :---- | :---- |
| Standard RPC `sendTransaction` | Landing-rate requirements are moderate, workload isn't time-critical | Simplest to operate, but weakest under congestion or contention |
| SWQoS-aware sender | Landing rate matters under congestion, no need for atomic bundling | Requires a staked or peered ingress relationship; still doesn't guarantee scheduler priority |
| Jito transaction or bundle | Ordering matters, or several transactions must land together or not at all | Adds tip economics and auction dynamics; irrelevant if the bid loses |
| Parallel submission across paths | Landing rate is critical and duplicate-handling is already solved | Higher operational complexity and cost; needs strict idempotency downstream |

A standard RPC endpoint treats a transaction as one of many undifferentiated jobs; a dedicated sender is built around getting that specific transaction through congestion, with routing and fee logic tuned for that one purpose. Regional availability, preflight behavior, and observability tooling still vary meaningfully between providers offering these paths — worth checking directly against current documentation rather than assuming parity across vendors. Some dedicated senders combine SWQoS and Jito routing. When a sender skips preflight, a successful response confirms forwarding only; the same confirmation loop still has to run on top. None of this is a permanent choice, either. A common operating pattern is to start with standard RPC and move to SWQoS only when congestion appears in landing-rate or expiry-rate metrics.

## Expiry-aware submission and confirmation

A production submission loop should cover blockhash tracking, compute estimation, one sign/serialize step, expiry-aware resending, and a clear stop condition. The example uses `@solana/kit`. If your codebase uses `@solana/web3.js`, adapt the pattern rather than copying the APIs directly — the transaction-building models differ. 

The core pattern is:

1. Fetch a fresh blockhash.  
2. Simulate with the intended instructions.  
3. Set compute budget from observed usage.  
4. Sign and serialize once.  
5. Resend the same signed bytes until confirmation or blockhash expiry.  
6. Rebuild only after expiry.

The example deliberately manages retries in application code instead of relying on the RPC node.

Types shown below reflect current @solana/kit exports; verify against the installed package version before shipping.

```ts
import {
  appendTransactionMessageInstructions,
  createSolanaRpc,
  createTransactionMessage,
  estimateComputeUnitLimitFactory,
  getBase64EncodedWireTransaction,
  getSignatureFromTransaction,
  pipe,
  setTransactionMessageComputeUnitLimit,
  setTransactionMessageFeePayerSigner,
  setTransactionMessageLifetimeUsingBlockhash,
  signTransactionMessageWithSigners,
  type Instruction,
  type Signature,
  type TransactionSigner,
} from "@solana/kit";

import {
  getSetComputeUnitPriceInstruction,
} from "@solana-program/compute-budget";

const rpc = createSolanaRpc(process.env.SOLANA_RPC_URL!);

const MAX_COMPUTE_UNITS = 1_400_000;
const COMPUTE_BUFFER = 1.1;

const sleep = (ms: number) =>
  new Promise((resolve) => setTimeout(resolve, ms));

async function sendWithExpiryAwareRetries(
  feePayerSigner: TransactionSigner,
  businessInstructions: readonly Instruction[],
  cuPriceMicroLamports: bigint,
): Promise<Signature> {
  // 1. Fetch a fresh blockhash immediately before building.
  const { value: latestBlockhash } = await rpc
    .getLatestBlockhash({ commitment: "confirmed" })
    .send();

  // 2. Build the message and estimate its compute usage.
  const messageForSimulation = pipe(
    createTransactionMessage({ version: 0 }),
    (message) =>
      setTransactionMessageFeePayerSigner(
        feePayerSigner,
        message,
      ),
    (message) =>
      setTransactionMessageLifetimeUsingBlockhash(
        latestBlockhash,
        message,
      ),
    (message) =>
      appendTransactionMessageInstructions(
        [
          getSetComputeUnitPriceInstruction({
            microLamports: cuPriceMicroLamports,
          }),
          ...businessInstructions,
        ],
        message,
      ),
  );

  const estimateComputeUnitLimit =
    estimateComputeUnitLimitFactory({ rpc });

  const estimatedUnits =
    await estimateComputeUnitLimit(messageForSimulation);

  const computeUnitLimit = Math.min(
    MAX_COMPUTE_UNITS,
    Math.ceil(estimatedUnits * COMPUTE_BUFFER),
  );

  // 3. Set the final CU limit, then sign and serialize once.
  const finalMessage = setTransactionMessageComputeUnitLimit(
    computeUnitLimit,
    messageForSimulation,
  );

  const signedTransaction =
    await signTransactionMessageWithSigners(finalMessage);

  const wireTransaction =
    getBase64EncodedWireTransaction(signedTransaction);

  const signature =
    getSignatureFromTransaction(signedTransaction);

  let delayMs = 250;
  let consecutiveSendErrors = 0;
  let lastSendError: unknown;

  // 4. Resend the same signed bytes until confirmation or expiry.
  for (;;) {
    const currentBlockHeight = await rpc
      .getBlockHeight({ commitment: "confirmed" })
      .send();

    if (
      currentBlockHeight >
      latestBlockhash.lastValidBlockHeight
    ) {
      throw new Error(
        "Blockhash expired before confirmation. " +
          "Rebuild and re-sign only after checking that " +
          "the business action is safe to repeat.",
      );
    }

    try {
      await rpc
        .sendTransaction(wireTransaction, {
          encoding: "base64",
          skipPreflight: true,
          maxRetries: 0n,
        })
        .send();

      consecutiveSendErrors = 0;
    } catch (error) {
      lastSendError = error;
      consecutiveSendErrors += 1;
    }

    const { value: statuses } = await rpc
      .getSignatureStatuses([signature])
      .send();

    const status = statuses[0];

    if (status?.err) {
      throw new Error(
        `Transaction landed but execution failed: ${
          JSON.stringify(status.err)
        }`,
      );
    }

    if (
      status?.confirmationStatus === "confirmed" ||
      status?.confirmationStatus === "finalized"
    ) {
      return signature;
    }

    // Do not suppress persistent RPC errors indefinitely.
    if (
      !status &&
      consecutiveSendErrors >= 3 &&
      lastSendError !== undefined
    ) {
      throw lastSendError;
    }

    await sleep(delayMs);
    delayMs = Math.min(Math.ceil(delayMs * 1.5), 1_500);
  }
}
```

The loop stops on three conditions: blockhash expiry, a terminal execution error, or confirmation. Until one of those happens, it resends the identical signed payload instead of rebuilding or re-signing. Everything in between is a resend of the identical signed payload — never a rebuild, never a re-sign, until the block-height check confirms the original blockhash is genuinely gone.

Durable nonces matter when signing itself is the slow part: custody approval flows, hardware-wallet queues, offline signing, or delayed signing. In those cases, a normal recent-blockhash window can expire before the transaction is ready to submit. A durable nonce removes that short-window dependency. On a hot, low-latency execution path, the expiry-aware retry pattern above is usually the better fit — durable nonces solve signing latency specifically, while landing rate still depends on the retry and fee logic covered earlier.

## How to measure transaction landing performance

A fast RPC response tells you the node answered quickly. What happened to the transaction afterward is a separate measurement, and treating response latency as a landing benchmark is a common mistake. Useful metrics include:

* landing rate (submitted vs. landed/processed)  
* confirmation rate and time-to-confirmed  
* expiry rate — transactions that hit `lastValidBlockHeight` before landing  
* execution failure rate, separated from landing failure  
* retries per transaction  
* first-landed slot or slots-to-land  
* results broken out by region and by submission path

Segmenting landing and expiry rates by path and region makes diagnosis straightforward. This breakdown usually shows whether the issue is isolated to one route, one region, or one transaction type before treating it as cluster-wide degradation.

## Transaction landing checklist

* Transaction is valid and passes simulation before anything else  
* Blockhash is fetched immediately before signing, with `lastValidBlockHeight` retained  
* Compute-unit limit reflects actual simulated usage, not a default maximum  
* Priority fee reflects current contention on the accounts involved  
* Submission path matches the workload's urgency and atomicity needs  
* No-preflight paths are simulated separately beforehand  
* Retries resend the same signed bytes until expiry is confirmed  
* Confirmation is tracked explicitly, not inferred from the send response  
* Landing-rate and expiry-rate metrics are logged per path and per region

## When a dedicated sender becomes the right solution

Some workloads reach a point where the retry loop is already solid, compute-unit budgeting matches real usage, and the priority fee tracks current contention — and landing rate still drops the moment the network gets congested. At that point the bottleneck is no longer transaction construction. It is delivery: getting the signed bytes into the leader's ingress path fast enough, through a route with enough priority to survive contention.

Dedicated senders exist for that gap specifically. Combining SWQoS routing, Jito integration, regional endpoint placement, and per-path observability into one submission layer covers ground a single standard RPC endpoint typically doesn't.

## When the delivery path is actually the bottleneck

Failing to land usually comes down to a handful of causes further up the stack: a stale blockhash, a compute budget that doesn't match real usage, a retry loop that gives up too early, or a rebuild-and-re-sign step fired before the original transaction had actually expired. Correcting those on standard RPC gets most workloads to a solid landing rate before a dedicated sender is ever needed.

The [Supanode by HighTower Sender documentation](https://supanode.xyz/docs/solana/sender/overview) covers the parallel submission path described above. Current availability and product details are listed on the [Supanode transaction landing](https://supanode.xyz/fast-transaction-sending) page.
