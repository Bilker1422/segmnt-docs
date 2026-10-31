---
title: "Quickstart: play your first signed video"
description: Get an API key, upload a video, wait until it is ready, create a playback token on your server, and embed the player on a verified domain.
icon: "play"
---

This tutorial takes you from a new account to a video that plays on your own
website. You upload one MP4 file, create a playback token on your server, and
embed the player. The last step shows that a copied link fails on another site.

You need:

- A Google or GitHub account, to sign in to the dashboard.
- A domain or subdomain that you control, for example `courses.example.com`, and
  access to its DNS settings.
- A short MP4 file under 100 MiB.
- A terminal with `curl` and `jq` installed.

Each step has one block of commands or code, and the output you should see after
it. Replace every value that starts with `YOUR_` or ends with `...`.

## Step 1: Create a project and an API key

1. Open [app.segmnt.cloud](https://app.segmnt.cloud) and sign in with Google or
   GitHub. Your free account is created on your first sign-in.
2. Click **New project**. Enter a name, for example `Course site`.
3. Open the project. Click **API keys**, then **Create key**. The dashboard
   creates a `full` key, which can call every route of this project.
4. Copy the key. It starts with `sk_`. We show it only once.

Set the values in your terminal:

```bash
export SEGMNT_BASE="https://api.segmnt.cloud"
export SEGMNT_KEY="sk_..."     # your API key. Keep it on your server.
export PROJECT_ID="prj_..."    # the project id, shown on the project page
```

Check the key. Run this command:

```bash
curl -sS "$SEGMNT_BASE/v1/projects/$PROJECT_ID/videos" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Expected output, for a new project:

```json
{"data":[],"nextCursor":null,"total":0}
```

If you get `401`, the key is wrong. Copy it again. See
[Authentication](/authentication).

## Step 2: Verify your domain

Playback works only on domains that you verify. Add the host in the dashboard.
A free account can verify one domain.

1. Open the project. Click **Domains**, then **Add domain**.
2. Enter the exact host, for example `courses.example.com`. A subdomain counts as
   its own domain.
3. Add the TXT record that the dashboard shows to your DNS provider. The name
   starts with `_segmnt-verify.`:

   ```text
   Type:   TXT
   Name:   _segmnt-verify.courses.example.com
   Value:  segmnt-verify=YOUR_TOKEN_FROM_THE_DASHBOARD
   ```

   Copy the name and the value. Do not retype them. The name starts with an
   underscore (`_`), and in Arabic text it can be shown at the wrong end of the
   name. A copied record always verifies.

   العربية: انسخ اسم السجل وقيمته ولا تُعِد كتابتهما؛ يبدأ الاسم بشرطة سفلية `_`.

4. Click **Verify**.

DNS changes can take time to spread. If **Verify** fails, wait and click it
again.

Expected output: the domain shows the status **Verified** in the dashboard.

## Step 3: Upload a video

Run these commands in the folder that holds your MP4 file. Change `FILE` to your
file name. A file under 100 MiB has one part. Larger files need more parts. See
[Direct upload](/direct-upload).

First, create the upload:

```bash
export FILE="lesson-01.mp4"
export SIZE=$(wc -c < "$FILE")

curl -sS -X POST "$SEGMNT_BASE/v1/projects/$PROJECT_ID/uploads" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"First lesson\",\"fileName\":\"$FILE\",\"sizeBytes\":$SIZE,\"contentType\":\"video/mp4\"}" \
  > upload.json
```

Expected output: none. The file `upload.json` now holds the answer. Read the IDs
with this command:

```bash
jq -r '.id, .video.id' upload.json
```

Expected output: an upload ID that starts with `upl_`, then a video ID that
starts with `vid_`.

Next, send the file to the part URL, then complete the upload. The part URL
needs no API key:

```bash
export UPLOAD_ID=$(jq -r '.id' upload.json)
export VIDEO_ID=$(jq -r '.video.id' upload.json)
export PART_URL=$(jq -r '.parts[0].url' upload.json)

curl -sS -X PUT "$PART_URL" --upload-file "$FILE" -D part.headers -o /dev/null
export ETAG=$(grep -i '^etag:' part.headers | cut -d' ' -f2- | tr -d '"\r\n')

curl -sS -X POST "$SEGMNT_BASE/v1/projects/$PROJECT_ID/uploads/$UPLOAD_ID/complete" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"parts\":[{\"partNumber\":1,\"etag\":\"$ETAG\"}]}"
```

Expected output: the video as JSON, with `"status":"processing"`. The encode
starts now.

You can also upload in the dashboard. Open the project, click **Upload**, and
choose the file. Then open the video, copy its ID, and set `VIDEO_ID` to that ID.

## Step 4: Wait until the video is ready

Run this loop. It checks the status every 10 seconds and stops when the video is
no longer processing:

```bash
while true; do
  STATUS=$(curl -sS "$SEGMNT_BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
    -H "Authorization: Bearer $SEGMNT_KEY" | jq -r '.status')
  echo "$STATUS"
  case "$STATUS" in ready|failed|held|uploading) break ;; esac
  sleep 10
done
```

Expected output: `processing` on each line, then `ready`. The time depends on
the length of the video.

| `status` | Meaning | What to do |
| --- | --- | --- |
| `processing` | We are encoding the video. | Wait. |
| `ready` | The video can play. | Go to step 5. |
| `held` | The video is held for review and does not play yet. | Read [Moderation](/moderation). |
| `uploading` | The upload is not complete. The loop stops here. | Run the complete call in step 3 again. |
| `failed` | The encode failed. | Read the `failure` field of the video. |

In a production system, subscribe to the `video.ready` event instead of polling.
See [Webhooks](/webhooks).

## Step 5: Create a playback token on your server

Run this command on your server, not in a browser. The API key stays on the
server. The token is safe to send to a browser, because it works only for this
video and this domain.

```bash
curl -sS -X POST "$SEGMNT_BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}' \
  > token.json

export TOKEN=$(jq -r '.token' token.json)
```

Expected output: none. The file `token.json` holds the answer. It looks like
this, with the values shortened:

```json
{
  "token": "v1.eyJ...",
  "expiresAt": "2026-11-01T06:00:00.000Z",
  "embedUrl": "https://cdn.segmnt.cloud/e/vid_9xR4tL?token=v1.eyJ...",
  "thumbnailUrl": "https://cdn.segmnt.cloud/v/vid_9xR4tL/thumb.jpg?token=v1.eyJ..."
}
```

The `domain` must be a verified domain of your project. Otherwise the answer is
`400 invalid_request`.

A token lasts 4 hours by default. For a longer video, add `expiresInSeconds` to
the body. The maximum is 86400. Create a new token each time you show the video.
For cURL, Node.js and PHP examples, see
[Create playback tokens on your server](/playback-tokens).

## Step 6: Embed the player

Paste this code into the page on your verified domain. Replace
`YOUR_PLAYBACK_TOKEN` with the value of `$TOKEN` from step 5. Use the `vid_` ID
from step 3.

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

Open the page. The video appears with its thumbnail. Press play.

Expected output: the video plays on your page.

## Step 7: Check the domain lock

Ask for the playlist with your token, and with an origin that you did not verify:

```bash
curl -i "https://cdn.segmnt.cloud/v/$VIDEO_ID/master.m3u8?token=$TOKEN" \
  -H "Origin: https://someone-else.example"
```

Expected output: a `403` answer with the header `X-Segmnt-Error: domain_mismatch`.

A copied link does not play on another site. Your video is now protected by a
token and a domain lock.

Free and Starter play through our embed (our player, with our logo). Pro and
Agency can use their own player.

## If a step fails

| Response | Cause | Fix |
| --- | --- | --- |
| `401 unauthorized` | The API key is wrong or revoked. | Check the key. Create a new key if needed. |
| `403 insufficient_scope` | The key has the `playback` scope, which can create only playback tokens. | Use a `full` key for steps 1 to 4. |
| `400 invalid_request` on the token call | The domain is not verified in this project. | Finish step 2. |
| `409 video_not_ready` on the token call | The video is still processing, or it is held. | Finish step 4. |
| `403 domain_mismatch` on the player | The page is not on the verified domain. | Embed the player on the verified domain. |
| `403 plan_limit_reached` on the player | Free and Starter play only through our embed page, not from your own site. | Use the embed code from step 6. Pro and Agency can use their own player. |

For every error code, see [Errors](/errors).

## Next steps

- [Authentication](/authentication): base URL, key scopes, and how playback
  tokens differ from API keys.
- [Direct upload](/direct-upload): files larger than 100 MiB, and parallel parts.
- [Create playback tokens on your server](/playback-tokens): cURL, Node.js and
  PHP examples.
- [Webhooks](/webhooks): get a `video.ready` event instead of polling.
