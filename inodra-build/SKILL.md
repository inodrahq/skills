---
name: inodra-build
description: Build Sui applications on Inodra. Use when writing code that reads Sui data, picks a gateway (gRPC, GraphQL, JSON-RPC), authenticates with an Inodra API key, or needs to understand credits, scopes and networks.
---

# Build with Inodra

You help the user integrate a Sui application with Inodra. Follow these rules.

## Pick the gateway

- **gRPC** (default): the Sui `sui.rpc.v2` services over real gRPC with TLS. Typed clients, field masks, full history for the `List*` calls. Use it for reads, writes and streams.
- **GraphQL**: the official Sui GraphQL schema with full history. Use it when one query must join several objects or when the app already uses GraphQL.
- **JSON-RPC**: a translation layer over gRPC for legacy code. Do not use it in new code.
- Do not build on the REST convenience endpoints. They are not a stable surface.

## Hosts

| Network | gRPC                         | HTTP (GraphQL, JSON-RPC, Warp, webhooks) |
| ------- | ---------------------------- | ---------------------------------------- |
| mainnet | `mainnet-grpc.inodra.com:443` | `https://mainnet-api.inodra.com/v1`      |
| testnet | `testnet-grpc.inodra.com:443` | `https://testnet-api.inodra.com/v1`      |

A key belongs to one project, and a project belongs to one network. A key sent to the other network's host gets `403 NETWORK_MISMATCH`.

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

```typescript
const page = await client.core.listTransactions({
  filter: { affectedAddress: "0x..." },
  limit: 50,
});
```

## GraphQL

```typescript
const res = await fetch("https://mainnet-api.inodra.com/v1/graphql", {
  method: "POST",
  headers: { ...headers, "content-type": "application/json" },
  body: JSON.stringify({
    query: `query ($a: SuiAddress!) {
      transactions(filter: { affectedAddress: $a }, last: 20) {
        nodes { digest effects { status } }
        pageInfo { hasPreviousPage startCursor }
      }
    }`,
    variables: { a: "0x..." },
  }),
});
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
