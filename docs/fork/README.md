# Our local MemPalace customisations (3.5.0 lineage)

**Branch:** `custom/3.5.0-installed` — cut from tag `v3.5.0`, contains our custom build exactly
as it was running on this box.

## What this is

The custom fork build that this server has been running since **2026-07-31**, captured so it is
version-controlled instead of existing only inside a virtualenv — or inside a `/tmp` directory
that no longer exists.

## Provenance

- Live module: `~/.hermes/hermes-agent/venv/lib/python3.11/site-packages/mempalace`
  (the Hermes MCP server runs `<venv>/bin/python3 -m mempalace.mcp_server` from this venv)
- dist-info: `mempalace-3.5.0.dist-info`; `direct_url.json` →
  `{"url":"file:///tmp/mempalace-fork","dir_info":{}}`
- Installed: 2026-07-31
- **`/tmp/mempalace-fork` no longer exists** — cleared by the 2026-10-04 reboots. The installed
  module was the last live copy of these customisations. The GitHub fork's `develop` held only
  upstream lineage (head = upstream PR #1960), not our work.

## Recipe (reproducible)

```sh
git checkout v3.5.0
patch -p1 -i docs/fork/3.5.0-vs-installed.patch      # 117 hunks / 23 files, 0 fuzz
# plus the two local-only additions:
#   mempalace/entities.py
#   mempalace/graphify-out/      (5 graphify metadata files)
```

**Verified:** the result is byte-identical to the installed module, excluding `__pycache__`.
95 files. Per-file hashes in `installed-manifest.sha256`.

## Change inventory (23 files, +1,934 / −297 lines)

| file | +/- | file | +/- |
|---|---|---|---|
| `mcp_server.py` | +658 / −85 | `hooks_cli.py` | +35 / −9 |
| `cli.py` | +226 / −1 | `embedding_wrapper.py` | +24 / −0 |
| `convo_miner.py` | +217 / −49 | `normalize.py` | +19 / −7 |
| `repair.py` | +162 / −? | `config.py` | +17 / −1 |
| `knowledge_graph.py` | +129 / −2 | `sqlite_exact.py` | +17 / −1 |
| `palace.py` | +100 / −74 | `daemon.py` | +16 / −1 |
| `chroma.py` | +93 / −26 | `searcher.py` | +15 / −? |
| `qdrant.py` | +62 / −0 | `base.py` | +9 / −0 |
| `miner.py` | +59 / −6 | `layers.py`, `entity_detector.py`, `llm_client.py`, `migrate.py`, `service.py`, `pgvector.py` | small |

Full per-file +/- table: `fork-vs-3.10.0-status.txt`.

## Status vs upstream

Upstream is at **3.10.0** (latest on PyPI as of 2026-10-05; upstream `develop` is 53 commits past
that tag). 3.10.0 is a *restructure*: `cli.py`, `mcp_server.py`, `palace.py` and `searcher.py` are
**deleted**, replaced by the `cli/`, `mcp_server/`, `palace/` and `searcher/` packages; new modules
include `backends/milvus.py`, `backends/rust_exact.py`, `hlc.py`, `hub_client.py`,
`replica.py`, `logsync.py`, `integrations/hermes/`.

Port triage of our 23 files against 3.10.0: **7 READY / 12 MANUAL / 4 PORT** — details in
`fork-vs-3.10.0-status.txt`.

## ⚠️ Do not

`pip install -U mempalace` — dist-info reports `3.5.0`, so the upgrade looks routine, but it would
overwrite the 23 customised files with upstream 3.10.0 and destroy this branch's content on disk.
