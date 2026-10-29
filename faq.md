---
title: Common questions
description: Short answers to the questions developers ask most about the API, uploads, playback, webhooks and the free plan. Each answer links to the full guide.
icon: "circle-help"
---

Each answer is short. The link goes to the full guide. If a status word says
**Coming soon**, the feature is not live yet. See [Feature status](/status).

## Integration

### How do I delete a video through the API?

Call `DELETE /v1/projects/{projectId}/videos/{videoId}`. The answer is `204`. A
second call for the same video answers `404`. See
[Manage videos → Delete a video](/manage-videos#delete-a-video).

### Can I upload from a browser on my own domain?

Partly. The file part `PUT` works from your own domain, and the `ETag` header is
readable. The `create` and `complete` calls are API calls, and today they allow
our dashboard origin only. So keep your API key on your server and make those
two calls there. See [Direct upload → Browser](/direct-upload#browser).

### Do abandoned or deleted uploads count against the monthly new-upload limit?

An upload that you create and never complete does **not** count. A video that
you complete and then delete **does** count, in the month you created it. See
[Usage and plan limits → Limits](/usage-and-limits#3-limits).

### How is the webhook signing secret used?

Use it exactly as you get it. Read it as a UTF-8 string and keep the `whsec_`
prefix. Do not decode it, and do not remove the prefix. See
[Webhooks → Verify the signature](/webhooks#verify-the-signature).

### Does a bulk import change the moderation hold?

No. A bulk import has no separate rule: each video in the import is one upload.
A paid plan is never held for the free-plan review, which is **Coming soon**. See
[Moderation → Bulk imports](/moderation#bulk-imports) and
[Webhooks → Are videos held](/webhooks#are-videos-held-and-does-a-bulk-import-change-it).

### What can a `playback` key do?

Only one thing: create playback tokens, with `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens`. Any other route answers `403` with `insufficient_scope`. See
[Accounts, projects, API keys and domains → Key scopes](/accounts-and-projects#key-scopes-full-and-playback).

### Can I verify `localhost` or a staging host as a domain?

`localhost` cannot be verified. A staging host needs its own verified domain: a
real public host with a TXT record. See
[Accounts, projects, API keys and domains → Local and staging hosts](/accounts-and-projects#local-and-staging-hosts).

### How do I rotate a webhook signing secret?

Call the rotate route. It is API only today, so there is no dashboard button.
Update your server with the new secret before you call it, because the old secret
stops working right away. See
[Webhooks → Rotate the signing secret](/webhooks#rotate-the-signing-secret).

### Can I read my plan and usage with an API key?

Usage, yes. `GET /v1/projects/{projectId}/usage` accepts a project API key. Your
plan, `GET /v1/billing/plan`, needs your dashboard session cookie, not an API key.
See [Usage and plan limits](/usage-and-limits).

### Do public videos need a token?

Not yet. The per-video public mode is **Coming soon**, so every video needs a
playback token today. See
[Playback tokens → Public videos need no token](/playback-tokens#public-videos-need-no-token).

## Sign-in and sessions

### Why did my session end?

A session ends after 7 days without use, and always 30 days after sign-in.
After that, or after you sign out, the dashboard tells you **Your session
ended. Please sign in again.** and API calls with the old cookie answer
`401 unauthorized`. Nothing is lost: sign in again with Google or GitHub and
keep working. See
[Accounts, projects, API keys and domains → Your session](/accounts-and-projects#2-your-session).

## Free plan

### What are the free-plan limits?

The free plan gives you 2 hours stored, 2 hours of new uploads per month, 2,000
watched minutes per month, 1 project, 1 verified domain, and playback up to 720p.
Our logo shows on the player. See
[Get started → Free-plan limits](/getting-started#free-plan-limits).

### Can I sign up with my work email instead of Google or GitHub?

Not today. Sign-up uses Google or GitHub only. See
[Accounts, projects, API keys and domains → Sign in](/accounts-and-projects#1-sign-in-sso-only).

### Is phone verification required to upload?

No. Phone verification at sign-up is **Not planned**, so it does not block an
upload. A verified domain is the gate. See
[Phone verification](/phone-verification).

### Will my first videos be held for review?

Not today. The free-plan review hold is **Coming soon**. A paid plan is never held
for it. A team hold can affect any plan. See [Moderation → Free accounts: the first videos](/moderation#4-free-accounts-the-first-videos).

### What quality can I play?

The free plan plays up to 720p. Starter plays up to 1080p. 4K playback is **Coming
soon**. See [Usage and plan limits → Limits](/usage-and-limits#3-limits).

### Can I hide your logo on the player?

Not today. Hiding the logo (white-label) needs a paid plan, and paid plans are
**Coming soon**. See [Plans and payment](/billing).

### How do I embed a video on my site?

Copy the embed code, or include the player script. Put a signed playback token in
the link. See [Embed a video on any website](/embed-a-video).

### Does deleting a video free my stored hours?

Yes, but not at once. Stored hours go down shortly after you delete the video. The
delete does not reset your watched minutes, and the video still counts against
your new-upload limit for the month. See
[Manage videos → Delete a video](/manage-videos#delete-a-video).

### How are videos protected from being shared?

Each playback uses a signed token, and the video can be locked to your verified
domain. A copied link answers `403` on another domain. See
[How playback protection works](/playback-protection).

### What about prices and payment?

The free plan works today. Paid plans are **Coming soon**, and checkout is paused.
See [Plans and payment](/billing) for the plan list and the paid-plan whitelist.
