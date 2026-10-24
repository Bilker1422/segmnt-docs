---
title: Use segmnt in a React app
description: Show a segmnt video in a React or Next.js app with the iframe embed or the player script, and keep the API key on your server.
icon: "react"
---

Put a segmnt video in a React app: a custom course platform, a dashboard or a
single-page app. React changes nothing about playback: your server still makes
the [playback token](/playback-tokens), and the video still plays only on your
[verified domain](/accounts-and-projects). What changes is **where the embed
code lives**: inside a component.

There are two ways to add the video:

| Path | Best for | What you do |
| --- | --- | --- |
| **iframe in JSX** | Most apps. No extra script. | Render the same `<iframe>` as the [embed guide](/embed-a-video), with the token from your backend. |
| **Player script** | Many videos on one page. | Load `segmnt-player.js` once, then call `SegmntPlayer.embed()` from a component. |

## Before you start

Check four things first:

1. Your video is **ready** (`vid_...`). See [Manage videos](/manage-videos).
2. The host that serves your React app is a **verified domain** of the project.
   `localhost` cannot be verified. Use a staging host you control, such as
   `staging.example.com`, while you develop. See
   [Integration guide, section 3](/integration-guide).
3. Your backend has a route that returns a playback token for a video. See
   [Create playback tokens on your server](/playback-tokens).
4. **The API key stays on your server.** Never put it in React code, a bundle
   or an environment variable that reaches the browser. The React app asks your
   backend for a token; it never talks to the segmnt API itself.

The token route answers with `token`, `expiresAt`, `embedUrl` and `playlistUrl`.
The embed paths below use `token`; the white-label direct-mount path uses
`playlistUrl`.

## Path 1: iframe in JSX

Your backend mints a fresh token per page load. Your component fetches it and
renders the iframe:

```jsx
import { useEffect, useState } from "react";

// Call YOUR backend, not the segmnt API. Your backend holds the API key.
async function fetchToken(videoId) {
  const res = await fetch(`/api/segmnt-token?videoId=${encodeURIComponent(videoId)}`);
  if (!res.ok) throw new Error(`token request failed: ${res.status}`);
  const data = await res.json();
  return data.token; // your backend returns the `token` from the playback-tokens route
}

export function SegmntVideo({ videoId, title = "segmnt video" }) {
  const [token, setToken] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    setToken(null);
    fetchToken(videoId)
      .then((t) => { if (!cancelled) setToken(t); })
      .catch((e) => { if (!cancelled) setError(e); });
    return () => { cancelled = true; };
  }, [videoId]);

  if (error) return <p>The video is not available right now.</p>;
  if (!token) return null;

  return (
    <div style={{ position: "relative", width: "100%", aspectRatio: "16 / 9" }}>
      <iframe
        src={`https://cdn.segmnt.cloud/e/${encodeURIComponent(videoId)}?token=${encodeURIComponent(token)}`}
        title={title}
        loading="lazy"
        style={{ position: "absolute", inset: 0, width: "100%", height: "100%", border: 0 }}
        allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
        allowFullScreen
        referrerPolicy="origin"
      />
    </div>
  );
}
```

Rules for this component:

- Fetch a token **per page load**. Do not store the token in `localStorage`,
  a context or a global store across sessions: it expires (4 hours by
  default), and a stored token stops working.
- A long-lived single-page app can outlive a token. If the player shows
  "expired", ask your backend for a new token and re-render. The [error
  codes](/embed-a-video#if-the-video-does-not-play) tell you which case you
  are in.
- Keep the wrapper `div` and the `aspect-ratio` style, so the player resizes
  with its container.

## Path 2: the player script

With many videos on one page, load the player script once and let a component
mount the embed. The script goes in your page `<head>` — in Create React App
or Vite, add it to the `index.html`:

```html
<script src="https://cdn.segmnt.cloud/segmnt-player.js" defer></script>
```

Then mount the video in a `useEffect`, and remove it on cleanup:

```jsx
import { useEffect, useRef, useState } from "react";
import { fetchToken } from "./fetchToken"; // the same backend call as Path 1

export function SegmntPlayerEmbed({ videoId, locale = "en" }) {
  const holder = useRef(null);
  const [token, setToken] = useState(null);

  useEffect(() => {
    let cancelled = false;
    fetchToken(videoId).then((t) => { if (!cancelled) setToken(t); });
    return () => { cancelled = true; };
  }, [videoId]);

  useEffect(() => {
    if (!holder.current || !token) return;
    const iframe = window.SegmntPlayer.embed(holder.current, {
      videoId,
      token,
      locale,
    });
    return () => {
      // React may mount and unmount twice in development (StrictMode).
      // Removing the iframe on cleanup keeps that from doubling the player.
      iframe.remove();
    };
  }, [videoId, token, locale]);

  return <div ref={holder} style={{ width: "100%" }} />;
}
```

`SegmntPlayer.embed(target, options)` builds the same responsive iframe as
Path 1 inside `target`. It returns the `<iframe>` element, so the cleanup can
remove it. The script defines `window.SegmntPlayer` after it loads; because the
token arrives from your backend after render, the effect runs after the script
is ready in practice. If your app renders before the script loads, gate the
effect on the script (for example, set state in the script's `load` event).

Options for `embed()`: `videoId`, `token`, `baseUrl`, `locale`, `title` and
`aspectRatio`. The locale is `"ar"` (default, RTL) or `"en"`.

## Next.js and other server-rendered setups

The embed is browser-only, so:

- Put the component in a **client component** (`"use client"` in the App
  Router) or render it only after mount. The iframe and `SegmntPlayer` exist
  only in the browser.
- Put the `<script src=...>` tag for Path 2 in your layout's `<head>` (App
  Router: the `head`/metadata of the root layout; Pages Router: `next/head`).
  Keep `defer`.
- Fetch the token from a **route handler or API route** on your server, which
  holds the API key in a server-side environment variable. Never put the key
  in `NEXT_PUBLIC_*` or any other variable that reaches the browser.

## Common problems

| Problem | Cause | Fix |
| --- | --- | --- |
| "This video needs a playback token" | The token fetch failed or returned nothing | Check your backend route. The answer must carry the `token` from the playback-tokens route. |
| The video plays, then stops later | The single-page app outlived the 4-hour token | Ask your backend for a new token and re-render when the player reports it. |
| The player appears twice | The effect ran on both StrictMode mounts without cleanup | Remove the iframe in the effect cleanup, as in Path 2. |
| `SegmntPlayer is not defined` | The component ran before the script loaded | Keep the script tag in the `<head>` with `defer`, and mount the embed after the token arrives. |
| "Origin does not match the allowed domain" | The app host is not the token's domain, or not verified | Verify the exact host (`localhost` cannot be verified) and make the token for it. See [Integration guide, section 3](/integration-guide). |

## Where to go next

- [Embed a video on any website](/embed-a-video): the plain-HTML version of both embeds.
- [Create playback tokens on your server](/playback-tokens)
- [How playback protection works](/playback-protection)

## Tested against the current release

Checked on 24 Oct 2026:

- The JSX iframe path renders the same markup as the dashboard's iframe embed:
  same `/e/{videoId}` URL shape, `allow` list, `referrerpolicy` and responsive
  wrapper.
- Both components from this page were run in a DOM test harness (React 18,
  StrictMode on) against the real player bundle: `SegmntPlayer.embed(target, {
  videoId, token, locale })` puts one iframe in the target, the cleanup removes
  it on unmount, and a StrictMode double-mount still leaves one player.
- `GET https://cdn.segmnt.cloud/segmnt-player.js` → `200`.
- `GET /e/{videoId}` with no token → `403` `token_missing` (checked on the
  TEST deploy), so an app with a broken token fetch shows the player's error
  message, not a blank box.
- The token response fields (`token`, `expiresAt`, `embedUrl`, `playlistUrl`)
  match [Playback tokens](/playback-tokens) and the OpenAPI spec.
