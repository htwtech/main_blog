---
title: "What Is NEAR—and What Are NEAR Intents"
slug: "what-is-near-and-near-intents"
date: "2026-02-24"
description: "Cross-chain swaps reimagined — one outcome request, one settlement layer."
author: "HighTower Research"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/HTW.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/NEAR%20Intents%20Explained.png"
coverAlt: "NEAR Intents Explained"
tags: ["near", "intents", "cross-chain", "infrastructure", "bridges"]
---

# What Is NEAR — and What Are NEAR Intents

Cross-chain swaps still feel like ops work: pick a bridge, manage gas twice, hope nothing breaks mid-route. NEAR Intents attempts to collapse that into a single “outcome request”, enforced by a settlement layer worth inspecting.

## What NEAR Is

NEAR is a Layer-1 blockchain with WebAssembly-compiled contracts and an account model that matters for what follows. Unlike most chains where an address is just a public key hash, NEAR accounts are named (“alice.near”) and can hold multiple keys with different permission scopes. One key might have full access; another might only be allowed to call specific contract functions. Apps and agents can hold scoped authority without ever touching the master key.

This matters for Intents because the moment you’re coordinating across multiple chains—with wallets, solvers, relayers all acting in sequence on a user’s behalf—delegating specific permissions without handing over full custody becomes a hard requirement. NEAR’s account model maps cleanly to that requirement.

Intents adds extra hops and coordination. Low per-step cost and fast finality (theoretical hard finality ~1.2s, empirical closer to ~3.5s depending on RPC and network conditions) make that overhead survivable.

## Chain Signatures

Before Intents make sense, there’s one more base layer primitive worth naming: Chain Signatures.

Chain Signatures use threshold MPC (multi-party computation) to allow NEAR accounts—including smart contracts—to sign and broadcast transactions on other blockchains. Bitcoin, Solana, EVM chains, all without any single party holding a complete private key.

In practice: a NEAR contract can construct a valid Bitcoin transaction and sign outbound transactions via distributed key shares. The actual trust model then depends on the bridge and deposit path you use—PoA vs OmniBridge—which is where the real assumptions sit today.

This is the primitive that makes NEAR Intents coherent as a system.

## NEAR Intents: The Inversion

The standard cross-chain swap experience is route assembly. A user finds a bridge, moves funds, manages gas on two chains, handles slippage in two windows, then maybe lands in the right place. The full sequence takes several transactions, and each one is a new failure point.

Instead of asking the user to assemble a route, Intents turns the route into a solver problem. A user—or a wallet acting on their behalf—signs an outcome: “I want USDC on Arbitrum, and I’m offering SOL from Solana.” Solvers compete on execution, and settlement is enforced on NEAR via the Verifier contract (intents.near).

![Intents separate coordination](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Intents%20separate%20coordination.png)

<sub>Intents separate coordination (off-chain) from enforcement (on-chain), with the Verifier contract and token bridges handling final execution https://docs.near-intents.org/getting-started/what-are-intents</sub>

In practice, “intent-based” here means outcome specification rather than route specification.

The docs frame this with a simple analogy: order an outcome, don’t micromanage the route. The protocol is designed to extend beyond swaps into any marketplace where competitive fulfillment makes sense, though whether it generalizes beyond token trading is still unproven.

NEAR’s own docs describe the system through three components:

- Distribution Channels—wallets, apps, and exchanges that create intent requests and surface them to solvers
- Market Makers / Solvers—liquidity providers and execution agents competing to fulfill those requests
- Verifier Contract—the on-chain settlement layer that mediates between user and solver, atomically

![NEAR Intents at a glance](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/NEAR%20Intents%20at%20a%20glance.png)

<sub>NEAR Intents at a glance: users express outcomes, distribution channels broadcast intents, market makers compete, settlement happens via the Verifier contract on NEAR. https://docs.near.org/chain-abstraction/intents/overview</sub>

There’s also a Message Bus, an off-chain system that handles price discovery and communication between distribution channels and solvers. The Message Bus is the default coordination layer. Solvers can also index on-chain intents directly, so the settlement layer isn’t structurally dependent on the bus.

![NEAR Intents components](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/NEAR%20Intents%20components.png)

<sub>NEAR Intents components: off-chain Message Bus coordinating quotes, Market Makers (solvers), and on-chain settlement via the Verifier https://docs.near-intents.org/integration/market-makers/message-bus/introduction</sub>

## How the Flow Actually Works

A useful mental model: a CEX-like deposit/trade/withdraw sequence—except execution and settlement are enforced on NEAR, while ingress and egress depend on the bridge path (PoA vs OmniBridge) and the operational model you choose.

**Deposit.** A user gets a deposit address on the origin chain, derived via Chain Signatures. They send funds; a bridge service detects the finalized deposit and mints a representation into intents.near.

Right now, most deposits use a PoA (Proof of Authority) bridge—fast, operationally simple, but trust-dependent. OmniBridge offers an alternative with stronger trust assumptions, combining light clients, Wormhole relay, and Chain Signatures depending on the target chain's constraints. It's rolling out chain by chain.

**Execution.** Solvers see the intent, submit competing quotes, and the Verifier settles atomically on NEAR. The base protocol fee is 0.0001% per transaction—effectively negligible at the protocol layer.

**Withdrawal.** The bridged representation burns on NEAR, and Chain Signatures constructs and signs an outbound transaction from the destination-chain treasury to the user’s address. NEAR publishes treasury addresses for all supported networks—Bitcoin, Solana, EVM chains, TRON, TON, Stellar, Starknet, and others—for transparency.

## 1Click: What It Abstracts Away

Most mainstream integrations—wallets, apps, exchanges—go through the 1Click API rather than the raw Intents protocol. 1Click is a REST layer that packages intent creation, solver coordination, and transaction handling into a simple request/poll flow. A distribution channel requests a quote, deposits funds, and polls for status. Either the swap delivers to the destination address or it refunds. That’s the full surface area from an integrator’s perspective.

![1Click in one picture](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/1Click%20in%20one%20picture.png)

<sub>1Click in one picture: the integrator requests a quote and receives a unique deposit address (POST /v0/quote)</sub>

Under the hood, 1Click introduces an extra trust surface: per the docs, assets temporarily transfer to a swapping agent coordinating with market makers. 1Click is developed by Defuse Labs Limited, incorporated in Gibraltar, and its ToS frames it as a non-regulated service—relevant when modeling compliance assumptions. It interacts with OmniBridge, HOT Bridge, or the PoA Bridge depending on origin chain and configuration.

![1Click swap lifecycle](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/1Click%20swap%20lifecycle.png)

<sub>1Click swap lifecycle</sub>

For wallet teams shipping cross-chain functionality fast, this packaging is the point. For anyone modeling trust assumptions in detail: the routing layer runs AML screening on quote requests (TRM Labs datasets for non-dry quotes, automatic blocking for flagged categories), and the Verifier contract is upgradeable and pausable under security council governance—a practical necessity for a fast-moving protocol, and worth knowing when calibrating what “on-chain atomic settlement” means in the current deployment.

## A Trust Map

Intents looks clean at the settlement layer. The real differentiator is how many assumptions you’re willing to take on ingress/egress and orchestration.

Since trust is distributed across layers, here’s the actual assumption map by component:

1. PoA Bridge—third-party custody during transit; fast, operationally dependent
2. OmniBridge—Chain Signatures MPC plus network-specific relay (e.g., Wormhole); trust profile varies by chain
3. Verifier Contract—on-chain, atomic settlement; upgradeable and pausable under governance
4. 1Click API—trusted swapping agent; third-party component interactions
5. Routing Layer—compliance-screened quote requests; permissioning above the protocol level

## Volume

As of February 24, 2026, per the public Intents Explorer:

- All-time volume: ~$13.46B
- Last 30 days: ~$2.67B
- Last 7 days: ~$369.66M
- 24 hours: ~$69.65M

![NEAR Intents Explorer](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/NEAR%20Intents%20Explorer.png)

<sub>NEAR Intents Explorer: https://explorer.near-intents.org/</sub>

The referral breakdown is more informative than the headline. Top affiliates in the Explorer include SwapKit, Zashi, Ledger SwapKit, Electric Coin Co, Trust SwapKit—embedded wallet integrations, not users navigating to near-intents.org directly. Volume coming through distribution partners is where you’d want it if the goal is to become default execution infrastructure rather than a destination app.

## Why the Architecture Is Interesting

Solver-based cross-chain settlement is an active design space. Across, deBridge, and Mayan Swift work in overlapping territory, and academic work on this class of bridges flags structural risks: solver liquidity concentration, delayed settlement under stress, availability failures in adversarial conditions. NEAR Intents operates in the same environment.

What NEAR’s stack contributes specifically is the combination of Chain Signatures as a native cross-chain signing layer, an account model designed for delegated authority, and a modular bridge architecture where swapping and bridging are genuinely separated.

The “no testnet” policy—no testnet exists and there are no plans to add one—is a production-first choice: testnet liquidity doesn’t model mainnet liquidity, and the team would rather have developers work with separate mainnet accounts.

The model assumes that if cross-chain execution becomes cheap and embeddable, and order flow shifts to wallets and apps that already own the user relationship. Wallet integrations embed order flow directly at the UX layer, which tends to be more durable than standalone DEX usage. If Chain Signatures expands coverage across chains and asset types, NEAR starts to look less like a destination L1 and more like a coordination and settlement layer sitting behind other frontends.

The remaining variable is how ingress trust surfaces and governance evolve—particularly OmniBridge’s rollout and council authority over the Verifier.
