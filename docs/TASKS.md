# PathwayPilot — Tasks

All implementation tasks in build order. Work top to bottom. Each task lists its dependencies, a time estimate and "done when" criteria. Tick the box and add a line to `MEMORY.md` when you finish one.

**Total estimate: about 10.5 to 11 hours** (0.5 setup + 1 data + 1 graph + 1 retrieval + 2 agent + 0.5 API + 1.5 evals + 1.5 UI + 1 ship + 0.75 README). It's a long day, so the cut line is marked below.

Legend: ⏱ estimate · ⛓ depends on · ✅ done when

---

## Phase 0 — Setup (30 min)

- [ ] **T0.1 Create repo and skeleton** ⏱ 15m
  - Create the folder structure from ARCHITECTURE §3, `pyproject.toml`, `.gitignore`, `.env.example`, `Makefile` (targets: `dev`, `test`, `lint`, `eval`, `eval-offline`, `docker`)
  - Copy all six docs into `docs/`
  - ✅ `git init`, first commit `chore: project skeleton`

- [ ] **T0.2 Install dependencies and tooling** ⏱ 10m · ⛓ T0.1
  - `uv venv && uv pip install -e ".[dev]"` (or pip)
  - ✅ `ruff check .` and `pytest` run (0 tests is fine)

- [ ] **T0.3 Config** ⏱ 5m · ⛓ T0.2
  - `config.py` with `Settings` (all env vars from ARCHITECTURE §8), default `LLM_PROVIDER=ollama`, `EMBEDDER=tfidf`
  - ✅ `Settings()` loads with no `.env` present

---

## Phase 1 — Data (1 h)

- [ ] **T1.1 Define models** ⏱ 10m · ⛓ T0.3
  - `models.py`: `Node`, `Edge`, `NodeType` and `EdgeType` enums, `PathwayData`, domain errors
  - ✅ Pydantic rejects an unknown node type

- [ ] **T1.2 Curate `data/pathways.json`** ⏱ 40m · ⛓ T1.1
  - Glycolysis (10 reactions), pyruvate → acetyl-CoA link (PDH complex), TCA cycle (8 reactions), shikimate pathway (7 reactions ending at chorismate)
  - Include cofactors as `COFACTOR_OF` (ATP, ADP, NAD⁺, NADH, NADPH, FAD, GTP)
  - Inhibitors: glyphosate → EPSP synthase; malonate → succinate dehydrogenase; fluorocitrate → aconitase; arsenite → PDH complex; citrate and ATP → PFK-1 (allosteric)
  - Every node gets a description written for search (include synonyms like "PFK-1", "EPSPS", "PEP")
  - Add Reactome / ChEBI IDs where available; spot-check 5 facts against Reactome
  - ✅ About 60 to 80 nodes; PEP is shared by glycolysis and shikimate

- [ ] **T1.3 Validation script** ⏱ 10m · ⛓ T1.2
  - `scripts/validate_data.py`: unique IDs, every edge endpoint exists, edge types match the node types (e.g. `CATALYZES` must go enzyme → reaction), every reaction has ≥ 1 substrate, ≥ 1 product, 1 enzyme and 1 pathway
  - ✅ Script passes; a broken copy of the file fails with a clear message

---

## Phase 2 — Graph (1 h)

- [ ] **T2.1 GraphStore** ⏱ 20m · ⛓ T1.3
  - Load JSON → `nx.DiGraph`, run validation, `get_node(id)`, `nodes_by_type()`
  - ✅ `test_store.py`: loads the real file; a dangling edge raises `DataValidationError`

- [ ] **T2.2 Graph queries** ⏱ 40m · ⛓ T2.1
  - `get_neighbors(node_id, direction, edge_types)`
  - `find_path(source, target)`: shortest path over `SUBSTRATE_OF`/`PRODUCES` only; return ordered steps `{reaction, substrates, products, enzyme}`
  - `get_inhibitors(node_id)`: for an enzyme, or for every enzyme in a pathway
  - `get_downstream(node_id, max_steps)`: from an enzyme (via its reaction) or a metabolite, follow products forward
  - ✅ `test_queries.py`: glucose → pyruvate gives 9 steps in order; cofactor edges are ignored; glyphosate → EPSP synthase; downstream of EPSP synthase reaches chorismate; unknown ID raises `NodeNotFound`

---

## Phase 3 — Retrieval (1 h)

- [ ] **T3.1 Embedders** ⏱ 20m · ⛓ T2.1
  - `Embedder` protocol; `TfidfEmbedder` (fit on node texts); `OpenAIEmbedder` (batch call, works for Azure too)
  - Node text = `name + synonyms + type + description`
  - ✅ TF-IDF embedder works with no network

- [ ] **T3.2 VectorIndex** ⏱ 25m · ⛓ T3.1
  - Build, cosine `search(query, k, types)`, save/load `.cache/index.npz` keyed by data file hash + embedder name
  - `scripts/build_index.py`
  - ✅ `test_retrieval.py`: "glyphosate" → `inh_glyphosate` in the top 3; "PFK-1" → `enz_pfk1` in the top 3; type filter works

- [ ] **T3.3 Quick retrieval check** ⏱ 15m · ⛓ T3.2
  - Try 10 questions by hand; improve node descriptions or synonyms where search misses
  - ✅ Findings logged in `MEMORY.md`

---

## Phase 4 — Agent (2 h)

- [ ] **T4.1 Tool layer** ⏱ 35m · ⛓ T2.2, T3.2
  - Arg models + JSON schemas for 5 tools; `dispatch(name, args_json)` → `(result_json, ToolCallRecord)`
  - Domain errors → `{"error": "..."}` with a next-step hint
  - Tool descriptions written like API docs (see RULES §3)
  - ✅ `test_tools.py`: each tool dispatches; bad args return an error message, not an exception

- [ ] **T4.2 LLM client factory + fake LLM** ⏱ 25m · ⛓ T0.3
  - `make_client(settings)` for `ollama` (default), `openai`, `azure`, `fake`
  - `FakeLLM` replays a script of tool calls and then a final answer, in the same response shape as the SDK
  - ✅ Fake client returns a tool call and then a final message

- [ ] **T4.3 System prompt** ⏱ 15m · ⛓ T4.1
  - Must use tools; cite `[node_id]`; answer only from tool results; say clearly when the graph doesn't cover a question; be concise; list steps in order for path questions
  - ✅ Prompt saved in `prompts.py`; version noted in `MEMORY.md`

- [ ] **T4.4 Agent loop** ⏱ 45m · ⛓ T4.1, T4.2, T4.3
  - Message history, tool dispatch, `MAX_STEPS` cap with a final "answer now" call, usage totals, latency
  - Citation parsing + validation against tool results; subgraph from all node IDs in tool results
  - ✅ `test_agent_loop.py` (fake LLM): happy path; step cap sets `truncated=True`; a made-up citation is dropped

- [ ] **T4.5 First real run** ⏱ ✂ inside T4.4 time
  - Run with the default `LLM_PROVIDER=ollama` (pull a small tool-calling-capable model first); ask the 4 example questions from a small script. Optionally repeat with `openai` or `azure` for comparison.
  - ✅ Answers look right; any surprises logged in `MEMORY.md`

---

## Phase 5 — API (30 min)

- [ ] **T5.1 FastAPI app** ⏱ 30m · ⛓ T4.4
  - Lifespan builds store, index and client on `app.state`
  - Routes: `POST /api/ask`, `GET /api/search`, `GET /api/node/{id}`, `GET /api/health`; serve `web/` at `/`
  - JSON request log line; 422 validation; 503 for provider failure
  - ✅ `test_api.py` (fake provider): health ok, ask returns every field, empty question → 422; `make dev` serves `/docs`

---

## Phase 6 — Evals (1.5 h) — never cut this phase

- [ ] **T6.1 Golden set** ⏱ 40m · ⛓ T1.2
  - 20 questions in `evals/golden.jsonl`: lookup 4, path 5, cross-pathway 3, impact/inhibition 5, out of scope 3
  - Fields: `id`, `category`, `question`, `gold_nodes`, `expected_tools`, `must_include` (lists of alternatives allowed, e.g. `["PEP", "phosphoenolpyruvate"]`), `should_refuse`
  - ✅ Every gold node ID exists (checked by a test)

- [ ] **T6.2 Metrics** ⏱ 20m · ⛓ T6.1
  - `recall_at_k`, `mrr`, `tool_selection_accuracy`, `answer_accuracy` (all `must_include` groups present, case-insensitive), `refusal_accuracy`, `citation_validity`, latency p50/p95, token totals
  - ✅ Unit tests for each metric on hand-made inputs

- [ ] **T6.3 Eval runner + reports** ⏱ 20m · ⛓ T6.2, T4.4
  - `python -m evals.run_eval --mode offline|full [--gate 0.80]`
  - Writes `results/<timestamp>.json` + `report.md` (summary table + per-question pass/fail)
  - Exit code 1 if the gate fails
  - ✅ Offline mode runs in under 5 s with no key

- [ ] **T6.4 Baseline + one improvement** ⏱ 10m run + any fixes · ⛓ T6.3
  - Run `--mode full`, record the baseline in `MEMORY.md`
  - Change **one** thing (tool descriptions, search `k`, prompt wording, node synonyms), run again, record the difference
  - ✅ Two runs logged with a one-line explanation of why the score moved

---

## ✂ CUT LINE — if you're behind schedule, the order to drop things below is: Phase 8 K8s (T8.2) → LLM judge (T10.1) → demo GIF (T9.2). Keep the UI (T7) if at all possible; it's what makes the demo.

---

## Phase 7 — Web UI (1.5 h)

- [ ] **T7.1 Layout + tokens** ⏱ 25m · ⛓ T5.1
  - `index.html`, `styles.css` with all DESIGN.md tokens (light and dark), header, ask bar, example chips, two-column grid that stacks below 1024px
  - ✅ Looks right at 400px and 1280px, in both themes

- [ ] **T7.2 Answer + trace rendering** ⏱ 30m · ⛓ T7.1
  - Call `/api/ask`; loading, error, refusal, truncated states; citation chips (safe rendering with `textContent`); metrics strip; trace rows with `<details>`
  - ✅ All 4 states render correctly with the fake provider

- [ ] **T7.3 Graph view** ⏱ 35m · ⛓ T7.2
  - Cytoscape (pinned CDN version); shapes and colors by node type; inhibitor edges dashed red with tee arrows; path edges in accent; legend; fit button; citation click → highlight node
  - ✅ Glyphosate question shows glyphosate → EPSP synthase → reaction → EPSP → … → chorismate

---

## Phase 8 — Ship (1 h)

- [ ] **T8.1 Docker** ⏱ 20m · ⛓ T5.1
  - `python:3.11-slim`, non-root user, `HEALTHCHECK`, `.dockerignore`
  - ✅ `docker run -p 8000:8000 -e LLM_PROVIDER=fake pathwaypilot:0.1.0` serves the UI (this smoke test overrides the `ollama` default to `fake` so it needs nothing reachable from inside the container; see ARCHITECTURE §8 for reaching a host Ollama instance)

- [ ] **T8.2 Kubernetes on kind** ⏱ 25m · ⛓ T8.1 · ✂ first to cut
  - `configmap.yaml`, `secret.example.yaml`, `deployment.yaml` (2 replicas, probes, resources), `service.yaml`
  - ✅ `kubectl get pods` shows 2/2 Ready; port-forward serves the UI

- [ ] **T8.3 GitHub Actions CI** ⏱ 15m · ⛓ T6.3
  - Jobs: ruff lint + format check → pytest → `eval --mode offline --gate 0.80` → docker build
  - ✅ A green run on GitHub; a deliberately broken synonym makes the gate fail (then revert)

---

## Phase 9 — Tell the story (45 min)

- [ ] **T9.1 README** ⏱ 30m · ⛓ everything above
  - Order: one-line pitch → demo GIF → "Why this is different from plain RAG" (3 bullets) → architecture diagram → eval results table (baseline vs improved) → quickstart (Ollama default, OpenAI, fake/test mode, Docker, kind) → design decisions (link to MEMORY.md) → roadmap
  - ✅ A reviewer understands the project in 60 seconds

- [ ] **T9.2 Demo GIF + screenshots** ⏱ 15m · ✂ can cut
  - Record the flow from DESIGN §9
  - ✅ GIF under 5 MB, embedded in the README

---

## Phase 10 — Stretch (only if time remains)

- [ ] **T10.1 LLM-as-judge faithfulness score** in the full eval (1–5 scale, judged against tool results)
- [ ] **T10.2 Hybrid search** (TF-IDF + embeddings, weighted) and compare recall@5
- [ ] **T10.3 Neo4j backend** implementing `GraphStore`
- [ ] **T10.4 Stream the trace** to the UI with server-sent events

---

## After the build
- [ ] Update your resume: add the project with the real eval numbers
- [ ] Pin the repo on GitHub; add a short LinkedIn post with the GIF
- [ ] Practice a 2-minute walkthrough: problem → why a graph → agent loop → how you measured it → what improved the score
