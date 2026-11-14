# Graphify — working guide

Notes from installing and running Graphify (fork: `moystakasgregory-glitch/graphify`,
upstream `Graphify-Labs/graphify`, Apache-2.0, version 0.9.108, PyPI package `graphifyy`).
Read this first in a new session instead of re-learning the repo.

## What it is
Turns a folder (code, docs, PDFs, images, video) into a **knowledge graph** that an AI coding
assistant queries instead of grepping files. Not a vector index: a real NetworkX graph with
communities, traversals and paths.

- **Code** is parsed locally with tree-sitter: deterministic, no LLM, no cost, nothing leaves the machine.
- **Docs / PDFs / images / video** need a model for a semantic pass (the assistant's own model, or an
  API key such as `GEMINI_API_KEY` / `GOOGLE_API_KEY`).
- Every edge has a confidence tag: `EXTRACTED` (explicit in source), `INFERRED` (resolved by
  graphify), `AMBIGUOUS`.
- Graphify Cloud (app.graphify.com) is the company's paid hosted version; the local tool works without it.

## Install
Needs Python 3.10–3.14.
```bash
uv tool install graphifyy            # or: pipx install graphifyy
graphify install                     # registers the /graphify skill in the detected assistant
# install from the fork instead of PyPI:
pip install /path/to/graphify        # (use a venv)
```
Then in the assistant: `/graphify .`

## Output (`graphify-out/`)
| File | What |
|---|---|
| `graph.html` | interactive map in the browser (click, filter, search) |
| `GRAPH_REPORT.md` | god nodes, surprising connections, import cycles, communities, suggested questions |
| `graph.json` | full graph (has `graph.schema_version`, `graph.graphify_version`) |
| `cache/`, `manifest.json`, `.graphify_*` | build state and caches |

## CLI commands (verified)
| Command | Use |
|---|---|
| `graphify update <path>` | re-extract **code** and rebuild the graph, no LLM. Flags: `--force`, `--no-cluster` |
| `graphify explain "X"` | a node, its source location, community, and connections (`--all` for every one) |
| `graphify path "A" "B"` | shortest path; directed by default, `--undirected` to ignore direction |
| `graphify query "<question>"` | BFS subgraph for a question (`--dfs`, `--seed X`, `--context C`) |
| `graphify cluster-only <path>` | re-cluster an existing graph and regenerate the report (`--no-viz`, `--no-label`) |
| `graphify label <path>` | (re)name communities with an LLM backend (`--missing-only`) |
| `graphify watch <path>` | rebuild on code changes |
| `graphify add <url>` | fetch a URL into `./raw` and update the graph |
| `graphify clone <github-url>` | clone a repo to `~/.graphify/repos/...` for /graphify |
| `graphify merge-graphs g1 g2` | merge graphs across repos |
| `graphify install` / `uninstall [--purge]` | add / remove the assistant skill |
| `graphify-mcp` | MCP server over a graph (`graphify/serve.py`) |

Gotcha: `--no-label` belongs to `cluster-only` / `label`, **not** `update` (`update --no-label` errors).

## Pipeline and modules (from ARCHITECTURE.md)
```
detect() → extract() → build() → cluster() → analyze helpers → report.generate() → export.to_*()
```
| Module | Role |
|---|---|
| `detect.py` | scan a directory, group files by category, count words |
| `extract.py` | `extract(paths, *, root=...)` → `{nodes, edges}`; always pass `root` |
| `extractors/` | per-language tree-sitter extractors (Python, JS/TS, Go, Rust, Java, C/C++, C#, Ruby, Kotlin, …) |
| `build.py` | extraction dicts → `nx.Graph` |
| `cluster.py` | Leiden communities |
| `analyze.py` | `god_nodes`, `surprising_connections`, `suggest_questions`, `find_import_cycles`, `graph_diff` |
| `report.py` / `export.py` / `wiki.py` | GRAPH_REPORT.md; JSON, HTML, Obsidian, SVG, GraphML, Canvas, Cypher; per-community wiki |
| `cache.py`, `validate.py`, `security.py` | caching, schema validation, URL/path safety |
| `serve.py`, `watch.py`, `benchmark.py` | MCP/HTTP server, file watcher, token-savings benchmark |
| `skills/<platform>/`, `skill-*.md` | the /graphify skill text for each assistant |

Extraction schema: nodes `{id, label, source_file, source_location}`, edges
`{source, target, relation, confidence}`; `validate.py` enforces it before `build()`.

Repo conventions: `AGENTS.md` says to prefer graph queries for codebase questions, read
`graphify-out/wiki/index.md` if present, and run `graphify update .` after code edits.
`tests/test_architecture_doc.py` imports every symbol in ARCHITECTURE.md, so that table stays accurate.
Very large `CHANGELOG.md` (600 KB) and `uv.lock`: don't read them whole.

## Test run: New Lexis website (Next.js), code only
Ran `graphify update .` on a copy of `components/`, `app/`, `lib/` (32 files, ~10k words):
- 124 nodes, 146 edges, 17 communities, a few seconds, **0 tokens**.
- God nodes: `Motion()` (9 edges), `Button`, `BookingSection()`, `AnimatedThemeToggle()`.
- No import cycles. 100% EXTRACTED.
- `.css` files are not classified (skipped).
- **Limitation seen:** JS cross-file calls are not always linked, e.g. no path from
  `BookingSection` to `mailtoHref` even though one calls the other. Treat "no path" in JS
  projects as "not found", not "not connected".

## Quick start for next time
```bash
python3 -m venv .venv && .venv/bin/pip install graphifyy   # or the fork path
cd <project> && ../.venv/bin/graphify update .
cat graphify-out/GRAPH_REPORT.md
graphify explain "SomeFunction"
```
