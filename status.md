---
title: Feature status
description: The public status labels used across segmnt docs and the website, and the current status of each feature.
icon: "list-checks"
---

We use four labels on every page and on the website. They mean the same thing
everywhere.

| Label | What it means |
| --- | --- |
| **Available** | The feature works today. Follow the guide as written. |
| **Beta** | The feature works, but it may change, and we have not finished checking it everywhere. Some plans cannot use it yet. |
| **Coming soon** | The feature is not available yet. We describe the plan, not a working feature. |
| **Not planned** | We are not going to build this. Do not wait for it. |

A page with no label is **Available**.

## What is available today

| Feature | Status | Notes |
| --- | --- | --- |
| Automatic encode at upload (HLS: 360p, 720p, 1080p) | Available | A free plan plays up to 720p. |
| Signed playback tokens | Available | Your server creates one token per video and domain. |
| Domain-locked playback | Available | A copied link returns `403` on another domain. |
| Embed codes (iframe and JavaScript) | Available | See [Embed a video on any website](/embed-a-video). |
| Thumbnails and review frames | Available | A thumbnail at 2 seconds, and one review frame every 10 seconds. |
| Video events (webhooks) | Available | `video.ready`, `video.failed`, `video.held` and `video.removed`. Register your own HTTPS receiver and upload a short video to try them. See [Webhooks](/webhooks). |
| Webhooks in the dashboard | Available | Add, list and remove endpoints, show the signing secret once, read the delivery log and send a test event. See [Webhooks](/webhooks). |
| Webhook secret rotation | Available | Replace an endpoint's signing secret without recreating the endpoint. API only today (no dashboard button). See [Webhooks](/webhooks#rotate-the-signing-secret). |
| Webhook delivery log | Available | Read each send attempt with its status and the last HTTP status your endpoint returned. See [Webhooks](/webhooks#read-the-delivery-log). |
| Send a test event | Available | Send one signed `webhook.test` event to your endpoint and see the result in the delivery log. See [Webhooks](/webhooks#send-a-test-event). |
| Browser upload token | Available | Your server mints a short-lived token. The browser uses it to upload one file without your API key. |
| Hold and removal notice | Available | When we hold or remove a video we e-mail the account owner with the reason, and a removal also sends the `video.removed` webhook. A legal matter is not notified. See [Reports and video review](/reports-and-review). |

## What is coming soon

| Feature | Status | Notes |
| --- | --- | --- |
| 1440p and 4K renditions | Coming soon | The higher ladder is built, but no customer can reach it in production today. |
| 4K playback through the CDN | Coming soon | 4K is not reachable while paid plans are not on sale. |
| Viewer network binding (IP binding) | Coming soon | An opt-in project setting that binds a playback token to the viewer's network (IPv4 `/24`, IPv6 `/64`). Built; we have not offered it publicly yet. Not DRM: viewers on the same network can still share a link. See [How playback protection works](/playback-protection#part-3-viewer-network-binding-optional). |
| Automatic video review (frame check) | Coming soon | There is no automatic frame check on a live account yet. |
| Free-plan review hold | Coming soon | A new free video is not held for review today. |
| Paid plans, checkout and overage | Coming soon | Plans and prices are listed; checkout is paused. See [Plans and payment](/billing). |
| White-label player | Coming soon | Built, and it needs a paid plan. Paid plans are not on sale yet. |
| Original-file download | Coming soon | The original stays with us; there is no download button yet. |

> The website uses this same legend. If a page and this table disagree, this
> table is right — tell us and we will fix the page.

## What is not planned

| Feature | Status | Notes |
| --- | --- | --- |
| Phone verification at sign-up | Not planned | We are not adding a phone step at sign-up. A verified domain is the gate. The phone endpoints stay in the API for now; see [Phone verification](/phone-verification). |

## How we keep this honest

Each row comes from our release status. We change a row only when a release
proves the new state. We do not write a shipped-sounding sentence for a feature
that is not available.

If you build on a **Beta** feature, expect the details to change. Ask us before
you depend on one in production.
