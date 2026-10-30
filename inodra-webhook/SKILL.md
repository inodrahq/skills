---
name: inodra-webhook
description: Create and manage Inodra webhooks for Sui. Use when an app must be notified by HTTP POST about Move events, address activity, coin balance changes or object changes, including field filters, payload handling and retries.
---

# Inodra webhooks

A webhook POSTs a JSON payload to your HTTPS endpoint when something happens on Sui. Creating and changing webhooks needs a key with the `manage` scope.

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
const { data } = await res.json(); // data.id, data.webhookSecret
```

Other create routes: `POST /webhooks/addresses` (`address`, `webhookUrl`), `POST /webhooks/coins` (`address`, `coinType`, `webhookUrl`), `POST /webhooks/objects` (`objectId`, `webhookUrl`).

Pick event types from the catalog: `GET /v1/catalog/protocols` and `GET /v1/catalog/events?protocol=...&q=...` (data scope).

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

- Return 2xx within 10 seconds. Queue the payload and process it later.
- Deliveries can repeat. Make the handler idempotent.
- Failed deliveries are retried with exponential backoff.
- Verify the payload with the webhook secret from the create response.
- Only public HTTPS endpoints are accepted.

The number of active webhooks is capped by the plan.
