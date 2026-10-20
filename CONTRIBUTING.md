# Docs style guide and contribution process

This file is for anyone who edits the customer docs. It is not a customer page.

The docs live in `mintlify-docs/` and are built by Mintlify at
https://docs.segmnt.cloud. A push to `main` mirrors this folder into the public
docs repository, and Mintlify builds it. Do not edit the public repository by
hand; a sync overwrites it.

## Run the docs locally

You need Node.js 24. From `mintlify-docs/`:

```bash
npx mintlify@4.2.1072 dev --no-open    # preview at http://localhost:3000
npx mintlify@4.2.1072 validate         # strict build check; must exit 0
npx mintlify@4.2.1072 broken-links     # checks internal links
```

`validate` and `broken-links` must pass before you open a pull request.

Use the version above, not `@latest`. CI runs this exact version, so a
different version can pass on your machine and fail in the build. The version
is bumped by hand in every file that names it, and the CI guard fails the
build if the files disagree.

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

Use the four labels from [Feature status](/status), and no others:

- **Available** — works today; follow the guide as written.
- **Beta** — works, may change, not fully verified, some plans cannot use it.
- **Coming soon** — not available yet; describe the plan, not a working feature.
- **Not planned** — we are not going to build this; do not tell the reader to
  wait for it or build on it.

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

## Truth check before you open a pull request

Run this short check on every change that touches a fact (a number, a limit, a
status, an endpoint, an error code). It takes a few minutes and it is what
keeps the docs from drifting away from the product again.

1. **Read the release status first.** The release-status list (the
   `release-status` document on YAH-876) is the only source of truth. If the
   list does not name the feature, do not describe it as working. A claim
   follows a release: nothing is described as shipped before the CTO marks it
   Available there.
2. **Match the status words.** Every feature you mention carries one of the
   four labels above, and the label matches the release status. If a page and
   [Feature status](/status) disagree, fix the page.
3. **Check each number against the code or the API spec.** Plan caps, limits,
   sizes, timeouts, token life, error codes: open the code (or the OpenAPI
   spec) and read the value. Do not copy a number from another page or an older
   draft.
4. **Check the API reference still matches.** If you touched an endpoint,
   re-check the count and the tables on [API overview](/api-overview) against
   the spec, and run the spec checks below.
5. **Check no inside words slipped in.** No internal environment names, no
   internal tool or component names, no storage-tier or server words. Write
   what the reader sees.
6. **Run the checks:**

   ```bash
   # Links (must report no broken links):
   npx mintlify@4.2.1072 broken-links

   # API spec, when you touched openapi.yaml or the overlay:
   npm run openapi:lint
   npm run openapi:overlay   # must build without errors
   ```

7. **Say how you verified it in the pull request.** One line per fact: where
   you read it (release status, code file, spec section). If you could not
   verify a fact, do not publish it — ask the CTO.

## Contribution process

1. Open an issue for the change. Say the page, the gap and the reader's job.
2. Branch from `main`. One branch, one page group.
3. Edit `mintlify-docs/` only.
4. Run `npx mintlify@4.2.1072 validate` and `npx mintlify@4.2.1072 broken-links`.
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
