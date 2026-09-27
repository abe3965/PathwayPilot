# PathwayPilot — Memory

The running log of decisions, bugs, eval runs and lessons during the build. Newest entries go **at the top** of each section.

**Why keep this:** it's your interview prep. "Tell me about a technical decision you made" or "tell me about a bug you fixed" is answered straight from this file.

**Rules for updating**
- Add an entry whenever you choose between options, fix a non-trivial bug, change a prompt or tool, or run an eval
- Keep entries short: what, why, and what you'd do differently
- Never delete an entry; if a decision is reversed, add a new entry that points to the old one

---

## 1. Decision log

Format:
```
### D-XX · YYYY-MM-DD · <short title>
- **Context:** what problem needed a decision
- **Options:** what was considered
- **Decision:** what was chosen
- **Why:** the main reason
- **Revisit if:** the condition that would change it
```

### D-09 · 2026-09-27 · `ollama` is the default LLM provider, not `fake` or `openai`
- **Context:** The config default needs to be something that actually answers questions out of the box, without a paid key. `fake` only replays scripted tool calls (fine for tests, not for real use); `openai` needs a paid API key.
- **Options:** (a) `openai` default; (b) keep `fake` default; (c) `ollama` default
- **Decision:** (c). `LLM_PROVIDER=ollama` is now the default in `Settings`, Docker docs and the README quickstart order. `fake` is still used explicitly by the test suite and CI (RULES.md §4) — that doesn't change.
- **Why:** keeps the whole stack runnable with free, local, open-weight models and no API key, matching "public data only" and the open-source-first direction for this project. `openai`/`azure` remain available for a hosted comparison run (see R-01/R-02 in §3).
- **Revisit if:** local tool-calling model quality is too unreliable for the golden-set eval; then note the tradeoff here and consider `openai` as the default for the demo build while keeping `ollama` as the OSS path.

### D-08 · 2026-09-27 · Glucose → pyruvate path is 9 reactions, not 10
- **Context:** Glycolysis has 10 reactions, but aldolase produces both DHAP and glyceraldehyde-3-phosphate (G3P). The shortest path goes F1,6BP → G3P directly and skips triose phosphate isomerase (DHAP → G3P).
- **Options:** (a) force TPI onto the path; (b) accept the 9-step shortest path and let the agent mention TPI via `get_neighbors`
- **Decision:** (b). The test expects 9 steps; the golden answer for "list the glycolysis steps" accepts either framing.
- **Why:** the graph is correct as modeled; forcing TPI would mean special-case code
- **Revisit if:** users expect "all 10 steps" answers; then add a `list_pathway_reactions(pathway_id)` tool

### D-07 · 2026-09-27 · Offline TF-IDF embedder as the default
- **Context:** Tests and CI must run without an API key
- **Options:** (a) mock the embedding API; (b) a real local fallback
- **Decision:** (b) TF-IDF via scikit-learn, selected by `EMBEDDER=tfidf`
- **Why:** real, deterministic retrieval in CI makes the recall gate meaningful; mocks would always pass
- **Revisit if:** TF-IDF recall can't reach 0.80 even after adding synonyms

### D-06 · 2026-09-27 · CI gate on offline recall@5 ≥ 0.80
- **Context:** Quality should be protected automatically, but full agent evals need a paid key
- **Decision:** CI runs only the offline retrieval eval and fails below 0.80; the full agent eval runs locally (and later on a manual workflow with a secret)
- **Why:** cheap, deterministic, and still catches the most common regression (someone edits node text and breaks search)
- **Revisit if:** a key can be added to CI with a budget cap

### D-05 · 2026-09-27 · No agent framework; hand-written loop
- **Context:** LangChain/LlamaIndex could build the agent faster
- **Decision:** Write the tool-calling loop directly with the OpenAI SDK (~100 lines)
- **Why:** the point of the project is to show an understanding of tool calling, message history, step caps and failure handling. A framework hides those.
- **Revisit if:** the project grows to multi-agent orchestration

### D-04 · 2026-09-27 · One SDK for all providers
- **Decision:** `openai` SDK for OpenAI, Azure OpenAI (`AzureOpenAI`) and Ollama (OpenAI-compatible `base_url`), plus a `fake` provider
- **Why:** one code path; switching providers is config only
- **Revisit if:** a provider's tool-calling format diverges

### D-03 · 2026-09-27 · Reactions modeled as nodes
- **Context:** Edges could be metabolite → metabolite labeled with the enzyme, or reactions could be nodes
- **Decision:** Reactions are nodes (`SUBSTRATE_OF`, `PRODUCES`, `CATALYZES`, `PART_OF`)
- **Why:** handles reactions with several substrates (S3P + PEP → EPSP); one shortest-path call returns the reactions; enzymes are one hop away; inhibition attaches cleanly to enzymes
- **Revisit if:** the graph grows past ~10k nodes and path queries slow down

### D-02 · 2026-09-27 · NetworkX instead of Neo4j for the MVP
- **Decision:** In-memory NetworkX `DiGraph` behind a `GraphStore` interface
- **Why:** zero setup, fits the one-day window, easy to test. Neo4j is already on the resume; the interface leaves room to add it.
- **Revisit if:** data grows beyond memory, or for the Neo4j stretch task (T10.3)

### D-01 · 2026-09-27 · Public data only; three pathways
- **Decision:** Glycolysis, TCA cycle, shikimate pathway (+ PDH link), curated from Reactome and ChEBI
- **Why:** no employer data; small enough to verify by hand; PEP connects glycolysis and shikimate (cross-pathway questions); the glyphosate → EPSP synthase story is relevant to agriculture
- **Revisit if:** adding a Reactome import script (roadmap)

---

## 2. Bug log

Format:
```
### B-XX · YYYY-MM-DD · <symptom in one line>
- **Symptom:** what you saw
- **Root cause:** why it happened
- **Fix:** what changed (file / commit)
- **Test added:** name of the test that now guards it
- **Lesson:** one line
```

*(No bugs logged yet. Likely candidates to watch for: the model inventing node IDs in citations; `find_path` routing through ATP/NAD⁺ if cofactor edges aren't excluded; index cache not rebuilding after a data change; dark-mode colors not applied to Cytoscape.)*

---

## 3. Eval run log

| Run | Date | Change from previous | Embedder | Model | Recall@5 | Tool acc. | Answer acc. | Refusal acc. | Citation valid. | p50 / p95 (s) | Tokens / q |
|---|---|---|---|---|---|---|---|---|---|---|---|
| R-00 | | Offline baseline | tfidf | — | | — | — | — | — | — | — |
| R-01 | | Full baseline | | | | | | | | | |
| R-02 | | *one change, e.g. better tool descriptions* | | | | | | | | | |

**Notes per run** (what failed and why):
- R-01: …
- R-02: …

---

## 4. Prompt and tool-description versions

| Version | Date | What changed | Linked eval run |
|---|---|---|---|
| P-1 | | Initial system prompt | R-01 |

---

## 5. Data changes

| Date | Change | Reason | Eval impact |
|---|---|---|---|
| | Initial curation: ~N nodes, ~M edges | | |

---

## 6. Open questions
- Should `find_path` also return alternative routes (e.g. through TPI), or only the shortest?
- Is keyword matching enough for answer accuracy, or does it need the LLM judge (T10.1)?
- Which chat model gives the best accuracy per dollar on this eval? (Try 2 models if time allows.)

---

## 7. Interview talking points (fill in at the end)
1. **Hardest bug:** …
2. **Decision I'd defend:** hand-written agent loop (D-05) — …
3. **What the eval taught me:** R-01 → R-02 moved answer accuracy from __% to __% because …
4. **What I'd do next with a week:** …
