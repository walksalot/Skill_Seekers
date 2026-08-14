# AGENTS.md — Skill Seekers

Single canonical rulebook for every AI agent (Codex natively; Claude Code via CLAUDE.md's
`@AGENTS.md` import; QWEN.md stays a short pointer here). The two former rulebooks (a
21.6 KB CLAUDE.md and a 13.8 KB AGENTS.md) carried contradictory versions and dispatch
descriptions; this merge resolves them — the drift-guarded CLAUDE.md content won on every
conflict, re-verified against the tree at merge time.

## Project Overview

**Skill Seekers** converts documentation from 18 source types into production-ready formats for 21+ AI platforms (LLM platforms, RAG frameworks, vector databases, AI coding assistants). Published on PyPI as `skill-seekers`.

**Version:** source of truth is `src/skill_seekers/_version.py` (which reads `pyproject.toml`) — never pin a version number in prose; two pinned copies of it in the old rulebooks contradicted each other. | **Python:** 3.10+ | **Website:** https://skillseekersweb.com/

**Architecture:** `docs/UML_ARCHITECTURE.md` (UML + module overview); StarUML at `docs/UML/skill_seekers.mdj`. Refactor state/history: `docs/UNIFICATION_PLAN.md` (Grand Unification — all 5 phases done).

## Essential Commands

```bash
# REQUIRED before running tests or CLI (src/ layout — tests hard-exit if not installed)
pip install -e .

# Run all tests (NEVER skip - all must pass before commits)
pytest tests/ -v

# Fast iteration (skip slow, integration, E2E, network, MCP)
pytest tests/ -m "not slow and not integration and not e2e and not network and not serial and not mcp_only" -q

# Fast parallel (pytest-xdist)
pytest tests/ -n auto --dist=loadfile -m "not slow and not integration and not e2e and not network and not serial and not mcp_only" -q

# 3-phase runner (recommended local dev)
bash scripts/run_tests_fast.sh

# Single test
pytest tests/test_scraper_features.py::test_detect_language -vv -s

# Code quality (must pass before push - matches CI; CI pins ruff==0.15.8)
uvx ruff check src/ tests/
uvx ruff format --check src/ tests/
mypy src/skill_seekers  # continue-on-error in CI

# Auto-fix
uvx ruff check --fix --unsafe-fixes src/ tests/
uvx ruff format src/ tests/

# Build & publish
uv build
uv publish
```

**Pytest config:** `asyncio_mode = "auto"`. Markers: `slow`, `integration`, `e2e`, `venv`, `bootstrap`, `benchmark`, `asyncio`, `serial`, `network`, `mcp_only`.

## CI

Runs on push/PR to `main` or `development`. Lint job (ruff + mypy) + test matrix (Ubuntu + macOS, Python 3.10–3.12). Test phases: `test-fast` (unit, xdist), `test-serial` (serial/integration/E2E/network), `test-mcp` (needs `[mcp]` extras). 7 workflows total incl. release.yml (tag → PyPI via `uv build`), docker-publish.yml (amd64+arm64 CLI + MCP images), test-vector-dbs.yml, quality-metrics.yml, scheduled-updates.yml, vector-db-export.yml.

## Git Workflow

- **`main`** — production, protected (tests + 1 review)
- **`development`** — default PR target (tests required)
- Feature branches `feature/{task-id}-{description}` from `development`; PRs always target `development`, never `main` directly.

## Architecture

### CLI: Unified create command

Entry point `src/skill_seekers/cli/main.py`. `create` is the **primary** entry point — auto-detects source type and routes to the appropriate `SkillConverter`. `scan` (issue #327) is a separate discovery step that emits one config per detected framework; run `create` on each.

```
skill-seekers create <source>     # Auto-detect: URL, owner/repo, ./path, file.pdf, etc.
skill-seekers scan <dir>          # AI-driven discovery → one config per framework + <project>-codebase.json
skill-seekers package <dir>       # Package (--target claude/gemini/openai/markdown/minimax/opencode/kimi/deepseek/qwen/openrouter/together/fireworks/atlas/langchain/llama-index/haystack/chroma/faiss/weaviate/qdrant/pinecone/ibm-bob)
```

Legacy-dispatch commands (see below): `enhance`, `enhance-status`, `package`, `upload`, `install` (scrape+enhance+package+upload), `install-agent`, `estimate`, `extract-test-examples`, `resume <job_id>`, `quality`, `config`, `workflows`, `sync-config`, `stream`, `update`, `multilang`.

**CLI dispatch** uses the `COMMAND_CLASSES` table in `main.py`: `create`, `scan`, and `doctor` dispatch as `Cls(args).execute()` on the parsed namespace directly. The remaining commands use the legacy `COMMAND_MODULES` dispatch (module `main(args=...)` receives the parsed central namespace; the old `_reconstruct_argv` round-trip is gone); flagged for migration. `ScanCommand.execute()` is the single `asyncio.run` boundary.

### Scan command (issue #327)

Pipeline in `src/skill_seekers/cli/scan_command.py`:

1. `collect_signals()` — deterministic, bounded gathering (per-kind byte budgets: 24 KB manifest / 6 KB README / 6 KB CI / 28 KB samples, 64 KB total). `_SOURCE_DIRS` covers ~14 layouts; walks root one level deep for flat-layout Python.
2. `detect_with_ai(bundle, AgentClient)` — one LLM call, structured JSON. Source signals are first-2-KB whole-file samples (no regex parsing). Canonical-slug prompt + canonical-name resolver are coupled — change one, update the other.
3. `resolve_or_generate_with_status()` — cache → `resolve_config_path` with canonical name candidates (handles CJK/European suffixes) → `generate_config_with_ai` last. Always appends `.json` to lookup names; always stamps `metadata.detected_version` (nested — `metadata.version` means config-schema version).
4. `emit_codebase_config()` — always writes `<project>-codebase.json`.
5. `diff_against_existing()` — keyed by filename slug so re-scans don't churn.
6. `_archive_removed()` — MOVE (never delete) to `out_dir/.archived/<UTC-timestamp>/`.
7. `maybe_publish()` — native async, opt-in registry submission; `GITHUB_TOKEN` pre-check, existing-issue idempotency guard, 0s/5s/15s retry backoff.

**Cost guardrails:** `--max-ai-generations N` (default 10), `--dry-run`, `--probe-urls`. **Safety:** all writes via `_atomic_write_json`; `_safe_size` guards broken symlinks; non-zero exit when nothing emitted. **Public constant:** `SourceDetector.CODE_PROJECT_MARKERS` (~50 manifest types), shared with signal_collectors.

### SkillConverter pattern (Template Method + Factory)

All 18 source types implement `SkillConverter` (`skill_converter.py`): `get_converter(type, config)` factory + `converter.run()` template (`extract()` → `build_skill()`). Registry: `CONVERTER_REGISTRY`. Converters have **no `main()`** — `create_command.py` builds config from `ExecutionContext` and runs centralized enhancement. The base resolves `skill_dir` once and derives `data_file` via `data_file_for()` — subclasses must not re-derive paths.

The 18: web (doc_scraper), github, pdf, word, epub, video, local (codebase_scraper), jupyter, html, openapi, asciidoc, pptx, rss, manpage, confluence, notion, chat, config (unified_scraper).

### DocumentSkillBuilder (build side of the 9 document scrapers)

`cli/document_skill_builder.py` sits between `SkillConverter` and the 9 document scrapers (epub, word, pptx, html, pdf, jupyter, man, rss, chat): owns `categorize_content`, reference-file writing, `index.md` + `SKILL.md` generation, `load_extracted_data`. Variation points are class attrs (`DOC_NOUN`, `SOURCE_LABEL`, …) + small hooks. Output pinned **byte-identical** by golden trees in `tests/golden/phase2/` — `UPDATE_GOLDENS=1` rewrites them, only deliberately.

### UnifiedScraper (multi-source configs)

`unified_scraper.py` dispatches via the class-level `SOURCE_DISPATCH` table; `_scrape_with_converter()` is the shared engine for the 13 mechanical types, so new `CONVERTER_REGISTRY` types work in the unified scrape engine automatically (unified-config validation still gates on `VALID_SOURCE_TYPES` — see Adding new features). documentation/github/local stay bespoke (commented why). `run()` deliberately does NOT follow the base template. Build side: `unified_skill_builder.py:UnifiedSkillBuilder` — pairwise synthesis for docs+github/docs+pdf-style combos, `_generic_merge()` for all other combinations.

### Data flow (5 phases)

1. **Scrape** → `output/{name}_data/pages/*.json` 2. **Build** → `output/{name}/SKILL.md` 3. **Enhance** (optional, `--enhance-level 0-3`) 4. **Package** (platform adaptor) 5. **Upload** (optional).

### Platform adaptor pattern (Strategy + Factory)

`get_adaptor(platform, config)` in `adaptors/__init__.py` → `SkillAdaptor` instance (base + `SkillMetadata` in `adaptors/base.py`; `openai_compatible.py` shared base). One module per target in `src/skill_seekers/cli/adaptors/`. All use `--target`; all imported with try/except ImportError so missing optional deps don't break the registry.

### CLI argument system (single-definition parsers)

`parsers/` holds the ONLY definition of each command's flags; `arguments/` holds shared defs (`common.py: add_all_standard_arguments()`, `create.py: UNIVERSAL_ARGUMENTS` etc.). Module `main(args=None)` paths build FROM the central parser class — **add/change a flag in `parsers/*.py` only**; drift-guard tests (`tests/test_cli_parsers.py`) fail CI on divergence. `ExecutionContext.override()` is context-local (ContextVar) — thread/async safe; propagate via `copy_context`.

### Standalone subsystems (outside the create flow)

- `embedding/` — FastAPI embedding server + cache + multi-backend generators; feeds vector-DB adaptors.
- `sync/` — real-time doc-sync: change detection, scheduled incremental re-scrapes, email/Slack/webhook notify.
- `benchmark/` — performance suite (timing, memory, CPU; comparison reports).
- `workflows/` — bundled enhancement-workflow presets.
- `cli/storage/` — cloud upload backends (S3/GCS/Azure) behind `BaseStorageAdaptor` (`storage/base_storage.py`).

### C3.x codebase analysis pipeline (all opt-out via `--skip-*`)

C3.1 pattern_recognizer (10 GoF patterns, 9 languages) · C3.2 test_example_extractor · C3.3 how_to_guide_builder · C3.4 config_extractor · C3.5 generate_router · C3.10 signal_flow_analyzer (Godot).

### MCP server

`src/skill_seekers/mcp/server_fastmcp.py` — 40 tools via FastMCP; stdio (Claude Code) or HTTP (Cursor/Windsurf). Optional dep: `pip install -e ".[mcp]"`.

- **Tools run in-process** via `run_cli_main()` in `mcp/tools/_common.py` (real parser, argv patch under a lock, identical `(stdout, stderr, returncode)` contract).
- **Exceptions BY DESIGN:** `enhance_skill` (LOCAL agent) and `install_skill`'s enhancement step stay subprocess — fork-bomb-guard env semantics (`SKILL_SEEKER_ENHANCE_ACTIVE`). Never make these in-process.
- **Domain logic lives in `skill_seekers.services/`** — importable without `[mcp]`; old `skill_seekers.mcp.*` paths are back-compat shims.

### Enhancement (AgentClient is the single AI transport)

Every AI call goes through `AgentClient` (`cli/agent_client.py`): central truncation gate, timeout policy, error classification. `API_PROVIDERS` + `AGENT_PRESETS` live ONLY there. Each provider entry declares wire `protocol` (`anthropic`/`openai`/`google`) and `supports_images` — `_call_api` branches on protocol, not provider name. Multimodal via `AgentClient.call_with_image()`.

- **API mode:** Anthropic, Gemini, OpenAI, Moonshot/Kimi, MiniMax — registry order; `SKILL_SEEKER_PROVIDER` forces one. Models via `SKILL_SEEKER_MODEL` or per-provider vars; `ANTHROPIC_BASE_URL` for compatible endpoints. Vision: `SKILL_SEEKER_VISION_PROVIDER`.
- **LOCAL mode (fallback):** Claude Code, Kimi Code, Codex, Copilot, OpenCode, custom — `build_local_agent_command()`.
- Control: `--enhance-level 0/1/2/3` · Agent: `--agent claude|codex|copilot|opencode|kimi|custom`.

## Key implementation details

- **Smart categorization** (`doc_scraper.py:smart_categorize()`): 3 pts URL match, 2 title, 1 content; threshold 2+; falls back to "other".
- **Content extraction** (`doc_scraper.py`): `FALLBACK_MAIN_SELECTORS` + `_find_main_content()`; links extracted from the full page before early return; `body` deliberately excluded from fallbacks.
- **Three-stream GitHub architecture** (`unified_codebase_analyzer.py`): code analysis / documentation / community; depth `basic` (1–2 min) or `c3x` (20–60 min).
- **sys.modules gotcha:** `test_swift_detection.py` deletes `skill_seekers.cli` modules from `sys.modules` — must save/restore both `sys.modules` entries AND parent package attrs.

## Code style (ruff, from pyproject.toml)

- Line length 100; target Python 3.10+; rules E, W, F, I, B, C4, UP, ARG, SIM (ignores: E501, F541, ARG002, B007, I001, SIM114).
- Imports: isort via ruff, `skill_seekers` first-party; guard optional imports with try/except ImportError (see `adaptors/__init__.py`).
- Naming: `snake_case.py` files, `PascalCase` classes, `snake_case` functions, `UPPER_CASE` constants, `_` private prefix.
- Types: gradual; modern syntax (`str | None`, `list[str]`); tests excluded from strict checking.
- Docstrings: module-level everywhere; Google-style for public functions/classes.
- Errors: specific exceptions, never bare `except:`; chain with `raise … from e`; clear install instructions on optional-dep failure.
- Lint suppressions inline: `# noqa: XXXX`.

## Testing

Known legitimate skips (~11): chromadb×Py3.14 (2), weaviate-client absent (2), Qdrant not running (2), langchain/llama_index absent (2), GITHUB_TOKEN unset (3). Fixtures in `tests/fixtures/`.

## Dependencies

Core: `langchain`, `llama-index`, `anthropic`, `httpx`, `PyMuPDF`, `pydantic`. Optional extras are enumerated in pyproject.toml `[project.optional-dependencies]` (per-platform, per-source-type, cloud storage, `[mcp]`, `[all]`, `[all-llms]` — read the file rather than a hand-copied list; the old rulebooks' lists had drifted). Dev deps use PEP 735 `[dependency-groups]`.

## Environment variables

`ANTHROPIC_API_KEY` (+ optional `ANTHROPIC_BASE_URL`) · `GOOGLE_API_KEY` · `OPENAI_API_KEY` · `GITHUB_TOKEN`. Never commit keys; `.env` is gitignored.

## Adding new features

**New platform adaptor:** 1) `cli/adaptors/{platform}.py` inheriting `SkillAdaptor` 2) register in `adaptors/__init__.py` (try/except import + `ADAPTORS` dict) 3) optional dep in pyproject.toml 4) tests.

**New source type converter:** 1) `cli/{type}_scraper.py` — document-shaped sources inherit `DocumentSkillBuilder`, else `SkillConverter`; set `SOURCE_TYPE` 2) register in `CONVERTER_REGISTRY` (the unified scrape engine picks it up automatically) 3) add the type to `config_validator.py:VALID_SOURCE_TYPES` — unified-config validation rejects unknown types 4) config building in `create_command.py:_build_config()` 5) auto-detection in `source_detector.py` 6) optional dep 7) tests.

**New CLI argument:** central parser class (`parsers/{cmd}_parser.py`) ONLY — drift-guard fails otherwise; universal args in `arguments/create.py`; shared scraper args in `arguments/common.py`.

## Deployment

Docker (multi-stage, Python 3.12 slim; CLI image runs non-root `skillseeker` UID 1000): `docker build -t skill-seekers:local .`; MCP image via `Dockerfile.mcp` (port 8765, non-root `mcp` user). MCP server: `skill-seekers-mcp` or `python -m skill_seekers.mcp.server_fastmcp`.

## Resources

PyPI: https://pypi.org/project/skill-seekers/ · Repo: https://github.com/yusufkaraaslan/Skill_Seekers · Website/docs/config browser: https://skillseekersweb.com/ · Project board: https://github.com/users/yusufkaraaslan/projects/2
