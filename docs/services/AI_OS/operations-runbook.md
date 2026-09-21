# 0028 — Operations runbook / คู่มือปฏิบัติการ

Updated: 2026-09-20. Scope: run, rebuild, migrate, back up, restore and roll back
the Mindwave-AI stack (docker compose). No secret values belong here; every
command reads credentials from `.env` inside the container or from your shell
environment. Related: `Docs/plans/0027-microservice-feature-services.md`,
`Docs/ops/key-backup.md`.

## 1. Run modes / โหมดการรัน

| Mode | Command | Console | Feature plugins |
|---|---|---|---|
| Monolith (default) | `docker compose up -d` | `http://localhost:5173` (vite) / API `:8000` | mounted in-process in `kernel-server` |
| Services | `MOUNT_FEATURE_PLUGINS=0` in `.env`, then `docker compose --profile services up -d --build --force-recreate kernel-server web` and `docker compose --profile services up -d --build` | `http://localhost:8080` (gateway) | one container per service behind nginx |

Notes:

- `kernel-server` = C kernel + control plane in ONE container (they share a Unix
  control socket). It is "core" in both modes.
- In services mode set `VITE_PROXY_TARGET=http://gateway:80` in `.env` if the vite
  dev `web` container must reach plugin routes.
- Only the gateway publishes a port in services mode. `/internal/*` is 404 there.
- Regenerate gateway config after adding/changing a manifest `service` key:
  `python3 Core/Plugin/gen_gateway_conf.py` (test fails if stale).

## 2. Rebuild rules / เมื่อไหร่ต้อง rebuild

Two different mount models — this is the most common source of "I changed the
code but nothing happened".

| Component | How code gets in | After a change |
|---|---|---|
| `kernel-server` (core: `Server/`, `Core/Plugin/`, `Core/Kernel/`) | bind-mount `.:/work` | Python: restart/recreate the container (uvicorn is PID 1, no reload). C kernel: entrypoint runs cmake+make on every start, so recreate. Node plugin source: new jobs read it from disk, no restart. |
| `web` (vite dev) | bind-mount `./Web` | hot reload. Plugin UI: edit `Core/Plugin/feature/<n>/ui`, run `python3 Core/Plugin/sync_ui.py`, check with `--check`. |
| Service containers (catalogs, ethics, ..., clinical-safety) | **baked image** (`Docker/service.Dockerfile` COPYs `Server` and `Core/Plugin`) | `docker compose --profile services up -d --build <service>` — a plain restart runs the OLD code. |
| `gateway` | bind-mount `Web/dist` + generated conf | rebuild web (`npx vite build` in the `web` container) and `nginx -s reload` / restart gateway after conf regeneration. |

Rule of thumb: if the code path is served by a service container, rebuild the
image; if it is served by `kernel-server`, recreate the container. A change to
`Server/` that both core and a service import needs BOTH.

Web check (must pass before handing over UI work):
`docker compose exec -T web sh -lc 'cd /work/Web && npx tsc -b && npx vite build'`.

Plugin tests:
`docker compose exec -T kernel-server bash -lc 'cd /work && python3 -m unittest discover -s Core/Plugin/tests -p "test_*.py"'`.

## 3. Migrations and one-off scripts / ลำดับ migration และสคริปต์ครั้งเดียว

Order matters. Always run each script as a **dry run first**, read the counts,
then `--apply`. Run scripts in the CORE container only (`kernel-server`); they
need `PDPA_ENCRYPTION_KEY`, `BLIND_INDEX_PEPPER`, `CHANNEL_CREDENTIAL_KEY`, which
services never hold. Scripts print counts only, never values.

1. **Back up first** (section 5) — DBs and keys.
2. **Schema migrations**: applied automatically on every `kernel-server` start
   (`Docker/kernel-server-entrypoint.sh`). Manual/idempotent form:
   `docker compose exec -T kernel-server bash -lc 'cd /work && python -m Server.db.migrate'`.
   `.sql` and `.py` files run in filename order, one transaction each, tracked in
   `schema_migrations`. `.py` migrations execute code: review them as code.
3. **`Server.db.encrypt_credentials`** — encrypts legacy plaintext provider keys,
   DB-credential passwords, channel secrets and invite emails. Requires
   migration `0080` applied first.
4. **`Server.db.redact_history`** — emails in `control_events`, `request_activity`,
   `invites.invited_by` become opaque account ids / masked strings.
5. **`Server.db.redact_reviewer_labels`** — email-shaped reviewer/author labels in
   the user DB and `keyword_proposals`.

Pattern for 3-5 (replace `<module>`):

```
docker compose exec -T kernel-server bash -lc 'cd /work && python -m Server.db.<module>'          # dry run, default
docker compose exec -T kernel-server bash -lc 'cd /work && python -m Server.db.<module> --apply'  # only after reading counts
docker compose exec -T kernel-server bash -lc 'cd /work && python -m Server.db.<module>'          # re-run: expect zero
```

All three are idempotent; a second run must report zero changes. None runs
automatically. Do not run 3-5 while the matching key is missing or wrong: check
`/health` `keys.*` first (section 6).

## 4. Databases / ฐานข้อมูล

- `ai_core` (system DB, `DB_NAME`): control plane, console documents, jobs, audit.
- User DB (`USER_DB_NAME`, own host/user via `USER_DB_*` when set): all personal
  data (perma, memory, suicide assessments, chat content). Kept separate on purpose;
  do not point both at one restore.
- Postgres is on the host (`host.docker.internal`), not in compose (the
  `postgres` service is profile `legacy-db`, not started normally).

## 5. Backup and restore / สำรองและกู้คืน

Backup (before any migration, one-off script, or upgrade). Take both DBs; use the
host `pg_dump` with credentials from your shell, never paste them into commands
that get logged:

```
set -a; . ./.env; set +a                       # load into THIS shell only
export PGPASSWORD="$DB_PASSWORD"
pg_dump -h "${DB_HOST:-localhost}" -p "${DB_PORT:-5432}" -U "$DB_USER" \
  -Fc -f "backups/ai_core-$(date +%Y%m%d-%H%M).dump" "$DB_NAME"
# user DB uses its own connection settings when USER_DB_* are set
export PGPASSWORD="${USER_DB_PASSWORD:-$DB_PASSWORD}"
pg_dump -h "${USER_DB_HOST:-$DB_HOST}" -p "${USER_DB_PORT:-$DB_PORT}" \
  -U "${USER_DB_USER:-$DB_USER}" -Fc \
  -f "backups/user_db-$(date +%Y%m%d-%H%M).dump" "$USER_DB_NAME"
unset PGPASSWORD
```

- Store dumps encrypted and **separately from the keys**. A dump plus its keys in
  one place defeats the encryption; a dump without the keys cannot decrypt
  (identities, credentials, user content).
- **Keys**: back up `PDPA_ENCRYPTION_KEY`, `PDPA_CONTENT_ENCRYPTION_KEY`,
  `BLIND_INDEX_PEPPER`, `CHANNEL_CREDENTIAL_KEY` and `INTERNAL_TOKEN_*` per
  `Docs/ops/key-backup.md` (offline vault, per environment, re-back-up after
  rotation). Losing a key is unrecoverable regardless of DB backups.

Restore (throwaway environment first; do the drill quarterly):

1. Stop writers: `docker compose stop kernel-server` (and services if running).
2. Create an empty target DB (or drop/recreate after confirming you have a good dump).
3. `pg_restore -h ... -U ... -d <db> --no-owner --clean --if-exists <file>.dump`
   (same connection pattern as backup; user DB restored into the user DB only).
4. Put the ORIGINAL keys back into `.env` — same values as when the dump was taken.
5. `docker compose up -d kernel-server` (recreate so env is re-read; migrations
   re-apply idempotently).
6. Verify: `/health` reports db/kernel up and every `keys.*` entry `ok`, an operator
   can log in, a test job completes. Never restore only one of the two DBs from
   different points in time without noting the skew (audit rows vs personal data).

## 6. Health endpoints / จุดตรวจสุขภาพ

| Where | Endpoint | Meaning |
|---|---|---|
| core (`:8000`) | `GET /health` (needs a session) | `status`, `db`, `kernel`, registry count, `test_mode`, `lockdown`, `keys.*` canaries |
| gateway (`:8080`) | `GET /gateway-health` | nginx alive (plain `ok`), says nothing about upstreams |
| each service | `GET /health` inside the container | compose healthcheck (python urllib to `127.0.0.1:8000`) |

Useful: `docker compose ps` (health column), `docker compose logs --tail=100 <svc>`,
`GET /admin/feature-plugins` (mount state/errors in monolith; `route_count: 0` is
expected for external plugins), `GET /admin/queue/jobs` for job state.

## 7. Common failures seen here / ปัญหาที่เคยเจอ

| Symptom | Cause | Fix |
|---|---|---|
| Job fails with an opaque error / generic `error` | real exception is swallowed in the per-job process | reproduce inside the container, not from the host (host lacks the container's env/paths): call the same Node through `POST /admin/nodes/test` (`tools:write`) or import its `handler.py` from a `docker compose exec kernel-server python` shell with the failing inputs, and read the traceback; then check `GET /admin/queue/jobs/{id}/events`. `run.py` itself cannot be fed stdin: it talks to the kernel over a per-job Unix socket |
| Changed `.env` but behaviour unchanged | env is read at container create; `restart` does not re-read `env_file` | `docker compose up -d --force-recreate <svc>` (or `--profile services ...`). Do not recreate casually in production: it drops in-flight jobs |
| Changed service code, still old behaviour | services run a baked image | `--build` that service (section 2) |
| `/api/...` returns the SPA HTML or a wrong-service 404 | gateway `/api/` fallthrough: prefix stripped for routing, unmatched paths go to core, SPA owns only non-`/api` paths; stale generated conf | regenerate with `gen_gateway_conf.py`, reload gateway; call the real path (`/admin/...`) to compare |
| Console shows "not yet connected" | before 2026-09-20 every 404 was flattened; now only a route-missing 404 (no JSON `detail`, or `Not Found`) shows it | a 404 carrying a specific `detail` is the API's real answer; otherwise the route/plugin is not mounted (disabled plugin, service down, old image) |
| curl/healthcheck to `[::1]:8000` refused but `127.0.0.1:8000` works (or reverse) | server binds IPv4 only; `localhost` may resolve to `::1` first | use `127.0.0.1` explicitly in scripts and healthchecks |
| Login broken / identities unreadable after restore | wrong or missing `PDPA_ENCRYPTION_KEY` / `BLIND_INDEX_PEPPER` | restore original keys (key-backup.md), recreate core, check `/health` `keys.*` |
| Plugin change not visible in the web console | `Web/src/plugins` is a generated copy | `python3 Core/Plugin/sync_ui.py`, rebuild web |
| Service returns 503 for one plugin | its secret is unset (e.g. `CLONE_SERVICE_TOKEN`) or `INTERNAL_TOKEN_<SERVICE>` missing on core | set in `.env`, recreate core and that service |

## 8. Rollback to monolith / ย้อนกลับเป็น monolith

1. Remove `MOUNT_FEATURE_PLUGINS=0` from `.env` (or set it to `1`).
2. `docker compose up -d --force-recreate kernel-server web` — core mounts all
   feature plugins again; routes are identical (71 verified).
3. Point the web proxy back: remove `VITE_PROXY_TARGET` override (default
   `http://localhost:8000`); console at `:5173`/`:8000`.
4. Stop services: `docker compose --profile services stop` (or `down` for the
   profile's containers). Feature plugin enable/disable state
   (`console_documents.feature_plugins`) is shared, so it carries over unchanged.
5. Verify: `/health`, `GET /admin/feature-plugins` shows plugins mounted with
   `route_count > 0`, a kernel job runs.

No schema change is involved in switching modes, so rollback needs no DB restore.
A DB restore is only needed to undo section-3 scripts; `--apply` runs are not
reversible except from the pre-run backup.

## Priority tiers (Settings > Priority Slots)

Resource policies double as tiers, lowest to highest priority: **free (30), plus (20),
premium (10), dev (0)**; `standard` (0) is the legacy default. The Kernel runs the
LOWER level number first, so dev jobs are dispatched before premium, plus and free,
and can pause a running job whose policy is preemptible (free and plus are; premium and
dev are not). Choose a tier per deployment (`policy_name` at deploy) or per direct job.
Defaults are seeded by migration 0170 (never overwrites edits). Effect is only visible
when jobs queue: Kernel capacity (`max_concurrent`) is 1 by default, raise it in the
same page to run several at once. Equal-level jobs are not strictly FIFO (observed).

## Kernel build type and memory (measured 2026-09-21)

The dev container builds the C kernel with Debug + AddressSanitizer by default. ASan
keeps freed memory in a quarantine, so `mwkernel` RSS climbs about 45 KB per job (49 MB
after ~100 jobs, ~2x slower, address space ~28 TB). That is not a leak. On servers set
`KERNEL_BUILD_TYPE=Release` and `KERNEL_ASAN=OFF` in `.env` (read by the entrypoint).
Release, measured: RSS 4.0 -> 4.3 MB after 300 jobs, 1 thread, 8 fds, flat.

What a finished job leaves behind (checked after 30 and 300 jobs): no node process, no
per-job cgroup, no socket file, no zombie, no thread or fd growth; the Python server RSS
stays flat. Job memory is bounded by the tier's cgroup limit and freed when the process
exits. Long-lived caches have TTLs (settings 2 s, sessions per SERVICE_SESSION_CACHE_SECONDS,
plugin state 5 s) or bounded rings (hook delivery log, queues).

## Dev web container (port 5173) failed to start

Symptom: `docker compose ps` shows `web` Exited (137) with
`joining network namespace of container: No such container`, and http://localhost:5173
does not answer. Cause (fixed 2026-09-21): `web` used `network_mode: service:kernel-server`,
so recreating kernel-server alone orphaned it. `web` now has its own network and publishes
5173 itself; its proxy target is `VITE_PROXY_TARGET` (default `http://kernel-server:8000`,
set `http://gateway:80` in `.env` for the microservice stack). After any `docker compose`
change, `docker compose --profile services up -d` brings every service back.
