---
title: Webhooks
description: Get a signed JSON message on your own HTTPS endpoint when a video changes. Create an endpoint, read the payload, verify the signature, and handle retries.
icon: "webhook"
---

A **webhook** is a small JSON message we `POST` to your server when something
happens to a video — for example, when encoding finishes and the video can play.
Use a webhook instead of polling the video route again and again.

> **Status: Available.** Video events are live. See [Feature status](/status).

This page shows the events we send, the payload, how to check the
`Segmnt-Signature` header, and what your receiver must do.

## Events we send

| Event | When we send it |
| --- | --- |
| `video.ready` | All qualities are ready and the video can play. |
| `video.failed` | The video failed, in encoding or in the complete call. `data.video.failure` says why. |
| `video.held` | Our team held the video and it does not play. When the hold ends, we send `video.ready` again. `data.reason` says why, when we have one. For a legal matter we do not send this event. |
| `video.removed` | Our team removed the video. It is deleted and playback stops. `data.reason` says why, when we have one. For a legal matter we do not send this event. |

Subscribe only to the events you handle. We send an event only to the endpoints
that subscribed to it.

## Create and manage endpoints

> **Status: Available.** Today you create endpoints with the API. The dashboard
> screen for webhooks is **Coming soon**. See [Feature status](/status).

| Method | Path | What it does |
| --- | --- | --- |
| `POST` | `/v1/projects/{projectId}/webhook-endpoints` | Add an endpoint. |
| `GET` | `/v1/projects/{projectId}/webhook-endpoints` | List the endpoints. The signing secret is **not** returned here. |
| `DELETE` | `/v1/projects/{projectId}/webhook-endpoints/{endpointId}` | Remove an endpoint. The answer is `204`. |

Add an endpoint:

```bash
curl -X POST "$BASE/v1/projects/$PROJECT/webhook-endpoints" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://courses.example.com/hooks/segmnt","events":["video.ready","video.failed"]}'
```

The answer is `201`:

```json
{
  "id": "whe_5mJ8kV",
  "url": "https://courses.example.com/hooks/segmnt",
  "events": ["video.ready", "video.failed"],
  "createdAt": "2026-10-12T09:00:00Z",
  "signingSecret": "whsec_2f9c..."
}
```

Rules:

| Item | Value |
| --- | --- |
| Endpoints per project | Up to 5. A sixth one gets `403` with `code: endpoint_limit_reached`. |
| URL | `https` only, and a public host. Private, loopback and link-local hosts are refused with `400`. |
| URL check | Repeated on every send. We do not follow redirects. |
| Events | A non-empty list, no repeats, from the table above. An unknown event gets `400`. |
| `signingSecret` | Returned **only** in this answer. Store it like an API key, and never log it. |

If you lose the signing secret, delete the endpoint and add a new one. The old
secret keeps working until you delete the endpoint. Secret rotation without
recreating the endpoint is **Coming soon**.

## The payload

Every event is one JSON object:

```json
{
  "id": "evt_1qA7zP",
  "type": "video.ready",
  "createdAt": "2026-10-12T09:12:00Z",
  "projectId": "prj_2bN6rD",
  "data": {
    "video": {
      "id": "vid_9xR4tL",
      "projectId": "prj_2bN6rD",
      "title": "Lesson 1",
      "status": "ready",
      "durationSeconds": 612.5,
      "qualities": ["360p", "720p", "1080p"],
      "sizeBytes": 734003200,
      "failure": null,
      "createdAt": "2026-10-12T09:00:00Z",
      "updatedAt": "2026-10-12T09:12:00Z",
      "readyAt": "2026-10-12T09:12:00Z",
      "accessMode": "protected"
    }
  }
}
```

- `id` — the event id. It is the **same on every retry** of the same event. Use
  it to skip duplicates.
- `type` — the event name, from the table above.
- `createdAt` — when we created the event. RFC 3339 in UTC.
- `projectId` — the project the video belongs to.
- `data.video` — the same object as
  `GET /v1/projects/{projectId}/videos/{videoId}`. Use `data.video.id` to find
  the video in your system.
- `data.reason` — only on `video.held` and `video.removed`, and only when we
  have a reason to give.

## Verify the signature

Every delivery carries a `Segmnt-Signature` header. It is the only way to know
the request really came from us. Do not trust a webhook without checking it.

```
Segmnt-Signature: t=1791648000,v1=5f2b...   (v1 is 64 hex characters)
```

- `t` is the Unix time in **seconds** of this delivery attempt.
- `v1` is the HMAC-SHA256 of the string `<t>.<raw body>`, in hex.
- The **key** is your signing secret, used **exactly as shown**. The `whsec_`
  prefix is part of the key, and the key is read as UTF-8. Do not decode it and
  do not strip the prefix.

To check a delivery:

1. Split the header on `t=` and `v1=`.
2. Refuse the request if `t` is more than 5 minutes old, or in the future.
3. Compute HMAC-SHA256 over `<t>.<raw body>`. Use the **raw body bytes**, not
   the parsed JSON.
4. Compare your result with `v1` in constant time. A simple `==` is not safe.

The Node.js example below does all four steps. It reads the raw body first and
does not call `JSON.parse` until after the signature is valid.

```js
import express from "express";
import { createHmac, timingSafeEqual } from "node:crypto";

const app = express();

/**
 * Check a segmnt webhook.
 * - secret: the endpoint signing secret (whsec_...), used as-is
 * - signatureHeader: the Segmnt-Signature header value
 * - rawBody: the exact request body bytes
 * Returns true only when the signature is valid and fresh.
 */
function verifySegmntSignature(secret, signatureHeader, rawBody, toleranceSeconds = 300) {
  const m = /^t=(\d+),v1=([0-9a-f]{64})$/.exec(signatureHeader ?? "");
  if (!m) return false;
  const t = Number(m[1]);
  if (Math.abs(Date.now() / 1000 - t) > toleranceSeconds) return false;
  const expected = createHmac("sha256", secret).update(`${t}.`).update(rawBody).digest();
  const got = Buffer.from(m[2], "hex");
  return expected.length === got.length && timingSafeEqual(expected, got);
}

// express.raw keeps the body as a Buffer, so the bytes we verify are the bytes we got.
app.post("/hooks/segmnt", express.raw({ type: "application/json" }), (req, res) => {
  const ok = verifySegmntSignature(
    process.env.SEGMNT_WEBHOOK_SECRET,
    req.get("segmnt-signature"),
    req.body,
  );
  if (!ok) return res.sendStatus(401);

  // Safe to parse now: the signature was checked against the raw bytes above.
  const event = JSON.parse(req.body.toString("utf8"));
  console.log(event.id, event.type, event.data.video.id);

  res.sendStatus(200); // answer fast, then do the real work in a queue
});
```

## How to handle deliveries

`video.ready`, `video.failed`, `video.held` and `video.removed` are delivered in
the same way.

- **Answer fast.** Return any `2xx` status within 10 seconds. Do the real work
  after you answer, for example in a queue. A slow answer looks like a failure.
- **At least once.** The same event can arrive more than once. Every delivery is
  "at least once": `id`, `type`, `createdAt` and the body are the same on each
  try, but `t` and the signature are new. Dedupe on `id` and ignore an event you
  have already processed.
- **Not in order.** Events can arrive out of order. Ignore an event whose
  `data.video.updatedAt` is older than the one you stored, or read the video
  again with `GET /v1/projects/{projectId}/videos/{videoId}` for the current
  state.
- **Retries.** Any answer that is not `2xx`, or no answer, means a retry. We
  retry with a growing wait, up to about 8 tries, then mark that delivery
  failed.

## Rotate, inspect and test (Coming soon)

> **Status: Coming soon.** The three calls below are planned. They are **not**
> live today, so do not build on them yet. We add them to the API reference when
> they ship.

### Rotate the signing secret

```
POST /v1/projects/{projectId}/webhook-endpoints/{endpointId}/rotate-secret
```

Returns a new `whsec_...` **once**. The endpoint, its URL and its event list stay
the same. The old secret stops working as soon as the call returns, so update
your server with the new secret before you call this route.

### Read the delivery log

```
GET /v1/projects/{projectId}/webhook-endpoints/{endpointId}/deliveries?limit=&cursor=
```

Newest first. Each row has the event type, the time we created it, the status
(`pending`, `delivered` or `failed`), the number of attempts, and the last HTTP
status your endpoint returned. `limit` sets the page size (1 to 100, default
20). For the next page, pass the `nextCursor` value back as `cursor`.

### Send a test event

```
POST /v1/projects/{projectId}/webhook-endpoints/{endpointId}/test
```

Queues one signed `webhook.test` event to that endpoint. It uses the same
signature header as every other event, and the result appears in the delivery
log. The body we `POST` to your endpoint is:

```json
{
  "id": "evt_1qA7zP",
  "type": "webhook.test",
  "createdAt": "2026-10-17T12:00:00.000Z",
  "projectId": "prj_C4LqLAGX",
  "data": { "test": true }
}
```

### Also coming soon

- A dashboard screen to add, list and remove endpoints, and to show the signing
  secret once.
- A "send test event" button in the dashboard.

## Test your endpoint today

To test your endpoint today, register your own HTTPS receiver and upload a short
video. Watch for the `video.ready` POST and check the signature with the code
above. To test `video.failed`, upload a file that is not a video (for example a
`.txt` file renamed to `.mp4`).

## Where to go next

- [API overview](/api-overview) — the base URL, authentication, and every
  endpoint. The webhook endpoints are in the "Webhooks, usage and billing"
  list, and under **Webhooks** in the API reference.
- [Direct upload](/direct-upload) — upload a video, then wait for the event.
- [Feature status](/status) — which features are Available, Beta or Coming soon.
