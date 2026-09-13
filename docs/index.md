# Organization Documentation

Central docs hub. Aggregates documentation pushed automatically from each
service repository in the organization.

## Services

See the [Services](services/) section in the nav for docs synced from each
repo. Each repo's `docs/` folder is mirrored here on every push to its
`main` branch.

## Adding a new repo to this hub

1. Add a `docs/` folder to your repo with at least an `index.md`.
2. Copy `templates/sync-to-central-docs.yml` from this repo into your
   repo's `.github/workflows/`.
3. Make sure the `DOCS_SYNC_PAT` secret is available to your repo
   (organization secret, or add it per-repo).
4. Push to `main` — your docs appear under `services/<your-repo-name>/`
   here automatically.
