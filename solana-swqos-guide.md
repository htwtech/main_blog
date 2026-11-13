---
title: "Solana SWQoS Explained: Stake-Weighted Transaction Delivery"
slug: "solana-swqos-guide"
canonical_url: "https://supanode.xyz/blog/solana-swqos-guide"
date: "2026-11-12"
description: "How Solana SWQoS changes leader ingress, how stake-backed peering differs from priority fees, and what to verify before using a transaction sender."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/htw-cover-solana-swqos.png"
coverAlt: "Solana SWQoS Explained: Stake-Weighted Transaction Delivery"
tags: ["Solana","SWQoS","Transaction Delivery","Infrastructure"]
---

# How Stake-Weighted QoS Changes Transaction Delivery on Solana

Solana stake-weighted quality of service (SWQoS) changes how a leader admits incoming transaction traffic. Connections the leader can tie to stake get preferential treatment at ingress. That stake can belong to the peer itself or be assigned to it through a trusted peering arrangement with a validator.

The difference shows up under load. When a leader's ingress is saturated, unstaked traffic is throttled first. A signed transaction forwarded over a stake-backed connection then has a better chance of reaching the current leader at all, and every later stage of processing depends on that step.

## What is Solana stake-weighted QoS?

SWQoS is a validator-side mechanism. The leader identifies the connecting peer and checks the stake associated with that identity. It then gives the peer a share of its ingress capacity roughly in proportion to that peer's share of total stake.

Identification became possible when Solana moved TPU ingestion from raw UDP to QUIC. Over plain UDP a leader sees packets arriving from an address and knows little about who sent them. A QUIC connection to a validator's TPU is set up with a certificate tied to the client's keypair, which gives the leader a public key to look up against stake.

Terms like "staked RPC" or "staked connections" are commercial names for access to this mechanism. On the network side, the stake sits with block-producing validators, and [Solana's SWQoS guide](https://solana.com/docs/defi/stake-weighted-qos) notes that delegating stake to RPC servers is neither necessary nor effective. An RPC or forwarding peer gets stake-weighted treatment through its relationship with a validator.

## Where SWQoS fits in the transaction path

The diagram shows a conceptual transaction path. Individual provider topologies may differ.

![Solana transaction flow from signing to confirmation, highlighting SWQoS at leader ingress before scheduling and execution.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/swqos-transaction-pathtw.png)

<sub>Where SWQoS acts in the Solana transaction path.</sub>

With a managed sender, the application submits transactions through the provider interface, and the sender's infrastructure handles validator-facing forwarding. Direct TPU submission skips the provider layer: the application opens its own QUIC connections to leaders. Without a stake-backed identity, though, that traffic still arrives at the leader as unstaked.

### Validator stake, RPC peers, and sender identity

A typical managed submission involves three keys. Only one of them is relevant to SWQoS.

The API key authenticates the application to its provider and stays at the HTTP layer. The transaction signer and fee payer authorize the transaction itself. Any stake they hold doesn't affect how the connection carrying the transaction is treated. The key SWQoS uses is the remote public key on the validator-facing QUIC connection. Agave extracts that key, looks up the associated stake, and classifies the connection internally as either staked (with a stake value) or unstaked.

So a wallet holding a large amount of staked SOL gets no ingress advantage from that stake when its transactions arrive through an unstaked peer.

### Trusted peering and virtual stake

A trusted RPC or forwarding peer doesn't have to become a voting validator. Solana's guide documents both sides of the arrangement:

* the validator operator lists the peer's identity together with a stake value in a `staked-nodes-overrides` configuration;  

* the RPC node forwards transactions to that validator's TPU endpoint.

The exact flags and file formats depend on the validator client version.

In practice, SWQoS access becomes a relationship between infrastructure operators. A managed sender maintains the validator relationships and forwarding identities behind its API, plus the routing between them. Applications submitting through that API use those relationships indirectly.

A virtual stake override lives in one validator's configuration and takes effect only on that validator's ingress, so coverage across the leader schedule is a real question. Leaders rotate every four slots on a published schedule, with a target slot time of roughly 400 milliseconds. A retried transaction can therefore be sent toward dozens of different leaders over the lifetime of a single blockhash. Some of them will recognize the sender's forwarding identity, and others may not. How much the stake-backed path helps depends on that set, and on how the sender routes traffic when the current leader sits outside it.

### Admission under congestion and implementation differences

Solana's SWQoS guide describes the original model: roughly 80% of a leader's TPU capacity is allocated by stake, and the rest is left for unstaked traffic. Connection allocation and throttling have since changed across Agave versions. Recent releases throttle in a congestion-aware way, so the gap between staked and unstaked peers can be small under light load and widen as ingress saturates. Per-peer quotas also depend on each validator's configuration, so current behaviour is best described against a specific client release.

## SWQoS and downstream transaction processing

SWQoS affects whether transaction traffic reaches the leader under congestion. For flows where a transaction loses most of its value within a few slots, that alone can decide the outcome. After admission, the transaction still has to get through three separate stages:

* Scheduling. The scheduler orders transactions by fee reward relative to estimated cost. The priority fee and the requested compute budget shape a transaction's position. Transactions that write to the same hot accounts can't run in parallel, so a well-delivered transaction may still wait behind competitors with stronger economics. Fee mechanics are covered in the guide to [Solana priority fees and compute units](/blog/solana-fees-priority-fees-transaction-cost).  

* Execution. Compute budget, block limits or the instruction logic itself can stop a transaction after it arrives.  

* Expiry and confirmation. A recent blockhash stays valid for roughly 150 blocks. After that, the transaction is rejected whatever path it took. A successful `sendTransaction` response means the RPC node accepted the signed transaction; it doesn't wait for confirmation. The [Solana transaction landing guide](/blog/solana-transaction-landing-guide) covers retry strategy and expiry handling across the broader submission lifecycle.

Jito Block Engine uses a separate auction path for bundle tips.

## Who configures the stake-weighted path?

The infrastructure provider and the application are responsible for different parts of the transaction lifecycle.

| Responsibility | Sender / infrastructure operator | Application |
| :---- | :---- | :---- |
| Trusted peering and the forwarding identity validators recognize | Establishes and maintains | — |
| Virtual-stake mapping | Arranges it with validator operators and keeps it current | — |
| Submission interface | Exposes it and returns provider-side errors | Calls it, classifies errors, decides which ones justify a retry |
| Instructions, blockhash, fee policy, compute budget | — | Builds and chooses them, including whether to skip preflight with `skipPreflight` |
| Signing | Only where a product explicitly documents delegated signing | Signs locally in client-signed flows; private keys stay with the application |
| Outcome verification | May offer tooling | Verifies inclusion and execution independently |

Most of the operator column is invisible from the application. A provider evaluation should cover these questions:

1. Which validators recognize the sender's forwarding identity? Is the stake behind those relationships real, virtual, or a mix?  

2. How does traffic reach leaders outside those relationships, and what happens to it during congestion?  

3. Does the sender retry or rebroadcast on its own? If it does, how does that interact with the application's retry logic and with `maxRetries`?  

4. Which errors come from the provider layer and which from the network, and can the two be told apart in responses?  

5. Does the transaction need anything extra to use the stake-backed path, such as a tip instruction or a specific endpoint?

## When should an application evaluate stake-weighted delivery?

Stake-weighted delivery is most relevant to latency-sensitive workloads where congestion at leader ingress may affect inclusion. Trading, liquidations and arbitrage fall into this group, along with any other flow where a late transaction is worth far less than one that lands on time.

Landing failures can start at ingress, scheduling, execution, blockhash expiry or status observation. From the application side, they often look like the same symptom.

Transactions included in blocks but failing execution have already passed ingress. The relevant diagnostics for those are the transaction errors and the program logs.

Sometimes valid transactions with a comparable fee policy keep ending up unresolved during high-load windows, while normal periods look fine. In that case a stake-backed path becomes a reasonable hypothesis to test.

Expiry, or a missing RPC status, doesn't prove an ingress problem by itself. By default, `getSignatureStatuses` checks only the node's recent status cache. With `searchTransactionHistory` enabled, the node also searches its ledger, but how far back it can see depends on the history that node keeps. An empty status result may reflect the queried node's available history rather than the transaction's network outcome.

If higher priority fees improve inclusion while changing sender paths has little effect, look next at scheduler competition and account contention.

A controlled comparison keeps the workload, fee policy and status-observation method consistent and varies only the delivery path. It should cover both quiet and busy periods, since the stake-aware preference matters mostly under load. A run limited to off-peak hours can easily show no difference and be mistaken for a conclusive result.

For low-volume applications with acceptable landing performance, a standard submission path may already meet the workload's requirements.

## How to measure stake-weighted transaction delivery

Track three outcomes separately:

1. Accepted by the sender. Did the API accept the signed transaction? This records that the request was accepted. Response latency is a separate metric and should be logged in its own field. Forwarding to the leader is a later event that only sender-side telemetry can confirm, if the sender exposes it.  

2. Included. Did the transaction appear in a block, and in which slot relative to submission?  

3. Executed successfully. Did the instructions succeed? An included transaction can still fail and still pay its fee.

Inclusion and execution should be recorded at one fixed commitment level (processed, confirmed or finalized), the same level the application uses when it acts on the result. Otherwise "landed" counts and business-level outcomes drift apart regardless of the sender.

A comparable benchmark also needs submission timestamps, signature tracking, consistent observation rules, and explicit handling of unresolved transactions. The journal fields are described in the methodology for how to [measure transaction delivery separately from execution](https://supanode.xyz/learn/measure-solana-transaction-landing).

Sending the same signed transaction through multiple paths improves redundancy but prevents reliable attribution of inclusion to a specific sender. Once the signature lands there is exactly one on-chain outcome, and nothing in it records which path was responsible for inclusion. A benchmark that sends through A and B and credits whichever replied first ends up measuring acknowledgement timing.

For sender attribution, split comparable transactions between routes and submit each signature through a single path. This gives up redundancy during the test but keeps a clearer relationship between delivery path and observed outcome.

## Using stake-weighted delivery through Supanode TPU Sender

Disclosure: Supanode TPU Sender is a product of Supanode, the publisher of this article.

Supanode TPU Sender provides a managed transaction-forwarding path with staked TPU connections using SWQoS. Applications submit already-signed, base64-encoded transactions over HTTP JSON-RPC. QUIK is Supanode's separately authorized submission interface; QUIC is the network transport used for validator-facing Solana connections. Endpoint and access details are in [TPU Sender endpoints and authorization](https://supanode.xyz/docs/solana/sender/endpoints).

The application builds and signs locally, and the send API returns a signature. Confirmation goes through regular Solana RPC status checks, as described in [submission requests and errors](https://supanode.xyz/docs/solana/sender/send).

→ [Explore Supanode TPU Sender](https://supanode.xyz/services/solana/tpu-sender) and review the connection requirements for your application.

