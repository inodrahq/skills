---
name: inodra-grpc
description: Use the Inodra gRPC gateway for Sui. Use when calling sui.rpc.v2 services (Ledger, State, MovePackage, Name, TransactionExecution, Subscription), choosing field masks, paging List* calls, streaming checkpoints, or migrating from JSON-RPC.
---

# Inodra gRPC gateway

Inodra serves the Sui `sui.rpc.v2` services over real gRPC (HTTP/2, protobuf, TLS). Plain JSON over HTTP does not work on the gRPC hosts; use a generated client.

Hosts: `mainnet-grpc.inodra.com:443`, `testnet-grpc.inodra.com:443`. Send the key as `x-api-key` metadata. The key needs the `data` scope.

## Client

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

`client.core.*` gives typed helpers. Raw service calls return a call object; await `.response`. Streaming calls expose `.responses` as an async iterable.

## Services

**LedgerService**: `GetServiceInfo`, `GetObject`, `BatchGetObjects`, `GetTransaction`, `BatchGetTransactions`, `GetCheckpoint`, `GetEpoch`, and the streaming `ListTransactions`, `ListEvents`, `ListCheckpoints`.

**StateService**: `ListOwnedObjects` (filter by `objectType`), `ListDynamicFields`, `GetBalance`, `ListBalances`, `GetCoinInfo`.

**MovePackageService**: `GetPackage`, `GetDatatype`, `GetFunction`, `ListPackageVersions`.

**NameService**: `LookupName`, `ReverseLookupName` (SuiNS).

**TransactionExecutionService**: `ExecuteTransaction`, `SimulateTransaction`.

**SubscriptionService**: `SubscribeCheckpoints`, `SubscribeTransactions`, `SubscribeEvents`, with filters. Inodra accepts `x-inodra-start-checkpoint` and `x-inodra-start-cursor` metadata to start a filtered stream from the past.

**SignatureVerificationService**: `VerifySignature`.

## Field masks

Always set `readMask`. It cuts bytes and credits.

- Object: `object_id,version,digest,object_type,owner,json`
- Transaction: `digest,transaction,effects,events,balance_changes,checkpoint,timestamp`
- Checkpoint: `sequence_number,digest,summary`
- Epoch validators: `epoch,system_state.validators`

## List* paging

Filters are in disjunctive normal form: `terms` are ORed, `literals` in a term are ANDed, at most 10 literals. Transaction literals: `sender`, `affected_address`, `affected_object`, `move_call`, `emit_module`, `event_type`, `package_write`. Event literals: `sender`, `emit_module`, `event_type`.

Each frame carries `watermark { cursor, checkpoint }`. The last frame carries `end { reason }`:

- `ITEM_LIMIT`: the page is full, continue from the cursor
- `SCAN_LIMIT`: the server stopped early, maybe with zero items, continue from the cursor
- `CHECKPOINT_BOUND`: the range is done

Descending scans continue with `options.before`, ascending with `options.after`. Inodra serves the full chain history for these calls.

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
- Unary paging (`ListOwnedObjects`, `ListBalances`, `ListDynamicFields`) uses `pageToken` bytes and `pageSize`.
- Gas from the address balance needs `SimulateTransaction` with `doGasSelection`; the SDK handles it.
