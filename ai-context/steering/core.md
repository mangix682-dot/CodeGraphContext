---
inclusion: always
---

# CodeGraphContext — Core Steering

> Primary instructions for any AI agent working in this repository. Read this **first**, then any other files under `ai-context/steering/`. Treat this as the single source of truth for project intent, architecture, and conventions. When in doubt, prefer this doc over assumptions; verify against source before changing behavior.

---

## 1. Project Identity

- **Name:** `codegraphcontext` (CLI: `cgc` / `codegraphcontext`)
- **Version:** `0.4.9` (see `@e:/claude-hub/CodeGraphContext/pyproject.toml:3`)
- **What it is:** An **MCP server** + **CLI toolkit** that parses local code repositories with Tree-sitter (and optional SCIP), persists a structural graph in a graph database (KuzuDB / FalkorDB / Neo4j / Nornic / LadybugDB), and exposes ~21 MCP tools and 55+ CLI commands so AI assistants and developers can query call chains, inheritance, dead code, complexity, etc.
- **License:** MIT
- **Python:** `>=3.10`, tested through 3.14 (Tree-sitter pinned out for 3.13)
- **Authoritative deep-dive:** `@e:/claude-hub/CodeGraphContext/ARCHITECTURE.md:1-50` (1500-line spec — read sections relevant to your task)

---

## 2. Tech Stack

- **Language:** Python (`src/`, package layout, `setuptools` build)
- **CLI:** `typer` + `rich` + `inquirerpy`
- **Parsing:** `tree-sitter` + `tree-sitter-language-pack` (20 languages) and optional **SCIP** indexers (`scip-clang`, `scip-dotnet`, `scip-python`, etc.)
- **Graph DB drivers:** `neo4j`, `falkordb` / `falkordblite`, `kuzu`, custom Nornic + Ladybug
- **MCP transport:** JSON-RPC 2.0 over stdio (no extra framework)
- **File watching:** `watchdog`
- **Viz:** FastAPI + `uvicorn` + Vite/React SPA in `@e:/claude-hub/CodeGraphContext/website` (built artifact synced to `src/codegraphcontext/viz/dist/`)
- **Testing:** `pytest`, `pytest-asyncio`, `typer.testing.CliRunner`
- **Formatting:** `black` (declared in `[project.optional-dependencies].dev`)

Always honor version pins in `@e:/claude-hub/CodeGraphContext/pyproject.toml:17-46` and conditional markers (`python_version != '3.13'`, `sys_platform != 'win32'`).

---

## 3. Repository Layout

```
e:/claude-hub/CodeGraphContext/
├── src/codegraphcontext/      # All runtime code (src layout)
│   ├── __main__.py            # `python -m codegraphcontext` entry
│   ├── server.py              # MCP JSON-RPC server (entry: MCPServer)
│   ├── tool_definitions.py    # 21 MCP tool schemas
│   ├── prompts.py             # LLM_SYSTEM_PROMPT shipped to clients
│   ├── cli/                   # Typer app (main.py is the root)
│   │   ├── main.py            # All `cgc …` commands
│   │   ├── cli_helpers.py     # Shared init + indexing progress
│   │   ├── config_manager.py  # ~/.codegraphcontext config + contexts
│   │   ├── setup_wizard.py    # Interactive Neo4j + MCP IDE setup
│   │   └── registry_commands.py
│   ├── core/                  # Database backends + watcher + jobs
│   │   ├── __init__.py        # get_database_manager() factory
│   │   ├── database*.py       # Neo4j / FalkorDB / Kuzu / Nornic / Ladybug
│   │   ├── watcher.py         # watchdog-based CodeWatcher
│   │   ├── jobs.py            # In-memory JobManager
│   │   ├── cgc_bundle.py      # .cgc bundle export/import
│   │   └── bundle_registry.py
│   ├── tools/
│   │   ├── graph_builder.py   # Indexing orchestrator
│   │   ├── code_finder.py     # 30+ Cypher query methods
│   │   ├── handlers/          # Per-category MCP handlers
│   │   ├── indexing/
│   │   │   ├── pipeline.py    # Tree-sitter full-repo flow
│   │   │   ├── scip_pipeline.py
│   │   │   ├── persistence/writer.py  # GraphWriter (all MERGE/CREATE)
│   │   │   ├── resolution/    # call + inheritance resolvers
│   │   │   └── schema_contract.py     # Canonical labels + rel types
│   │   ├── languages/         # 20 Tree-sitter parsers (one file per lang)
│   │   ├── query_tool_languages/
│   │   ├── scip_indexer.py + scip_pb2.py
│   │   └── package_resolver.py
│   ├── utils/                 # debug_log, path_ignore, tree_sitter_manager
│   └── viz/dist/              # Synced Vite build (generated)
├── tests/                     # unit / integration / e2e / perf
├── docs/                      # MkDocs site + reference docs
├── website/                   # Vite + React SPA
├── extensions/                # IDE integrations (VS Code, etc.)
├── k8s/, scripts/, organizer/
├── ARCHITECTURE.md            # 1500-line authoritative architecture
├── README.md                  # Public-facing intro
├── TESTING.md                 # Testing strategy (canonical)
├── CONTRIBUTING.md
└── pyproject.toml
```

**Rule:** All importable code lives under `src/codegraphcontext/…`. Do **not** create a top-level `codegraphcontext/` package — the build is configured for the `src/` layout (`@e:/claude-hub/CodeGraphContext/pyproject.toml:61-70`).

---

## 4. Architectural Overview

```
AI IDE / Terminal / Browser
        │
        ▼
┌──────────────────────────────────────────────────────┐
│ Entrypoints                                          │
│  • MCPServer (server.py)   — stdio JSON-RPC          │
│  • Typer CLI (cli/main.py) — `cgc …`                 │
│  • FastAPI viz server      — local browser viz       │
└─────────────────┬────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────────────────┐
│ Handlers Layer (tools/handlers/*.py)                 │
│ Dispatches MCP/CLI calls to core engines             │
└─────────────────┬────────────────────────────────────┘
                  │
   ┌──────────────┼──────────────────────┐
   ▼              ▼                      ▼
GraphBuilder   CodeFinder            CodeWatcher / JobManager
(indexing)     (querying)            (live updates / async jobs)
   │              │                      │
   ▼              ▼                      ▼
┌──────────────────────────────────────────────────────┐
│ Parsing: Tree-sitter (20 langs) + optional SCIP      │
│ Persistence: GraphWriter → DriverWrapper             │
└─────────────────┬────────────────────────────────────┘
                  ▼
       get_database_manager() picks one of:
       KuzuDB | FalkorDB Lite | FalkorDB Remote | Neo4j | Nornic | LadybugDB
```

Backend selection priority (`@e:/claude-hub/CodeGraphContext/src/codegraphcontext/core/__init__.py:63-184`):

1. Runtime override `CGC_RUNTIME_DB_TYPE` (CLI `--database` flag, MCP context)
2. Configured default `DEFAULT_DATABASE` (set via `cgc config db <backend>`)
3. Implicit auto-detect: `FALKORDB_HOST` → remote; else Unix+Py3.12+ → FalkorDB Lite; else Kuzu; else Neo4j; else Nornic
4. **Default on Windows:** KuzuDB (FalkorDB Lite is Unix-only)

All backends implement a uniform interface: `get_driver()`, `close_driver()`, `is_connected()`, `session()`, with wrappers (`DriverWrapper`, `SessionWrapper`, `RecordWrapper`, `ResultWrapper`) that normalize result access across engines. **Never bypass these wrappers** when adding new query code.

---

## 5. Graph Schema Contract

Source of truth: `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/schema_contract.py:9-69` and `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/prompts.py:57-99`.

**Canonical node labels:** `Repository`, `Directory`, `File`, `Function`, `Class`, `Trait`, `Variable`, `Interface`, `Macro`, `Struct`, `Enum`, `Union`, `Record`, `Property`, `Annotation`, `Module`, `Parameter`, `MavenModule`, `GradleModule`, `ExternalLibrary`, `Datasource`, `DbTable`, `DbColumn`, `RedisKeyPattern`.

**Canonical relationship types:** `CONTAINS`, `CALLS`, `IMPORTS`, `INHERITS`, `HAS_PARAMETER`, `INCLUDES`, `IMPLEMENTS`, `INJECTS`, `EXPOSES_ENDPOINT`, `PROVIDES_BEAN`, `MODULE_DEPENDS_ON`, `USES_LIBRARY`, `CHILD_MODULE`, `FILE_BELONGS_TO`, `READS`, `WRITES`, `MAPS_TO`, `HAS_COLUMN`, `STORED_IN`.

**Merge keys (identity):**
- `Function` / `Class`: `(name, path, line_number)`
- `File` / `Repository` / `Directory`: `(path,)` — always the **absolute** path

**Critical conventions:**
- `path` is always an absolute filesystem path on `Function`, `Class`, `File`. Use `path`, not `relative_path`, in Cypher queries.
- Workspace prefix `/workspace/` is stripped from outgoing MCP responses by `_strip_workspace_prefix` (`@e:/claude-hub/CodeGraphContext/src/codegraphcontext/server.py:69-78`).
- Any new node label or relationship type **must** be added to `schema_contract.py`; tests assert that writers stay within this set.

---

## 6. Indexing Pipeline (Reading Order)

When debugging or extending indexing, walk the code in this order:

1. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/graph_builder.py` — `GraphBuilder.build_graph_from_path_async`
2. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/pipeline.py` — Tree-sitter orchestrator
3. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/discovery.py` — file walking + `.cgcignore`
4. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/pre_scan.py` — import-map pre-pass
5. `tools/languages/<lang>.py` — per-language `parse(path, is_dependency, **kwargs) -> Dict`
6. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/persistence/writer.py` — `GraphWriter` (all MERGE/CREATE Cypher)
7. `tools/indexing/resolution/{calls.py,inheritance.py}` — second-pass relationship resolution
8. `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/scip_pipeline.py` — SCIP path (only when `SCIP_INDEXER=true`)

**Rule:** Parsers must not write to the database; only return dicts. All persistence flows through `GraphWriter`. All Cypher MERGE keys must match `schema_contract.py`.

---

## 7. Adding a New Language Parser

1. Create `src/codegraphcontext/tools/languages/<lang>.py` with class `<Lang>TreeSitterParser` exposing `parse(path, is_dependency, **kwargs) -> Dict` and module-level `pre_scan_<lang>(...)`.
2. Register the extension(s) in `tools/graph_builder.py` (parser dispatch table).
3. Mirror existing parsers (e.g. `python.py`, `go.py`) for output shape: `functions`, `classes`, `imports`, `module_name`, language-specific arrays.
4. Add a unit test in `tests/unit/parsers/test_<lang>_parser.py` using `get_tree_sitter_manager()` and a small fixture string. Follow the pattern in `@e:/claude-hub/CodeGraphContext/TESTING.md:75-92`.
5. Optional: add SCIP support in `tools/indexing/scip_pipeline.py` if a `scip-<lang>` binary exists.

Never duplicate cross-cutting logic — reuse helpers from `tree_sitter_manager.py` and `path_ignore.py`.

---

## 8. Adding a New MCP Tool

1. Define schema in `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tool_definitions.py` (JSON-Schema input).
2. Implement handler in the right module under `src/codegraphcontext/tools/handlers/` (`analysis_handlers.py`, `indexing_handlers.py`, `management_handlers.py`, `query_handlers.py`, `watcher_handlers.py`).
3. Register the tool in `MCPServer._init_tools()` (`server.py`).
4. Update the LLM tool manifest in `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/prompts.py:45-56`.
5. Add an integration test under `tests/integration/mcp/`.
6. Document it in `@e:/claude-hub/CodeGraphContext/docs/MCP_TOOLS.md`.

Handlers must be **synchronous functions** — the server wraps them with `asyncio.to_thread`. Return JSON-serializable dicts; let `_strip_workspace_prefix` handle path scrubbing.

---

## 9. Adding a New CLI Command

1. Add the Typer command in `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/cli/main.py` under the appropriate sub-app (`mcp`, `analyze`, `find`, `bundle`, `registry`, `config`, `context`, `neo4j`).
2. Push business logic into `cli/cli_helpers.py` (`*_helper` functions) — keep `main.py` thin.
3. Update `@e:/claude-hub/CodeGraphContext/docs/CLI_COMPLETE_REFERENCE.md` (the canonical CLI table).
4. Add an integration test in `tests/integration/cli/` using `typer.testing.CliRunner`.

Print to stderr via `console = Console(stderr=True)` so stdout stays reserved for piping data.

---

## 10. Configuration & Runtime

- Per-user config dir: `~/.codegraphcontext/` (loaded by `cli/config_manager.py`). Stores `.env`, contexts, workspace mappings, embedded DB files.
- Per-repo config dir: `.codegraphcontext/` in the workspace root (used for context discovery in `MCPServer`).
- Notable env vars: `NEO4J_URI`, `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `FALKORDB_HOST`, `FALKORDB_PORT`, `DEFAULT_DATABASE`, `CGC_RUNTIME_DB_TYPE`, `CGC_HOME`, `SCIP_INDEXER`, `SCIP_LANGUAGES`, `ENABLE_APP_LOGS`, `DEBUG_LOGS`, `LIBRARY_LOG_LEVEL`, `MAX_TOOL_RESPONSE_TOKENS`. See `@e:/claude-hub/CodeGraphContext/.env.example` for canonical examples.
- Ignore patterns: `.cgcignore` (gitignore-style; merged with `.gitignore` and built-in defaults in `tools/indexing/constants.py`).

`MAX_TOOL_RESPONSE_TOKENS` is read **per call** — config edits take effect without a restart.

---

## 11. Common Commands

```bash
# Install (editable + dev extras)
pip install -e ".[dev,parsing]"

# CLI entry points (both work)
cgc --help
codegraphcontext --help

# Daily dev loop
cgc index .                     # one-shot index of cwd
cgc watch .                     # index + live update
cgc list                        # see what's indexed
cgc find name MyFunction
cgc analyze callers MyFunction
cgc analyze complexity --threshold 10
cgc analyze dead-code
cgc query "MATCH (f:Function) RETURN f.name LIMIT 10"

# MCP
cgc mcp setup                   # interactive IDE/client setup wizard
cgc mcp start                   # start stdio MCP server (used by IDEs)
cgc mcp tools                   # list registered MCP tools

# Config
cgc config show
cgc config db kuzudb            # quick switch backend
cgc neo4j setup                 # configure Neo4j

# Bundles
cgc bundle export my.cgc --repo .
cgc bundle load flask
cgc registry search web
```

PowerShell note (Windows is a primary platform): the test runner is a Bash script. On Windows use `pytest` directly:

```powershell
pytest tests\unit\ tests\integration\ -v
```

---

## 12. Testing Strategy

Authoritative doc: `@e:/claude-hub/CodeGraphContext/TESTING.md:1-115`. TL;DR:

- **Pyramid:** `tests/unit/` (mocked, <100 ms) → `tests/integration/` (Typer/MCP, ~1 s) → `tests/e2e/` (real subprocesses, >10 s) → `tests/perf/`.
- **Fast loop (preferred locally):**
  ```bash
  ./tests/run_tests.sh fast      # unit + integration
  ```
  On Windows: `pytest tests/unit/ tests/integration/ -v`
- **Full suite:** `./tests/run_tests.sh all`
- **Markers (`pyproject.toml`):** `integration`, `e2e`, `slow`. Use them on new tests.
- **Excluded from collection:** `tests/fixtures/`, `tests/e2e/sample_projects/`, `venv`, `__pycache__` (`@e:/claude-hub/CodeGraphContext/pyproject.toml:75`).
- **Discipline:** Write or update tests **before/with** implementation. Never delete or weaken existing tests without explicit user approval. Add a regression test for every bug you fix.

When you cannot run tests yourself, give the user a copy-pastable `pytest …` command targeting only the affected files.

---

## 13. Coding Conventions

- **Style:** `black` defaults; existing code is the canonical reference. Match indentation, import grouping, and naming of neighboring code.
- **Imports:** Always at top of file. Imports inside functions are reserved for *intentional* lazy loading (e.g. backend selection in `core/__init__.py`) — preserve those patterns.
- **Comments / docstrings:** Do **not** add or delete comments unless the user asks. Many modules have docstrings that are part of the public contract — leave them intact.
- **Logging:** Use the helpers in `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/utils/debug_log.py` (`debug_log`, `info_logger`, `warning_logger`, `error_logger`, `debug_logger`). Never `print()` from library code; CLI output goes through `rich.Console(stderr=True)`.
- **Async vs sync:** `MCPServer.run()` is async; tool handlers are sync and dispatched via `asyncio.to_thread`. Don't make handlers async unless you also update the dispatcher.
- **Paths:** Always use `pathlib.Path` and absolute paths in graph properties. Never hard-code OS-specific separators.
- **Cypher:** Run all queries through `DatabaseManager.session()` so the wrapper layer normalizes record access across Kuzu / FalkorDB / Neo4j. Avoid backend-specific Cypher unless gated by a backend check.
- **Schema additions:** Update `tools/indexing/schema_contract.py` *and* the writer *and* `prompts.py`'s schema reference together — they are intentionally coupled.
- **No emojis** in source/docs unless explicitly requested by the user (README and a few public docs are exceptions, already in place).

---

## 14. Bug-Fix Discipline (Project-Specific)

1. **Find the root cause** — prefer minimal upstream fixes over downstream workarounds. The codebase has many specialized parsers; verify the bug is in the parser vs. resolver vs. writer before patching.
2. **One-line fixes are OK** when sufficient. Don't expand scope unless asked.
3. **Always add a regression test** — usually under `tests/unit/parsers/` or `tests/integration/`.
4. **Never weaken or delete existing tests** to make a change pass.
5. **Don't introduce a new dependency** unless absolutely required and aligned with version pins in `pyproject.toml`. Prefer stdlib.
6. **Keep edits scoped to the failing area.** Avoid drive-by formatting changes.

---

## 15. Common Pitfalls

- **Tree-sitter on Python 3.13:** parsing extras are intentionally excluded (see `@e:/claude-hub/CodeGraphContext/pyproject.toml:25-27`). Don't unconditionally import tree-sitter at module top-level if your code may run under 3.13 without parsing extras.
- **FalkorDB Lite on Windows:** unsupported. Selection logic falls back to Kuzu — preserve this branch when editing `core/__init__.py`.
- **`relative_path` vs `path`:** Cypher queries against `Function`/`Class` must use absolute `path`. Many existing helpers normalize this; don't fight them.
- **Workspace prefix:** MCP responses strip `/workspace/`. If you add a path-bearing key, name it `*_path` so `_is_path_key` detects it (`@e:/claude-hub/CodeGraphContext/src/codegraphcontext/server.py:51-66`).
- **Job IDs:** All long-running indexing returns a job ID. Tools must register with `JobManager` (`core/jobs.py`) and update status; otherwise progress polling breaks.
- **Bundle drift:** `.cgc` bundles encode the schema. If you change node labels or merge keys, also bump bundle handling in `core/cgc_bundle.py`.
- **Generated files:** `tools/scip_pb2.py` is generated from protobuf — never hand-edit. `viz/dist/` is synced from `website/` build output.
- **Two CLI entry points:** both `cgc` and `codegraphcontext` map to `cli.main:app`. Don't add a third without updating `[project.scripts]`.

---

## 16. PR / Change Hygiene

- Branch from `main`, scope each PR to a single feature or fix (`@e:/claude-hub/CodeGraphContext/CONTRIBUTING.md:9-10`).
- Run `./tests/run_tests.sh fast` (or the Windows equivalent) before declaring done.
- Update `docs/CLI_COMPLETE_REFERENCE.md` when CLI changes, `docs/MCP_TOOLS.md` for MCP tool changes, and `ARCHITECTURE.md` only for architectural shifts.
- For user-visible changes, mention them where relevant in `README.md` (and translated READMEs only if the change is large/stable).
- Commit messages: imperative, descriptive (e.g. `fix(parser/python): handle decorator with kwargs`).

---

## 17. Where to Look (Quick Index)

| Task | Start here |
| :--- | :--- |
| Understand whole system | `@e:/claude-hub/CodeGraphContext/ARCHITECTURE.md:1-50` |
| Add/fix MCP tool | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tool_definitions.py:1-80`, `src/codegraphcontext/tools/handlers/` |
| Add/fix CLI command | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/cli/main.py:1-100`, `cli/cli_helpers.py` |
| Add/fix language parser | `src/codegraphcontext/tools/languages/<lang>.py`, register in `graph_builder.py` |
| Change graph schema | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/indexing/schema_contract.py:9-69` + `persistence/writer.py` + `prompts.py` |
| Add/fix DB backend | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/core/__init__.py:63-184` + `core/database_<x>.py` |
| Query work | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/tools/code_finder.py` |
| Write tests | `@e:/claude-hub/CodeGraphContext/TESTING.md:1-115`, `tests/conftest.py` |
| LLM behavior in MCP clients | `@e:/claude-hub/CodeGraphContext/src/codegraphcontext/prompts.py:1-125` |

---

## 18. Non-Negotiables

- Read every file under `ai-context/steering/` at the start of each session. They override anything in this file when they conflict.
- Never commit secrets, generated `viz/dist/` content, or local DB files (`.codegraphcontext/`, `*.db`). All are already in `@e:/claude-hub/CodeGraphContext/.gitignore`.
- Never bypass `GraphWriter` or the `DriverWrapper` abstraction.
- Never widen `pyproject.toml` dependency ranges silently.
- Never add or remove tests, comments, or docstrings without an explicit reason tied to the user's request.
- Confirm before destructive actions: deleting bundles, dropping graph contents, mutating `~/.codegraphcontext/`.

If a requested change conflicts with anything above, surface the conflict and ask before proceeding.
