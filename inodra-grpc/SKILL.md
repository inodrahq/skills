---
name: inodra-grpc
description: Use the Inodra gRPC gateway for Sui. Use when calling sui.rpc.v2 services (Ledger, State, MovePackage, Name, TransactionExecution, Subscription), choosing field masks, paging List* calls, streaming checkpoints, or migrating from JSON-RPC.
---

# Inodra gRPC gateway

Inodra serves the Sui `sui.rpc.v2` services over real gRPC (HTTP/2, protobuf, TLS). Plain JSON over HTTP does not work on the gRPC hosts; use a generated client.

Hosts: `mainnet-grpc.inodra.com:443`, `testnet-grpc.inodra.com:443`. Send the key as `x-api-key` metadata. The key needs the `data` scope. The gateway serves the key's network on either host and never answers `NETWORK_MISMATCH`; confirm the network with `GET https://mainnet-api.inodra.com/v1/auth/me` if in doubt.

With the Inodra MCP server installed, `inodraGrpc` makes any of these calls as a tool; use it to check what a call returns before writing code.

## Client

Install `@mysten/sui`, `@grpc/grpc-js` and `@protobuf-ts/grpc-transport`.

```typescript
import { credentials } from "@grpc/grpc-js";
import { SuiGrpcClient } from "@mysten/sui/grpc";
import { GrpcTransport } from "@protobuf-ts/grpc-transport";

const client = new SuiGrpcClient({
  network: "mainnet",
  transport: new GrpcTransport({
    host: "mainnet-grpc.inodra.com:443",
    channelCredentials: credentials.createSsl(),
    meta: { "x-api-key": process.env.INODRA_API_KEY! },
  }),
});
```

`client.core.*` gives typed helpers with plain JS types, for example `client.core.listBalances({ owner, limit, cursor })` returns `{ balances, hasNextPage, cursor }`. Raw service calls return a call object; await `.response`. Streaming calls expose `.responses` as an async iterable. Raw responses carry `bigint` for numbers and `Uint8Array` for bytes and cursors.

## Services

**LedgerService**: `GetServiceInfo`, `GetObject`, `BatchGetObjects`, `GetTransaction`, `BatchGetTransactions`, `GetCheckpoint`, `GetEpoch`, and the streaming `ListTransactions`, `ListEvents`, `ListCheckpoints`.

**StateService**: `ListOwnedObjects` (filter by `objectType`), `ListDynamicFields`, `GetBalance`, `ListBalances`, `GetCoinInfo`.

**MovePackageService**: `GetPackage`, `GetDatatype`, `GetFunction`, `ListPackageVersions`.

**NameService**: `LookupName`, `ReverseLookupName` (SuiNS).

**TransactionExecutionService**: `SimulateTransaction` (unsigned BCS, dry run) and `ExecuteTransaction` (BCS plus signatures). Coin sends draw from the address balance; see the `inodra-build` skill, Send coins.

**SubscriptionService**: `SubscribeCheckpoints`, `SubscribeTransactions`, `SubscribeEvents`, with filters. Inodra accepts `x-inodra-start-checkpoint` and `x-inodra-start-cursor` metadata to start a filtered stream from the past.

**SignatureVerificationService**: `VerifySignature`.

## Field masks

Always set `readMask`. It cuts bytes and credits.

- Object: `object_id,version,digest,object_type,owner,json`
- Transaction: `digest,transaction,effects,events,balance_changes,checkpoint,timestamp`
- Checkpoint: `sequence_number,digest,summary`
- Epoch validators: `epoch,system_state.validators`

## List* paging

Filters are in disjunctive normal form: `terms` are ORed, `literals` in a term are ANDed, at most 10 literals. Transaction literals: `sender`, `affected_address`, `affected_object`, `move_call`, `emit_module`, `event_type`, `package_write`. Event literals: `sender`, `emit_module`, `event_type`, `event_stream_head`.

`options.ordering`: `0` = ASCENDING, `1` = DESCENDING. Each frame carries `watermark { cursor, checkpoint }`. The last frame carries `end { reason }`:

| `end.reason`           | Meaning                                                   | Next step                     |
| ---------------------- | --------------------------------------------------------- | ----------------------------- |
| `1` `ITEM_LIMIT`       | The page is full                                          | Continue from the cursor      |
| `2` `SCAN_LIMIT`       | The server stopped early, maybe with zero items           | Continue from the cursor      |
| `3` `CHECKPOINT_BOUND` | The requested checkpoint range is done                    | Stop                          |
| `4` `CURSOR_BOUND`     | The scan reached the cursor bound, nothing more that way  | Stop                          |
| `5` `LEDGER_TIP`       | The scan reached the newest checkpoint                    | Stop, or poll later           |

A page can end with fewer items than `limit`. Descending scans continue with `options.before`, ascending with `options.after`; pass the `Uint8Array` cursor back as-is. Inodra serves the full chain history for these calls.

Transaction status lives at `effects.status`: `{ success: boolean, error?: { description } }`.

Events by type, newest first:

```typescript
const call = client.ledgerService.listEvents({
  readMask: { paths: ["event_type", "json", "checkpoint", "transaction_digest"] },
  filter: { terms: [{ literals: [{ negated: false, predicate: {
    oneofKind: "eventType", eventType: { eventType: normalizeStructTag("0x2::display::DisplayCreated") },
  } }] }] },
  options: { limit: 3, ordering: 1 },
});
for await (const frame of call.responses) if (frame.event) console.log(frame.event);
```

```typescript
const call = client.ledgerService.listTransactions({
  readMask: { paths: ["digest", "checkpoint", "effects.status"] },
  filter: { terms: [{ literals: [{ negated: false, predicate: { oneofKind: "affectedAddress", affectedAddress: { address: "0x..." } } }] }] },
  options: { limit: 50, ordering: 1 },
});
for await (const frame of call.responses) {
  if (frame.transaction) handle(frame.transaction);
  if (frame.watermark?.cursor) cursor = frame.watermark.cursor;
}
```

## From JSON-RPC

| JSON-RPC                               | gRPC                                              |
| -------------------------------------- | ------------------------------------------------- |
| `sui_getObject`                        | `LedgerService/GetObject`                         |
| `sui_multiGetObjects`                  | `LedgerService/BatchGetObjects`                   |
| `sui_getTransactionBlock`              | `LedgerService/GetTransaction`                    |
| `suix_queryTransactionBlocks`          | `LedgerService/ListTransactions`                  |
| `suix_queryEvents`                     | `LedgerService/ListEvents`                        |
| `sui_getLatestCheckpointSequenceNumber`| `LedgerService/GetServiceInfo`                    |
| `suix_getBalance` / `getAllBalances`   | `StateService/GetBalance` / `ListBalances`        |
| `suix_getOwnedObjects`                 | `StateService/ListOwnedObjects`                   |
| `suix_resolveNameServiceAddress`       | `NameService/LookupName`                          |
| `sui_executeTransactionBlock`          | `TransactionExecutionService/ExecuteTransaction`  |

## Rules

- Responses contain `bigint` and `Uint8Array`. Convert before `JSON.stringify`.
- Addresses and types in filters must be full length (`0x000...002::coin::Coin`, 64 hex digits). The SDK helpers normalize them; raw calls with `0x2` get `INVALID_ARGUMENT`. Omit `filter` entirely instead of sending empty `terms`.
- Unary paging (`ListOwnedObjects`, `ListBalances`, `ListDynamicFields`) uses `pageToken` bytes and `pageSize`.
- Build coin sends with `tx.coin({ type, balance })` and `0x2::coin::send_funds`, not by splitting coin objects.
