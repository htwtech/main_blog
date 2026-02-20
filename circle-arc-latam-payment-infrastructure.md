---
title: "Circle & ARC: The Payment Infrastructure Stack Emerging in LATAM"
slug: "circle-arc-latam-payment-infrastructure"
date: "2026-02-20"
description: "A deep-dive into how Circle and ARC are supporting payment startups, expanding into LATAM markets, and what the USDC growth curve actually looks like from the inside."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/1Circle%20ARC%20Builds%20LATAM%20Rails.png"
coverAlt: "Circle ARC Builds LATAM Rails"
tags: ["circle", "usdc", "arc", "stablecoins", "latam", "payments", "infrastructure"]
---

# Circle & ARC: The Payment Infrastructure Stack Emerging in LATAM

A deep-dive into how Circle and ARC are supporting payment startups, expanding into LATAM markets, and what the USDC growth curve actually looks like from the inside.

## The Numbers First

![Circle Transparency](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Circle%20Transparency%20.png)

<sub>Circle Transparency. Source: https://www.circle.com/transparency</sub>

According to Circle's Q3 2025 Earnings Presentation, USDC in circulation hit $73.7 billion — up 108% year-over-year. As of February 13, 2026, Circle's own transparency page shows $73.4 billion in circulation, backed by $73.7 billion in reserves.

![USDC circulation growth](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/USDC%20circulation%20growth.png)

<sub>USDC circulation growth (end-of-period, $B). Source: Circle Q3 2025 Earnings Presentation, slide 17</sub>

Its share of stablecoin circulation went from 23% to 29% over twelve months (per CoinMarketCap data cited in the Q3 deck), and by Q3'25 USDC accounted for 40% of total stablecoin transaction volume — per Visa Onchain Analytics. Cumulatively, USDC has supported over $41 trillion in on-chain transactions since 2018, with CCTP cross-chain transfer volumes reaching $31 billion in Q3'25 alone (up 7.4x year-over-year).

![USDC share of stablecoin circulation](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/USDC%20share%20of%20stablecoin%20circulation%20.png)

<sub>USDC share of stablecoin circulation and market growth. Source: Circle Q3 2025 Earnings Presentation, slide 7</sub>

We'll come back to what those cross-chain numbers mean for builders in a moment.

## What Circle Actually Offers the Builder Ecosystem

The product suite is layered, and for payment startups specifically, it covers a lot of ground:

- Circle Mint + API infrastructure — USDC/EURC issuance, CCTP for cross-chain transfers, Paymaster and Gateway modules for multi-chain liquidity management
- Circle Payments Network (CPN) — by November 2025, 29 financial institutions were live on the network, with 55 more in verification; single integration, access to all corridors
- Circle Alliance Program — over a hundred projects in the partner directory, with Circle Ventures taking equity positions in select companies building on USDC rails
- Grant and go-to-market support — early-stage funding and product validation before launch, with a clear graduation path into CPN

The Alliance Program is specifically where the LATAM-oriented case studies concentrate. It's a builder-facing program — the companies in it are building infrastructure and payment products, operating directly on USDC rails.

## Startup Case Studies: Who's Actually Building in LATAM

Félix is the cleanest example. A Miami-based fintech, they built a WhatsApp chatbot for US-to-Mexico remittances — layering Circle Mint, Stellar, Bitso, and Mexico's SPEI system to cut transfer fees from ~$5 to ~$3 per $100 sent (roughly 40% cheaper) while enabling 24/7 transfers including weekends. The CEO was direct about the mechanics: USDC settles instantly, which reduces prefunding costs. That's the kind of infrastructure edge that actually moves unit economics for a remittance product.

BUFI (Brazil/LATAM) built an invoicing and payment platform for freelancers and agencies, turning international payouts into near-instant settlements. Their founder Tomas put it plainly: without USDC and Circle's infrastructure, the product wouldn't have been viable at the economics they needed at launch.

Alfred (Miami, LATAM focus) started with a Circle grant, validated product before launch, and then graduated into CPN itself. They built a single API that routes USDC into bank accounts in Mexico and Brazil in minutes — and after CPN integration, they effectively became a payment rail that banks can use to offer USDC withdrawals directly to their customers.

Conduit went the venture route: Circle Ventures led their $36M Series A. They use CPN to open new payment corridors, and their claim is that within the network, launching a new corridor takes days rather than the traditional months. They're now expanding into Asia markets on the back of that raise.

EdenFi (Africa, but worth noting) is building a smart wallet for the African diaspora with USDC as the core stable unit — their reasoning being that it's a stable currency users can trust for cross-border flows.

The Alliance directory has over 100 companies across regions and verticals — the cases above are a sample, not a summary.

## ARC: Quick Context for Those Who Need It

For readers already familiar — skip ahead. For those who aren't: ARC is Circle's new public L1, launched to testnet in October 2025, with over 100 companies participating. The design is specifically optimized for payment flows:

1. 1–2 second deterministic finality — not probabilistic, actually deterministic
2. Gas fees denominated in stablecoins (USDC, EURC) — predictable cost structure for B2B payment products
3. Native multi-stablecoin support and built-in tokenized asset interoperability
4. On-chain FX layer — FX conversion happens at the settlement layer itself

Predictable fee costs in dollar terms matter for treasury management at scale — that much is straightforward for anyone building B2B payment flows on volatile-gas networks.

## Who's In the ARC Testnet

The participant list is more informative than the announcement itself. On the payments and fintech side:

- Amazon Web Services, Brex, Corpay — infrastructure and corporate spend
- dLocal (Uruguay) and EBANX (Brazil) — two of LATAM's most significant PSPs
- LianLian, Paysafe, Nuvei — established global processors evaluating ARC for merchant settlement
- SBI, HSBC, Societe Generale — banking participants exploring the network

Nuvei's framing was specific: they're evaluating ARC for its sub-second finality, predictable dollar-denominated fees, and the built-in FX layer for multi-currency merchant payouts. That's an unusually concrete use-case description from a company processing billions annually.

On the stablecoin issuer side, ARC already has local currency stablecoin issuers from Brazil (Avenia, BRL-backed) and Mexico (Juno/Bitso, MXN-backed). For LATAM startups, that means settlement in local currency, through local rails, with global dollar liquidity accessible underneath.

Visa's public comment on ARC: the stable USDC-denominated fees, fast finality, and programmable interoperability create a strong environment for payment networks to connect and scale infrastructure. Mastercard's blockchain lead called it "a meaningful step toward programmable money."

## LATAM: Where the Infrastructure Gap Is Most Visible

Circle's LATAM work has been methodical. In 2024, they integrated PIX (Brazil) and SPEI (Mexico) — meaning businesses and users can now convert between local currency and USDC without multi-day correspondent banking delays.

The partnership with Brazilian firm Matera gives banks the ability to hold USDC and reals in the same system — effectively removing the correspondent bank layer for certain payment flows. That's a real cost reduction for mid-sized institutions currently routing everything through US correspondent banks.

Circle's own framing on this: local USDC availability makes it attractive for businesses in LATAM where flows are already dollar-denominated. We think that's accurate — the dollar preference in the region predates stablecoins by decades, and what Circle is doing is giving that preference a programmable, low-cost infrastructure layer.

## Why the LATAM Focus Makes Sense

![Revenue growth and reserve income contribution](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Revenue%20growth%20and%20reserve%20income%20contribution%20.png)

<sub>Revenue growth and reserve income contribution (Q3 2025). Source: Circle Q3 2025 Earnings Presentation, slide 18</sub>

The stablecoin layer is where actual user activity concentrates in LATAM, and dollar access is the demand driver. Remittances, savings, cross-border freelance payments — USDC is already the instrument people reach for in the region, and has been for a while.

What Circle is doing with CPN, ARC, PIX/SPEI integration, and the Alliance grant program is giving that existing dollar preference a programmable, low-cost infrastructure layer. The Félix and BUFI cases illustrate what that looks like in practice: startups building on top of a stack where settlement, liquidity, and compliance are already handled at the infrastructure level, so the product layer can actually focus on the user.

That's a meaningful shift in what it costs — in time and capital — to launch a payment product in the region. And given how much unmet demand there is in LATAM for dollar-denominated financial services, Circle's infrastructure position here has a lot of room to compound.

