# Deployment — high throughput on CPU (target: 10,000 users/hour)

vCPU24 / RAM24, no GPU. ~2.8 req/s average. Pure-CPU LLM generation cannot hit
that on unique generations alone — **the semantic cache is the capacity lever.**
Architecture is built around offloading repeat/near-duplicate questions to the
cache so the CPU only generates genuinely new answers.

## Request path

```
client ──► /api/chat/stream (FastAPI, async SSE)
              │
              ├─ semantic cache hit (e5-small, cosine ≥ threshold) ──► replay instantly (~10ms)
              │
              └─ miss ──► Celery queue (Redis) ──► worker ──► llama-server (GGUF, cont-batching)
                                                                  │
                                                          tokens ─┴─► Redis pub/sub ─► SSE ─► client
                                                          (answer stored back into cache)
```

Two inference backends exist. For scale, use **`llama_server`** (CPU-quantized
GGUF via llama.cpp continuous batching). The in-process `local` transformers
path is single-generation and far slower — do **not** use it as the default
under load. The Celery worker is a separate process and never loads the
in-process model, so the queued path only works with `llama_server` (the API
auto-routes `local` → `llama_server` for queued requests).

## One-time setup

1. Set keys (production): `API_KEY` and `ADMIN_API_KEY` (separate). See `.env`.
2. Boot: `docker compose up -d` (api + worker + redis + nginx).
3. Open `/admin`, paste the admin key.
4. Model Management → Repo ID `mradermacher/llama3.1-typhoon2-8b-instruct-GGUF`
   → Load Files → pick `…Q4_K_M.gguf` → Download.
5. Activate it → llama-server starts with the model.
6. Verify `GET /api/admin/llama-server/status` → `healthy: true`.

Model pick: Typhoon2-8B (Thai-tuned) Q4_K_M ≈ 4.9 GB weights. For lower latency
at some quality cost, use a 3B–4B GGUF.

## Control plane — env is the source of truth

There is **no settings.json**. Every default is seeded from `.env` at process
start and held in memory. To change the persistent baseline, **edit `.env` and
restart**. See `.env.example` for the full list of tunables (generation,
sampling, penalties, threads, ports, cache, providers).

The dashboard can still tweak things at **runtime** (Active Provider card,
Fine-tuning Parameters) — but those edits are **in-memory and revert to the env
base on restart**, and they apply to the **API process only**, not the separate
Celery worker (which has its own env-seeded copy). So:

- Single-process / non-queued chat (`/v1/chat/completions`): dashboard edits
  take effect immediately.
- Queued path (`/api/chat/stream` → worker): the worker uses the **env** values;
  to change them for the worker, set the env var and restart the worker.

Remote-provider **API keys are always env-driven** (`*_API_KEY` / `HF_TOKEN`),
never stored on disk.

## Throughput knobs

| Knob | Env | Default | Notes |
|------|-----|---------|-------|
| Decode slots | `LLAMA_SERVER_N_PARALLEL` | 6 | Concurrent in-flight generations. More slots = more concurrency but slower per-token and more KV-cache RAM. |
| Per-slot context | `LLAMA_SERVER_N_CTX` | 8192 | Total `-c` = n_ctx × n_parallel. 8192×6 = 49k tokens of KV cache — watch RAM. Lower to 4096 if tight. |
| HTTP threads | `LLAMA_SERVER_THREADS_HTTP` | 16 | Must be ≥ n_parallel. |
| Decode/batch threads | `LLAMA_SERVER_THREADS` / `_THREADS_BATCH` | =cores | Don't oversubscribe past physical cores. |
| Worker concurrency | compose `--concurrency` | 12 | ≥ n_parallel. Only handles cache misses. |
| Cache threshold | `CACHE_THRESHOLD` | 0.94 | Lower (~0.90) = more hits/aggressive reuse, higher = stricter. 0.94 separates multi-turn contexts sharing a last line. Watch hit-rate. |

## Watch the cache hit-rate

`GET /api/admin/cache/stats` → `{hits, misses, hit_rate, entries}`.

Hit-rate **is** the capacity multiplier. At 70% hit-rate the CPU only generates
30% of 2.8 req/s ≈ 0.85 gen/s — comfortable on an 8B Q4 with 6 slots. If
hit-rate is low and latency climbs: lower `CACHE_THRESHOLD` toward 0.88, or
shrink the model. If answers feel wrongly reused, raise the threshold.

## RAM budget (RAM24)

- 8B Q4_K_M weights ≈ 4.9 GB
- KV cache: ~scales with total context (n_ctx × n_parallel) — keep total ≤ ~16–24k unless you've measured headroom
- e5-small embedder ≈ 0.5 GB (loaded in the API process for the cache)
- Redis + OS overhead

If you hit OOM at model load, lower `LLAMA_SERVER_N_PARALLEL` or
`LLAMA_SERVER_N_CTX` first.
