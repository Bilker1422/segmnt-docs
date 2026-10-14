---
title: Add segmnt to WordPress
description: Put a segmnt video in a WordPress page with a Custom HTML block, and learn where the playback token comes from.
icon: "globe"
---

This guide puts a segmnt video inside a WordPress page. It works with the
**Gutenberg editor** and any theme. You use a **Custom HTML** block, and the
video's **playback token** comes from your server.

> The examples use the **TEST** hosts. In production the API host is
> `api.segmnt.cloud` and the embed host is `cdn.segmnt.cloud`.

## What you need

1. A **ready** segmnt video (`vid_...`).
2. A **verified** domain: the WordPress site's own domain, for example
   `blog.example.com` or `courses.example.com`. If WordPress is at
   `www.example.com`, verify `www.example.com` too, because we match the exact
   host. See [Accounts, projects, API keys and domains](/accounts-and-projects).
3. A **playback token** for that video. Your server makes it; see below.

## Step 1: Add a Custom HTML block

1. Edit the page in WordPress.
2. Click **+**, search for **Custom HTML**, and add the block.
3. Paste this, and replace `vid_9xR4tL` with your video id and
   `YOUR_PLAYBACK_TOKEN` with a real token:

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://test-cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

4. Preview the page. The video plays on the preview domain only if that domain
   is verified and the token was made for it.

## Step 2: Where the token comes from

A token **expires** (4 hours by default), so pasting one token into the block is
only good for a short test. For a real page, the token must be **fresh** on each
page load. There are two ways.

### Option A: A quick test (paste a token by hand)

Make a token once from your server with the playback-tokens call, paste it into
the block, and preview the page. This is fine to check the setup. It stops
playing when the token expires.

```bash
curl -X POST "https://test-api.segmnt.cloud/v1/projects/$PROJECT_ID/videos/vid_9xR4tL/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}'
```

Copy the `token` value from the answer. See
[Create playback tokens on your server](/playback-tokens).

### Option B: A fresh token on every page load (recommended)

WordPress runs on PHP on your server, so it can call our API and print the
iframe with a fresh token. Add a small **shortcode** to your theme or a small
plugin, then use `[segmnt video="vid_9xR4tL"]` in any page (or keep the Custom
HTML block and paste the built HTML).

Keep the API key in `wp-config.php`, **never** in a page or the block:

```php
// wp-config.php — keep this out of version control and out of any page.
define('SEGMNT_API_KEY', 'sk_...');
define('SEGMNT_PROJECT', 'prj_7kQ2mB');
define('SEGMNT_API_BASE', 'https://test-api.segmnt.cloud');
define('SEGMNT_CDN', 'https://test-cdn.segmnt.cloud');
```

```php
// functions.php (a small plugin is cleaner) — a [segmnt] shortcode.
add_shortcode('segmnt', function ($atts) {
    $atts = shortcode_atts(['video' => '', 'domain' => $_SERVER['HTTP_HOST'] ?? ''], $atts);
    $video  = sanitize_text_field($atts['video']);
    $domain = strtolower(preg_replace('/:\d+$/', '', sanitize_text_field($atts['domain'])));

    if ($video === '' || $domain === '') {
        return '';
    }

    $res = wp_remote_post(SEGMNT_API_BASE . '/v1/projects/' . SEGMNT_PROJECT . '/videos/' . $video . '/playback-tokens', [
        'timeout' => 10,
        'headers' => [
            'Authorization' => 'Bearer ' . SEGMNT_API_KEY,
            'Content-Type'  => 'application/json',
        ],
        'body' => wp_json_encode(['domain' => $domain]),
    ]);

    if (is_wp_error($res) || wp_remote_retrieve_response_code($res) !== 201) {
        return '<!-- segmnt: could not create a playback token -->';
    }

    $token = json_decode(wp_remote_retrieve_body($res), true)['token'] ?? '';
    if ($token === '') {
        return '';
    }

    $src = SEGMNT_CDN . '/e/' . rawurlencode($video) . '?token=' . rawurlencode($token);
    return '<div style="position:relative;width:100%;aspect-ratio:16 / 9;">'
        . '<iframe src="' . esc_url($src) . '" title="segmnt video" loading="lazy"'
        . ' style="position:absolute;inset:0;width:100%;height:100%;border:0;"'
        . ' allow="autoplay; fullscreen; picture-in-picture; encrypted-media"'
        . ' allowfullscreen referrerpolicy="origin"></iframe></div>';
});
```

Then in the page use the shortcode block:

```
[segmnt video="vid_9xR4tL"]
```

Notes:

- The shortcode runs in PHP on your server, so the **API key never reaches the
  browser**.
- WordPress caches pages. If your site serves cached HTML, keep the token's
  life longer than the cache (for example `expiresInSeconds` of 6 hours for a
  4-hour page cache), or refresh the cache before the token expires.
- If the page is public and long-lived, a short token is safer. Ask for a new
  token per page load, and do not cache the HTML that holds it.

## Common problems

| Problem | Cause | Fix |
| --- | --- | --- |
| The block shows code, not a video | The editor escaped the HTML, or the block is a Code block, not a Custom HTML block | Use the **Custom HTML** block, or the `[segmnt]` shortcode. |
| "This video needs a playback token" | `YOUR_PLAYBACK_TOKEN` was left in the code | Replace it with a real token, or use Option B. |
| "This playback link has expired" | The token is older than its `expiresAt` | Use Option B, or make a longer token. |
| "Origin does not match the allowed domain" | The site's domain is not the token's domain, or it is not verified | Verify the exact host (`www.` counts as its own host) and make the token for it. |

## Where to go next

- [Embed a video on any website](/embed-a-video)
- [Create playback tokens on your server](/playback-tokens)
- [How playback protection works](/playback-protection)

## Tested in TEST

Checked on TEST on 15 Oct 2026:

- The embed host serves the player: `GET /e/{videoId}` with no token → `403`
  `token_missing`; `GET /segmnt-player.js` → `200`.
- The playback-tokens route is live: with no key it answers `401 unauthorized`.
- The WordPress shortcode uses the public WordPress HTTP API
  (`wp_remote_post`), the same call as the cURL example, which is covered by
  `api/src/playback-tokens.test.ts`.
