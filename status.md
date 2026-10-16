---
title: Feature status
description: The public status labels used across segmnt docs and the website, and the current status of each feature.
icon: "list-checks"
---

We use three labels on every page and on the website. They mean the same thing
everywhere.

| Label | What it means |
| --- | --- |
| **Available** | The feature works today. Follow the guide as written. |
| **Beta** | The feature works, but it may change, and we have not finished checking it everywhere. Some plans cannot use it yet. |
| **Coming soon** | The feature is not available yet. We describe the plan, not a working feature. |

A page with no label is **Available**.

## What is available today

| Feature | Status | Notes |
| --- | --- | --- |
| Automatic encode at upload (HLS: 360p, 720p, 1080p) | Available | A free plan plays up to 720p. |
| Signed playback tokens | Available | Your server creates one token per video and domain. |
| Domain-locked playback | Available | A copied link returns `403` on another domain. |
| Embed codes (iframe and JavaScript) | Available | See [Embed a video on any website](/embed-a-video). |
| Thumbnails and review frames | Available | A thumbnail at 2 seconds, and one review frame every 10 seconds. |
| Video events (webhooks) | Available | `video.ready`, `video.failed` and `video.held`. No customer endpoint has received one yet: register your own receiver and upload a short video. |

## What is coming soon

| Feature | Status | Notes |
| --- | --- | --- |
| 1440p and 4K renditions | Coming soon | The higher ladder is built, but no customer can reach it in production today. |
| 4K playback through the CDN | Coming soon | 4K is not reachable while paid plans are not on sale. |
| Phone verification at sign-up | Coming soon | The API is available, but the sign-up step is not on. See [Phone verification](/phone-verification). |
| Automatic video review (frame check) | Coming soon | There is no automatic frame check on a live account yet. |
| Free-plan review hold | Coming soon | A new free video is not held for review today. |
| Removal notice and the `video.removed` webhook | Coming soon | We will e-mail the account owner with the reason when we hold or remove a video, and send `video.removed` on a removal. A legal matter is not notified. |
| Paid plans, checkout and overage | Coming soon | Plans and prices are listed; checkout is paused. See [Plans and payment](/billing). |
| White-label player | Coming soon | Built, and it needs a paid plan. Paid plans are not on sale yet. |
| Original-file download | Coming soon | The original stays with us; there is no download button yet. |

> The website uses this same legend. If a page and this table disagree, this
> table is right — tell us and we will fix the page.

## How we keep this honest

Each row comes from our release status. We change a row only when a release
proves the new state. We do not write a shipped-sounding sentence for a feature
that is not available.

If you build on a **Beta** feature, expect the details to change. Ask us before
you depend on one in production.
