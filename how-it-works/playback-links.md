---
title: Playback links
description: Playback links are signed for one video and one domain. A link used on another website stops working. Here is what that check does and does not stop.
icon: "link"
---

Every protected video plays with a signed link. The link names the video, the
time it expires and your verified domain. We check it on every request.

## What we decided

- Every protected video needs a link signed for that video.
- A link works until it expires. The default life is **4 hours**. You can set
  it from 60 seconds to 24 hours when you create the link.
- A link works only on a verified domain. A request from another website gets
  **403**.

## Why

- Someone who can play a video already has its link. The check protects the
  video from the common case: another website puts your video on its page.
- A short life would stop long videos in the middle. The life must be longer
  than the video.
- A link signed for one video cannot play a different video.

## What it does not stop

- A person who copies a link and sends their own request headers can still
  play the video until the link expires. The domain check is a browser rule.
  It is not a lock on the file.
- A viewer who can play the video can record it. See [No DRM](/how-it-works/no-drm).
- Link sharing is a separate topic. See [How playback protection works](/playback-protection).

## What it means for you

- Create the link on your server. Do not put your API key in a page. See
  [Playback tokens](/playback-tokens).
- Verify your domain before you test. Without a verified domain, the video
  does not play.
