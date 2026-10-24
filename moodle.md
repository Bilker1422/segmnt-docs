---
title: Add segmnt to Moodle
description: Put a segmnt video in a Moodle course with an iframe in a Label or Page, and keep a fresh playback token on every page load.
icon: "graduation-cap"
---

Put a segmnt video in a Moodle course: in a Label on the course page, in a Page
or Book chapter, or in an HTML block. Moodle is PHP on your server, so it can
also make a fresh playback token on every page load — the same job the
[WordPress plugin](/wordpress) does there.

Moodle treats pasted iframes carefully. An **administrator** can always paste
an iframe. A **teacher** can only save iframe code when the site allows it.
Section 2 shows the two switches an admin can set.

## Before you start

1. Your video is **ready** (`vid_...`). See [Manage videos](/manage-videos).
2. The Moodle site's host, for example `moodle.example.com`, is a **verified
   domain** of the project. See
   [Accounts, projects, API keys and domains](/accounts-and-projects).
3. A **playback** API key (`sk_...`) is stored on the Moodle server, ready for
   the token step. Create a playback key in the segmnt dashboard; it can only
   make playback tokens.

## 1. Let iframes into course content

By default, Moodle cleans the HTML that teachers save, and an iframe is often
removed. An administrator can change this in two ways. Pick one.

**Option A: allow embed tags site-wide.** In
**Site administration → Security → Site policies**, turn on
**Allow EMBED and OBJECT tags**. This lets authors paste third-party embed
code anywhere. It is the broad switch: check it fits your site's rules first.

**Option B: trust a teacher role instead.** Give the teacher role the
capability **`moodle/site:trustcontent`**. Users with that capability keep
their iframe code; everyone else stays cleaned. This is the narrower switch.

Test with the same role that will add the videos in real courses. If the iframe
still disappears, paste it again with an administrator account.

> Moodle's editor may also tidy pasted code. If the iframe code vanishes while
> you type, use the editor's **HTML / source** view (`</>` in the toolbar) to
> paste the block.

## 2. Add the video to a course page

1. Make a playback token for the video and your Moodle host, with the cURL
   call from [Create playback tokens on your server](/playback-tokens).
2. Edit the Label, Page or HTML block, open the HTML/source view, and paste:

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

3. Replace `vid_9xR4tL` with your video id and `YOUR_PLAYBACK_TOKEN` with the
   real token.
4. Save and view the page as a **student** would. The video plays only on the
   verified host, so a look at the page as a real user is the check that
   matters.

This hand-made token is good for a short test. It stops working when it
expires (4 hours by default). A real course page needs a fresh token on every
page load — the next section.

## 3. A fresh token on every page load

For a page students open every day, do not paste one token. Let the Moodle
server mint a new one each time it builds the page. The recommended way is a
small **text filter** plugin: it replaces a short placeholder in any course
text with the finished iframe.

The placeholder in the teacher's text:

```
[[segmnt:vid_9xR4tL]]
```

The filter (the shape of a Moodle `filter` plugin's `filter()` method):

```php
<?php
// filter/segmnt/filter.php — a minimal Moodle text filter.
// Install as filter/segmnt, enable it in Site administration -> Plugins ->
// Text filters, then write [[segmnt:vid_...]] in any course text.

class filter_segmnt extends moodle_text_filter {
    public function filter($text, array $options = []) {
        if (strpos($text, '[[segmnt:') === false) {
            return $text;
        }
        return preg_replace_callback('/\[\[segmnt:([A-Za-z0-9_-]+)\]\]/', function ($m) {
            return $this->segmnt_iframe($m[1]);
        }, $text);
    }

    private function segmnt_iframe(string $videoid): string {
        $key     = get_config('filter_segmnt', 'apikey');   // set on the plugin's settings page
        $project = get_config('filter_segmnt', 'projectid');
        $host    = parse_url(config::wwwroot(), PHP_URL_HOST); // the verified domain

        $curl = new curl();
        $res = $curl->post(
            'https://api.segmnt.cloud/v1/projects/' . $project . '/videos/' . $videoid . '/playback-tokens',
            json_encode(['domain' => $host]),
            ['CURLOPT_HTTPHEADER' => [
                'Authorization: Bearer ' . $key,
                'Content-Type: application/json',
            ], 'CURLOPT_TIMEOUT' => 10],
        );
        $data = json_decode($res, true);
        $token = $data['token'] ?? '';
        if ($token === '') {
            return '<!-- segmnt: could not create a playback token -->';
        }

        $src = 'https://cdn.segmnt.cloud/e/' . rawurlencode($videoid) . '?token=' . rawurlencode($token);
        return '<div style="position:relative;width:100%;aspect-ratio:16 / 9;">'
            . '<iframe src="' . s($src) . '" title="segmnt video" loading="lazy"'
            . ' style="position:absolute;inset:0;width:100%;height:100%;border:0;"'
            . ' allow="autoplay; fullscreen; picture-in-picture; encrypted-media"'
            . ' allowfullscreen referrerpolicy="origin"></iframe></div>';
    }
}
```

Notes on this sample:

- **The API key lives in the plugin's admin settings** (a site
  administrator pastes it once, like the WordPress plugin does), never in a
  course page or a code comment in a lesson.
- **The domain comes from the server** (`wwwroot`), not from the page, so a
  page author cannot pick another domain.
- The HTTP call is the same `POST .../playback-tokens` call shown in
  [Create playback tokens on your server](/playback-tokens). That route, its
  answer shape (`token`, `expiresAt`, `embedUrl`) and the key rules are
  covered by the API tests.

> **Caching.** Moodle caches the output of text filters, and your host may
> cache whole pages. A cached page serves an old token, and the video stops
> when the token expires. Before you rely on the filter in a live course,
> check the caching section of the [Moodle filter plugin
> documentation](https://moodledev.io/general/plugins/types/filter) and keep
> any page-cache lifetime under about 4 hours, as in the
> [WordPress guide](/wordpress).

Building the filter plugin takes care. If your course count is small, the
hand-made token in section 2, renewed by an editor, can be enough to start.

## Common problems

| Problem | Cause | Fix |
| --- | --- | --- |
| The iframe disappears after saving | The author's role is not trusted, and embed tags are not allowed | Section 1: allow embed tags, or give the role `moodle/site:trustcontent`, or paste as an administrator. |
| The iframe shows but the video does not play | The Moodle host is not a verified domain, or the token is for another domain | Verify the exact host and make the token for it. |
| "This video needs a playback token" | `YOUR_PLAYBACK_TOKEN` was left in the code | Make a real token, or set up the filter in section 3. |
| "This playback link has expired" | The page served a cached, expired token | Keep page caches under about 4 hours, or use the fresh-token filter. |
| The video plays for staff, not for students | Students see a cached page, or a different host (for example a mobile app web view) | Check the page as a student and compare the host with the token's domain. |

## Where to go next

- [Embed a video on any website](/embed-a-video)
- [Create playback tokens on your server](/playback-tokens)
- [Add segmnt to WordPress](/wordpress): the same two paths, on WordPress.

## Tested against the current release

Checked on 24 Oct 2026:

- The embed block is the same markup the dashboard's **Copy code** box builds:
  `/e/{videoId}?token=…`, the `allow` list, `referrerpolicy="origin"` and the
  responsive wrapper.
- `GET https://cdn.segmnt.cloud/segmnt-player.js` → `200`; `GET /e/{videoId}`
  with no token → `403` `token_missing` (checked on the TEST deploy).
- The filter's HTTP call is the same playback-tokens request as the cURL
  example, which the API tests cover.
- **Unverified:** the filter class itself has not run on a Moodle install —
  the TEST environment has no Moodle site yet. The Moodle-specific parts
  (plugin settings, the `moodle_text_filter` shape, filter output caching)
  follow the Moodle filter plugin documentation. Verify section 1 and 2 on
  your Moodle site before a live course, and treat section 3 as a starting
  point, not a finished plugin.
