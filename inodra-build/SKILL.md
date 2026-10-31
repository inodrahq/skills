---
name: inodra-build
description: Build Sui applications on Inodra. Use when writing code that reads Sui data, picks a gateway (gRPC, GraphQL, JSON-RPC), authenticates with an Inodra API key, or needs to understand credits, scopes and networks.
---

# Build with Inodra

You help the user integrate a Sui application with Inodra. Follow these rules.

## MCP server (optional)

Everything in these skills works with plain HTTP and gRPC calls; no MCP server is required. If the Inodra MCP server is installed, use its tools for one-off lookups and to check results: `inodraAccount`, `inodraObject`, `inodraTransaction`, `inodraChain`, `inodraNaming`, `inodraGrpc`, `inodraRpc`, `inodraWarp`, `inodraWebhook`, `inodraManagement`. Write code when the result must run again or ship in an app. To install it, the user runs `claude mcp add inodra -- npx -y @inodra/mcp-server` in Claude Code, or adds `npx -y @inodra/mcp-server` as a stdio server in another client; the first `inodraConnect` call signs them in. See the package README for details.

## Pick the gateway

- **gRPC** (default): the Sui `sui.rpc.v2` services over real gRPC with TLS. Typed clients, field masks, full history for the `List*` calls. Use it for reads, writes and streams.
- **GraphQL**: the Sui GraphQL RPC 2.0 schema with full history. Use it when one query must join several objects or when the app already uses GraphQL.
- **JSON-RPC**: a translation layer over gRPC for legacy code. Do not use it in new code.
- **Platform routes** under `/v1` are supported: `/webhooks`, `/warp`, `/catalog`, `/usage/stats`, `/auth/me`, `/account`, `/keys`, `/projects`, `/billing`. The data convenience routes (`/v1/accounts`, `/coins`, `/objects`, `/packages`, `/chain`, `/validators`, `/suins`) are not a stable surface. Read chain data over gRPC or GraphQL.

## Hosts

| Network | gRPC                         | HTTP (GraphQL, JSON-RPC, Warp, webhooks) |
| ------- | ---------------------------- | ---------------------------------------- |
| mainnet | `mainnet-grpc.inodra.com:443` | `https://mainnet-api.inodra.com/v1`      |
| testnet | `testnet-grpc.inodra.com:443` | `https://testnet-api.inodra.com/v1`      |

A key belongs to one project, and a project belongs to one network. On HTTP, a key sent to the other network's host gets `403 NETWORK_MISMATCH`. On gRPC there is no such error: both gRPC hosts serve the key's network, so a mainnet key on `testnet-grpc` silently returns mainnet data. Check the key's network with `GET /v1/auth/me` before you trust a "testnet" result.

## Authenticate

Send the key in the `x-api-key` header, or as `x-api-key` call metadata on gRPC.

```typescript
const headers = { "x-api-key": process.env.INODRA_API_KEY! };
```

Scopes on a key:

- `data`: gRPC, GraphQL, JSON-RPC, and the protocol event catalog
- `streams`: connect to an existing Warp stream
- `manage`: create and change webhooks and Warp streams, usage stats, key info, keys and projects

`403 INSUFFICIENT_SCOPE` means the key lacks the scope. Give an agent a `data` key unless it must manage webhooks or streams. Never give an agent a `manage` key only to read data.

## gRPC client

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

// Core helpers (SDK v2 takes `owner`, not `address`)
const balance = await client.core.getBalance({ owner: "0x...", coinType: "0x2::sui::SUI" });

// Raw service call with a field mask
const tx = await client.ledgerService.getTransaction({
  digest: "...",
  readMask: { paths: ["digest", "effects", "events"] },
}).response;
```

## Address history

Use `LedgerService.ListTransactions` with an `affected_address` or `affected_object` filter. Inodra serves the full chain history for these calls. Pages come as a stream; each frame has a `watermark.cursor`. Continue a descending scan with `options.before`, an ascending one with `options.after`. A frame with `end.reason = SCAN_LIMIT` can arrive with zero items: resume from the cursor, do not stop.

The `client.core.listTransactions` helper only accepts `sender` or `function` filters; address history needs the raw call:

```typescript
const call = client.ledgerService.listTransactions({
  readMask: { paths: ["digest", "checkpoint", "effects.status"] },
  filter: { terms: [{ literals: [{ negated: false, predicate: {
    oneofKind: "affectedAddress", affectedAddress: { address: normalizeSuiAddress("0x...") },
  } }] }] },
  options: { limit: 50, ordering: 1 }, // 1 = DESCENDING, newest first
});
for await (const frame of call.responses) {
  if (frame.transaction) handle(frame.transaction);
  if (frame.watermark?.cursor) cursor = frame.watermark.cursor; // pass back as options.before
}
```

Addresses and type strings in filters must be full length (`0x` plus 64 hex digits); `normalizeSuiAddress` and `normalizeStructTag` from `@mysten/sui/utils` do that.

## Send coins

When the user asks to send SUI or any other coin, draw the amount from the sender's address balance. Do not list coin objects, split one and transfer the piece. Sui is moving coins into address balances; when the sender holds the funds as balance there is no coin object to split.

```typescript
import { Transaction } from "@mysten/sui/transactions";

const SUI = "0x2::sui::SUI";
const tx = new Transaction();
tx.setSender(sender);
const coin = tx.coin({ type: SUI, balance: 1_000_000_000n }); // 1 SUI from the address balance
tx.moveCall({
  target: "0x2::coin::send_funds", // lands in the recipient's address balance
  typeArguments: [SUI],
  arguments: [coin, tx.pure.address(recipient)],
});
const bytes = await tx.build({ client });
```

`tx.coin` (or the `coinWithBalance` intent) withdraws from the address balance when it covers the amount and falls back to owned coins only when it does not. `client.core.getBalance` returns both `balance` (total) and `addressBalance`. Use `tx.transferObjects([coin], recipient)` only when the recipient must get a coin object, for example a contract that takes `Coin<T>` by object.

Sign `bytes` with the user's wallet or keypair. Dry run with `TransactionExecutionService.SimulateTransaction`, then submit with `ExecuteTransaction`. With the MCP server, `inodraGrpc simulateTransaction` and `inodraGrpc executeTransaction` take the base64 BCS; the server never signs.

## GraphQL

Inodra serves the Sui GraphQL RPC 2.0 schema (`object(address, version, atCheckpoint)`, `effects { checkpoint }`), not the legacy one. Connections are ascending. For newest first, ask for `last` with `before` and reverse the page on the client. `pageInfo` is exact: a short page means there is no more data. Scalars: `SuiAddress`, `UInt53`.

```typescript
const res = await fetch("https://mainnet-api.inodra.com/v1/graphql", {
  method: "POST",
  headers: { ...headers, "content-type": "application/json" },
  body: JSON.stringify({
    query: `query ($a: SuiAddress!, $before: String) {
      transactions(filter: { affectedAddress: $a }, last: 20, before: $before) {
        nodes { digest effects { status checkpoint { sequenceNumber } } }
        pageInfo { hasPreviousPage startCursor }
      }
    }`,
    variables: { a: "0x..." },
  }),
});
```

An object as it was at a checkpoint:

```graphql
query { object(address: "0x6", atCheckpoint: 300000000) { version asMoveObject { contents { json } } } }
```

Introspection is on, so any GraphQL IDE can read the live schema.

## Credits

Every call costs credits. A standard read costs 1. gRPC and GraphQL charge per entity returned, so use field masks and small pages. Batch lookups (`BatchGetObjects`, `BatchGetTransactions`) are cheaper than loops. Read usage with `GET /v1/usage/stats` (manage scope).

## No account

An agent with a Sui wallet can pay per request with USDC or USDsui through x402. Send the request without a key, read the `402` challenge, sign the payment, and resend with `PAYMENT-SIGNATURE`. See https://docs.inodra.com/getting-started/pay-per-request.md.

## Errors

- 401: missing or invalid key
- 403 `INSUFFICIENT_SCOPE`: key lacks the scope
- 403 `NETWORK_MISMATCH`: wrong host for the key's network
- 429: rate limit, back off with jitter
- 5xx: retry with backoff; 5xx responses are not billed

Validate inputs before calling: addresses are `0x` plus 64 hex characters, digests are base58.
