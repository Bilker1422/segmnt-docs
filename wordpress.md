---
title: Add segmnt to WordPress
description: Put a segmnt video in a WordPress page with the segmnt video plugin and a shortcode, or with a Custom HTML block and a token from your server.
icon: "globe"
---

There are two ways to put a segmnt video on a WordPress page:

| Path | Best for | What you do |
| --- | --- | --- |
| **Plugin (easiest)** | Most sites. No code. | Install the `segmnt-video` plugin, save your key once, then add `[segmnt id="vid_..."]` to a page. |
| **Custom HTML block** | Any platform, or a site that cannot install a plugin. | Paste an HTML block, and build the playback token on your server. |

Both paths need the same two things: a **ready** video (`vid_...`) and a
**verified domain**. The plugin does **not** remove the verified-domain step.

## Easy path: the segmnt video plugin

The plugin keeps your API key on your server and gets a signed playback token
for the page (from a short cache, and re-mints it before it expires). It ships
no second player: it frames the same player we use everywhere (hls.js,
automatic quality, Arabic RTL).

> **Status: Available.** The plugin is version 1.0.0. Download it from
> [segmnt-video-1.0.0.zip](https://segmnt.cloud/downloads/segmnt-video-1.0.0.zip).
> It is not in the WordPress.org directory yet — that is a later phase.

### Step 1: Install the plugin

1. Download
   [segmnt-video-1.0.0.zip](https://segmnt.cloud/downloads/segmnt-video-1.0.0.zip).
2. In WordPress, go to **Plugins → Add New → Upload Plugin**.
3. Upload the zip and click **Install Now**.
4. Click **Activate**.

### Step 2: Paste your key and project id once

1. Go to **Settings → segmnt video**.
2. Paste:
   - your **project id** (`prj_...`), and
   - a **playback** API key (`sk_...`). Create a playback key in the segmnt
     dashboard; it can only make playback tokens — it cannot upload, delete or
     read your account.
3. Click **Save Changes**.

The key is stored on your server and is never sent to the browser.

### Step 3: Make the domain verified

The domain of this WordPress site must be a **verified domain** of the project,
for example `blog.example.com` or `courses.example.com`. If WordPress is at
`www.example.com`, verify `www.example.com` too, because we match the exact
host. See [Accounts, projects, API keys and domains](/accounts-and-projects).

If the domain is not verified, the API answers `400` and the video does not
play.

### Step 4: Add the shortcode to a page

Edit the page and add a **Shortcode** block:

```
[segmnt id="vid_9xR4tL"]
```

Replace `vid_9xR4tL` with your video id. Attributes are optional:

| Attribute | Required | Default | Meaning |
| --- | --- | --- | --- |
| `id` | yes | - | The video id, `vid_...` |
| `title` | no | `segmnt video` (localized) | iframe title, for screen readers |
| `locale` | no | the site locale, else `ar` | player language: `ar` or `en` |
| `aspect` | no | `16 / 9` | responsive aspect ratio |

### Page cache and the 4-hour token

The plugin asks for the normal **4-hour** playback token and re-mints a few
minutes before it expires. WordPress and your host may cache the built HTML, so
keep the **page-cache lifetime under about 4 hours**, or leave `[segmnt]` pages
**uncached**. A page cached for longer can serve a token that has already
expired, and the video stops.

## Custom HTML block (portable to other platforms)

Use this path for any other site or page builder, or when you cannot install a
plugin. It works with the **Gutenberg editor** and any theme.

### Step 1: Add a Custom HTML block

1. Edit the page in WordPress.
2. Click **+**, search for **Custom HTML**, and add the block.
3. Paste this, and replace `vid_9xR4tL` with your video id and
   `YOUR_PLAYBACK_TOKEN` with a real token:

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

4. Preview the page. The video plays on the preview domain only if that domain
   is verified and the token was made for it.

### Step 2: Where the token comes from

A token **expires** (4 hours by default), so pasting one token into the block is
only good for a short test. For a real page, the token must be **fresh** on each
page load. There are two ways.

#### Option A: A quick test (paste a token by hand)

Make a token once from your server with the playback-tokens call, paste it into
the block, and preview the page. This is fine to check the setup. It stops
playing when the token expires.

```bash
curl -X POST "https://api.segmnt.cloud/v1/projects/$PROJECT_ID/videos/vid_9xR4tL/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}'
```

Copy the `token` value from the answer. See
[Create playback tokens on your server](/playback-tokens).

#### Option B: A fresh token on every page load (recommended)

WordPress runs on PHP on your server, so it can call our API and print the
iframe with a fresh token. Add a small **shortcode** to your theme or a small
plugin, then use it in any page (or keep the Custom HTML block and paste the
built HTML).

Keep the API key in `wp-config.php`, **never** in a page or the block:

```php
// wp-config.php — keep this out of version control and out of any page.
define('SEGMNT_API_KEY', 'sk_...');
define('SEGMNT_PROJECT', 'prj_7kQ2mB');
define('SEGMNT_API_BASE', 'https://api.segmnt.cloud');
define('SEGMNT_CDN', 'https://cdn.segmnt.cloud');
```

```php
// functions.php (a small plugin is cleaner) — a [segmnt_embed] shortcode.
// The tag is different from the plugin's [segmnt] so the two never clash.
add_shortcode('segmnt_embed', function ($atts) {
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
[segmnt_embed video="vid_9xR4tL"]
```

Notes:

- The shortcode runs in PHP on your server, so the **API key never reaches the
  browser**.
- The domain must be **verified** here too. This path does not remove that step.
- WordPress caches pages. Keep the page-cache lifetime **under about 4 hours**,
  or leave the page uncached, so the token is still valid when the page is
  served.
- If the page is public and long-lived, a short token is safer. Ask for a new
  token per page load, and do not cache the HTML that holds it.

## segmnt in an LMS: Tutor LMS and LearnDash

Course plugins keep lesson text in the WordPress editor, so both paths above
work inside a lesson. The short version: put `[segmnt id="vid_9xR4tL"]` (plugin
path) in the lesson body. The video plays on the same verified domain as the
rest of the site, and the token rules do not change.

**Tutor LMS.** Open the lesson (or the topic) in the course builder and add a
**Shortcode** block in the lesson body with `[segmnt id="vid_9xR4tL"]`. Tutor
renders lesson content through the normal WordPress content filters, so the
shortcode builds the iframe with a fresh token, exactly as on a normal page.

**LearnDash.** Lessons and topics use the standard WordPress editor, so a
**Shortcode** block with `[segmnt id="vid_9xR4tL"]` works in the lesson body.
If you build lessons with a page builder (for example Elementor with the
LearnDash module), use its Shortcode or HTML widget with the same shortcode.

If a lesson shows the shortcode as plain text, the builder rendered the content
without WordPress's shortcode filters. Use the **Custom HTML** block with the
iframe code instead, and keep the token fresh with the manual-path shortcode
from [Option B](#option-b-a-fresh-token-on-every-page-load-recommended) above.

Two rules carry over unchanged:

- **Domain.** The lesson plays on the site's host, so that host must be the
  verified domain. A staging or preview host is refused until you verify it.
- **Cache.** An LMS that caches lesson HTML has the same 4-hour token rule as
  a cached page: keep the cache lifetime under about 4 hours, or leave lesson
  pages uncached.

## Common problems

| Problem | Cause | Fix |
| --- | --- | --- |
| The block shows code, not a video | The editor escaped the HTML, or the block is a Code block, not a Custom HTML block | Use the **Custom HTML** block, or the shortcode block. |
| The lesson shows `[segmnt ...]` as text | The LMS builder skipped the shortcode filters | Use the Custom HTML block with the iframe, or the `[segmnt_embed]` shortcode from Option B. |
| The video does not play (plugin) | The site domain is not a verified domain, the key is not a playback key, or the video is not ready | Check **Settings → segmnt video**, verify the domain, and make sure the video is ready and not held for review. |
| The video plays, then stops after a while (plugin) | The page cache is holding the page longer than the token lives | Keep the page-cache lifetime under about 4 hours, or leave `[segmnt]` pages uncached. |
| "This video needs a playback token" | `YOUR_PLAYBACK_TOKEN` was left in the code | Replace it with a real token, or use Option B. |
| "This playback link has expired" | The token is older than its `expiresAt` | Use Option B, or make a longer token. |
| "Origin does not match the allowed domain" | The site's domain is not the token's domain, or it is not verified | Verify the exact host (`www.` counts as its own host) and make the token for it. |

If you revoke a playback key, new page views stop getting a token at once. A
viewer who already loaded the page keeps playing until that token expires.

## Where to go next

- [Embed a video on any website](/embed-a-video)
- [Create playback tokens on your server](/playback-tokens)
- [How playback protection works](/playback-protection)

## Tested against the current release

Checked on 15 Oct 2026:

- The **segmnt video** plugin v1 was verified on a real WordPress site, in
  English and Arabic (RTL), including the "the key never leaves the server"
  and "revoke the key" checks.
- The embed host serves the player: `GET /e/{videoId}` with no token → `403`
  `token_missing`; `GET /segmnt-player.js` → `200`.
- The playback-tokens route is live: with no key it answers `401 unauthorized`.
- The plugin shortcode mints through the same route as the cURL example, which
  is covered by our API tests.
- The manual-path shortcode uses the public WordPress HTTP API
  (`wp_remote_post`), the same call as the cURL example.
