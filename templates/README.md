# Sync workflow template

`sync-to-central-docs.yml` is meant to be copied into **satellite repos**
(not this one) — it pushes their `docs/` folder into this repo under
`docs/services/<repo-name>/` on every push to `main`.

## Install in a satellite repo

1. Copy this file to `.github/workflows/sync-to-central-docs.yml` in the
   satellite repo.
2. Create a fine-grained PAT (or org secret) named `DOCS_SYNC_PAT` with
   `contents: write` access to this `organization-docs` repo, and add it
   as a secret in the satellite repo (or as an org-level secret shared
   across repos).
3. Make sure the satellite repo has a `docs/` folder with at least an
   `index.md`.
4. Push to `main` — docs show up here at `docs/services/<repo-name>/`
   and in the nav automatically (via `mkdocs-awesome-pages-plugin`).
