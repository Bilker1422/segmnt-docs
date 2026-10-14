# segmnt developer docs

Public developer documentation for segmnt, built and hosted on
[Mintlify](https://mintlify.com). Live at <https://docs.segmnt.cloud> — the
production domain the board confirmed on 14 Oct 2026 (YAH-530). The Mintlify
preview host is <https://segmnt.mintlify.site>.

**This repository is synced automatically.** The source of truth is the private
`Bilker1422/segmnt` repository, folder `mintlify-docs/`. The
`Sync docs to segmnt-docs` workflow there mirrors that folder into this repo on
every push to `main`. Do not edit files here by hand; the next sync overwrites
them. Make changes in `mintlify-docs/` in the private repo instead.

Mintlify builds this repo and serves the site, including the agent tools
(`/llms.txt`, `*.md` pages, `/mcp`, `/.well-known/mcp`).

## Files

| Path | What it is |
|---|---|
| `docs.json` | Mintlify site config: theme, colours, logo, favicon, fonts, navigation, and the API reference (`api/openapi.yaml`). |
| `index.md`, `accounts-and-projects.md`, `direct-upload.md`, `manage-videos.md`, `phone-verification.md`, `usage-and-limits.md`, `billing.md`, `reports-and-review.md`, `moderation.md` | English guide pages. This is the **source of truth** for the English docs. Every customer page that ships must land here, or CI fails. |
| `logo/` | segmnt logos (`segmnt-logo.svg` light, `segmnt-logo-dark.svg` dark). |
| `favicon.svg` | segmnt symbol mark. |
| `api/openapi.yaml` | Copy of the segmnt API contract (`api/openapi.yaml` in the private repo). |

English only. The board decided (YAH-530) not to host the Arabic docs, so the
English pages do not link to an Arabic docs site. Arabic is supported in the
product (UI and player), but Arabic is a feature, not the pitch.

## Run it locally

You need Node.js 24. From this folder:

```bash
npx mintlify@latest dev --no-open   # http://localhost:3000
npx mintlify@latest validate        # strict build check (exits on warnings/errors)
npx mintlify@latest broken-links    # checks internal links
```

`dev`, `validate` and `broken-links` work offline and do not need a Mintlify
login. The agent tools only exist on the hosted site, so they return 404 from
the local preview.

The JSON schema for `docs.json` is at <https://mintlify.com/docs.json>.
