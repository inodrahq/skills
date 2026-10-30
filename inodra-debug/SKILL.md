---
name: inodra-debug
description: Troubleshoot an Inodra integration for Sui. Use when a request to Inodra fails with 401, 403, 429, a gRPC status, or returns empty data, or when the user asks why their Inodra API key, scope, network or stream does not work.
---

# Debug an Inodra integration

Work through the steps in order.

## 1. Check the key

```bash
curl -s https://mainnet-api.inodra.com/v1/auth/me -H "x-api-key: $INODRA_API_KEY"
```

- 200: shows organization, project, network and tier
- 401: the key is wrong or revoked
- 403 `INSUFFICIENT_SCOPE`: the key is valid but data-only. That is normal for a data key; the network is confirmed.
- 403 `NETWORK_MISMATCH`: the key belongs to the other network. Use that network's host.

## 2. Check a data call

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
console.log(await client.ledgerService.getServiceInfo({}).response);
```

## 3. Check usage

`GET /v1/usage/stats` (manage scope). Compare consumed credits with the limit, and look at the error rate.

## Errors

**401**: missing `x-api-key`, or the key is invalid or revoked.

**403**: `INSUFFICIENT_SCOPE` means the route needs a scope the key lacks. Webhook and Warp management, usage stats, `/auth/me`, `/keys` and `/projects` need `manage`. Connecting to a Warp stream needs `streams`. `NETWORK_MISMATCH` means the wrong host.

**429**: rate limit of the plan, or a per-key cap. Back off: 1s, 2s, 4s, 8s with jitter.

**404 / NOT_FOUND**: the address, digest or object does not exist on this network. Addresses are `0x` plus 64 hex characters. Digests are base58. Check mainnet versus testnet.

**400 / INVALID_ARGUMENT**: malformed body, wrong field names in a read mask, or a field filter with the wrong Move type.

**5xx / UNAVAILABLE**: retry with backoff. 5xx responses are not billed.

**gRPC**: `UNAUTHENTICATED` means missing or wrong `x-api-key` metadata. The gRPC hosts speak real gRPC only; a JSON POST to them fails.

**Empty results**: the resource exists but has no data, the network is wrong, or a `List*` scan ended on `SCAN_LIMIT` before finding items. Resume from the watermark cursor.

**Warp**: a stream that never delivers is usually paused (`active: false`), on the wrong network, or connected with a key that lacks `streams`.

## Checklist

1. `/v1/auth/me` answers 200 or `INSUFFICIENT_SCOPE`
2. The host matches the key's network
3. The key's scopes cover the route
4. Inputs are well formed
5. A known-good query works, for example `GetObject` on `0x6` (the Clock)
6. The failure is intermittent (retry) or consistent (bug)
