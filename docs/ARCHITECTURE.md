# Architecture

Everything lives in one file, `competancy-matrix.py` (~450 lines): a custom lightweight agent framework, four agent implementations, and the Streamlit UI. This document maps what is actually there.

## Component map

| Component | Location | Responsibility |
|---|---|---|
| `Agent` | `competancy-matrix.py` | Abstract base; one method, `process(query)` |
| `Memory` | `competancy-matrix.py` | Append-only list of `{query, learning_path}` records |
| `Team` | `competancy-matrix.py` | Orchestrator: generator → critic → per-level reference gathering |
| `LearningPathAgent` | `competancy-matrix.py` | Calls GPT-4 Turbo and Claude 3.5 Sonnet with the same prompt; returns the longer response with provider name and latency |
| `MetaReviewerAgent` | `competancy-matrix.py` | Same dual-provider loop, prompted to critique and improve the draft |
| `WebSearchAgent` | `competancy-matrix.py` | Tavily search (5 results) + Serper POST (5 results), URL-deduplicated, capped at 10 |
| `VideoSearchAgent` | `competancy-matrix.py` | `youtube_search` scrape (10 results), mapped to title/URL/views, capped at 5 |
| `main()` | `competancy-matrix.py` | Streamlit page: Excel upload, three-step selection, generation button, two-column results |

## Data flow, end to end

1. **Input.** The user optionally uploads an `.xlsx` with columns `Function`, `Competency`, `Role`; `main()` pivots it into a nested dict driving three cascading selectboxes. Without an upload, a hard-coded function list and free-text inputs are used.
2. **Prompt construction.** On button click, a fixed prompt template interpolates the three selections and demands three Markdown tables (Module/Course, URL, Description) plus a "General Suggestions" section.
3. **Client construction.** OpenAI, Anthropic, and Tavily clients and a Serper config dict are built from env vars — freshly, on every click.
4. **Generation.** `Team.process_query` runs `LearningPathAgent.process`: each provider is called in sequence inside try/except; each response is timed; the longer text wins.
5. **Critique.** The draft is wrapped in a "review and critique … suggest improvements" prompt and sent through `MetaReviewerAgent` (same dual-provider, pick-longer logic). The critic's output replaces the draft **only if** its length ≥ 80% of the draft's — a guard against the critic returning a short critique instead of a full revision.
6. **Reference gathering.** For each of `beginner`/`intermediate`/`advanced`, a query string `"{level} level {function} {competency} {role} training"` is sent to `WebSearchAgent` and `VideoSearchAgent`.
7. **Store + render.** The result is appended to `Team.memory`, put in `st.session_state.current_result`, and rendered: learning path (with `unsafe_allow_html=True`), latency/provider metrics, and per-level reference expanders.

## Orchestration analysis: what runs parallel vs sequential

Nothing is parallel. One button click performs, strictly in sequence:

- 2 LLM calls (generator: OpenAI, then Anthropic)
- 2 LLM calls (critic: OpenAI, then Anthropic)
- 3 × (Tavily + Serper + YouTube) = 9 search calls

Total wall-clock time is the sum of all 13 network calls. The calls are mutually independent within each stage — the two provider calls could race concurrently, and all nine reference calls could fan out — but the code is synchronous and Streamlit's script-rerun model makes plain `for` loops the path of least resistance. This is the single largest latency lever in the system (see "Extending this system").

There is also no cross-provider truncation control: Anthropic calls set `max_tokens` (4000 generator / 2000 critic) while OpenAI calls set none, which biases the pick-the-longer heuristic in ways that are provider-default-dependent.

## State and context engineering

- **`Memory`** is an in-process list on the `Team` instance. Because `Team` is constructed inside the button-click branch, memory lives for exactly one generation — it is written once and never read. It is a seam for persistence, not a working memory.
- **`st.session_state.current_result`** carries the latest result across Streamlit reruns so the page redraws without regenerating.
- **`st.session_state.history`** is *read* by the "Document Versions" expander but never *written* anywhere — the version-history UI is a dormant feature and always shows "No previous versions."
- **Context assembly** is one-shot: a single user prompt per LLM call, no system prompt, no conversation history, no retrieved context injected into generation. The reference-gathering stage is independent of the generated text — searches key off the raw Function/Competency/Role selections, not off the modules the LLM proposed. Context is bounded implicitly by the fixed prompt template; a large chunk of the "General Suggestions" content is verbatim boilerplate embedded in the prompt rather than model-generated.

## Design decisions and trade-offs visible in the code

- **Custom micro-framework over LangChain/LangGraph.** The `Agent`/`Team`/`Memory` trio is ~30 lines and fully inspectable. Trade-off: no retries, no streaming, no callbacks, no tracing — all of which the heavier frameworks would provide.
- **Length as the quality metric.** Both the dual-provider selection and the accept-the-critique decision compare `len(content)`. Cheap and deterministic, but it rewards verbosity and cannot detect a wrong or malformed table.
- **Dual-provider redundancy as availability.** If one provider errors, its exception is caught, shown via `st.error`, and the other provider's output is used. Availability is the real benefit of the dual calls; "best result" is only nominal given the length heuristic.
- **Coupled web-search error domain.** Tavily and Serper calls share one try/except, so a Tavily exception discards any Serper results that would have succeeded (and vice versa is moot — Serper runs second). Fine for a demo; a production system would isolate them.
- **Prompted-URL references vs searched references.** The LLM is asked to include URLs in its tables (hallucination-prone), while verifiable URLs arrive separately from the search agents. The two are never reconciled.
- **Clients rebuilt per click.** Simple and safe under Streamlit's rerun model; costs a little setup time and precludes connection reuse.

## Extending this system

Grounded in the seams that already exist:

1. **Parallelize the fan-out.** The generator's two provider calls and the nine reference calls are independent. Wrapping them in `concurrent.futures.ThreadPoolExecutor` (the clients are synchronous, so threads suffice) would cut wall-clock latency to roughly the slowest single call per stage — likely a 3–5× improvement — with no behavioral change.
2. **Make the critic the judge.** `MetaReviewerAgent` already receives the draft; prompting it for a structured verdict (JSON rubric: table completeness, URL plausibility, level coverage) and selecting on that verdict would replace both `len()`-based decisions with an actual quality signal, at zero new infrastructure.
3. **Wire up the dormant version history.** The UI for `st.session_state.history` already renders versions and reload buttons; appending each result to it (and persisting `Team.memory` to disk or SQLite) turns an existing dead code path into a working feature.
4. **Validate references before display.** The search agents already return URLs; a HEAD-request liveness check plus reconciliation against the URLs the LLM emitted in its tables would ground the learning path in verified links — the architecture already separates generation from retrieval, which makes this insertion point clean.
5. **Externalize model configuration.** Model IDs, `max_tokens`, and temperature are hard-coded at four call sites. Lifting them to env vars (the `.env` pattern is already in place) allows model upgrades and A/B provider tests without code edits, and lets the two providers be given symmetric token budgets.
