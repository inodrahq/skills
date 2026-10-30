---
name: inodra-warp
description: Set up Inodra Warp real-time streams for Sui over SSE or WebSocket. Use when an app must receive Move events, address activity, coin balance changes or object changes live, or when managing Warp stream subscriptions.
---

# Inodra Warp streams

Warp pushes Sui activity to a client over SSE or WebSocket. The client connects to Inodra; nothing is sent to your server (that is a webhook).

Scopes: creating, updating and deleting streams needs `manage`. Connecting to a stream needs `streams`.

## Stream types

1. **Event**: Move events of a type, optional `sender` and field filters
2. **Address**: all activity that touches an address
3. **Coin**: balance changes of one coin type for an address
4. **Object**: changes to one object

## Create a stream

```typescript
const base = "https://mainnet-api.inodra.com/v1";
const headers = { "x-api-key": process.env.INODRA_API_KEY!, "content-type": "application/json" };

const res = await fetch(`${base}/warp/events`, {
  method: "POST",
  headers,
  body: JSON.stringify({
    eventType: "0xpackage::module::EventType",
    sender: "0x...", // optional
    eventFieldFilters: [{ field: "amount", moveType: "u64", operator: "gt", value: "1000000000" }], // optional
  }),
});
const { data } = await res.json();
const streamId = data.id;
```

Other create routes: `POST /warp/addresses` (`address`), `POST /warp/coins` (`address`, `coinType`), `POST /warp/objects` (`objectId`).

## Connect

SSE: `GET /v1/warp/{id}/stream`. Send the key as `x-api-key`, as `Authorization: Bearer`, or as the `api_key` query parameter. The browser `EventSource` cannot set headers, so it uses the query parameter. Keep the key out of logs.

```typescript
const es = new EventSource(`${base}/warp/${streamId}/stream?api_key=${apiKey}`);
es.onmessage = (e) => handle(JSON.parse(e.data));
```

`EventSource` sends `Last-Event-ID` on reconnect. Buffered events after that id are replayed before live events resume.

WebSocket: `wss://mainnet-api.inodra.com/v1/warp/{id}/ws?api_key=...`. The socket supports an ACK protocol and reconnect modes; see https://docs.inodra.com/warp/setup.md.

## Manage

```
GET    /v1/warp           list
GET    /v1/warp/{id}      details
PUT    /v1/warp/{id}      { active: false } pause
DELETE /v1/warp/{id}      delete
```

## Rules

- Handle reconnects. Networks drop.
- Pause unused streams with `active: false` instead of deleting them.
- Field filters use the same syntax as webhooks: operators `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`.
- A connected stream costs credits over time and per delivered event. Watch usage.
- The number of active streams is capped by the plan.
