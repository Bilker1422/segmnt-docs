---
title: "What is segmnt"
description: What segmnt is, who it is for, when to use it, when not to, and how it compares with self-hosting video or using a public video platform.
icon: "info"
---

segmnt is signed-URL video infrastructure. You upload a video once, through the API or the dashboard. segmnt encodes it to adaptive HLS, stores it, and streams it on your own domain. A video plays only when your server creates a playback token for it, and only on a domain you have verified.

## Who it is for

- Course platforms and edtech companies that sell or host courses.
- Training companies and agencies that deliver video for their clients. One account can hold several projects, so each client's videos stay apart.
- Creators and SaaS apps that need video inside their own product.

## When to use it

Use segmnt when:

- Your viewers should watch the video on your own pages.
- You want playback protected. Your server creates a token, and the video plays only on verified domains.
- You want your product to know when a video is ready. segmnt sends a `video.ready` webhook.
- You do not want to run your own encoding or video storage.

## When not to use it

Do not use segmnt when:

- You want a public video site with search, discovery or comments. segmnt is infrastructure. It has no public video platform.
- You need DRM. See [No DRM](/how-it-works/no-drm).

## How it compares

| | Self-host the files | Public video platform | segmnt |
| --- | --- | --- | --- |
| Encoding | You build and run it yourself. | Done by the platform. | Done once at upload, to adaptive HLS. |
| Playback protection | You build it yourself. | Depends on the platform, and videos are often public. | Signed playback tokens and domain lock, included. |
| Where the video plays | Your pages. | Mostly the platform's own pages. | Your pages, with the embed code or the player. |
| Built for | Teams with a media engineering team. | Public audiences. | Products that need video inside their own product. |

## How to start

1. Sign in with Google or GitHub. A free account is created on your first sign-in.
2. Create a project and an API key.
3. Verify your domain. Videos do not play until a domain is verified.
4. Upload a video, then wait for the `video.ready` webhook or check the video status.
5. Create a playback token on your server and embed the player on your page.

The full steps are in the [quickstart](/quickstart).

Every call uses the same base URL and header:

```bash
curl -sS "https://api.segmnt.cloud/v1/projects/$PROJECT_ID/videos" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Base URL: `https://api.segmnt.cloud`. Send the API key in the `Authorization: Bearer <token>` header. Keep the key on your server. See [Authentication](/authentication).

## Related pages

- [Quickstart](/quickstart): your first signed video in a few steps.
- [API overview](/api-overview): base URL, authentication and every public endpoint.
- [Playback protection](/playback-protection): how signed playback and the domain lock work.
- [Glossary](/glossary): the words segmnt uses.
