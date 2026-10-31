---
title: API overview
description: Base URL, authentication, conventions, and every public endpoint in one list.
icon: "square-terminal"
---

The segmnt API is a REST API. Every request and answer is JSON.

**Base URL:** `https://api.segmnt.cloud`

This reference covers **42 public endpoints across 33 paths**. Every public
endpoint has an entry in the sidebar, with its request and response shapes.

## Authentication

There are two ways to authenticate.

| Method | Header or cookie | Use it for |
| --- | --- | --- |
| **API key** | `Authorization: Bearer sk_...` | Server-to-server calls for one project: uploads, videos, playback tokens, webhooks, usage. |
| **Session** | `segmnt_session` cookie | Account and dashboard calls: projects, API keys, domains, billing. It comes from SSO sign-in. |

Rules:

- Keep the **API key on your server**. It can create playback tokens, so anyone
  with it can play your videos anywhere. Never put a key in a page, a browser or
  a mobile app.
- A key works only for its own project's routes. A key from another project
  answers `404`.
- A revoked key answers `401` immediately.
- Account routes (projects, API keys, domains, billing) accept only the session.
  An API key cannot manage keys, projects or domains.
- A key has a **scope**: `full` (every route of its project) or `playback` (only
  the playback-token route). See
  [Accounts, projects, API keys and domains](/accounts-and-projects).
- Sign-in is SSO only: Google and GitHub. We never store a password.

Full detail: [Accounts, projects, API keys and domains](/accounts-and-projects).

## Conventions

- **IDs** are strings with a type prefix: `acc_`, `usr_`, `prj_`, `vid_`,
  `upl_`, `utk_`, `key_`, `dom_`, `whe_`, `evt_`, `rep_`.
- **Times** are RFC 3339 in UTC.
- **Pagination** is by cursor. Pass the `nextCursor` from an answer back as
  `cursor` to get the next page.
- **Errors** are `application/problem+json` with a stable `code`. See
  [Errors](/errors).
- **`Idempotency-Key`** is scoped to the project. Only upload create and video
  reports read it. The same key returns the first result. We do not compare the
  body, so use a new key for a new request.
- **Money** is an integer in the smallest currency unit (halalas for SAR;
  1 SAR = 100 halalas).
- Bodies are `application/json`. Another content type gets `415`.

## Rate limits

| Area | Limit |
| --- | --- |
| `/v1/**` | 1,200 requests per minute, per client IP |
| `/auth/**` | 60 requests per minute, per client IP |

Over a limit you get `429` with `Retry-After`. See [Errors](/errors).

## The public endpoints

### System and auth

| Method | Path |
| --- | --- |
| `GET` | `/health` |
| `GET` | `/auth/providers` |
| `GET` | `/auth/{provider}/start` |
| `GET` | `/auth/{provider}/callback` |
| `POST` | `/auth/logout` |
| `GET` | `/v1/me` |

### Projects, keys and domains

| Method | Path |
| --- | --- |
| `GET` `POST` | `/v1/projects` |
| `GET` `PATCH` `DELETE` | `/v1/projects/{projectId}` |
| `GET` `POST` | `/v1/projects/{projectId}/api-keys` |
| `DELETE` | `/v1/projects/{projectId}/api-keys/{keyId}` |
| `GET` `POST` | `/v1/projects/{projectId}/domains` |
| `DELETE` | `/v1/projects/{projectId}/domains/{domainId}` |
| `POST` | `/v1/projects/{projectId}/domains/{domainId}/verify` |

### Uploads and videos

| Method | Path |
| --- | --- |
| `POST` | `/v1/projects/{projectId}/upload-tokens` |
| `POST` | `/v1/projects/{projectId}/uploads` |
| `DELETE` | `/v1/projects/{projectId}/uploads/{uploadId}` |
| `POST` | `/v1/projects/{projectId}/uploads/{uploadId}/complete` |
| `GET` | `/v1/projects/{projectId}/videos` |
| `GET` `PATCH` `DELETE` | `/v1/projects/{projectId}/videos/{videoId}` |
| `GET` | `/v1/demo/playback` |
| `POST` | `/v1/projects/{projectId}/videos/{videoId}/playback-tokens` |
| `POST` | `/v1/projects/{projectId}/videos/{videoId}/preview-token` |
| `POST` | `/v1/projects/{projectId}/videos/{videoId}/reports` |

To upload a video, see [Direct upload](/direct-upload). To rename it or **delete**
it, see [Manage videos](/manage-videos). To let a browser upload on your own
domain, see the browser upload token in [Direct upload](/direct-upload#browser).

### Webhooks, usage and billing

For the events we send, the payload, and how to check `Segmnt-Signature`, see
[Webhooks](/webhooks).

| Method | Path |
| --- | --- |
| `GET` `POST` | `/v1/projects/{projectId}/webhook-endpoints` |
| `DELETE` | `/v1/projects/{projectId}/webhook-endpoints/{endpointId}` |
| `POST` | `/v1/projects/{projectId}/webhook-endpoints/{endpointId}/rotate-secret` |
| `GET` | `/v1/projects/{projectId}/webhook-endpoints/{endpointId}/deliveries` |
| `POST` | `/v1/projects/{projectId}/webhook-endpoints/{endpointId}/test` |
| `GET` | `/v1/projects/{projectId}/usage` |
| `GET` | `/v1/billing/plan` |
| `GET` | `/v1/billing/plans` |
| `POST` | `/v1/billing/checkout` |
| `GET` `POST` | `/v1/billing/whitelist` |

## Where to go next

- [Quickstart](/quickstart) — the end-to-end path, from upload to a `403` off-domain.
- [Manage videos](/manage-videos) — rename or delete a video.
- [Webhooks](/webhooks) — get a signed event when a video changes.
- [Errors](/errors) — every code, with the `403` and `409` causes.
- [Changelog](/changelog) — what changed and when.
- [Glossary](/glossary) — the words used here.
