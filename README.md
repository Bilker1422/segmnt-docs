# segmnt developer docs

Public developer documentation for segmnt, built and hosted on
[Mintlify](https://mintlify.com). Live at <https://segmnt.mintlify.site>.

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
| `docs.json` | Mintlify site config: theme, colours, logo, favicon, fonts, navigation, `noindex` for TEST, and the API reference (`api/openapi.yaml`). |
| `index.md`, `accounts-and-projects.md`, `direct-upload.md` | English guide pages. |
| `logo/` | segmnt logos (`segmnt-logo.svg` light, `segmnt-logo-dark.svg` dark). |
| `favicon.svg` | segmnt symbol mark. |
| `api/openapi.yaml` | Copy of the segmnt API contract (`api/openapi.yaml` in the private repo). |

English only. Arabic is out of scope for the Mintlify docs; the product stays
Arabic-first.

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
