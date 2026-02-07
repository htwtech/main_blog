---
title: "Citrea Bitcoin L2 Mainnet Campaign: Dashboard, Bridge, ctUSD & Metrics"
slug: "citrea-bitcoin-l2-mainnet-campaign-dashboard-bridge-ctusd"
date: "2026-02-07"
description: "An in-depth look at Citrea’s Bitcoin L2 mainnet launch: on-chain campaign metrics, ctUSD stablecoin design, BitVM bridge architecture, and early network performance."
author: "Katrin HighTower"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Katrin.jpg"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/logos/citrea.jpeg"
coverAlt: "Citrea Bitcoin Layer-2 mainnet campaign and infrastructure overview"
tags: ["citrea", "bitcoin", "l2", "btcl2", "infrastructure", "stablecoins", "bitvm", "zkrollup"]
---

@citrea_xyz went live on **MAINNET on January 27**. By early February, **157,213 transactions** had settled on-chain, with **1.23K total active accounts** and **636 cBTC transfers** recorded.  

The team deployed a dashboard tracking **six distinct participation categories**. This is what bootstrap activity looks like when the infrastructure actually works.

---

## How the Dashboard Works (And Why It Matters)

The campaign tracks **real on-chain behavior**, not abstract point systems. Instead of rewarding clicks, Citrea measures **capital deployment, liquidity depth, and trading activity**, pulling participation in a fundamentally different direction.

![Citrea dashboard overview](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/logos/cit3.jpeg)

**Citrea Mainnet Dashboard Status (Jan 27 – Feb 3):**

### Live Categories
- **Bridger**  
  Measures net asset inflow and duration. How much BTC or stablecoins enter Citrea, and how long capital stays deployed. The longer you hold, the higher your score.

- **Liquidity Provider**  
  Pool depth across pairs and duration of commitment. Covers activity on @SatsumaDEX, @JuiceSwap_com, and other AMMs.

- **Trader**  
  On-chain swap volume and frequency.

### Coming Soon
- **Lender**  
  BTC-backed lending markets via @ZentraFinance, tracking cBTC deposits and ctUSD borrows.

- **Yield Strategist**  
  Participation in structured vaults and institutional yield products.

- **Adventurer**  
  Experimental features: prediction markets, privacy tools, Lightning swaps.

What stands out is **retroactive accounting**. Bridging activity from day one counted toward scores even before the Bridger tracker went live on January 30. Early users received **Week 1 boosts**, after which metrics ran independently. Farming is not eliminated — but it’s far less obvious.

---

## The Stablecoin Play: ctUSD as an Institutional Bridge

Source:  
https://www.blog.citrea.xyz/introducing-citrea-usd-ctusd-the-native-stablecoin-for-bitcoin-issued-by-moonpay-and-powered-by-m0/

MoonPay issued **ctUSD** as Citrea’s native stablecoin, powered by **M0 infrastructure**. The advantage is clear: **instant fiat on/off ramps without exchange gatekeeping**.

What matters operationally:

- **Instant fiat access**  
  Users in **49 US states** can buy ctUSD via MoonPay using a credit card. No exchange account required. (New York and several countries remain excluded due to regulation.)

- **Institutional settlement**  
  ctUSD redeems to real dollars. Lock BTC, borrow ctUSD, and cash out to a bank account — a direct bridge to traditional finance workflows.

- **Unified liquidity**  
  One canonical stablecoin means liquidity concentrates instead of fragmenting. Pairs like **cBTC/ctUSD** formed immediately with minimal slippage.

- **Compliance infrastructure**  
  MoonPay operates across **160+ countries**, giving Citrea an operational edge most BTCL2s lack.

In week one, ctUSD held its peg, remained liquid under large swaps, and generated real LP fees. That’s the baseline — novelty traction fades, but infrastructure performance remains measurable.

---

## The Bridge Architecture: BitVM in Practice

![Citrea bridge architecture](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/logos/cit4.jpeg)

Citrea’s **Clementine bridge** uses **Bitcoin itself as the security layer**, avoiding federated multisigs or custodial vaults.

### How It Works
- **Ten signers** (Galaxy Digital, Nansen, Nethermind, Luxor, HashKey Cloud, Finoa, Chainway Labs, Coinsummer, 0xmakase_jp) control a shared BTC vault via **N-of-N MuSig2**.
- All ten must collaborate to release BTC — but security only requires **one honest signer** to prevent theft.
- Withdrawal rules are **pre-signed**. Any invalid withdrawal can be challenged on Bitcoin L1 via **BitVM**, proving fraud directly on-chain.

### Cost Compression
- Bridge validation costs dropped from **~$15,000 per batch to under $100**.
- Roughly **5,000 L2 transactions** settle to Bitcoin for about **$0.02 per transaction**.
- Batches post frequently with minimal Bitcoin footprint using Taproot proofs.

Citrea combines **ZK-rollup execution**, **BitVM dispute resolution**, and **signer accountability** — without Bitcoin soft forks or native SNARK verification.

---

## Early Metrics: What We’re Seeing

Explorer:  
https://explorer.mainnet.citrea.xyz/stats

- **Total transactions:** 156.841K  
- **cBTC transfers:** 660  
- **Total accounts:** 1.23K  
- **Total addresses:** 4.094K  

![Citrea mainnet metrics](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/logos/cit5.jpeg)

Transaction success rates stayed above **99%** throughout the first month. Gas usage remains negligible — around **0.001 cBTC** in 24 hours — indicating real usage, not subsidy farming.

Early UX friction (users holding only stablecoins without cBTC for gas) was acknowledged quickly. **Account abstraction improvements** are already planned, suggesting hands-on operational ownership rather than outsourced maintenance.

---

## What’s Next: Remaining Categories

- **Lender**  
  Unlocks BTC-backed lending — the core of Bitcoin capital markets.

- **Yield Strategist**  
  Structured products and institutional vaults. Execution quality will determine adoption.

- **Adventurer**  
  Experimental primitives including privacy tools like @crest_btc and novel Bitcoin-native features.

---

## The Partnership Lineup (Why It Matters)

The partner list isn’t branding — it’s **operational responsibility**:

- **@GalaxyHQ** — capital + infrastructure.
- **@nansen_ai, @Nethermind, @Luxor, @HashKeyCloud, @Finoa_io** — institutional ops teams running nodes and signers.
- **@MoonPay** — 30M+ users, real banking rails.
- **@Morpho** — lending infrastructure.
- **@Keyrock** — institutional market making and yield strategies.

These aren’t hypothetical integrations. If something breaks, it’s their ops teams on call.

---

## Final Take

The architecture is verifiable.  
The metrics track **capital retention**, not clicks.  
The stablecoin works.  
The bridge settles cleanly.  

Week one delivered on the promise: **Bitcoin infrastructure that actually works**. The real test now is scale — whether Citrea evolves into a durable Bitcoin capital markets layer.

As of early February, all the pieces are in place.

---

## Links

- Website: https://citrea.xyz  
- Explorer: https://explorer.mainnet.citrea.xyz  
- Blog: https://www.blog.citrea.xyz  
- X / Twitter: https://x.com/citrea_xyz
