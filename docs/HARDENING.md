# Hardening

## Current posture

- **Authentication: none.** The Streamlit app is open to anyone who can reach the port. Every page view can trigger paid LLM and search-API calls on the operator's keys.
- **Secrets handling: correct at HEAD.** All four API keys are read from environment variables via `python-dotenv`; no credentials are committed. There is no `.gitignore`, so an accidentally created `.env` would be one `git add .` away from being committed — the `.gitignore` added alongside this document closes that.
- **Error handling: user-facing only.** Exceptions are caught and rendered as `st.error` banners. There is no logging, so failures leave no trace for the operator; a hung LLM or Tavily call (no timeout) blocks the run indefinitely.
- **Observability: none.** No logs, no metrics export, no tracing. Latency and provider are computed per generation but only displayed, never recorded.
- **Injection surface:** the generated learning path is rendered with `st.markdown(..., unsafe_allow_html=True)` — model output (which itself embeds unvetted search-derived context in future extensions) is injected as raw HTML into the page. Uploaded Excel cell values flow directly into LLM prompts and search queries without sanitization.
- **Dependencies:** `requirements.txt` mixes pinned and unpinned entries, contains duplicates (`langchain`, `python-dotenv`, `serpapi` appear twice with and without pins), and lists many packages the app never imports (pyautogen, autogen, langchain/langgraph, pandasai + extras, chromadb, pinecone, faiss-cpu, semantic-router, …). This inflates install time and attack surface, and the pinned LangChain 0.1.x line is old. Meanwhile `openpyxl`, which `pd.read_excel` needs, is absent. The `youtube_search` package scrapes YouTube rather than using an official API — a fragility and terms-of-service consideration.
- **Cost control: none.** One click costs four LLM calls plus nine search calls; there is no rate limiting, quota, or caching.

## Ladder to production

### Stage 1 — Identity and keys
- Put the app behind authentication: Streamlit Community Cloud viewer allowlist, an OAuth reverse proxy, or at minimum a shared-secret gate — anything that stops anonymous spend on the operator's API keys.
- Move keys from `.env` files to a secrets manager appropriate to the deployment target (Streamlit `secrets.toml`, cloud key vault); scope each key minimally (e.g. OpenAI project-scoped keys).
- Keep the repository's `.gitignore` covering `.env*` (except `.env.example`) and any local data files.

### Stage 2 — Reliability and monitoring
- Add explicit timeouts to the OpenAI, Anthropic, and Tavily calls (only Serper has one today) and bounded retries with backoff for transient failures.
- Replace `st.error`-only handling with structured logging (stdlib `logging`, JSON formatter) covering provider errors, latencies, and win-rates; ship logs wherever the deployment runs.
- Render model output with `unsafe_allow_html=False` (Streamlit's Markdown covers the tables the prompt requests) to close the raw-HTML injection path.
- Split the Tavily/Serper try blocks so one search provider's failure does not discard the other's results.

### Stage 3 — Deployment
- Prune `requirements.txt` to the nine actual runtime dependencies, pinned, with `openpyxl` added; generate a lockfile (pip-tools or uv).
- Containerize (slim Python base, non-root user) and run behind TLS; Streamlit itself should not be internet-facing without a proxy.
- Add per-session and per-day generation caps, and cache results keyed on `(function, competency, role)` — identical inputs currently re-spend the full 13-call budget.
- Wire the evaluation harness proposed in [EVALUATION.md](EVALUATION.md) into CI as a pre-deploy gate.

### Stage 4 — Compliance and data governance
- Uploaded Excel files may contain internal role taxonomies; document that their contents are sent to two LLM providers and two search APIs, and gate uploads accordingly (or add an on-request redaction step).
- Pin and review the data-handling terms of each provider (OpenAI, Anthropic, Tavily, Serper) against the operating organization's requirements; replace the `youtube_search` scraper with the official YouTube Data API for terms-of-service cleanliness.
- Add dependency and secret scanning (Dependabot/`pip-audit`, gitleaks) to CI.

## Secrets audit

A scan of the tree at HEAD found no committed credentials: API keys are referenced only via `os.getenv`, and no `.env`, key, or certificate files are present. Nothing required redaction.
