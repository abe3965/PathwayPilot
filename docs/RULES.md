# PathwayPilot — Rules

The rules for building this project, for a human or for an AI coding assistant. When a rule and a shortcut conflict, follow the rule and log the conflict in `MEMORY.md`.

---

## 1. Ground rules
1. **Public data only.** Never use Corteva data, employer data, or any non-public dataset. Every node in `pathways.json` must be checkable against Reactome, ChEBI or a textbook source.
2. **Evals are not optional.** Every change that affects retrieval, prompts or tools gets an eval run, and the result goes in `MEMORY.md`.
3. **Read the docs first.** Before starting a task, read `TASKS.md` (what's next), `ARCHITECTURE.md` (where it goes) and the relevant part of this file.
4. **Update the log.** After each task: tick it in `TASKS.md` and add any decisions or bugs to `MEMORY.md`.
5. **No secrets in git.** Keys come from the environment or a Kubernetes Secret only. `.env` is in `.gitignore`.
6. **Small commits.** One task (or less) per commit, using Conventional Commits.

---

## 2. Approved libraries

| Purpose | Use | Version guidance |
|---|---|---|
| Web API | `fastapi`, `uvicorn[standard]` | Latest stable; pin in lockfile |
| Validation & config | `pydantic` v2, `pydantic-settings` | v2 APIs only (`model_validate`, `model_dump`) |
| Graph | `networkx` | |
| LLM + embeddings | `openai` | Use `OpenAI` / `AzureOpenAI` clients; chat completions with `tools` |
| Offline retrieval | `scikit-learn` (`TfidfVectorizer`), `numpy` | |
| HTTP (tests) | `httpx` via FastAPI `TestClient` | |
| Tests | `pytest` | |
| Lint / format | `ruff` | |
| Types (optional) | `mypy` | Non-blocking in the MVP |
| Frontend graph | `cytoscape` (UMD from cdnjs, pinned exact version) | No npm build step |

### Not allowed in the MVP, and why
| Library | Reason |
|---|---|
| **LangChain, LlamaIndex, CrewAI, Semantic Kernel** | Write the agent loop by hand (about 100 lines). You need to explain tool calling, message history and step caps in interviews, and a framework hides exactly that. A framework comparison can be a later task. |
| Vector databases (Pinecone, Chroma, pgvector, …) | 100 vectors fit in a NumPy array. Mention the upgrade path in the README. |
| React / Vite / any frontend build | One HTML page doesn't need a build pipeline |
| Streamlit / Gradio | The UI must use the real API, the same way any client would |
| Neo4j driver | Post-MVP (see PRD roadmap) |

Adding any other dependency needs a one-line justification in `MEMORY.md`.

---

## 3. Python coding standards

### Style
- Python **3.11+**. Use modern syntax: `list[str]`, `X | None`, `match` where it reads better.
- `ruff format` (line length 100) and `ruff check` must pass. Rule sets: `E, F, I, B, UP, SIM, RUF`.
- **Type hints on every function signature.** Return types included.
- Names: `snake_case` functions and variables, `PascalCase` classes, `UPPER_SNAKE` constants.
- Docstrings on public functions: one line on *what* it does, plus args or returns if they aren't obvious. No docstrings that just repeat the name.
- Keep functions short (about 40 lines or fewer). If one grows past that, split it.

### Structure
- Follow the dependency direction: `api → agent → tools → graph / retrieval → models / config`. **Never import upward.**
- `graph/queries.py` holds **pure functions**: input store and arguments, output Pydantic models. No logging side effects, HTTP or LLM calls.
- **Pydantic models at every boundary:** API request and response, tool arguments, tool results, config, data file.
- No module-level mutable globals. Singletons live on `app.state`, built in the FastAPI lifespan.
- Config is read **only** through `Settings`. Never call `os.getenv` elsewhere.

### Errors
- Define domain errors in `models.py`: `NodeNotFound`, `DataValidationError`, `ToolArgumentError`, `LLMUnavailable`.
- Tool code raises domain errors. `tools.dispatch` catches them and returns `{"error": "<helpful message>"}` to the model.
- Never write a bare `except:`. Catch specific exceptions only.
- Error messages say **what went wrong and what to do next**, e.g. `"Node 'enz_pfk' not found. Call search_entities to find valid IDs."`

### Logging
- Use stdlib `logging` with one JSON formatter configured in `api/main.py`.
- Log one summary line per request (see ARCHITECTURE §10). Never log API keys or full prompts at INFO level; full prompts go at DEBUG only.

### LLM and agent rules
- `temperature=0` for the agent and for evals.
- Tool descriptions are part of the product. Write them like API docs: what the tool does, when to use it, what each argument means, and one example.
- The system prompt lives in `agent/prompts.py` as a constant. Log every prompt change in `MEMORY.md` with its eval before and after.
- The agent must cite as `[node_id]`, and only IDs returned by tools count.
- Keep the provider-specific code inside `agent/llm.py`; nothing else should know which provider is in use.

---

## 4. Testing standards
- **Every module in `src/` has a test file.** Tests run with **no network and no API key.**
- Use the `fake` LLM provider (scripted tool calls) to test the agent loop, and the TF-IDF embedder to test retrieval.
- Keep a small fixture graph in `conftest.py` (about 10 nodes) so query tests are readable.
- Test names describe behavior: `test_find_path_ignores_cofactor_edges`, not `test_path_2`.
- Required tests for the MVP:
  - Data loads; a dangling edge raises `DataValidationError`
  - `find_path` glucose → pyruvate returns 9 reactions in order (aldolase feeds glyceraldehyde-3-phosphate straight into GAPDH, so the triose phosphate isomerase step is a side branch and not on the shortest path)
  - `find_path` ignores `COFACTOR_OF` edges
  - `get_inhibitors` works for an enzyme and for a whole pathway
  - `get_downstream` from EPSP synthase reaches chorismate
  - Search returns an inhibitor node for the query "glyphosate"
  - The agent loop stops at `MAX_STEPS` and sets `truncated=True`
  - The agent drops citations that weren't in tool results
  - `/api/ask` returns 422 for an empty question
- CI must pass before merging to `main`.

---

## 5. Evaluation standards
- `evals/golden.jsonl` is the source of truth. **Never edit a gold answer to make a failing run pass** unless the gold answer itself was wrong, and log that in `MEMORY.md`.
- The golden set has 20 questions covering all 5 categories: lookup (4), path (5), cross-pathway (3), impact / inhibition (5), out of scope (3).
- Every run writes a timestamped JSON file. The README table shows the latest baseline and the best run.
- The CI gate is offline recall@5 ≥ 0.80. Raise it as quality improves, but never lower it without a note in `MEMORY.md`.

---

## 6. Frontend standards
- One `index.html`, one `styles.css`, one `app.js`. No framework.
- All colors, fonts and spacing come from CSS custom properties defined in `DESIGN.md`.
- `app.js` only renders what the API returns. No business logic in the browser.
- Accessible by default: real `<button>` and `<label>` elements, visible focus, `aria-live="polite"` on the answer region, and a result for keyboard users as well as mouse users.
- Escape all text from the API before inserting it into the DOM (use `textContent`, never `innerHTML` with API strings).

---

## 7. Git and repo standards
- Branch: `main` is always green. Feature branches are named `feat/<short-name>`.
- Commit style: Conventional Commits: `feat:`, `fix:`, `test:`, `docs:`, `refactor:`, `chore:`, `ci:`.
  - e.g. `feat(agent): add get_downstream tool`
- `.gitignore` covers: `.env`, `.cache/`, `evals/results/*.json`, `__pycache__/`, `.venv/`.
- The README stays current. It's the first thing a reviewer sees.

---

## 8. Docker and Kubernetes standards
- Base image `python:3.11-slim`; run as a non-root user; no build tools in the final image.
- Copy only `src/`, `web/`, `data/` and the lockfile or pyproject.
- `HEALTHCHECK` hits `/api/health`.
- K8s: every container has resource requests and limits, plus readiness and liveness probes.
- The image is tagged with a version (`0.1.0`), never `latest` in manifests.

---

## 9. Definition of done (for every task)
- [ ] Code follows these rules; `ruff check` and `ruff format --check` pass
- [ ] Tests added or updated; `pytest` passes offline
- [ ] If retrieval, prompts or tools changed: eval run and result logged in `MEMORY.md`
- [ ] `TASKS.md` ticked
- [ ] Committed with a Conventional Commit message

---

## 10. Using an AI coding assistant on this repo
- Point it at this file and `ARCHITECTURE.md` at the start of each session.
- Give it **one task ID from `TASKS.md` at a time.**
- Review every diff before committing. You must be able to explain every line in an interview.
- Ask it to add `MEMORY.md` entries for any decision it made that isn't already in the docs.
