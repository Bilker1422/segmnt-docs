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
| `video.removed` | The video is deleted and playback stops. We send it when our team removes the video, when you delete it (`data.reason` is `deleted_by_customer`), and when it is deleted because a paid plan stopped (`plan_ended`). For a legal matter we do not send this event. |

Subscribe only to the events you handle. We send an event only to the endpoints
that subscribed to it.

### Are videos held, and does a bulk import change it?

A video can be **held** for review. A held video does not play, and we send
`video.held`; when the hold ends we send `video.ready` again.

- The **free-plan review hold** is **Coming soon** and **off** today, so a new
  video is not held for it. A **paid** plan (**Starter**, **Pro** and **Agency**)
  is **never** held for it.
- A **team hold**, or a video the **automatic check** flags, can affect any plan.
  See [Moderation and held videos](/moderation).
- **A bulk import has no separate rule.** Every video in the import is one
  upload, so the same rule above applies to each one.

## Create and manage endpoints

> **Status: Available.** Create and manage endpoints with the API or in the
> dashboard (**Project settings → Webhooks**). See [Feature status](/status).

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

If you lose the signing secret, delete the endpoint and add a new one, or
replace it with `rotate-secret` without recreating the endpoint (the dashboard
has no rotate button today) - see
[Rotate, inspect and test](#rotate-inspect-and-test).

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
  have a reason to give. On `video.removed` the value is `deleted_by_customer`
  or `plan_ended`.

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
does not call `JSON.parse` until after the signature is valid. It also reads the
signing secret **once, when the server starts**, and stops the server if the
secret is missing. Without the secret, every delivery fails, so it is better to
stop at start than to fail every delivery in silence.

```js
import express from "express";
import { createHmac, timingSafeEqual } from "node:crypto";

// Read the signing secret once, when the server starts. If it is missing, stop
// now: without it, every delivery fails.
const SEGMNT_WEBHOOK_SECRET = process.env.SEGMNT_WEBHOOK_SECRET;
if (!SEGMNT_WEBHOOK_SECRET) {
  console.error("SEGMNT_WEBHOOK_SECRET is not set. Set it to your endpoint signing secret (whsec_...).");
  process.exit(1);
}

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
    SEGMNT_WEBHOOK_SECRET,
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

## Rotate, inspect and test

> **Status: Available.** All three calls below are live. Add/list/delete, the
> delivery log and send test event are also in the dashboard; rotation is
> **API only** today. See [Feature status](/status).

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

### In the dashboard

The dashboard covers most of these calls:

- **Project settings → Webhooks** adds, lists and removes endpoints, and shows
  the signing secret **once** when you add one.
- The **delivery log** shows each send attempt with its status and the last HTTP
  status your endpoint returned.
- **Send test event** queues one signed `webhook.test` to the endpoint.

Rotating the signing secret is **API only** today: there is no rotate button in
the dashboard yet.

## Troubleshooting

### Every delivery fails

If every delivery in the delivery log is `failed` (or keeps retrying), and the
last HTTP status from your endpoint is `401` or `500`, your server most likely
does not have the right signing secret.

1. Check that `SEGMNT_WEBHOOK_SECRET` (or the name you use) is set **in this
   environment** - staging and production each need their own value. A secret
   set on your laptop is not set on your server.
2. Check that the value is the **whole** secret, including the `whsec_` prefix,
   with no extra spaces or quotes.
3. Check that it is the secret of **this endpoint**. Each endpoint has its own
   secret, and a rotated secret replaces the old one at once.
4. Check that your server reads the **raw body** before anything parses it. If
   a JSON parser runs first, the signature never matches, even with the right
   secret. See [Verify the signature](#verify-the-signature).

Where to find the secret:

- In the dashboard, **Project settings → Webhooks** shows the `whsec_...` value
  **once**, when you add the endpoint.
- In the API, it is the `signingSecret` field in the answer to
  [add an endpoint](#create-and-manage-endpoints) or to
  [`rotate-secret`](#rotate-the-signing-secret).

We cannot show a secret again. If you no longer have it, call `rotate-secret`
to get a new one, put it on your server, and restart the server. Then use
**Send test event** to check that the delivery shows as `delivered`.

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
