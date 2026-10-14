---
title: Accounts, projects, API keys and domains
description: Sign in with SSO, then set up projects, per-project API keys and verified domains.
icon: "user"
---

This page covers sign-in, your session, and how to set up **projects**,
**API keys** and **domains**.

You sign in with SSO only. **We never store a password.**

This guide is written in simple English. It is in English only; the Arabic
version is on the Arabic docs site at
[test-docs.segmnt.cloud](https://test-docs.segmnt.cloud/ar/accounts-and-projects/).

## Before you start

- Dashboard in TEST: `https://test-app.segmnt.cloud`
- API in TEST: `https://test-api.segmnt.cloud`
- IDs: accounts start with `acc_`, users with `usr_`, projects with `prj_`,
  API keys with `key_`, domains with `dom_`.

## 1. Sign in (SSO only)

Open the dashboard and pick a provider:

- **Google**
- **GitHub**

The buttons come from the API, so the dashboard only shows providers that work
in this environment:

```bash
curl https://test-api.segmnt.cloud/auth/providers
```

```json
{"data":[{"id":"google","label":"Google"},{"id":"github","label":"GitHub"}]}
```

When you pick a provider, the browser goes to `/auth/{provider}/start`, the
provider signs you in, and it returns the browser to
`/auth/{provider}/callback`. The callback creates your account on the first
sign-in and starts your session.

```bash
curl -i "https://test-api.segmnt.cloud/auth/google/start?return_to=/videos"
```

Answer: `302` to the provider. We also set a short-lived flow cookie
(`segmnt_auth`, 10 minutes) that holds the CSRF `state`, the PKCE verifier and
the OIDC `nonce`.

Notes:

- `return_to` can be a dashboard path (`/videos`) or a full dashboard URL.
  Anything else goes to the dashboard home, so this route cannot be used to
  send you to another site.
- We link a new sign-in to your account **only** when the provider says your
  e-mail is verified. Sign in with Google and GitHub using the same verified
  e-mail and you land in one account. A provider that gives no verified e-mail
  makes a new account.
- **Apple** and **Reddit** come later. Apple waits for the board to approve the
  paid Apple Developer account; Reddit waits for Reddit to approve our OAuth
  app. Adding a provider is a small change on our side.

## 2. Your session

After sign-in we send a `segmnt_session` cookie:

| Property | Value |
| --- | --- |
| Lifetime | 30 days |
| Flags | `HttpOnly`, `Secure`, `SameSite=Lax`, `Path=/` |

Sessions are **server-side, not JWTs**. The cookie holds a random id, and we
store only the SHA-256 hash of that id. Signing out deletes the session row. If
you lose the cookie, sign in again.

Every `/v1/projects...` call needs this session cookie. Without a valid session
you get `401 unauthorized`:

```bash
curl https://test-api.segmnt.cloud/v1/projects
```

```json
{"type":"https://docs.segmnt.cloud/errors/unauthorized","title":"Missing or expired session.","status":401,"code":"unauthorized","detail":"Missing or expired session."}
```

`GET /v1/me` returns the signed-in user and account. It uses the session cookie
and never returns any hidden account field:

```json
{
  "user": { "id": "usr_...", "email": "you@example.com", "displayName": "...", "role": "owner" },
  "account": {
    "id": "acc_...",
    "name": "...",
    "planId": "free",
    "planStatus": "active",
    "createdAt": "...",
    "phoneVerified": false,
    "phoneVerificationRequired": true
  }
}
```

`phoneVerified` is `true` once a phone number is verified. `phoneVerificationRequired`
is `true` when the account must verify a Saudi mobile number before its first
upload (a new free account). See [Phone verification](/phone-verification).

Sign out:

```bash
curl -X POST https://test-api.segmnt.cloud/auth/logout
```

Answer `204`. Safe to call twice.

## 3. Projects

Use one project per client, site or course. An account can have many projects,
so an agency can keep each client separate.

In the dashboard you can create, rename and delete a project. **Delete works
only for an empty project.** Delete the videos first, or you get
`409 project_not_empty`. Deleting a project also stops its API keys and domains.

REST endpoints (all use the session cookie):

| Method | Path | Body / notes |
| --- | --- | --- |
| `GET` | `/v1/projects` | `limit` (1-100, default 20) and `cursor`. Newest first. |
| `POST` | `/v1/projects` | `{"name": "Client A courses"}` (1-100 characters) → `201` |
| `GET` | `/v1/projects/{projectId}` | one project |
| `PATCH` | `/v1/projects/{projectId}` | `{"name": "New name"}` → `200` |
| `DELETE` | `/v1/projects/{projectId}` | empty project → `204`; not empty → `409 project_not_empty` |

A list page looks like this. Pass `nextCursor` back as `cursor` to get the next
page:

```json
{
  "data": [ { "id": "prj_...", "name": "Client A courses", "createdAt": "2026-10-12T07:00:00.000Z" } ],
  "nextCursor": null
}
```

## 4. API keys

API keys are **per project** and are for servers, not browsers. Create them in
the dashboard, then copy the key once.

Format: `sk_{8 characters}_{64 hex characters}`, for example
`sk_Abcd1234_<64 hex>`. We show the full key **only in the create answer**, and
we store only the SHA-256 hash of the secret. Keep it secret. If you lose it,
revoke it and make a new one.

Your server sends the key on the `Authorization` header:

```
Authorization: Bearer sk_...
```

The [Direct upload](/direct-upload) guide shows the calls that use it.

Endpoints (all use the session cookie; an API key cannot manage keys, projects
or domains):

| Method | Path | Body / notes |
| --- | --- | --- |
| `GET` | `/v1/projects/{projectId}/api-keys` | list; the secret is never returned |
| `POST` | `/v1/projects/{projectId}/api-keys` | `{"name": "Course site backend"}` → `201`, with `secret` once |
| `DELETE` | `/v1/projects/{projectId}/api-keys/{keyId}` | revoke → `204` |

A list entry has `id`, `name`, `prefix`, `createdAt` and `lastUsedAt`. The
secret is not in the list. **A revoked key gets `401` right away.** A key works
only for its own project's routes (for example uploads).

## 5. Domains

A domain tells us which sites may play a project's videos. Playback matches the
**exact** host, so a subdomain needs its own line: add `courses.example.com`,
not `example.com`.

Steps:

1. Add the host in the dashboard, or call `POST /v1/projects/{projectId}/domains`
   with `{"hostname": "courses.example.com"}`. The answer has status `pending`
   and a `verification` block:

   ```json
   { "type": "TXT", "name": "_segmnt-verify.courses.example.com", "value": "segmnt-verify=b41c09e2f7a3d6c1" }
   ```

2. Create that TXT record at your DNS provider.
3. Call verify. When the record is found, `status` becomes `verified`. If it is
   not found yet, it stays `pending` and you can try again later.

Endpoints (all use the session cookie):

| Method | Path | Body / notes |
| --- | --- | --- |
| `GET` | `/v1/projects/{projectId}/domains` | list |
| `POST` | `/v1/projects/{projectId}/domains` | `{"hostname": "courses.example.com"}` → `201` |
| `POST` | `/v1/projects/{projectId}/domains/{domainId}/verify` | check the DNS record |
| `DELETE` | `/v1/projects/{projectId}/domains/{domainId}` | remove → `204` |

Plan limit: a **free** account can verify **1** domain. More gives
`403 plan_limit_reached`; remove the old one or upgrade first. If the host is
already used, you get `409 hostname_taken`. Removing a domain refuses new tokens
for it. Tokens already issued keep working until they expire.

> The domain routes are live in TEST now. The customer-facing parts - the
> dashboard flow and the playback domain check - ship with the domains release.

## 6. Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text. The `type` field links to `https://docs.segmnt.cloud/errors/{code}`.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Missing or wrong field, or a bad cursor | Fix the request. Do not retry. |
| 400 | `invalid_state` | The sign-in flow expired or was changed | Start sign-in again. |
| 400 | `auth_denied` | You cancelled sign-in at the provider | Start sign-in again. |
| 401 | `unauthorized` | Missing, wrong, expired or revoked session or key | Sign in again, or check the key. |
| 403 | `plan_limit_reached` | Over your plan limit (for example 1 verified domain on free) | Remove something or upgrade. |
| 403 | `domain_not_verified` | Playback was asked for on a domain you have not verified | Verify the domain first. |
| 404 | `not_found` | Wrong id, or it is in another account | Check the id. |
| 404 | `not_found` | Unknown or disabled sign-in provider | Use a provider from `/auth/providers`. |
| 409 | `project_not_empty` | You tried to delete a project that has videos | Delete the videos first. |
| 409 | `hostname_taken` | The host is already added | Use another host, or claim it with its owner. |
| 500 | `internal_error` | We could not create your session | Try again. |
| 502 | `provider_error` | The sign-in provider returned an error | Try again. |
| 503 | `not_configured` | Sign-in is off in this environment | Nothing to fix on your side. |

## Tested in TEST

Checked against `https://test-api.segmnt.cloud` on 12 Oct 2026:

- `GET /auth/providers` → `200` with `google` and `github`.
- `GET /auth/google/start?return_to=/videos` → `302` to Google, with the
  `segmnt_auth` flow cookie (`Path=/auth`, `Max-Age=600`, `HttpOnly`, `Secure`,
  `SameSite=Lax`).
- `GET /v1/me` and `GET /v1/projects` with no cookie → `401 unauthorized`.
- `POST /auth/logout` with no cookie → `204`.
- `GET /auth/google/callback` with no `state` → `400 invalid_state`.
- `GET /v1/projects/{projectId}/domains` with no cookie → `401` (the domain
  routes are live in TEST; a route that is not deployed answers `404`).

We also ran the session routes end-to-end in TEST with a real session. That run
checked: `/v1/me`; create, list, rename and delete a project; the
non-empty-project `409`; key format `sk_{8}_{64 hex}`; the secret is not in the
list; the key works for that project's uploads; a key from another project gets
`404`; a revoked key gets `401`; no session gets `401`; and the domain routes
(add a pending domain with its TXT record, verify stays `pending` without DNS,
remove gets `204`). The run aborts the open upload with the API key and asserts
that both project deletes answer `204`, so it leaves no video row and no R2
multipart upload behind.

The route behaviour for projects, API keys and SSO is also covered by our API
unit tests.
