# PathwayPilot — Product Requirements Document

| | |
|---|---|
| **Owner** | Abhinaya Nagisetty |
| **Status** | Draft v1 — ready to build |
| **Build window** | 1 day (~10 focused hours) |
| **Last updated** | 2026-09-27 |

---

## 1. Product overview

PathwayPilot is a question-answering agent for metabolic pathways. A user asks a question in plain English, such as *"Which enzyme does glyphosate inhibit, and what does that block downstream?"* The agent answers by **calling tools that walk a knowledge graph** of pathways, enzymes, metabolites, reactions and inhibitors.

Every answer comes with:
- **Citations**: the exact graph nodes the answer is based on
- **A tool trace**: every tool the agent called, with arguments, results and timing
- **A graph view**: the subgraph the agent touched, drawn on screen

The project also ships an **evaluation harness** that scores the agent on a golden question set, and a **CI gate** that fails the build if quality drops.

The MVP covers three public pathways: **glycolysis, the citric acid (TCA) cycle, and the shikimate pathway**. That's about 60 to 80 nodes of curated data from Reactome.

---

## 2. Problem statement

Researchers regularly need answers that require **following connections across a pathway**:
- "How does glucose become acetyl-CoA, and which enzymes are involved?"
- "Which metabolite links glycolysis to aromatic amino acid synthesis?"
- "If this enzyme is inhibited, which later steps lose their input?"

Today they get these answers by:
1. Clicking through pathway databases and diagrams by hand
2. Asking a general chatbot, which answers confidently but can't show its sources and sometimes invents enzymes or steps

**Standard RAG doesn't fix this.** RAG finds text chunks that are *similar* to the question. Multi-hop questions need **traversal**: enzyme → reaction → product → next reaction. No single chunk holds the answer.

**The gap:** there's no lightweight tool that answers multi-hop pathway questions with **verifiable, cited, graph-grounded** answers, and whose accuracy is **measured** rather than assumed.

---

## 3. Goals and non-goals

### Goals
| # | Goal | How we'll know |
|---|---|---|
| G1 | Answer multi-hop pathway questions correctly | ≥ 80% answer accuracy on the golden set |
| G2 | Ground every answer in the graph | 100% of citations point to real nodes the agent actually retrieved |
| G3 | Refuse questions the graph can't answer | 100% correct refusals on out-of-scope questions |
| G4 | Make agent behavior observable | Every response includes a full tool trace |
| G5 | Make quality measurable and protected | Eval runs with one command; CI fails if retrieval recall@5 drops below 0.80 |
| G6 | Deployable like a real service | Runs in Docker and on a local Kubernetes cluster (kind) |

### Non-goals (MVP)
- Covering all of metabolism or loading full Reactome/KEGG exports
- User accounts, authentication, saved history
- Streaming responses
- Fine-tuning models
- Production cloud deployment (local `kind` only)
- Medical, clinical or agronomic advice
- Any proprietary or employer data. **Public data only.**

---

## 4. Target users

### Primary: research scientist
- A biologist, biochemist or plant scientist who knows the domain but not graph query languages
- **Needs:** fast, trustworthy answers with sources they can check
- **Frustration:** chatbots that can't show where an answer came from

### Secondary: bioinformatics or data engineer
- Builds internal tools for scientists
- **Needs:** a reference pattern for graph-grounded agents, with traces and evals they can adapt

### Portfolio audience: hiring manager or interviewer
- Reviews the repo and demo in 2 to 5 minutes
- **Needs:** to quickly see production habits: tool calling, evaluation, tests, CI, containers, clear docs
- **Success:** the README shows the architecture, an eval score table and a demo GIF above the fold

---

## 5. User stories

| ID | As a… | I want to… | So that… |
|---|---|---|---|
| US1 | scientist | ask a pathway question in plain English | I don't need to learn a query language |
| US2 | scientist | see which nodes the answer came from | I can verify it |
| US3 | scientist | see the relevant part of the pathway drawn | I can understand the context visually |
| US4 | scientist | be told clearly when the data can't answer my question | I don't act on a made-up answer |
| US5 | engineer | inspect every tool call the agent made | I can debug wrong answers |
| US6 | engineer | run the eval suite with one command | I can see if a change improved or hurt quality |
| US7 | engineer | swap the LLM provider through config | I can use local Ollama (default), OpenAI or Azure OpenAI |
| US8 | reviewer | run the app with one Docker command | I can try it without setup pain |

---

## 6. MVP scope

### In scope
| Area | MVP includes |
|---|---|
| **Data** | 3 pathways (glycolysis, TCA cycle, shikimate), plus the pyruvate → acetyl-CoA link. About 60 to 80 nodes: metabolites, enzymes, reactions, pathways, inhibitors. Every node carries a Reactome or ChEBI reference where one exists. |
| **Graph** | In-memory directed graph (NetworkX), loaded and validated at startup |
| **Retrieval** | Semantic search over node descriptions (embeddings). An offline TF-IDF fallback runs without an API key. |
| **Agent tools** | `search_entities`, `get_neighbors`, `find_path`, `get_inhibitors`, `get_downstream` |
| **Agent loop** | Hand-written tool-calling loop with a step cap, argument validation and a full trace |
| **API** | `POST /api/ask`, `GET /api/search`, `GET /api/node/{id}`, `GET /api/health` |
| **UI** | Single page: question box, example questions, answer with citation chips, tool trace timeline, graph view |
| **Evals** | 20 golden questions across 5 categories; metrics report in JSON and Markdown |
| **Quality** | pytest suite, ruff lint and format, GitHub Actions CI with an eval gate |
| **Deploy** | Dockerfile; Kubernetes Deployment, Service, ConfigMap and Secret template, tested on `kind` |

### Out of scope, planned for later
- Neo4j backend (the store interface is ready for it)
- Hybrid search (BM25 plus vectors) and re-ranking
- Streaming the trace live to the UI
- A full Reactome import script
- An LLM-as-judge faithfulness score in CI
- Cloud deployment (AKS or EKS)

---

## 7. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR1 | Load `data/pathways.json`; fail fast on a missing node, dangling edge or unknown type | Must |
| FR2 | `search_entities(query, k, types?)` returns the top-k nodes with scores | Must |
| FR3 | `get_neighbors(node_id, direction, edge_types?)` returns adjacent nodes and edge types | Must |
| FR4 | `find_path(source_id, target_id)` returns an ordered list of reaction steps (substrate → enzyme → product) along directed edges, ignoring cofactors | Must |
| FR5 | `get_inhibitors(node_id)` returns inhibitors of an enzyme, or of all enzymes in a pathway | Must |
| FR6 | `get_downstream(node_id, max_steps)` returns reactions and metabolites that lose their input if the node is blocked | Must |
| FR7 | The agent answers only from tool results and cites node IDs in `[node_id]` form | Must |
| FR8 | The agent says it can't answer from the graph when tools return nothing relevant | Must |
| FR9 | The agent stops after at most 6 tool steps and returns a partial answer with a note | Must |
| FR10 | `/api/ask` returns the answer, citations, trace, subgraph, token usage and latency | Must |
| FR11 | The UI renders citations as chips colored and shaped by node type; clicking one highlights the node in the graph | Should |
| FR12 | The eval runner writes `evals/results/<timestamp>.json` and `report.md` | Must |
| FR13 | CI runs lint, tests and the offline retrieval eval; it fails if recall@5 < 0.80 | Must |
| FR14 | The LLM provider is chosen by config: `ollama` (default), `openai`, `azure` or `fake` | Must |

---

## 8. Non-functional requirements

| Area | Target |
|---|---|
| **Latency** | p95 under 8 s per question with a hosted model; search under 100 ms |
| **Cost** | Token usage logged per question; the eval report shows the total |
| **Reliability** | Tool errors are returned to the model as messages, never crash the request; LLM call timeout 30 s |
| **Security** | API keys only from environment or a Kubernetes Secret; never committed; `.env` git-ignored |
| **Testability** | All tool logic tested without an LLM; the agent loop tested with a scripted fake LLM |
| **Portability** | Runs offline for tests and CI (TF-IDF plus fake LLM); no key needed to run `make test` |
| **Accessibility** | UI meets WCAG AA contrast; node types use color **and** shape **and** a label |

---

## 9. Success metrics (reported in README)

| Metric | Definition | MVP target |
|---|---|---|
| Retrieval recall@5 | Share of gold nodes found in the top 5 search results | ≥ 0.85 (OpenAI embeddings), ≥ 0.80 (TF-IDF, CI gate) |
| Tool selection accuracy | Share of questions where the expected tools were called | ≥ 90% |
| Answer accuracy | Share of questions whose answer contains every required fact | ≥ 80% |
| Refusal accuracy | Share of out-of-scope questions correctly declined | 100% |
| Citation validity | Share of cited IDs that exist and appeared in tool results | 100% |
| Latency p50 / p95 | Wall-clock time per `/ask` | Reported |
| Tokens per question | Prompt plus completion tokens | Reported |

The README records **two runs**: a baseline, and one improvement (a better tool description, or a different `k`), with the difference explained. That before/after comparison is the story to tell in interviews.

---

## 10. Example questions (golden set seeds)

| Category | Example |
|---|---|
| Lookup | "What reaction does phosphofructokinase-1 catalyze?" |
| Multi-hop path | "How is glucose converted to pyruvate? List the enzymes in order." |
| Cross-pathway | "Which metabolite is shared by glycolysis and the shikimate pathway?" (answer: phosphoenolpyruvate) |
| Inhibition / impact | "What does glyphosate inhibit, and which product can no longer be made downstream?" (EPSP synthase → EPSP → chorismate) |
| Out of scope | "How does insulin regulate glycogen synthase?" (glycogen metabolism isn't in the graph, so it must refuse) |

---

## 11. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Curated data contains a biology error | Every node cites a Reactome or ChEBI ID; a validation script checks the structure; spot-check against Reactome |
| The model skips tools and answers from memory | System prompt requires tool use; eval checks tool selection and citation validity |
| TF-IDF recall is too low for the CI gate | Add synonyms to node text (e.g. "PFK-1", "phosphofructokinase 1"); log it in MEMORY.md |
| One day isn't enough | TASKS.md marks a cut line: drop K8s manifests, then LLM-judge, then the demo GIF. Never drop evals. |
| API cost during eval runs | Use a small, low-cost chat model; cache embeddings to disk |

---

## 12. Future roadmap
1. Neo4j backend behind the same `GraphStore` interface
2. Full Reactome import for more pathways
3. Hybrid retrieval with re-ranking; compare the eval scores
4. Stream the trace to the UI with server-sent events
5. Nightly LLM-judge eval in CI using a secret key
6. Deploy to AKS with Azure OpenAI
