---
title: Phone verification
description: Phone verification is not planned. We do not ask for a phone number or an SMS code; a verified domain is the gate.
icon: "smartphone"
---

Phone verification is **Not planned**. We do not ask for a phone number or an
SMS code, at sign-up or at upload. A **verified domain** is the gate: verify one
domain before your videos play. See
[Accounts, projects, API keys and domains](/accounts-and-projects#5-domains).

This page stays up because older docs described a phone step. There is nothing
to build on your side, and nothing to wait for.

## What this means for you

- **Sign-up:** no phone number, no SMS code. Sign in with SSO and you are in.
- **Upload:** no phone step before the first video.
- **Trust:** keep the gate you already have — a verified domain, plus account
  and device bans for abuse. See [How playback protection works](/playback-protection).

## The old phone endpoints

An older version of the API had two phone endpoints
(`POST /v1/me/phone/start` and `POST /v1/me/phone/verify`). They are still in
the API for now, but there is **no SMS service behind them**: in production they
answer `503 sms_not_configured`. Do not build on them. They may be removed.

If you see an older guide or code sample that uses these endpoints, ignore it.

This guide is written in simple English.
