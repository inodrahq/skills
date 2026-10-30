---
name: inodra-webhook
description: Create and manage Inodra webhooks for Sui. Use when an app must be notified by HTTP POST about Move events, address activity, coin balance changes or object changes, including field filters, payload handling and retries.
---

# Inodra webhooks

A webhook POSTs a JSON payload to your HTTPS endpoint when something happens on Sui. Creating and changing webhooks needs a key with the `manage` scope. With the Inodra MCP server installed, the `inodraWebhook` and `inodraManagement` tools do the create, list and catalog steps below.

## Types

1. **Event**: Move events of a type, optional `sender` and field filters
2. **Address**: all activity that touches an address
3. **Coin**: balance changes of one coin type for an address
4. **Object**: changes to one object

## Create

```typescript
const base = "https://mainnet-api.inodra.com/v1";
const headers = { "x-api-key": process.env.INODRA_API_KEY!, "content-type": "application/json" };

const res = await fetch(`${base}/webhooks/events`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    eventType: "0xpackage::module::EventType",
    webhookUrl: "https://your-app.com/webhook",
    sender: "0x...", // optional
    eventFieldFilters: [{ field: "amount", moveType: "u64", operator: "gt", value: "1000000000" }],
  }),
});
const { data } = await res.json(); // data.id, data.webhookSecret, data.active
```

Responses are wrapped in `{ data }`. The create, get and list responses all include `webhookSecret`. Never log them.

Other create routes: `POST /webhooks/addresses` (`address`, `webhookUrl`), `POST /webhooks/coins` (`address`, `coinType`, `webhookUrl`), `POST /webhooks/objects` (`objectId`, `webhookUrl`).

Pick event types from the catalog (data scope): `GET /v1/catalog/protocols`, then `GET /v1/catalog/events?protocol=cetus&category=swap`. Each entry has `category` and `types[]` with `eventType`, `primary` and `verified`; take the `primary: true` type of the category you want. `q=` also matches descriptions, so `q=swap` can return fee events.

## Field filters

- Operators: `eq`, `neq`; `gt`, `gte`, `lt`, `lte` on numeric types; `contains` on string types
- Numeric Move types: `u8`, `u16`, `u32`, `u64`, `u128`, `u256`
- String-like: `address`, `vector<u8>`, `0x1::string::String`, `0x1::ascii::String`
- `bool` with `eq` or `neq`
- At most 10 filters per webhook

## Manage

```
GET    /v1/webhooks           list
GET    /v1/webhooks/{id}      details
PUT    /v1/webhooks/{id}      { webhookUrl?, active? }
DELETE /v1/webhooks/{id}      delete
```

## Receive

The body is `{ payloadVersion: 1, payload: { id, activityType, txDigest, eventSequence, checkpoint, timestamp, type, sender, data } }`, where `data` holds the decoded event fields.

- Return 2xx within 10 seconds. Queue the payload and process it later.
- Deliveries can repeat. Make the handler idempotent on `payload.id`.
- Failed deliveries are retried with exponential backoff.
- Only public HTTPS endpoints are accepted. Private and internal hosts are refused.

### Verify the signature

Headers: `X-Inodra-Signature: t=<unix seconds>,v1=<hex>` and `X-Inodra-Timestamp`. The signature is `HMAC-SHA256(webhookSecret, t + "." + rawBody)`. Compare with a constant-time check and reject timestamps older than 5 minutes.

```typescript
import { createHmac, timingSafeEqual } from "node:crypto";
const [t, v1] = sig.split(",").map((p) => p.split("=")[1]);
const expected = createHmac("sha256", secret).update(`${t}.${rawBody}`).digest("hex");
const ok = timingSafeEqual(Buffer.from(expected), Buffer.from(v1));
```

## Costs and limits

- 2 credits for the first delivery attempt, 1 per retry, at most 11 per event. Attempts are charged when sent.
- Delivery history and manual replay are in the dashboard. There is no API route for delivery logs.
- The number of active webhooks is capped by the plan.
