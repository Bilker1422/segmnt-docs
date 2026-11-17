---
title: Keeping and deleting videos
description: When you delete a video, what we remove and when. What happens to your videos if a paid plan stops.
icon: "trash-2"
---

This page explains how long we keep your files, and what happens when you
delete a video or when a plan stops. The full policy is on the
[Data retention and deletion](https://segmnt.cloud/retention) page.

## What we decided

- **Deleting a video** removes its files, and playback stops. The review frames
  are removed when the video is deleted, unless a review or an abuse report
  about it is still open. In every case they are removed no later than **30 days**
  after the delete, unless a legal matter about that video is open.
- **Your original file** stays until you delete the video.
- **Removed or held videos.** When we hold or remove a video, we send a
  `video.held` or `video.removed` webhook and e-mail the account owner. When we
  can, we include the reason.
- **If a paid plan stops**, we delete the stored videos 30 days after the paid
  plan stops, and never earlier than 7 days after we e-mail the account owner a
  notice. Until then you can renew your plan or export your videos.
- **Legal matters.** If the law requires us to keep a file, we keep it while the
  matter is open.

Paid plans are not on sale yet. The plan-end rule does not apply to any account
today.

## Why

- We delete on your request, so you control your files.
- The warning gives you time to renew, or to export your videos. We do not want
  a delete without notice, so the rule waits for the warning.
- Review frames exist to check content. We keep review frames while the video
  exists. After you delete the video, the review frames are removed when the
  video is deleted, unless a review or an abuse report about it is still open. In
  every case they are removed no later than 30 days after the delete, unless a
  legal matter about that video is open.

## What it means for you

- Export your videos before a plan stops, if you need to keep them.
- If we cannot send the warning e-mail, we do not delete the videos.
- See [Account and billing](/billing) for plan states.
