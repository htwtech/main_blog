---
title: "Canton Network: How Validation Actually Works"
slug: "canton-validation-how-it-works"
date: "2026-03-17"
description: "How validation works in Canton Network: local privacy enforcement, Global Synchronizer consensus, Super Validators, and governance mechanisms."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Canton%20Validation.png"
coverAlt: "Canton validation architecture overview"
tags: ["canton", "institutional blockchain", "validation", "bft", "daml", "digital asset"]
---


# Canton Network: How Validation Actually Works

Canton is a privacy-first institutional blockchain where validation is scoped by design. Each validator operates on exactly the portion of the ledger it participates in, and the whole consensus architecture is built around that assumption.

## What you're looking at

Canton is a [privacy-preserving blockchain built on Daml smart contracts](https://www.htw.tech/blog/canton-network-public-when-users-are-banks), developed by Digital Asset. The Global Synchronizer (Canton's decentralized synchronization infrastructure) is live on mainnet, and the network has been running coordinated multi-party financial transactions across distributed applications.

This matters now because the validation architecture here works differently from most chains in the space, and the governance layer has been evolving actively through Q1 2026.

Earlier this March, HighTower joined the Canton network as a validator node — a good moment to break down how validation in Canton actually works in practice.

So before anything else: what does "validation" even mean here?

## Your validator sees only its slice

![Canton UST collateral pilot](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Canton%20UST%20collateral%20pilot%20was%20structured%20across%20custodians,%20apps,%20and%20market%20participants..png)

<sub>Canton’s shared coordination layer: validators keep private slices of state, while the Global Synchronizer provides network-wide ordering, interoperability, and atomicity.</sub>

Canton's own documentation is direct about this: validators "are only active in transactions to which they are a party," and data is "segmented and replicated only to those validators permissioned to view the data."

This sounds like a constraint.

In practice, it's the design principle that makes Canton work for institutional use cases at all. A bank running a validator on Canton doesn't need to replicate every trade from every other participant. It needs its own state to be correct and final, and it needs atomic cross-party transactions to settle reliably.

The 2024 U.S. Treasuries pilot gives a concrete illustration: 11 distributed applications, 21 participant nodes, all coordinated through the Global Synchronizer "keeping everything in sync and enabling atomic transactions across the different applications and parties."

Local privacy, global atomicity. That's the core claim the architecture is built around.

## Three layers worth distinguishing

![Global Synchronizer diagram](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/The%20Global%20Synchronizer%20(2).png)

<sub>Canton’s shared coordination layer: validators keep private slices of state, while the Global Synchronizer provides network-wide ordering, interoperability, and atomicity.</sub>

Validation on Canton breaks into three layers.

### Layer 1 — Local validity

This is your validator's domain. Daml smart contract rules are enforced here, privacy sets are applied here, and the "need-to-know" replication boundary is maintained here. The validator doesn't replicate the world — it stays synchronized with the portion it's supposed to see.

### Layer 2 — Global ordering and atomicity

This is where the Global Synchronizer lives, and where Super Validators operate a Byzantine Fault Tolerant consensus layer. The docs specify CometBFT keys used "in CometBFT quorums (>⅔ of SVs)" — required for advancing the chain and changing configuration. That ">⅔" threshold is what separates global finality from local privacy enforcement.

### Layer 3 — Public verifiability without a single trusted operator

Canton uses redundant Scan services and BFT-backed proxy queries to make "public" meaningful without making all data transparent by default.

## Who the Super Validators are

Super Validators are the operator set at the synchronizer layer — currently 13 identities.

Public dashboards expose an SV weight distribution, but that weight should not be conflated with the >⅔ SV quorum the docs describe for CometBFT — those are node/operator counts, not weighted votes.

The weight distribution is public via canton.wiki: the Foundation holds the largest share by a significant margin, with the remaining weight distributed across institutional operators including Cumberland, Tradeweb Markets, Liberty City Ventures, and others.

Weight exists as a network primitive — it plays a role in governance and issuance mechanics — but the exact relationship between the public weight table and configuration-change quorums is not fully specified in the sources available here.

What the docs do confirm: CometBFT configuration and validator-set changes require >⅔ of SVs.

The distribution also reflects who has completed the full onboarding process — which, as it turns out, is a meaningful filter.

## Onboarding is a security boundary

Getting a validator onto Canton mainnet requires more than spinning up a node. The network treats admission as a controlled process, and the docs are explicit about this.

There are three environments — DevNet, TestNet, and MainNet — each with progressively tighter requirements:

1. **DevNet** — open for development, but requires your validator's egress IP on an allowlist maintained by SV operators; resets every 3 months.

2. **TestNet** — requires approval by a tokenomics committee, an allowlisted IP, and an onboarding secret from a sponsoring Super Validator; resets every 3–6 months.

3. **MainNet** — same requirements as TestNet, no resets.

Allowlist approval typically takes 2–7 days. Onboarding secrets are time-bound and one-time-use.

The "SV sponsor" relationship in that process is real and operational — it's how operators are admitted and how their connectivity is anchored to an accountable SV identity.

One detail worth noting: the Scan web UI is described as "not (yet) fully public," and without a static IP, accessing it may require a VPN operated by an SV.

Canton can be public in terms of code and standards, while still being governed in terms of access paths and operator admission. That tension is probably part of the design.

## What "Writes Denied" actually means

Traffic management on Canton is deliberate. The synchronizer charges for usage beyond a free base rate — to amortize Super Validator operational costs, improve SV incentives, and make DoS attacks economically unattractive.

If both free and paid-for traffic are exhausted, "attempted writes are denied by the sequencer."

Liveness is measurable too. Every Canton node exposes `/readyz` and `/livez` endpoints.

A dedicated ingestion timestamp metric —

`splice_store_last_ingested_record_time_ms`

— tracks whether the validator is staying current with the ledger.

For validators collecting liveness rewards, ingestion happens every round, and acceptable lag is under 20 minutes.

The liveness reward model is being actively redesigned.

CIP-0096 (approved December 31, 2025) proposes reducing rewards in stages and setting them to zero on April 30, 2026.

The motivation is explicit: validators were collecting rewards "without otherwise participating… and adding utility."

The network is shifting toward rewarding real activity, not just uptime.

## How verifiability works without transparency

Because the ledger is private by design, "how do you trust it?" is a real question. Canton's answer involves two mechanisms working together.

The Scan app is hosted redundantly by each Super Validator. Different SV-hosted instances are designed to be consistent with one another, and the goal spelled out in the docs is direct: comparing multiple Scan UIs "allows the public to inspect multiple Scan UIs and compare their data, so that they do not need to trust a single Super Validator."

The Scan Proxy API adds another layer.

When a validator app is part of a Canton Node, each proxy query is broadcast to the scan services of multiple SV nodes, and the consensus result is returned.

Public verification is implemented as BFT-backed aggregation rather than global data exposure — a design that fits financial infrastructure better than a transparent-by-default chain would.

## Governance, weight, and CIP-0105

![Canton network metrics](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/A%20snapshot%20of%20Canton%20s%20network%20level%20metrics%20in%20March%202026.png)

<sub>A snapshot of Canton’s network-level metrics in March 2026</sub>

Canton Coin issuance runs on Activity Records with a round-based cycle — a round starts every 10 minutes, under parameters that SVs adjust through governance votes.

Minting and activity recording are distinct steps, and the whole loop is governance-controlled.

For external parties who sign with keys they control independently (a common institutional pattern), Canton supports minting delegation: a one-time sign-off that delegates reward collection without granting broader rights.

This is live functionality, implemented as part of CIP-0073.

The more significant development is CIP-0105, approved March 2, 2026.

It introduces a "Super Validator Locking & Long-Term Commitment Framework" — SVs can elect to lock a portion of lifetime-earned rewards to maintain forward weight, with unlocks vesting at **1/365.25 per day once initiated**.

The stated purpose: **"replace reputational assurances with cryptographic proof of alignment," applied uniformly with "no exemptions."**

The design logic holds up.

The link between "who can finalize and govern" and "who has durable stake in correct operation" is exactly the link that makes a BFT-governed network credible over time. CIP-0105 is trying to make that formal and enforceable.

---

That's how Canton's validation layer works — from local privacy enforcement at each node, through BFT consensus among Super Validators, to the governance mechanisms that define who participates and on what terms.
