# Evaluation

## What exists today

**There is no automated test suite.** The repository contains no test files, no CI configuration, and no evaluation scripts. No quality or performance metrics are produced anywhere in the code; any figures previously attached to this project were unverifiable and have been removed from the documentation.

The only quantitative signals the app itself emits are per-generation **latency** and the **winning provider name**, both measured in `LearningPathAgent.process` / `MetaReviewerAgent.process` and displayed in the "Generation Details" panel. They are shown to the user and not recorded anywhere.

## Edge cases the code visibly handles

Enumerated from `competancy-matrix.py`:

- **Provider failure fallback.** Each LLM call (both agents, both providers) is wrapped in its own try/except; a failing provider surfaces as `st.error` and the other provider's output is used. If both fail, the agent returns a stub (`"No learning path generated."` / `"No review generated."`).
- **Weak-critique guard.** The critic's output replaces the draft only if its length is ≥ 80% of the draft's, protecting against the reviewer returning a short critique instead of a full revision.
- **Search failure.** `WebSearchAgent` and `VideoSearchAgent` catch all exceptions and return `[]`; the UI renders "No web resources found" / "No video guides found" rather than crashing.
- **Serper timeout.** The Serper POST carries a 10-second timeout. (The Tavily and LLM calls have no explicit timeout.)
- **Result deduplication.** Web results are deduplicated by URL and capped at 10; videos capped at 5; missing titles default to "Untitled".
- **Malformed Excel upload.** Parsing is wrapped in try/except with a sidebar error; the app degrades to a default function list and free-text competency/role inputs.
- **Missing inputs.** Clicking Generate without all three selections shows a warning instead of calling any API.
- **Defensive result access.** Rendering uses `.get(...)` with defaults throughout (content, provider, latency, references).

## What the code does **not** handle

- No retries or backoff on any network call.
- No timeout on LLM or Tavily calls (a hung call hangs the Streamlit run).
- No validation that the generated Markdown actually contains the three required tables or the "General Suggestions" section.
- No verification that URLs emitted by the LLM inside the learning-path tables exist (search-agent URLs are real; LLM-authored URLs are unchecked).
- No schema validation of the uploaded Excel beyond "the three columns exist" (a `KeyError` on missing columns is caught by the generic handler).

## Proposed evaluation harness

*This section is a design proposal; none of it exists in the repository.*

**Golden dataset.** 20–30 triples of `(function, competency, role)` spanning the shipped defaults (HR, IT, Finance) plus free-text edge cases (unusual roles, long strings, non-English input). Store as a checked-in CSV with, per row, the structural expectations below.

**Deterministic structural gates** (pytest, no LLM required to grade):
- Output contains exactly three level sections, each with a Markdown table bearing the `Module / Course | URL | Description` header.
- A "General Suggestions" section and a concluding paragraph are present.
- Every table row's URL parses as an absolute `http(s)` URL.
- Reference payload shape: each level has `web` (≤10, unique URLs) and `videos` (≤5, `youtube.com/watch` URLs).

**Link-liveness gate.** HEAD-request every URL (search-agent and LLM-authored); fail the run if more than a threshold (e.g. 20%) of LLM-authored URLs are dead — this directly measures the hallucinated-link risk noted in ARCHITECTURE.md.

**LLM-judge rubric** (scored 1–5, run per golden row): level-appropriateness of modules, role relevance, progression coherence between levels. Gate on mean ≥ 4 with no row below 3. Keep judge prompts and scores in version control so regressions are diffable.

**Unit seams.** The agent classes take their clients via constructor, so `LearningPathAgent`, `MetaReviewerAgent`, and `WebSearchAgent` are mockable today without refactoring: assert the pick-longer selection, the 80% critique guard, deduplication, and the empty-result fallbacks with stub clients.

**Operational metrics worth recording** (currently displayed but discarded): per-stage latency, provider win-rate, provider error-rate. A one-line append to a local JSONL per generation would make the latency claims in any future documentation checkable.
