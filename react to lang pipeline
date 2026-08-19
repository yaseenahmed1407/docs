# docs-pipeline (`bh-dev`) — Single-Language (Python) Migration Analysis

**Repo:** OpenAgriNet/docs-pipeline, branch `bh-dev`
**Scope of this pass:** every file in the branch, not just `pipeline/`

## 1. What language the branch is actually in today

The backend is already Python. The "polyglot" surface is smaller than it might feel from the docker-compose file:

| Component | Language | Size | Role |
|---|---|---|---|
| `pipeline/` | Python 3.10 (FastAPI + Temporal SDK) | ~11,600 lines across ~35 files | API, worker, OCR/translation/chunking/domain-tagging providers, auth, DB, vector store adapters, master catalog |
| `scripts/` | Python (18 files) + Bash (6 files) + 1 PowerShell file | small | Ops tooling: reindex, backfill, Keycloak bootstrap, backup/deploy, test uploads |
| `lang-detect/` | **TypeScript / Node.js** (Express + `franc-min`) | 1 file, ~120 lines | Standalone HTTP microservice: language detection before translation |
| `ui/` | **JavaScript/JSX** (React 18 + Vite) | ~50 files | Operator console SPA (dashboard, review, search workbench, admin) |
| `keycloak/import/*.json` | config, not code | — | Realm/role definitions |

So there are really only **two non-Python islands**: the `lang-detect` Node service, and the React UI. Everything else (API, worker, Temporal activities, all six processing providers, DB layer, auth, ops scripts) is already one language.

## 2. `lang-detect` — the real "second language" to fix

### What it does
A tiny Express server with two endpoints (`/detect`, `/detect/batch`) that runs `franc-min` (an n-gram statistical language guesser) over text, then maps ISO 639-3 → 639-1 codes for the languages this pipeline cares about (Hindi, Gujarati, Marathi, Tamil, Telugu, Kannada, Malayalam, Punjabi, Bengali, Odia, Urdu, Sanskrit, Nepali, English).

It is called from exactly one place: `pipeline/translation/service.py::detect_page_languages()`, which does an `httpx` POST to `{LANG_DETECT_URL}/detect/batch` with a list of text lines per page, then folds the results down to a single "non-English winner" language per page before deciding whether to translate.

It is deployed as its own container in `docker-compose.yml`, health-checked, and explicitly called out as horizontally scalable (`docker compose up -d --scale lang-detect=3`) — i.e. it was built as an independent, stateless microservice, not as a library, likely so OCR/translation-heavy workers wouldn't need Node in their image.

### Why it's a good migration candidate
- It's pure CPU-bound text classification with no state, no GPU, no external API calls — a textbook "should just be a function" component.
- Python has direct equivalents to `franc`: `py3langid` / `langid.py` (~97 languages, no dependencies, similar n-gram approach to franc), `lingua-language-detector` (higher accuracy, especially on short text, includes Hindi/Bengali/Tamil/etc. but check exact language coverage against your list), or `fasttext`'s `lid.176` model (176 languages including gu/mr/ta/te/kn/ml/pa/bn/or/ur — closest breadth match to franc, and the repo already ships `torch`/`transformers` so adding a small fastText model is cheap).
- Removing it deletes an entire Docker image, a Node/TS toolchain, a `Dockerfile`, health-check wiring, a `depends_on` edge on both `api` and `worker`, and one network hop per translation batch.

### Three ways to re-home it, and the trade-offs

**Option A — In-process library call (recommended default).**
Replace the `httpx.post(.../detect/batch)` call in `detect_page_languages()` with a direct Python function call (`py3langid.classify(text)` or similar), wrapped in a small `pipeline/langdetect/` module mirroring the existing `pipeline/ocr/`, `pipeline/translation/` provider pattern (a `base.py` interface + one concrete implementation, so it stays swappable). Runs inside the `worker` process, no network hop, no extra container, no health check to maintain. This matches how the rest of the pipeline already treats CPU-only text utilities (chunking's deterministic splitter, tokenization) — they're library calls, not services.
*Trade-off:* loses independent horizontal scaling of language detection specifically — but since it's sub-millisecond CPU work per page and already bottlenecked by the GPU-backed OCR/translation steps upstream/downstream, that scaling knob was never the bottleneck.

**Option B — Keep it as an HTTP microservice, rewritten in Python (FastAPI).**
Same container topology, same `/detect` and `/detect/batch` contract (so `LANG_DETECT_URL` and the existing `httpx` call in `service.py` don't need to change at all — only the service's own Dockerfile becomes `python:3.11-slim` + `fastapi`/`uvicorn` + `py3langid`), if you specifically want to keep it independently deployable/scalable or want to reuse it from something other than this worker later.
*Trade-off:* keeps a network hop and a container for something that doesn't need one; only worth it if another consumer (e.g. a future ingestion service) is actually expected to call it.

**Option C — Expose it as an agent "tool" call.**
Relevant only if the pipeline evolves from a deterministic Temporal workflow into something where an LLM is making runtime decisions (e.g., "should this page be re-OCR'd, retranslated, or routed to a human?" decided by a model rather than fixed retry logic). In that world you'd wrap the same Python function in a tool schema (JSON-schema args/return, e.g. via MCP or a function-calling tool definition) so an agent could call `detect_language(text) -> {lang, confidence}` as one of several tools alongside `run_ocr`, `translate_page`, etc.
*Trade-off:* today nothing in this codebase is agentic — Temporal workflows call fixed activity functions in a fixed order with fixed retry policies (`pipeline/workflows.py`/`activities.py`), there's no LLM deciding control flow. Building a tool-call interface now would be speculative scaffolding with no caller. **Recommendation: don't do this yet** — but Option A's module boundary (`pipeline/langdetect/base.py` interface) is deliberately the same shape a tool wrapper would need, so this stays a cheap follow-on if/when an agent loop is introduced, not a rewrite.

**Bottom line for `lang-detect`: Option A.** It deletes a language, a container, a Dockerfile, and a network hop, for a component that is 120 lines of stateless text classification with one caller.

## 3. The React UI — technically the second non-Python component, but a different kind of decision

`ui/` is a full operator console: 9 views, Keycloak SSO (PKCE), PDF viewer, tables, review workflows, ~50 component files, Tailwind + Radix UI. This is not a "wrapper around a Python function" the way `lang-detect` is — it's the browser-side rendering layer, and browsers execute JavaScript, not Python.

If "single language" is meant to include the UI, the honest options are:
- **Keep React** (recommended). This is a full-featured, permission-aware, multi-tab admin console. Python-native web UI frameworks (Reflex, NiceGUI, Streamlit, Dash) exist but are a much bigger rewrite (~50 component files, routing, PDF viewer, Keycloak PKCE flow, Radix-based design system) for a framework family that is generally weaker for this class of dense, stateful admin UI than React + a component library. This would be a multi-week rewrite for no functional gain.
- **If the goal is narrower** — e.g. "no Node.js in the deploy/build story" rather than literally "no JavaScript anywhere" — that's already almost true: Node is only needed to `vite build` the UI (build-time) and to run `lang-detect` (runtime). Removing `lang-detect` per §2 means Node is only a *build tool* for static assets, not a runtime language your services execute in. That's a meaningfully different (and much cheaper) claim than "rewrite the UI in Python," and is probably what's actually motivating the ask.

**Recommendation:** treat "single language" as "single *runtime service* language" (Python for every long-running process) rather than "no JavaScript file exists in the repo." Fixing `lang-detect` gets you there; rewriting the UI does not pay for itself.

## 4. Ops scripts — small cleanup, not urgent

`scripts/` has 6 Bash files and 1 PowerShell file (`upload_test_docs.ps1`) alongside 18 Python files doing the same class of work (bootstrap, reindex, transfer, backup). These are genuinely low-risk, low-effort conversions if consistency matters:

- `ingest_test_docs.sh`, `upload_test_docs.ps1` → trivial to fold into one Python `click`/`argparse` CLI script (`scripts/upload_test_docs.py`) using `httpx`, removing the OS-specific duplication (today you maintain the same "upload PDFs to the API" logic twice, once per OS).
- `backup_volumes.sh`, `deploy-compose.sh`, `export-prod-images.sh`, `transfer-prod-to-dev.sh`, `create_marqo_passage_index.sh` are mostly thin wrappers around `docker`/`rsync`/`ssh`/`curl` — portable to Python via `subprocess`, but arguably *idiomatic* to leave as shell since they're literally sequences of shell commands for ops runbooks. Converting these buys consistency, not capability.

**Recommendation:** convert `upload_test_docs.ps1` + `ingest_test_docs.sh` into one Python script (removes actual duplication); leave the docker/rsync/ssh orchestration scripts as Bash unless the team specifically dislikes maintaining shell.

## 5. What's *not* actually a language problem

Worth naming so it doesn't get bundled into "re-engineer into Python" by accident: `pipeline/ocr/chandra_vllm.py`, `pipeline/translation/gemma_vllm.py`, `pipeline/domain_tags/gemma_tagger.py`, `pipeline/chunking/qwen_vllm.py`, and `pipeline/ocr/mistral_ocr.py` all call *external* model-serving endpoints (vLLM, Mistral API) over HTTP from Python already — that's correct as-is; those model servers are separate deployables by necessity (GPU inference), not a language-consistency issue.

## 6. On the "library / API / agent-tool" framing specifically

Mapping every processing step in the current architecture to that trichotomy:

| Step | Today | Right shape |
|---|---|---|
| Language detection (`lang-detect`) | External HTTP microservice (Node) | **Library call** (Python function inside `worker`) — see §2 |
| OCR (Chandra/Mistral) | HTTP call to external vLLM/Mistral | **API call** — correct as-is, GPU inference belongs outside the app process |
| Translation (Gemma) | HTTP call to external vLLM | **API call** — correct as-is |
| Domain tagging (Gemma) | HTTP call to external vLLM | **API call** — correct as-is |
| Chunking (Qwen) | HTTP call to external vLLM | **API call** — correct as-is |
| Workflow control flow | Temporal activities, fixed order, fixed retries | Deterministic orchestration, **not agentic** — no LLM decides "what happens next" today |

The general rule this suggests: things that are pure CPU logic with no external dependency (language ID, deterministic chunking, tokenization) should be **library calls** in-process; things that require a model server (OCR, translation, tagging, embeddings) should stay **API calls** to that server, regardless of what language the caller is written in; **agent tool-calls** only make sense once/if you introduce an LLM that chooses *which* of these steps to invoke and in what order — which this pipeline deliberately does not do today (it's a reviewed, deterministic Temporal state machine by design, per `docs/SYSTEM_DESIGN.md` §1 non-goals: "Real-time chat / low-latency LLM product surface" is explicitly out of scope). If that changes later, the migration in §2 Option A already leaves the code in the right shape (a typed function with a clean interface) to wrap as a tool with minimal extra work.

## 7. Suggested migration order

1. **Replace `lang-detect` with an in-process Python module** (Option A, §2). Delete the `lang-detect/` directory, its `Dockerfile`, its `docker-compose.yml` service block, `LANG_DETECT_URL` env plumbing (or keep the env var but make it a no-op / config toggle for a brief transition period), and the `depends_on: lang-detect` edges on `api` and `worker`. Update `detect_page_languages()` to call the new module directly. This is the one change that actually removes a runtime language.
2. **Fold `upload_test_docs.ps1` into a single Python script** alongside the existing `ingest_test_docs.sh` logic, to remove the Windows/Unix duplication.
3. **Leave `ui/` as React** and treat "single language" as achieved at the *service* level once step 1 lands — Node remains only as a UI build-time tool, not a running service.
4. Optionally, decide as a team whether the remaining ops shell scripts (`backup_volumes.sh`, `deploy-compose.sh`, etc.) are worth porting to Python for consistency; this is cosmetic, not architectural.

Net effect of steps 1–2: one Docker service removed, one language (TypeScript/Node runtime) eliminated from the deployed system, no behavior change to the translation pipeline's inputs/outputs, and the codebase converges to "every long-running process is Python" without touching the UI.
