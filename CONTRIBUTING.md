# Docs style guide and contribution process

This file is for anyone who edits the customer docs. It is not a customer page.

The docs live in `mintlify-docs/` and are built by Mintlify at
https://docs.segmnt.cloud. A push to `main` mirrors this folder into the public
docs repository, and Mintlify builds it. Do not edit the public repository by
hand; a sync overwrites it.

## Run the docs locally

You need Node.js 24. From `mintlify-docs/`:

```bash
npx mintlify@latest dev --no-open    # preview at http://localhost:3000
npx mintlify@latest validate         # strict build check; must exit 0
npx mintlify@latest broken-links     # checks internal links
```

`validate` and `broken-links` must pass before you open a pull request.

## The four page types (Diátaxis)

Pick the type first. One page has one type. Do not mix them.

| Type | Job for the reader | Example |
| --- | --- | --- |
| **Tutorial** | Learn by doing, start to finish | `quickstart.md` |
| **How-to** | Do one task | `direct-upload.md` |
| **Reference** | Look up a fact | `errors.md`, `api-overview.md` |
| **Explanation** | Understand a concept | `playback-protection.md` |

- A tutorial is a lesson: a single path, numbered steps, no choices.
- A how-to is a recipe: it assumes some knowledge and gives the steps for one job.
- Reference is dry and complete. No story.
- Explanation answers "why" and "how it fits together".

## Rules that override everything

1. **Document only what is released.** The release status is the source of
   truth. If the release status does not list a feature, do not write it as
   working.
2. **Never write a shipped-sounding sentence for an unshipped feature.** Use
   the status labels below.
3. **Never name our internal environments, internal tools, storage tiers or
   server components on a page.** Write what the reader sees. To show
   verification, write "tested against the current release".
4. **Every code sample must run.** If you cannot run it, label it **unverified**;
   do not publish it as fact.
5. **Never put a secret on a page.** No API keys, no tokens, no customer data.
   Use `sk_...` and `YOUR_PLAYBACK_TOKEN` placeholders.
6. **Do not write marketing copy, prices or legal text.** Route those to Product
   Marketing or the CEO.

## Status labels

Use the three labels from [Feature status](/status), and no others:

- **Available** — works today; follow the guide as written.
- **Beta** — works, may change, not fully verified, some plans cannot use it.
- **Coming soon** — not available yet; describe the plan, not a working feature.

A page with no label is **Available**.

## Language

Write in simple English (close to ASD-STE100):

- One idea per sentence. Short sentences.
- Active voice. "We encode the video", not "the video is encoded".
- Talk to the reader as **you**.
- Define a term the first time you use it. Link to the [Glossary](/glossary).
- One word, one meaning. Use the same word for the same thing every time.
- Name a page after the job the reader has, not after our feature name.

## Page shape

Start each page with frontmatter:

```markdown
---
title: Embed a video on any website
description: Copy the embed code, replace the token, paste it into your page. One sentence.
icon: "code"
---

One short paragraph: what this page does and who it is for.
```

- `description` is one sentence and shows in search and link previews.
- Put the three pages a new developer needs first in the navigation
  (`docs.json`): Get started, then Integrate.
- Link with site-relative paths and no trailing slash: `/quickstart`,
  `/errors`. Do not hard-code `https://docs.segmnt.cloud` in an internal link.

## Navigation

`docs.json` holds the navigation, in five groups, in this order:

1. **Get started** — welcome, quickstart, setup, status.
2. **Integrate** — upload, embed, playback tokens, WordPress.
3. **Concepts** — playback protection, glossary.
4. **Account & billing** — accounts, phone, usage, billing.
5. **Trust & safety** — moderation, reports and review.

The **API reference** tab is generated from `api/openapi.yaml`. Keep the
documentation copy byte-identical to `api/openapi.yaml`; CI checks this with
`diff`.

## Contribution process

1. Open an issue for the change. Say the page, the gap and the reader's job.
2. Branch from `main`. One branch, one page group.
3. Edit `mintlify-docs/` only.
4. Run `npx mintlify@latest validate` and `npx mintlify@latest broken-links`.
   Both must pass.
5. Open a pull request. Put in the description:
   - the preview link,
   - the before/after list for the pages you touched,
   - how you checked the code samples.
6. Review:
   - the **CTO** checks facts and release status,
   - the **Docs Writer** checks language, structure and links.
7. Merge to `main` when both reviews pass. Mintlify rebuilds the site.

## Definition of done

- The page has the right type, the right title, and one job.
- Every claim matches the release status.
- Every code sample runs, or is labelled unverified.
- `validate` and `broken-links` pass.
- The preview link is in the pull request.

## Handover

The Docs Writer owns day-to-day maintenance after this contract. This file and
`docs.json` are the two files to read first.
