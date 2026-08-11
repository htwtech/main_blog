---
title: 'Pump AMM Data Streaming on Solana: gRPC vs WebSocket'
slug: 'pump-amm-data-streaming'
canonical_url: 'https://supanode.xyz/blog/pump-amm-data-streaming'
date: "2026-08-11"
description: "Compare Yellowstone gRPC and WebSocket for streaming Pump.fun and PumpSwap activity, including lifecycle tracking, event decoding, account updates, reserves, backfill, and reconciliation."
author: "Ilya Sekretarev"
authorImage: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/images/Ilya%20Sekretarev_logo.png"
cover: "https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Pump%20AMM%20Data%20Streaming_%20gRPC%20vs%20WebSocket.png"
coverAlt: "Pump AMM Data Streaming on Solana: gRPC vs WebSocket"
tags: ["Pump AMM", "gRPC", "WebSocket"]
---

A Pump.fun token can move from a fresh mint to a completed bonding curve to a live PumpSwap pool within minutes. Monitoring this lifecycle requires watching two distinct Solana programs: the Pump program for token creation and bonding-curve trading, and PumpSwap once liquidity migrates.

For high-volume streaming, [Yellowstone gRPC](/blog/yellowstone-grpc-solana-guide) is better suited when the pipeline requires full transaction and account updates across both programs. Native WebSocket subscriptions can be enough for single-program setups, small account sets, or log-only alerts. Decoding accuracy depends on using current camel-cased IDLs, tracking migration separately from curve completion, maintaining proper commitment handling, and calculating quote reserves with the virtual component included.

## What Pump AMM Data Streaming Means on Solana

Pump.fun launches tokens against a [bonding curve controlled by the Pump program](https://github.com/pump-fun/pump-public-docs/blob/main/docs/PUMP_PROGRAM_README.md). Every buy and sell against that curve happens inside Pump itself, tracked through virtual reserves rather than a separate pool account or LP shares — that arrangement holds until the token completes.

Completion and migration should be stored as separate state transitions. A token can fill its curve before the `migrate` instruction creates or fills the PumpSwap pool, so a pipeline should not treat a completed curve as an existing AMM pool.

Once migration runs, liquidity moves into PumpSwap through the Pump program's `migrate` instruction, and trading continues on a [constant-product AMM](https://github.com/pump-fun/pump-public-docs/blob/main/docs/PUMP_SWAP_README.md). This is the "Pump AMM" most search queries are actually asking about. Pump-created migrated pools use index 0, but canonical status should be derived from the Pump migration transaction or `CompletePumpAmmMigrationEvent`, not inferred from the index alone.

| Program | Program ID | Role |
| :---- | :---- | :---- |
| Pump | `6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P` | token creation, bonding-curve trading, migration |
| PumpSwap | `pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA` | post-migration constant-product AMM, pools, swaps, liquidity |

Subscribing to only one program creates an immediate blind spot: you either lose track of tokens the moment they migrate, or process PumpSwap trades without knowing their origin. The same live-data problem appears across [Solana data streaming](/blog/solana-data-streaming-guide), but Pump AMM adds enough lifecycle state to need its own pipeline design.

![Diagram showing the Pump.fun token lifecycle from creation through bonding-curve trading, curve completion, migration, and PumpSwap pool swaps.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Pump.fun%20token%20lifecycle_%20%20from%20launch%20to%20PumpSwap%20trades.png)

<sub>What Pump AMM data streaming means on Solana.</sub>

## Pump and PumpSwap Data You Need to Monitor

A Pump AMM pipeline needs more than swap events. Some of it comes from transaction events, some only from account state, and the two don't always agree at the same instant.

* Pump `CreateEvent` — new mint, bonding curve account, creator, initial reserves.
* Pump `TradeEvent` — every fill against the curve, from `buy_v2`, `sell_v2`, or `buy_exact_quote_in_v2`.
* Pump `CompleteEvent` — the curve has filled. Migration hasn't necessarily run yet.
* Pump `CompletePumpAmmMigrationEvent` — the actual handoff into PumpSwap.
* PumpSwap pool creation, either via migration or independently through `create_pool`.
* PumpSwap `Pool` account state — base and quote mints, vault addresses, LP mint and supply, `coin_creator`, `is_mayhem_mode`, `is_cashback_coin`, and `virtual_quote_reserves`. Raw vault balances live in the associated token accounts, while trade events provide post-trade reserve snapshots.
* PumpSwap `BuyEvent` / `SellEvent` — swap fills, including a post-trade reserves snapshot.
* Deposit and withdraw activity on the LP side.

Events give point-in-time fills tied to a specific transaction. Account state gives the current picture at whatever slot you happened to read it. Current `BuyEvent` and `SellEvent` payloads carry `poolBaseTokenReserves`, `poolQuoteTokenReserves`, and `virtualQuoteReserves` after the trade. This allows the post-trade pool price to be reconstructed directly from the event once token decimals are known. Account reads remain useful for independent reconciliation.

The schema should include the newer fields from the start. `coin_creator` and [current fee-recipient accounts](https://github.com/pump-fun/pump-public-docs/blob/main/docs/BREAKING_FEE_RECIPIENT.md) matter for tracking creator-fee accrual. `is_cashback_coin` and `is_mayhem_mode` record whether specific program flags are set on the pool account; applications should not infer economic mechanics from the flags alone. `quote_mint` matters because not every pool is SOL-denominated — parsing it explicitly avoids a class of bugs tied to non-SOL pairs.

## Yellowstone gRPC vs WebSocket for Pump AMM Monitoring

Workload requirements determine the protocol choice—latency numbers alone do not tell the whole story.

| Requirement | Yellowstone gRPC | Native WebSocket |
| :---- | :---- | :---- |
| Program transaction stream | full transaction filtering per program | logs or block subscription |
| Account updates | rich, composable account filters | `programSubscribe` / `accountSubscribe` |
| Multiple program filters | supported, subject to provider limits | separate subscription per program |
| Payload richness | full transaction and account payloads | depends on the method used |
| High-volume workloads | better suited to sustained high-volume filtering | more operational overhead at scale |
| Simple launch or migration alert | works, more than the job strictly needs | usually enough on its own |
| Reconnect and gap recovery | application responsibility, assisted by client | application responsibility |
| Historical backfill | separate RPC/indexer path | separate RPC/indexer path |

For teams indexing both Pump and PumpSwap at real volume — every trade, not a sampled slice — gRPC tends to be the practical default: filtering by program and getting full transaction payloads back removes a round trip to `getTransaction` for every event of interest. Teams that only need a migration alert can usually start with WebSocket and move to gRPC once volume or account coverage grows.

The important implementation details are the filter shape, [provider limits](/docs/solana/grpc/limits), reconnect behavior, and how much enrichment the application still needs after the stream arrives. For a migration-only alert, standard PubSub patterns such as [`programSubscribe`](https://solana.com/docs/rpc/websocket/programsubscribe) and [`logsSubscribe`](https://solana.com/docs/rpc/websocket/logssubscribe) may be enough without standing up a gRPC client.

## Production Architecture for Pump.fun-to-PumpSwap Tracking

A working pipeline has three live inputs: Pump transactions, PumpSwap transactions, and PumpSwap account updates. These flow into an IDL-aware decoder, then into a normalizer and an idempotent processor. The processor writes to a state store, which feeds the database, alerts, APIs, and analytics. A checkpoint and backfill worker runs alongside the whole thing, not after it.

A minimal state registry needs to track, per token:

* mint and bonding curve address
* completion slot, if reached
* migration signature, if migrated
* PumpSwap pool address, base and quote mint
* a canonical-pool flag, derived from the migration transaction or `CompletePumpAmmMigrationEvent` rather than from pool index alone
* last processed slot and signature per stream
* the parser/IDL version used to decode each record

The parser or IDL version should be stored because Pump and PumpSwap have both shipped interface changes — the `_v2` instruction variants being one example. A pipeline that skips tracking which IDL version decoded a given record has no clean path to reprocess old data once the schema shifts again.

![Production architecture diagram for streaming Pump and PumpSwap transactions through a decoder, normalizer, idempotent processor, and checkpointed state store.](https://bvkmqlnhnryztqknjjah.supabase.co/storage/v1/object/public/content/news/Pump%20+%20PumpSwap%20streaming%20architecture.png)

<sub>Pump and PumpSwap streaming pipeline architecture.</sub>

## Stream Pump and PumpSwap Transactions with Yellowstone gRPC

The fragments below show the live subscription and decoding path — subscribe, verify invocation, parse events, normalize them, build an idempotency key, and checkpoint. Durable storage, replay, and historical backfill stay outside the live subscription loop and should be designed as separate parts of the pipeline.

The code below targets `@triton-one/yellowstone-grpc` 5.1.0, `@coral-xyz/anchor` 0.32.1, and `@solana/web3.js` 1.98.4, using the [official `pump.json`](https://github.com/pump-fun/pump-public-docs/blob/main/idl/pump.json) and [`pump_amm.json`](https://github.com/pump-fun/pump-public-docs/blob/main/idl/pump_amm.json) IDLs. Because the JSON IDLs use Rust-style names, both are passed through `convertIdlToCamelCase`. With this setup, the decoded account name is `pool`; decoded event names and fields also use lower camelCase.

```bash
npm install \
  @triton-one/yellowstone-grpc@5.1.0 \
  @coral-xyz/anchor@0.32.1 \
  @solana/web3.js@1.98.4 \
  bs58 \
  decimal.js
```

Setup and the two event coders:

```ts
import Client, {
  CommitmentLevel,
  SubscribeRequest,
  SubscribeUpdate,
  SubscribeUpdateAccount,
} from "@triton-one/yellowstone-grpc";

import { PublicKey } from "@solana/web3.js";

import {
  BorshCoder,
  EventParser,
  convertIdlToCamelCase,
  type Idl,
  type Event,
} from "@coral-xyz/anchor";

import bs58 from "bs58";

import pumpIdl from "./idl/pump.json";
import pumpAmmIdl from "./idl/pump_amm.json";

const PUMP_PROGRAM_ID = new PublicKey(
  "6EF8rrecthR5Dkzon8Nwu78hRvfCKubJ14M5uBEwF6P"
);

const PUMP_AMM_PROGRAM_ID = new PublicKey(
  "pAMMBay6oceH9fJKBRHGP5D4bD4sWpmSwMn52FMfXEA"
);

const pumpIdlCamel = convertIdlToCamelCase(pumpIdl as Idl);
const pumpAmmIdlCamel = convertIdlToCamelCase(pumpAmmIdl as Idl);

const pumpCoder = new BorshCoder(pumpIdlCamel);
const pumpEvents = new EventParser(PUMP_PROGRAM_ID, pumpCoder);

const pumpAmmCoder = new BorshCoder(pumpAmmIdlCamel);
const pumpAmmEvents = new EventParser(PUMP_AMM_PROGRAM_ID, pumpAmmCoder);

const POOL_ACCOUNT_NAME = "pool" as const;

const POOL_DISCRIMINATOR =
  pumpAmmCoder.accounts.accountDiscriminator(POOL_ACCOUNT_NAME);

const EVENT_NAMES = {
  create: "createEvent",
  complete: "completeEvent",
  migration: "completePumpAmmMigrationEvent",
  buy: "buyEvent",
  sell: "sellEvent",
} as const;
```

The [subscription request](/docs/solana/grpc/overview) covers all three inputs from the architecture above — transactions for both programs, plus an accounts filter for PumpSwap pool state. The PumpSwap program owns several account types beyond `Pool` — `BondingCurve`, `FeeConfig`, `GlobalConfig`, `GlobalVolumeAccumulator`, `SharingConfig`, `UserVolumeAccumulator` among them — so the filter narrows to the `Pool` discriminator specifically rather than returning everything the program owns:

```ts
function buildRequest(): SubscribeRequest {
  return {
    accounts: {
      pumpAmmPools: {
        account: [],
        owner: [PUMP_AMM_PROGRAM_ID.toBase58()],
        filters: [
          {
            memcmp: {
              offset: "0",
              base58: bs58.encode(POOL_DISCRIMINATOR),
            },
          },
        ],
      },
    },
    slots: {},
    transactions: {
      pump: {
        vote: false,
        failed: false,
        accountInclude: [PUMP_PROGRAM_ID.toBase58()],
        accountExclude: [],
        accountRequired: [PUMP_PROGRAM_ID.toBase58()],
      },
      pumpAmm: {
        vote: false,
        failed: false,
        accountInclude: [PUMP_AMM_PROGRAM_ID.toBase58()],
        accountExclude: [],
        accountRequired: [PUMP_AMM_PROGRAM_ID.toBase58()],
      },
    },
    transactionsStatus: {},
    entry: {},
    blocks: {},
    blocksMeta: {},
    accountsDataSlice: [],
    commitment: CommitmentLevel.CONFIRMED,
  };
}
```

The official PumpSwap IDL defines the `Pool` discriminator as `[241, 154, 109, 4, 17, 177, 109, 188]`. The code derives it from the same camel-cased IDL used by the decoder, preventing the subscription filter and account decoder from drifting apart. The discriminator begins at byte offset `0`.

Because `failed: false` is set, the two transaction filters only return successful transactions. Failed transactions are excluded from the stream by design.

`accountInclude` tells the server "this account key appears somewhere in the transaction" — close to what's needed, but with a gap: a key can appear without the program at that key ever being invoked, and versioned transactions can load extra accounts through address lookup tables that don't show up in the static key list at all. Invocation is confirmed against the combined account list — static keys plus loaded addresses — and the compiled instruction list, top-level and inner:

```ts
type TransactionInfo = NonNullable<
  NonNullable<SubscribeUpdate["transaction"]>["transaction"]
>;

function programWasInvoked(
  txInfo: TransactionInfo,
  programId: PublicKey
): boolean {
  const message = txInfo.transaction?.message;
  if (!message) return false;

  const staticKeys = message.accountKeys.map((key) =>
    new PublicKey(key).toBase58()
  );

  const loadedWritable =
    txInfo.meta?.loadedWritableAddresses.map((key) =>
      new PublicKey(key).toBase58()
    ) ?? [];

  const loadedReadonly =
    txInfo.meta?.loadedReadonlyAddresses.map((key) =>
      new PublicKey(key).toBase58()
    ) ?? [];

  const accountKeys = [
    ...staticKeys,
    ...loadedWritable,
    ...loadedReadonly,
  ];

  const programIndex = accountKeys.indexOf(programId.toBase58());
  if (programIndex === -1) return false;

  const topLevelInstructions = message.instructions;

  const innerInstructions =
    txInfo.meta?.innerInstructions.flatMap(
      (group) => group.instructions
    ) ?? [];

  return [...topLevelInstructions, ...innerInstructions].some(
    (instruction) => instruction.programIdIndex === programIndex
  );
}
```

Idempotency and checkpointing sit next to each other, since both track how far the pipeline has progressed. Transactions and account updates need separate cursors — a transaction has a slot, a transaction index, and a signature; an account write has a slot and a write version, with no transaction signature at all:

```ts
type Uint64String = string;

type EventContext = {
  signature: string;
  slot: Uint64String;
  transactionIndex: Uint64String;
};

const processedKeys = new Set<string>(); // back with a durable store in production
const seenAccountWrites = new Set<string>();

let txCheckpoint = {
  slot: "0",
  transactionIndex: "0",
  signature: "",
};

let accountCheckpoint = {
  slot: "0",
  writeVersion: "0",
  pubkey: "",
};

function makeEventKey(
  ctx: EventContext,
  programId: PublicKey,
  eventName: string,
  eventIndex: number
): string {
  return [
    ctx.slot,
    ctx.transactionIndex,
    ctx.signature,
    programId.toBase58(),
    eventName,
    eventIndex,
  ].join(":");
}

async function commitTxCheckpoint(
  slot: Uint64String,
  transactionIndex: Uint64String,
  signature: string
): Promise<void> {
  txCheckpoint = {
    slot,
    transactionIndex,
    signature,
  };

  // persist durably — a single row, a key-value entry
}

async function commitAccountCheckpoint(
  slot: Uint64String,
  writeVersion: Uint64String,
  pubkey: string
): Promise<void> {
  accountCheckpoint = {
    slot,
    writeVersion,
    pubkey,
  };

  // persist durably, separate from the transaction cursor
}
```

The idempotency key includes the program ID because Pump and PumpSwap each parse events independently, starting their own index from zero — without the program ID, a Pump event and a PumpSwap event at the same index in the same transaction would collide on the same key.

The dispatcher branches on whether an update is an account or a transaction, then hands transactions off per program. `EventParser.parseLogs()` returns a generator, so it requires a `for...of` loop:

```ts
async function handleUpdate(
  update: SubscribeUpdate
): Promise<void> {
  if (update.account) {
    await handleAccountUpdate(update.account);
    return;
  }

  const transactionUpdate = update.transaction;
  const txInfo = transactionUpdate?.transaction;

  if (!transactionUpdate || !txInfo) return;

  const message = txInfo.transaction?.message;
  const meta = txInfo.meta;

  if (!message || !meta) return;

  const signature = bs58.encode(txInfo.signature);

  // Yellowstone 5.1.0 generates uint64 fields as strings.
  const slot = transactionUpdate.slot;
  const transactionIndex = txInfo.index;
  const logs = meta.logMessages;

  const ctx: EventContext = {
    signature,
    slot,
    transactionIndex,
  };

  if (programWasInvoked(txInfo, PUMP_PROGRAM_ID)) {
    let eventIndex = 0;

    for (const event of pumpEvents.parseLogs(logs)) {
      await emitPumpEvent(event, eventIndex, ctx);
      eventIndex += 1;
    }
  }

  if (programWasInvoked(txInfo, PUMP_AMM_PROGRAM_ID)) {
    let eventIndex = 0;

    for (const event of pumpAmmEvents.parseLogs(logs)) {
      await emitPumpAmmEvent(event, eventIndex, ctx);
      eventIndex += 1;
    }
  }

  await commitTxCheckpoint(
    slot,
    transactionIndex,
    signature
  );
}
```

`emitPumpEvent` normalizes Pump-side events, writes them, and only then marks them processed — reversing that order means a failed write gets treated as a duplicate on retry instead of getting retried:

```ts
async function emitPumpEvent(
  event: Event,
  eventIndex: number,
  ctx: EventContext
): Promise<void> {
  if (
    event.name !== EVENT_NAMES.create &&
    event.name !== EVENT_NAMES.complete &&
    event.name !== EVENT_NAMES.migration
  ) {
    return;
  }

  const key = makeEventKey(
    ctx,
    PUMP_PROGRAM_ID,
    event.name,
    eventIndex
  );

  if (processedKeys.has(key)) return;

  if (event.name === EVENT_NAMES.create) {
    await store.saveLaunch({
      idempotencyKey: key,
      mint: event.data.mint.toString(),
      bondingCurve: event.data.bondingCurve.toString(),
      creator: event.data.creator.toString(),
      quoteMint: event.data.quoteMint.toString(),
      signature: ctx.signature,
      slot: ctx.slot,
      transactionIndex: ctx.transactionIndex,
      eventIndex,
    });

    processedKeys.add(key);
    return;
  }

  if (event.name === EVENT_NAMES.complete) {
    await store.saveCompletion({
      idempotencyKey: key,
      mint: event.data.mint.toString(),
      bondingCurve: event.data.bondingCurve.toString(),
      quoteMint: event.data.quoteMint.toString(),
      completedSlot: ctx.slot,
      completionSignature: ctx.signature,
      transactionIndex: ctx.transactionIndex,
      eventIndex,
    });

    processedKeys.add(key);
    return;
  }

  await store.saveMigration({
    idempotencyKey: key,
    mint: event.data.mint.toString(),
    bondingCurve: event.data.bondingCurve.toString(),
    pumpSwapPool: event.data.pool.toString(),
    quoteMint: event.data.quoteMint.toString(),
    migrationSignature: ctx.signature,
    migrationSlot: ctx.slot,
    transactionIndex: ctx.transactionIndex,
    eventIndex,
  });

  processedKeys.add(key);
}
```

In production, an idempotent insert backed by a unique constraint on the same key is a safer backstop than an in-memory set, which disappears on restart.

`emitPumpAmmEvent` follows the same write-then-mark shape for a PumpSwap trade fill. `BuyEvent` and `SellEvent` don't share a single generic amount field — a buy reports `baseAmountOut`/`quoteAmountIn`, a sell reports `baseAmountIn`/`quoteAmountOut`:

```ts
async function emitPumpAmmEvent(
  event: Event,
  eventIndex: number,
  ctx: EventContext
): Promise<void> {
  const isBuy = event.name === EVENT_NAMES.buy;
  const isSell = event.name === EVENT_NAMES.sell;

  if (!isBuy && !isSell) return;

  const key = makeEventKey(
    ctx,
    PUMP_AMM_PROGRAM_ID,
    event.name,
    eventIndex
  );

  if (processedKeys.has(key)) return;

  const baseAmount = isBuy
    ? event.data.baseAmountOut
    : event.data.baseAmountIn;

  const quoteAmount = isBuy
    ? event.data.quoteAmountIn
    : event.data.quoteAmountOut;

  await store.saveSwap({
    idempotencyKey: key,
    pool: event.data.pool.toString(),
    signature: ctx.signature,
    slot: ctx.slot,
    transactionIndex: ctx.transactionIndex,
    eventIndex,
    side: isBuy ? "buy" : "sell",
    baseAmount: baseAmount.toString(),
    quoteAmount: quoteAmount.toString(),
    poolBaseTokenReserves:
      event.data.poolBaseTokenReserves.toString(),
    poolQuoteTokenReserves:
      event.data.poolQuoteTokenReserves.toString(),
    virtualQuoteReserves:
      event.data.virtualQuoteReserves.toString(),
  });

  processedKeys.add(key);
}
```

The account branch decodes pool state through the same Anchor coder, checks the discriminator before decoding — since an `owner` filter alone would return every PumpSwap account type — and dedupes by write version rather than by signature, since account updates don't have one:

```ts
async function handleAccountUpdate(
  accountUpdate: SubscribeUpdateAccount
): Promise<void> {
  const accountInfo = accountUpdate.account;
  if (!accountInfo) return;

  const pubkey = new PublicKey(
    accountInfo.pubkey
  ).toBase58();

  const slot = accountUpdate.slot;
  const writeVersion = accountInfo.writeVersion;
  const data = Buffer.from(accountInfo.data);

  const dedupeKey = [
    pubkey,
    slot,
    writeVersion,
  ].join(":");

  if (seenAccountWrites.has(dedupeKey)) return;

  if (!data.subarray(0, 8).equals(POOL_DISCRIMINATOR)) {
    return;
  }

  const pool = pumpAmmCoder.accounts.decode(
    POOL_ACCOUNT_NAME,
    data
  );

  await store.savePoolState({
    idempotencyKey: dedupeKey,
    pool: pubkey,
    poolIndex: pool.index,
    baseMint: pool.baseMint.toString(),
    quoteMint: pool.quoteMint.toString(),
    poolBaseTokenAccount:
      pool.poolBaseTokenAccount.toString(),
    poolQuoteTokenAccount:
      pool.poolQuoteTokenAccount.toString(),
    lpMint: pool.lpMint.toString(),
    coinCreator: pool.coinCreator.toString(),
    isMayhemMode: pool.isMayhemMode,
    isCashbackCoin: pool.isCashbackCoin,
    virtualQuoteReserves:
      pool.virtualQuoteReserves.toString(),
    slot,
    writeVersion,
  });

  seenAccountWrites.add(dedupeKey);

  await commitAccountCheckpoint(
    slot,
    writeVersion,
    pubkey
  );
}
```

`store` above stands for whatever persistence layer sits underneath — a database client, a queue producer, or both.

Yellowstone Node client 5.1.0 provides native reconnect, bounded replay, and deduplication layers. Authentication credentials (`xToken`) are configured directly with the endpoint:

```ts
async function run(): Promise<void> {
  const endpoint = process.env.YELLOWSTONE_ENDPOINT;
  const xToken = process.env.YELLOWSTONE_X_TOKEN;

  if (!endpoint) {
    throw new Error(
      "YELLOWSTONE_ENDPOINT is required"
    );
  }

  const client = new Client(
    endpoint,
    xToken,
    {},
    {
      backoff: {
        initialIntervalMs: 100,
        multiplier: 2,
        maxRetries: 10,
      },
      slotRetention: 250,
    }
  );

  await client.connect();

  const stream = await client.subscribe(
    buildRequest()
  );

  let processing = Promise.resolve();

  stream.on("data", (update: SubscribeUpdate) => {
    processing = processing
      .then(() => handleUpdate(update))
      .catch((error) => {
        console.error(
          "update processing failed",
          error
        );
      });
  });

  stream.on("error", (error) => {
    console.error(
      "Yellowstone stream terminated",
      error
    );
  });

  stream.on("end", () => {
    void processing.finally(() => {
      console.error("Yellowstone stream ended");
    });
  });
}

void run().catch((error) => {
  console.error("fatal Yellowstone client error", error);
  process.exitCode = 1;
});
```

Under normal delivery, the transaction filters above return successful, matching Pump and PumpSwap transactions observed by the endpoint at the selected commitment level. This provides targeted coverage of the filtered dataset, excluding failed transactions by design.

## Detect Launches, Completion, and PumpSwap Migrations

`CreateEvent` and `CompleteEvent` are handled inside `emitPumpEvent()` above with the same durable-write-before-dedupe pattern as `CompletePumpAmmMigrationEvent`.

The normalized lifecycle record can combine the exact decoded fields shown above with stream context:

```json
{
  "mint": "...",
  "bondingCurve": "...",
  "creator": "...",
  "quoteMint": "...",
  "completedSlot": "0",
  "migrationSignature": "...",
  "pumpSwapPool": "..."
}
```

A token's `complete` flag confirms the curve is full. Migration status is a separate check, confirmed only once `CompletePumpAmmMigrationEvent` arrives.

## Decode PumpSwap Pools, Swaps, and Liquidity Changes

PumpSwap's current IDL covers pool creation, trading, and liquidity instructions — `create_pool`, `buy`, `buy_exact_quote_in`, `sell`, `deposit`, `withdraw` — alongside `BuyEvent` and `SellEvent` for trade fills, shown decoded above. Pump-created migrated pools use index 0, but canonical status should be derived from the migration transaction or `CompletePumpAmmMigrationEvent` rather than inferred from the index alone.

For decoding and storage:

1. Token decimals vary by mint — apply them before any human-readable price shows up in a dashboard or alert.
2. `quote_mint` isn't fixed to SOL across every pool; parse it per-pool.
3. `is_mayhem_mode` and `is_cashback_coin` record whether specific program flags are set on the pool account; applications should not infer economic mechanics from the flags alone.
4. IDLs evolve — fields get appended, instruction variants get added. Storing raw instruction/event bytes alongside the decoded record keeps reparsing possible without replaying the chain.

Across the pipeline, five record types cover the full lifecycle:

| Record type | Required fields |
| :---- | :---- |
| Token launch | mint, creator, bonding curve, quote mint, slot |
| Completion | mint, completion slot, signature |
| Migration | mint, PumpSwap pool, migration signature |
| Swap | pool, side, base amount, quote amount, reserves, event index |
| Pool state | pool, base mint, quote mint, balances, virtual reserves, slot |

A normalized swap record, stored alongside the raw payload:

```json
{
  "pool": "...",
  "signature": "...",
  "slot": "0",
  "transactionIndex": "0",
  "eventIndex": 0,
  "side": "buy",
  "baseAmount": "...",
  "quoteAmount": "...",
  "poolBaseTokenReserves": "...",
  "poolQuoteTokenReserves": "...",
  "virtualQuoteReserves": "..."
}
```

The trade event contains the post-trade base and quote pool reserves. Effective quote reserves are obtained by adding `virtualQuoteReserves` to `poolQuoteTokenReserves`; token decimals are then applied before calculating the pool price. Pool-account and vault-account reads remain a separate reconciliation path.

## Calculate PumpSwap Price and Liquidity Correctly

PumpSwap pricing breaks if virtual quote reserves get left out of the quote side. The quote side should include both the quote vault balance and `virtual_quote_reserves` before decimals are applied:

`effective_quote_reserves = pool_quote_token_account.amount + pool.virtual_quote_reserves`

Base reserves stay as the raw base-vault token balance, with no virtual component on that side. Reserves are `u64` and `virtual_quote_reserves` can be a wider signed integer type — large enough that standard JavaScript `Number` conversions silently lose precision above `2^53 - 1`. A decimal-safe division keeps the result exact:

```ts
import Decimal from "decimal.js";

function computePostTradePoolPrice(input: {
  poolBaseTokenReserves: string;
  poolQuoteTokenReserves: string;
  virtualQuoteReserves: string;
  baseDecimals: number;
  quoteDecimals: number;
}): Decimal {
  const rawBaseReserves = new Decimal(
    input.poolBaseTokenReserves
  );

  const effectiveQuoteReserves = new Decimal(
    input.poolQuoteTokenReserves
  ).plus(input.virtualQuoteReserves);

  if (rawBaseReserves.lte(0)) {
    throw new Error(
      "Pool base-token reserves must be positive"
    );
  }

  if (effectiveQuoteReserves.lte(0)) {
    throw new Error(
      "Effective quote-token reserves must be positive"
    );
  }

  const baseReserves = rawBaseReserves.div(
    new Decimal(10).pow(input.baseDecimals)
  );

  const quoteReserves = effectiveQuoteReserves.div(
    new Decimal(10).pow(input.quoteDecimals)
  );

  return quoteReserves.div(baseReserves);
}
```

Event-derived reserves and account-snapshot reserves can reflect different slots if read at different moments. Mixing the two produces a price that never actually existed on-chain. Keep every reserve figure tagged with the slot it came from, and avoid combining values across slots even when the mismatch looks small. Pool price, trade execution price, liquidity, and market cap are four distinct numbers — conflating any pair of them tends to produce a metric that looks plausible while meaning something else entirely.

## Make the Stream Reliable in Production

A production stream needs recovery behavior from the start. The main requirements are straightforward:

* Commitment level matters here. [`processed` data can roll back](https://solana.com/docs/rpc); we default to `confirmed` unless there's a specific reason to wait for `finalized`.
* Idempotency key: signature, program ID, event or instruction index — two programs can each emit an event at the same index within one transaction, so the program ID has to be part of the key.
* Out-of-order updates are common — order transactions by slot and transaction index, and account updates by slot and write version, rather than trusting delivery order alone.
* Reconnect options in `@triton-one/yellowstone-grpc` enable native stream reconnection, bounded replay, and deduplication while preserving the original subscription request. Durable application checkpoints are still required to coordinate successful storage commits and to recover beyond the configured replay window.
* Checkpoints should be persisted outside process memory, with separate cursors for transactions (`slot`, `transactionIndex`, `signature`) and accounts (`slot`, `writeVersion`, `pubkey`).
* Account updates need their own dedupe key — pubkey plus slot and write version — separate from the signature-based key used for transaction events.
* Gap detection must use stream-specific continuity: transaction or block indexes where available, observed rooted slots, provider replay cursors, and reconciliation against RPC or an indexer. A missing slot number alone does not confirm data loss, since [Solana permits skipped slots](https://solana.com/docs/references/terminology) when a leader produces no accepted block.
* Route parser failures into a dead-letter queue so schema changes do not silently drop undecoded events.
* Useful metrics: stream lag, reconnect count, decode failures, gaps, events per second.

Most of this applies to any high-volume Solana stream. The pace of memecoin trading just surfaces gaps faster and makes them matter sooner.

## When WebSocket Is Enough—and When to Use gRPC

WebSocket tends to cover the job when:

* monitoring one program or a small, fixed set of accounts
* log-only alerts cover the use case
* volume stays manageable without heavy filtering
* occasional RPC calls for enrichment are acceptable

gRPC becomes the better fit when:

* full transactions and account updates both matter
* Pump and PumpSwap need tracking together, at scale
* the goal is an indexer, a trading feed, or a production analytics pipeline
* precise filters and lower serialization overhead change the design or operating cost

A minimal WebSocket version of the migration alert reuses the same event parser defined above:

```ts
import { Connection } from "@solana/web3.js";

const connection = new Connection(
  RPC_HTTP_URL,
  { wsEndpoint: RPC_WS_URL }
);

connection.onLogs(
  PUMP_PROGRAM_ID,
  (logInfo, ctx) => {
    if (logInfo.err) return;

    const events = pumpEvents.parseLogs(logInfo.logs);

    for (const event of events) {
      if (event.name === EVENT_NAMES.migration) {
        console.log("migration detected", {
          signature: logInfo.signature,
          slot: ctx.slot,
          mint: event.data.mint.toString(),
          pool: event.data.pool.toString(),
        });
      }
    }
  },
  "confirmed"
);
```

This covers one program per subscription — a second `onLogs` call handles PumpSwap the same way. `@solana/web3.js` v1 automatically reconnects and attempts to restore subscriptions after an implicit socket close, but it does not replay notifications missed during the disconnected interval. Production use still needs the same checkpoint and reconciliation discipline as the gRPC path.

Performance beyond that depends on node placement, provider architecture, workload shape, and filter precision — not on a single benchmark number that applies everywhere equally.

## Build the Live Stream and Historical Data Path Together

A live stream only covers what happens after the subscription opens. Production setups split the work three ways: live ingestion — the gRPC or WebSocket layer above — historical backfill through RPC or [an indexer](/docs/solana/indexer/overview) for anything before the subscription started, and periodic reconciliation that compares live-derived state against canonical account state to catch whatever the stream missed.

Trading bots in particular need all three running at once: live fills for execution, historical state for context, reconciliation to catch drift before it compounds. The same split shows up across [Solana trading bot infrastructure](/blog/solana-trading-bot-infrastructure), where execution logic depends on fresh streams while strategy and risk checks often need stored state.

For PumpSwap backfill and reconciliation, decoded historical swap and migration records are usually easier to query through an indexer than to reconstruct from live-stream logic alone.

## Conclusion

Full-lifecycle Pump AMM monitoring means tracking Pump and PumpSwap as two distinct programs, decoded against whichever IDL is currently live rather than an outdated schema. WebSocket covers simple alerts; gRPC covers structured, high-volume ingestion across both programs at once, including account state. Reserves, read correctly with virtual components included, drive accurate pricing. Backfill and reconciliation belong in the design from day one, before the first production gap has to be reconstructed.

[Supanode by HighTower Yellowstone gRPC](https://supanode.xyz/services/solana/grpc) can provide the live Pump and PumpSwap transaction and account streams for this setup.
