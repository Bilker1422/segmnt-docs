---
title: Create playback tokens on your server
description: Create a signed playback token with cURL, Node.js and PHP. Keep the API key on your server and never in a web page.
icon: "key"
---

A **playback token** is a short, signed pass that lets one video play on one
domain for a short time. Your server creates it, and the CDN checks it on every
request.

> **Never put your API key in a web page.** The API key creates tokens, so
> anyone with the key can play your videos anywhere. Tokens are safe in a page.
> Keys are not.

## Public videos need no token

> **Status: Coming soon.** The per-video **public** access mode is not released
> yet, so every video needs a token today. This section describes how it will
> work. See [Feature status](/status).

When the mode ships, a video whose access mode is **Public** will play with no
token, so you will **not** call this endpoint for it. Build the embed URL with no
`?token=...`: `https://cdn.segmnt.cloud/e/{videoId}`. See
[Protected and public videos](/playback-protection#protected-and-public-videos).
Protected is the default today; the rest of this page is about protected videos.

## The call

`POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens`

- Auth: your project API key, `Authorization: Bearer sk_...`.
- Call it **from your server**, each time you show the video (or per page load).
- The video must be `ready` and not held for review.
- The `domain` must be a **verified** domain of the project.

Body:

| Field | Required | Notes |
| --- | --- | --- |
| `domain` | Yes | A verified domain of the project. A host name only: `courses.example.com`. No `https://`, no path. |
| `viewerIp` | Only with IP binding | Coming soon. The viewer's IP address. Will be required when the project has IP binding on; ignored otherwise. Send the viewer's IP, not your server's. |
| `expiresInSeconds` | No | 60 to 86400. Default `14400` (4 hours). Make it longer than the video so playback does not stop. |

Answer (`201`):

```json
{
  "token": "v1....",
  "expiresAt": "2026-10-15T17:00:00.000Z",
  "embedUrl": "https://cdn.segmnt.cloud/e/vid_9xR4tL?token=v1....",
  "thumbnailUrl": "https://cdn.segmnt.cloud/v/vid_9xR4tL/thumb.jpg?token=v1...."
}
```

**Free and Starter play through our embed (our player, with our logo). Pro and
Agency can use their own player.**

On a plan with **white-label** (Pro and Agency), the answer also has
`playlistUrl` for your own hls.js player:

```json
{ "playlistUrl": "https://cdn.segmnt.cloud/v/vid_9xR4tL/master.m3u8?token=v1...." }
```

Put `embedUrl` in an iframe, or `playlistUrl` in your own player. See
[Embed a video on any website](/embed-a-video).

## Bind a token to the viewer's network (optional)

> **Status: Coming soon.** See [Feature status](/status).

A project will be able to turn on **network binding** (`ipBinding`, off by
default). Then each new token is bound to the **viewer's** network, so a link
that is shared to another network stops working. See
[Viewer network binding](/playback-protection#part-3-viewer-network-binding-optional).

You will pass the viewer's IP address as `viewerIp`. Our API sees **your
server's** IP, not the viewer's, so your server is the one that knows the
viewer's address: read it from the request socket or a trusted proxy header, and
put it here.

Send the viewer's own address, never your server's:

```js
// Node.js
createPlaybackToken(videoId, domain, undefined, viewerIp);
```

```php
// PHP / Laravel
segmntPlaybackToken($videoId, $domain, viewerIp: $viewerIp);
```

On a bound project a token will never be minted without it: a missing or invalid
`viewerIp` will be `400 invalid_request`. When `ipBinding` is off, `viewerIp` is
ignored, so sending it is always safe. The setting applies to tokens you create
after you change it; a token created earlier keeps working (or stays bound) until
it expires. The CDN keeps only the viewer's **network prefix** (IPv4 `/24`,
IPv6 `/64`) and refuses another network with `403 ip_mismatch`.

## cURL

```bash
export SEGMNT_KEY="sk_..."      # keep this on your server
export PROJECT_ID="prj_7kQ2mB"
export VIDEO_ID="vid_9xR4tL"

curl -X POST "https://api.segmnt.cloud/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}'
```

## Node.js (18+)

```js
const BASE = "https://api.segmnt.cloud";
const { SEGMNT_KEY, PROJECT_ID } = process.env;

export async function createPlaybackToken(videoId, domain, expiresInSeconds, viewerIp) {
  const body = { domain };
  if (expiresInSeconds) body.expiresInSeconds = expiresInSeconds;
  if (viewerIp) body.viewerIp = viewerIp; // needed only when the project has ipBinding on
  const res = await fetch(
    `${BASE}/v1/projects/${PROJECT_ID}/videos/${videoId}/playback-tokens`,
    {
      method: "POST",
      headers: { Authorization: `Bearer ${SEGMNT_KEY}`, "Content-Type": "application/json" },
      body: JSON.stringify(body),
    },
  );
  if (!res.ok) throw new Error(`playback token failed: ${res.status} ${await res.text()}`);
  const { token, expiresAt, embedUrl, playlistUrl } = await res.json();
  return { token, expiresAt, embedUrl, playlistUrl };
}
```

## PHP / Laravel

```php
use Illuminate\Support\Facades\Http;

function segmntPlaybackToken(
    string $videoId,
    string $domain,
    ?int $expiresInSeconds = null,
    ?string $viewerIp = null,
): array {
    $api = 'https://api.segmnt.cloud/v1/projects/' . config('services.segmnt.project');

    $body = ['domain' => $domain];
    if ($expiresInSeconds !== null) {
        $body['expiresInSeconds'] = $expiresInSeconds;
    }
    if ($viewerIp !== null) {
        $body['viewerIp'] = $viewerIp; // the viewer's IP; needed only when ipBinding is on
    }

    return Http::withToken(config('services.segmnt.key'))
        ->acceptJson()
        ->post("$api/videos/$videoId/playback-tokens", $body)
        ->throw()
        ->json(); // token, expiresAt, embedUrl, thumbnailUrl [, playlistUrl]
}
```

Keep `services.segmnt.key` in your server environment (`.env`), never in a Blade
template or a page.

## Put the token in the page

Build the iframe on your server and send only the finished HTML:

```php
$t = segmntPlaybackToken($videoId, 'courses.example.com');
$src = 'https://cdn.segmnt.cloud/e/' . $videoId . '?token=' . urlencode($t['token']);
?><iframe src="<?= htmlspecialchars($src) ?>" title="segmnt video"
  style="width:100%;aspect-ratio:16/9;border:0" allowfullscreen></iframe>
```

The page sees a token, never the key.

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | `domain` is missing or not a host name, or `expiresInSeconds` is out of range | Fix the body. |
| 400 | `invalid_request` | `domain` is not a **verified** domain of the project | Verify the domain first. |
| 400 | `invalid_request` | Coming soon: IP binding is on and `viewerIp` is missing or not an IP address | Send the viewer's IP address. |
| 401 | `unauthorized` | Missing, wrong or revoked key | Check the key. |
| 404 | `not_found` | Unknown video, or a key from another project | Check the ids. |
| 409 | `video_not_ready` | The video is not `ready`, or it is held for review | Wait, or ask support. |
| 403 | `plan_limit_reached` | The account is over its watched-minutes limit for the month | Free, Starter and Pilot stop here. Upgrade to keep playing. |

## Tested against the current release

Checked on 15 Oct 2026:

- `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens` with no key →
  `401 unauthorized` (the route is live).
- The route, the verified-domain check, the token TTL and the held-video `409`
  are covered by our API tests.

## Where to go next

- [Embed a video on any website](/embed-a-video)
- [Add segmnt to WordPress](/wordpress)
- [How playback protection works](/playback-protection)
- [Accounts, projects, API keys and domains](/accounts-and-projects) — verify
  the domain and create the key.
