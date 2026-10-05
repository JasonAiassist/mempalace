# Merge plan: our MemPalace fork → upstream 3.10.0

**TL;DR — there is nothing to port.** Our customisations are already in upstream 3.10.0. This is an
**upgrade + parity verification**, not a merge. Hand-porting effort: **zero files**; every "residual"
turned out to be reformatting or an upstream superset.

---

## 1. What we actually have

| | |
|---|---|
| Fork base (true ancestor) | upstream **`4291fecf`** (2026-06-22) — verified: **65 of 65** files we never touched are byte-identical to that commit |
| Our delta vs that base | **30 files, +3,375 / −346** (29 source + `version.py` 3.4.1→3.5.0) |
| Install origin | `file:///tmp/mempalace-fork`, installed 2026-07-31 — **the directory is gone** (cleared by the 2026-10-04 reboots) |
| Version-controlled as | branch **`custom/3.5.0-installed`** (commit `ab9eb3d`) + `3.5.0-vs-installed.patch` + `installed-manifest.sha256` |

Our fork's real base is *not* the PyPI 3.5.0 wheel: the wheel contains upstream commits our fork never
had, so a wheel-vs-fork diff shows phantom "missing" code. All comparison here is against `4291fecf`.

## 2. How the verdict was reached (reproducible)

1. **Dry-run port**: our 23-file patch vs a real `v3.10.0` tree → 2 CLEAN / 17 "conflict" / 4 file-deleted.
2. **Symbol sweep**: 86 functions/classes added by our fork — 68 present by name in 3.10.0.
3. **Body comparison** (difflib on extracted source): 100% identical for `_startup_integrity_size_limit_bytes`,
   `_http_origin_allowed`, `_reset_palace_lock_state_after_fork`, `build_where_filter`,
   `_load_or_create_server_token`; 82/74/71/70% for the rest (gaps = upstream's added parameters/logging).
4. **Residual check**: every flagged item resolved — see §5.
5. **Tool-name set difference** (`mempalace_*` in our monolith vs the 3.10.0 package) = **∅** — no local-only tool.

## 3. Parity matrix — our change → 3.10.0 home

| Our change (fork) | 3.10.0 home | Status |
|---|---|---|
| MCP HTTP/TLS, allowed-host + origin checks, server token, `--read-only`, startup-integrity gate, writer-lock self-heal, sqlite-integrity payload, facets fast path, `kg_supersede` tool, `list_drawers` date window, log filter | `mcp_server/{http,_guards,protocol,_logging,_session,tools_read,tools_write,tools_kg,schemas}.py`, `mcp_proxy.py`, `mcp_light_server.py`, `date_window.py` | **absorbed** (relocated; upstream superset on writer-lock) |
| CLI `serve` (+ token helpers), `hallways`, `repair rebuild-index` gate, parser wiring | `cli/{cmd_serve,cmd_query,cmd_repair,parser}.py`, `server_registry.py` | **absorbed**, evolved (token path via `server_registry`; autoheal via preflight helper) |
| Palace-lock re-entrancy (process-wide) + at-fork reset; `prefetch_mined_set` dict; `bulk_check_mined` removed | `palace/{palace_lock,mined}.py` | **absorbed verbatim** (+ upstream `chunk_total` grouping) |
| `authored_at` tie-break + field on all result paths | `searcher/{ranking,sqlite_bm25,query}.py` | **absorbed** (+ explicit `authored_at_source`) |
| `facet_counts` (qdrant), `get_all_metadata` + `with_document` projection (pgvector), NUL stripping, SQLite magic detect, HNSW marker intact | `backends/{qdrant,pgvector,chroma,sqlite_exact,base,embedding_wrapper,_magic}.py` | **absorbed** (identical or superset) |
| No-follow file helpers, unique backup path, isolated-FTS5 autoheal | `repair.py` | **absorbed**; upstream richer (`_fts5_content_census`, dry-run, rollback) |
| Regular-file guards, size caps, mtime re-mine skip, hallways safe-wrap, `.tex`/`.bib` | `convo_miner.py`, `miner.py`, `normalize.py`, `entity_detector.py` | **absorbed** (+ content-hash dedup) |
| KG `supersede()`, half-open temporal filter (`valid_to > as_of`) | `knowledge_graph.py:375`, `:111` | **absorbed verbatim** |
| Hook transcript-path validation, safe wing slug, `CREATE_NO_WINDOW` | `hooks_cli.py`, `daemon.py` | **absorbed** |
| `strip_nul_bytes`; `authored:` layer render | `config.py:68`; `layers.py:394` | **absorbed** |
| `entities.py` | `entities.py` | **byte-identical** |

## 4. Why the earlier triage said "MANUAL / conflict"

`fork-vs-3.10.0-status.txt` counts exact-line matches and runs a textual 3-way merge. Upstream
reformatted files and re-homed code into packages, so identical functions read as "conflict" and its
"added lines already in 3.10.0" column is wrong for at least knowledge_graph (109/109), repair, chroma,
qdrant, pgvector, sqlite_exact, cli and palace. Treat that file's verdict column as superseded by §3.

## 5. Residual lines (verified — nothing to port)

86 of our added lines are not byte-present upstream: 16 are comments/docstrings; the rest are superseded
variants —

- `os.O_RDONLY | O_NOFOLLOW` → upstream adds `O_NONBLOCK` + `EAGAIN` retry
- `_read_text_no_follow() -> Optional[str]` → upstream returns `(content, mtime)`
- `shutil.copytree(..., symlinks=True)` → `copy_palace_dir(..., symlinks=True, log=print)`
- `expanduser(DEFAULT_PALACE_PATH)` → config-dir default refactor
- error-message wording (`normalize.py`)

Two flagged and cleared: chroma `_collection_has_sync_threshold_metadata` (identical, `chroma.py:1049`);
sqlite_exact `_rows` (upstream **superset** with `_IncludeSpec`, `sqlite_exact.py:789`).

**Behavioural note (only one):** upstream removed **both** sticky `_MCP_WRITER_LOCK_FAILED`
short-circuits; our fork kept one. Upstream is a strict superset — adopt it, then re-verify the
peer-writer path.

## 6. What to do

**Phase 1 — do not replay the patch.** Applying `3.5.0-vs-installed.patch` onto 3.10.0 risks
double definitions (`convo_miner._is_unchanged_since_last_mine`, `_compute_hallways_for_wing_safe`) and
would downgrade `_read_text_no_follow` from `(content, mtime)`, regressing re-mine safety.

**Phase 2 — rehearse away from the live install**

```sh
uv venv /tmp/mp310 && /tmp/mp310/bin/pip install mempalace==3.10.0
MEMPALACE_PALACE_PATH=~/.hermes/palace /tmp/mp310/bin/python -m mempalace.cli --help
```

**Phase 3 — parity checklist** (each maps to a row in §3)

- [ ] `mempalace_*` tool set identical to our fork's (expect ∅ diff)
- [ ] `kg_supersede` callable; temporal filter half-open
- [ ] palace lock re-entrant in-process, survives fork
- [ ] peer-writer refusal fires (second writer refused)
- [ ] read-only mode refuses mutating tools with the -32003 code
- [ ] `--read-only`, `--tls-cert/--tls-key`, `--token`, non-loopback-without-token guard
- [ ] facets fast path (chroma); `facet_counts` (qdrant/pgvector)
- [ ] SQLite magic detect on a truncated DB; `strip_nul_bytes` on a NUL-bearing doc
- [ ] `repair --dry-run rebuild-index` gate + FTS5 autoheal decision
- [ ] `list_drawers --since/--before`; `authored_at` + `authored_at_source` in results
- [ ] hooks: transcript path validation, wing slug, no console window (Windows)
- [ ] `.tex`/`.bib` mined; transcript re-mine triggers on mtime change

**Phase 4 — cut over (rollback ready)**

1. stop the MCP server / gateway
2. back up `~/.hermes/palace` and `~/.mempalace/knowledge_graph.sqlite3`; `PRAGMA quick_check`
3. install 3.10.0 pinned into a dedicated venv (or the Hermes venv), point `config.yaml`'s mcp command at it
4. restart; run the Phase-3 checklist + `mempalace repair --dry-run`
5. rollback = reinstall the fork from `custom/3.5.0-installed` using the README recipe

**Phase 5 — re-baseline the repo.** Track `upstream/develop`; drop the stale local `develop`
(`a20d769`, 559 commits behind the tag). For post-tag work target `43122ae` and re-verify the files
that changed after `v3.10.0`: `convo_miner`, `miner`, `repair`, `chroma`, `sqlite_exact`,
`cli/parser`, `cli/cmd_repair`, `cli/cmd_query`, `palace/mined.py`, `knowledge_graph`, `hooks_cli`,
`llm_client`, `config`, `backends/embedding_wrapper`.

## 7. Do not

- **Don't `pip install -U mempalace`** into the live venv without the Phase-4 backup — dist-info says
  `3.5.0`, so the upgrade looks routine and would silently replace the fork.
- **Don't use the local `develop` branch as "upstream"** — it is the fork's stale branch, not the
  MemPalace line. `upstream/develop` is the real one.
- **Don't replay the fork patch onto 3.10.0** (see Phase 1).

## Appendix — evidence

- `3.5.0-vs-installed.patch` (portable patch), `installed-manifest.sha256`, `fork-vs-3.10.0-status.txt`
- Raw analysis JSON: `~/.hermes/cache/scratch/mergeplan-{dryrun,absorbed,body-match,gone-map,residuals}.json`
- Base hunt: `~/.hermes/cache/scratch/mp-base-hunt.json`
