---
title: "Raiku: Current State, February 2026"
slug: "raiku-current-state-february-2026"
date: "2026-02-01"
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Raiku%20COVER.png"
coverAlt: "Raiku Current State February 2026"
tags: ["solana", "execution", "validator", "raiku", "infrastructure"]
---

# Raiku: Current State, February 2026

Retry logic ships as a core feature in most Solana integrations we've touched. When a tx drops, you resubmit. When congestion spikes, you resubmit faster. That's the current architecture of "reliable" execution on Solana.

We've been routing around this in practice - testing submission paths, watching drops during load spikes, building infra that accounts for the gap between "submitted" and "landed." Raiku's slot reservation model landed in that specific context.

## The failure rate

Up to 40% of Solana transactions fail under load - Raiku's own slot marketplace page. Blockworks documented spikes around 59-75% non-vote tx failure during peak congestion. A 2025 academic study found bots hitting ~73.4% failure rates, with errors dominated by "price/profit not met" and "invalid status." Failure as a structural outcome of competitive execution.

![The cancellation race](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Raiku%20The%20cancellation%20race.png)

<sub>The cancellation race: why execution becomes a speed game under competition  
https://www.raiku.com/blog/solana-will-beat-tradfi</sub>

Blockspace markets price compute units and fees. Raiku prices execution windows. Miss your slot and the outcome becomes unpredictable regardless of fee paid.

Their Slot Marketplace introduces siQoS (slot-inclusion QoS) - reservation at a specific ledger point, two paths: JIT (sealed-bid, sub-50ms pre-confirmation) and AOT (35+ slots ahead, English-style auction for future slots). AOT targets workflows with hard deadlines: oracle updates, payment settlements, scheduled rebalances, DePIN intervals. Infrastructure-level operations that currently absorb failure through retry because there's no reservation layer to rely on.

## Cryptographic receipts

The Coordination Engine tracks transaction status in real time, reroutes around failures, anchors finality to a commitment level that avoids minority fork ambiguity. The more concrete UX piece is what they call "cryptographic receipts" - verifiable proof that a guaranteed transaction landed, or verifiable proof that something went wrong. Today, dropped transactions require log-level inspection to diagnose. A receipt is a different starting point.

![Raiku extension architecture](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Raiku%20extension%20architecture.png)

<sub>Raiku extension architecture: execution environments + Ackermann nodes + RFQ layer  
https://docs.raiku.com/overview/novel-app-architectures</sub>

Enforcement runs through the Sidecar: a validator module that checks slot leadership, applies bandwidth allocation, filters spam, injects transactions at the right moment, falls back to standard paths when a participating validator isn't the leader. The Node/Sidecar split, with planner and enforcer as separate components, reflects the coverage constraint directly: the guarantee is only as wide as Sidecar deployment.

## $13.5M and a testnet

Kiln and Everstake are active on the validator testnet. Dawn Labs joined as a launch partner deploying the Sidecar while maintaining client diversity. September 2025 brought the raise:

- $13.5M total (seed + pre-seed), led by Pantera Capital  
- Jump Crypto, Lightspeed Faction on the seed round  
- Pre-seed co-led by Figment and Big Brain Holdings  

Limited testnet seats, curated onboarding. The current phase is validator network formation - getting enough Sidecar coverage for slot guarantees to be something a protocol can depend on across a meaningful share of blocks.

## The block-building competition inside Solana

![Two fault models on Solana](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Raiku%20Two%20fault%20models%20on%20Solana.png)

<sub>Two fault models on Solana: TowerBFT (33% byzantine) vs Alpenglow (20% malicious + 20% faulty)  
https://www.raiku.com/blog/alpenglow-the-solana-upgrade-that-rivals-nasdaq</sub>

Syndica data from January 2026 shows Solana stake fragmenting across client variants, with block-building mechanics becoming a visible competition at the infrastructure level. BAM adoption has already reached >25% stake share and >110M SOL staked among ~275 eligible validators (Jito JIP-31 / Trillium) - also targeting ordering reliability, through TEEs, different trust and latency profile than the Sidecar model.

Raiku's public positioning frames validator economics as the central problem: block production is an economic decision made before finality, and designs that redistribute the production monopoly shift extraction rather than reduce it. The ecosystem runs on private execution advantages already: stake-weighted QoS, private block engine channels, multi-endpoint routing. Raiku's bet is that making those advantages explicit, priced, and slashable is the more durable architecture.

The diagnosis is grounded in current validator economics. The mechanism is the open question.

## What we're watching

![Finality time comparison](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Raiku%20Finality%20time%20comparison.png)

<sub>Finality time comparison  
https://www.raiku.com/blog/alpenglow-the-solana-upgrade-that-rivals-nasdaq</sub>

Validator coverage expansion and the share of slots running active Sidecars, cryptographic receipts becoming third-party auditable onchain, AOT reservation showing up as a real validator revenue line beyond MEV. That last one is Raiku's core economic bet: execution SLAs as a validator business model, enforceable commitments as a revenue category.

If that takes hold, the downstream effect is a shift in how applications design around execution certainty rather than probabilistic landing. A category of onchain applications begins building around schedules instead of absorbing execution uncertainty as a constraint.
