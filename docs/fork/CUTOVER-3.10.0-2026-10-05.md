# Cutover record — fork 3.5.0 → upstream mempalace 3.10.0

**Date:** 2026-10-05 · **Executed by:** Aura (session `20261005_125650_4f10dc`) · **Approved by:** Steve ("implement your plan ensuring to have a rollback facility")

Follows `docs/fork/MERGE-PLAN-3.10.0.md`. There was nothing to port; this is an upgrade + parity
verification. **No Hermes venv was modified** — the fork package in the Hermes venv is untouched and
still present, which is what makes the rollback a one-liner.

## What changed (exactly one line)

`~/.hermes/config.yaml` → `mcp_servers.mempalace.command`

| | |
|---|---|
| before | `/home/kraythorne/.hermes/hermes-agent/venv/bin/python3` (venv's fork 3.5.0) |
| after | `/home/kraythorne/.local/share/mempalace-310/venv/bin/python` (PyPI mempalace 3.10.0) |

Unchanged: `args: [-m, mempalace.mcp_server]`, `env.MEMPALACE_PALACE_PATH=/home/kraythorne/.hermes/palace`,
`connect_timeout: 30`, `timeout: 60`.

**No gateway restart was required** — contrary to MERGE-PLAN §6 Phase 4 step 1. The gateway reads
`mcp_servers` at **spawn time**, not only at boot: a mempalace server (pid 72445) came up at
`13:59:04` — the same second `config.yaml` was written — on the new venv
(`/home/kraythorne/.local/share/uv/python/cpython-3.11.15…`), with `MEMPALACE_PALACE_PATH` set and an
open fd on the live `chroma.sqlite3`. Sessions already running keep their existing server; every new
session gets 3.10.0. (A restart inside the gateway process is refused by design: SIGTERM would
propagate to the caller.)

## Artifacts

| | |
|---|---|
| Dedicated venv | `/home/kraythorne/.local/share/mempalace-310/venv` (CPython **3.11.15** via uv-managed interpreter — parity with the live server's 3.11) |
| Pinned source | git worktree at tag **`v3.10.0`** (`22fd87f`) → `/home/kraythorne/.local/share/mempalace-310/src` |
| Install source | **PyPI `mempalace==3.10.0`** — verified byte-identical to the tag tree (`diff -rq` → only `__pycache__` differs) |
| Palace copy used for every destructive test | `/home/kraythorne/.local/share/mempalace-310/palace-copy` |
| Rollback bundle | `/home/kraythorne/.hermes/backups/mempalace-upgrade-20261005/` |

### Rollback bundle contents

- `db-snapshots/palace-chroma.sqlite3` + `mempalace-kg.sqlite3` — `VACUUM INTO` snapshots, `PRAGMA quick_check = ok` on both live and snapshot
- `palace-dir-20261005.tgz` (41 entries), `dotmempalace-dir-20261005.tgz` (19 entries)
- `venv-mempalace-3.5.0-fork.tgz` (185 entries) — **the exact package that was live**, the real rollback artifact
- `venv-pip-freeze.txt` (288 lines), `config-mempalace-block.yaml`, `MANIFEST.txt` (sha256), `build-report.json`
- `rollback.sh` — `verify | package [target] | data | config | all`

`rollback.sh verify` was run and passed **before** anything live changed: it extracts the fork package
from the tarball, imports it under the venv interpreter (3.5.0, py 3.11.15) and `quick_check`s both DB
snapshots. Rollback is exercised, not merely documented.

## Evidence (all on copies or read-only against live)

1. **Tool surface** — MCP `tools/list`: fork 36, 3.10.0 45; **fork-only tools: ∅**. New: `event_*` (4), `artifact_get/put`, `task_create`, `mesh_peers`, `patch_submit`. `mempalace_kg_supersede` present in both.
2. **Data parity** — 3.10.0 on the palace copy: **2,270 drawers**, 19 wings, 93 rooms, wing map byte-identical to the fork's `/api` status; `repair-status`: sqlite 2,270 / HNSW 2,270, divergence 0, closets 76/76.
3. **No schema migration** — 41 schema objects both sides, 0 added/removed, 0 DDL text changes; only `acquire_write` 5605→5606 (lock audit append). 3.5.0 consumers (hooks, cron) can keep sharing the palace safely.
4. **Upstream's own tests** (from the tag worktree, 3.10.0 venv): locks ×5, knowledge_graph ×4, date_window/date_provenance/backfill_authored_at, non-regular-file guards, clean_nul_bytes, miner_fts5_validation, convo_miner_unit, format_miner, mcp_http_transport, `tests/mcp/**` → **874 passed, 2 skipped, 0 failed** (88.86s).
5. **Read-only surface** — `--read-only`: 24 tools, zero mutating tools exposed; direct dispatch of `mempalace_add_drawer` → refused `-32003` "Server is in read-only mode; this tool is disabled"; read tools still work. Verified against the **live palace** before cutover (live `quick_check` still ok afterwards).
6. **KG half-open temporal filter** — `supersede(..., at=T)` then `query_entity(as_of=T)` returns the **new** value (old already expired at the boundary) — half-open `valid_to > as_of` confirmed live on a KG copy.
7. **Provenance** — PyPI wheel == pinned tag tree; `knowledge_graph.supersede` at `knowledge_graph.py:375`, `config.strip_nul_bytes` at :68, `backends/_magic.has_sqlite_magic` at :114, `palace/palace_lock.py:123` `os.register_at_fork(_reset_palace_lock_state_after_fork)`, `mcp_server/_session.py:432 _supports_metadata_facets`.

## Behavioural delta to watch

- Upstream **removed both** sticky `_MCP_WRITER_LOCK_FAILED` short-circuits (superset). Re-verify the peer-writer path under real contention.
- New: `EmbedderIdentityUnknownWarning` — the palace had no recorded embedder identity; both fork and 3.10.0 resolve to `minilm`, so this was new *noise*, not a behaviour change. **Closed 2026-10-05:** `mempalace --palace ~/.hermes/palace palace set-embedder --model minilm` → `✓ recorded embedder identity: minilm (dim=384)`. That also confirms the stored vectors are all-MiniLM-L6-v2. The record stops a future silent model swap from going unnoticed.

## Rollback

```sh
BUNDLE=/home/kraythorne/.hermes/backups/mempalace-upgrade-20261005
"$BUNDLE/rollback.sh" config          # config command back to the Hermes venv python3
systemctl --user restart hermes-gateway.service
```
Package and data restore paths exist too (`rollback.sh package`, `rollback.sh data`, `rollback.sh all`).
The live venv was never modified, so `config` + restart is the whole rollback in the normal case.

## Dependency note

The production venv's interpreter is uv-managed CPython 3.11.15 (`~/.local/share/uv/python/…`). If that
interpreter is ever removed, the venv breaks loudly (the MCP server fails to start) — recoverable via
`rollback.sh config` or by rebuilding the venv from the pinned tag worktree.
