---
title: "Perp DEX: March Snapshot"
slug: "perp-dex-march-snapshot"
date: "2026-03-13"
description: "Perp DEX design in 2026 is converging around infrastructure, constraint design, and stablecoin collateral."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Perp%20DEX_%20%20March%20Snapshot.png"
coverAlt: "Perp DEX March Snapshot"
tags: ["perps", "dex", "lighter", "variational", "edgex", "hibachi"]
---

# Perp DEX Snapshot: Lighter, Variational, edgeX, Hibachi

Perp DEX design in March 2026 is converging around a few clear ideas — venues are starting to compete on infrastructure design, and four projects illustrate that shift well: Lighter, Variational, edgeX, and Hibachi.

Stablecoin collateral is becoming the default foundation for perp venues — and the past couple of months of product updates from Lighter, Variational, edgeX, and Hibachi show how differently teams are approaching it.

## Lighter

The most revealing update from Lighter is the Partner Attribution program. In practice, this means third-party frontends can plug into Lighter's trading infrastructure, configure their own fees (within global limits — perps up to 10 bps, spot up to 1%), and earn fees from their own flow. The user consent layer is explicit: signed approvals, revocable, optionally expiring.

Effectively, this signals that Lighter is competing for distribution at least as much as for end-users. The backend becomes the product.

The fee model underneath it is fairly straightforward:

- Standard accounts trade for free  
- Premium accounts pay maker/taker fees and buy execution advantages through staking tiers  
- LIT Fee Credits let semi-pro users rent higher tier access time-boxed, with proceeds streamed to LIT stakers  

And then there's the ARC perp stress event from late February — a whale liquidation that became a very visible test of the system. The system limited LP losses to roughly $75k despite ~$50M open interest, after which the team added a $40M OI cap and moved the pair to a capped strategy with about $100k USDC allocated capital, with ADL triggered once that's exhausted. The explicit caps are the point. In 2026, perp venues increasingly try to earn trust by making failure limits explicit, and Lighter is leaning into that.

## Variational

Variational's "zero fees + redistribution" model is well established by now. The newer part is how the protocol is managing the cost of that promise.

Loss Refunds are funded by 10% of spreads and operate as a tier-based lottery, with one hard constraint: a maximum of one refund per account per week. At its core, the weekly cap is an actuarial limit — the team is pricing the real cost of refunds.

On the product side, TP/SL orders now incorporate explicit slippage limits. Triggers are checked every 0.1 seconds and can stay pending rather than executing into worst-case volatility — the design logic being that a trigger held is better than a fill at a terrible price.

![Variational economic loop](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Variationals%20March%20update%20shows%20the%20full%20economic%20loop.png)

*Variational’s March update shows the full economic loop*  
Source: https://discord.com/channels/1196857788220067943/1196907421575164015/1479535156397080841

The OLP (Omni Liquidity Provider) framing stays consistent throughout: OLP quotes spreads, takes the other side, hedges externally, and the protocol takes a percentage of spreads (currently 10%, routed to treasury, still being tested). Put together, Variational's recent posture reads as: safer UX for traders, tighter economics on the refund hook, and gradual formalization of protocol revenue routing.

## edgeX

edgeX's product direction looks increasingly like an attempt to replicate a full CEX-style trading terminal. Two recent examples are worth singling out.

Scaled orders: the help center defines these as automated multiple limit orders within a chosen price range — 2 to 50 suborders, with multiple distribution modes (average, ascending, descending, random). Classic pro-trader tooling for reducing market impact when entering or exiting large positions.

Stock perpetuals: edgeX has operationalized equity-style exposure with strict microstructure rules for weekends and holidays. Market orders are rejected during closed-market periods, limit orders are constrained to a defined price range, mark price is clamped to 0.5% per 3 seconds, and conditional orders triggered while the underlying is closed get special handling. The asset class itself matters less here — what edgeX is doing systematically is adding explicit, legible constraints to reduce pathological conditions.

The stablecoin layer is where edgeX becomes part of a larger story. Circle announced native USDC and CCTP integration on EDGE Chain on February 10 (alongside a disclosed Circle Ventures investment into edgeX), then on March 9 confirmed both are live, with USDC positioned as core margin collateral for EDGE apps, and a migration path from bridged USDC.e toward native USDC. CCTP's burn-and-mint model handles cross-chain movement without fragmented liquidity or bridge risk. The collateral layer is being rebuilt at the infrastructure level, and edgeX is early on that path.

## Hibachi

Hibachi's most interesting update this cycle is the decision to build a stablecoin-focused FX venue on Arc — and the reasoning behind that choice is worth unpacking.

The February 12 announcement frames Hibachi as an Arc Builders Fund participant developing an FX exchange targeting spot and derivatives across stablecoin FX pairs, with explicit KYC and AML screening for institutional trading firms. The positioning is clear: a venue stablecoin treasuries will actually need, which targets a very different market than the retail perp trader crowd.

The numbers behind that thesis are straightforward:

- BIS data puts global FX trading at $9.6T per day (April 2025)  
- Stablecoins closed 2025 at ~$311B market cap  
- Circle is simultaneously pushing Arc's built-in FX engine narrative (StableFX on Arc testnet)  

Hibachi's Arc-based FX direction aligns with what Arc is actually designed for, which makes this look less like a casual ecosystem partnership and more like a deliberate bet on where stablecoin capital markets are heading.

On the current-product side, Hibachi is running a points-driven growth loop with unusually explicit mechanics:

- 500,000 points per epoch base schedule, distributed weekly (Mondays 3:00 UTC)  
- During "March Madness," weekly issuance doubled to 1M points temporarily  
- Multi-level referral program: 10% first degree, 5% second, 1% third  
- Explicit abuse-prevention stance against wash trading and multi-account farming  

Around 1,000,000 points distributed to roughly 2,800 traders in a single week, with total points in circulation at 44.5M. The incentive structure is legible and the abuse guardrails are documented. The posture is: spend points to buy attention now, but keep the program enforceable and tied to real trading volume while the FX-on-Arc narrative develops.

---

Perps venues in 2026 are competing on constraint design as much as on features. The team that makes its failure limits most explicit — OI caps, weekend market rules, slippage-bounded triggers, actuarial refund ceilings — is, probably, the one institutions will trust the most over time.

We think the Hibachi FX bet is the most structurally interesting direction in this group right now. It's the clearest case of a perp venue deliberately creating a new product category, and the Arc alignment gives it a more coherent foundation than a standalone announcement would.
