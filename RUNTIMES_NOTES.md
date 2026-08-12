# Running gpt-researcher through llm_runtimes

`llm_runtimes/` (vendored at the repo root) exposes an OpenAI-compatible
server for two keyless chat backends:

- `claudecli-sonnet` / `claudecli-opus` / `claudecli-haiku` — routed through
  the local `claude -p` CLI (subscription auth, no API key).
- `local-qwen` — an in-process vLLM engine (Qwen3-8B-AWQ by default).

No routing-specific code changes are needed. The stock config already supports
this setup:

- `Config.parse_llm` only validates the provider prefix (`openai:` is in
  `_SUPPORTED_PROVIDERS`); the model name after the colon is passed through
  unchanged (`gpt_researcher/config/config.py`).
- The `openai` provider honors `OPENAI_BASE_URL` in both
  `gpt_researcher/llm_provider/generic/base.py` and
  `gpt_researcher/utils/llm.py`.

## Start the server

```bash
python -m llm_runtimes.server --port 8399
# or embedded, from any process in this repo:
#   from llm_runtimes import ensure_server; ensure_server()  # -> http://127.0.0.1:8399/v1
```

Quick check: `curl http://127.0.0.1:8399/v1/models`

## Env recipes

### A. Usual hosted APIs (upstream default, needs keys)

```bash
export OPENAI_API_KEY=sk-...          # real key
export TAVILY_API_KEY=tvly-...        # default retriever
# defaults: FAST_LLM=openai:gpt-5.4-mini, SMART_LLM/STRATEGIC_LLM=openai:gpt-5.4,
#           EMBEDDING=openai:text-embedding-3-small, RETRIEVER=tavily
```

### B. Claude CLI via the runtime server (keyless)

```bash
export OPENAI_BASE_URL=http://127.0.0.1:8399/v1
export OPENAI_API_KEY=llm-runtimes    # any non-empty value; ChatOpenAI just requires presence
export FAST_LLM=openai:claudecli-haiku
export SMART_LLM=openai:claudecli-sonnet
export STRATEGIC_LLM=openai:claudecli-sonnet   # or openai:claudecli-opus
export EMBEDDING=huggingface:sentence-transformers/all-MiniLM-L6-v2
export RETRIEVER=duckduckgo
```

### C. Local Qwen via the runtime server (keyless, needs a free GPU)

Same as B, but:

```bash
export FAST_LLM=openai:local-qwen
export SMART_LLM=openai:local-qwen
export STRATEGIC_LLM=openai:local-qwen
```

The first request loads the model into vLLM (slow); pre-warm with
`python -m llm_runtimes.server --port 8399 --preload-local`.

## Embeddings: keep them OFF the runtime server

The runtime server has no `/v1/embeddings` endpoint. Because the `openai`
embedding provider also honors `OPENAI_BASE_URL`
(`gpt_researcher/memory/embeddings.py`), leaving `EMBEDDING` at its default
`openai:text-embedding-3-small` while `OPENAI_BASE_URL` points at the runtime
server sends embedding calls to a 404. Always override `EMBEDDING`.

Keyless options (both already supported by `Memory`):

- **Recommended:** `EMBEDDING=huggingface:sentence-transformers/all-MiniLM-L6-v2`
  — fully local SentenceTransformers on CPU. Requires two extra packages that
  are not in `requirements.txt`:
  `pip install langchain-huggingface sentence-transformers`
  (first use downloads the ~90 MB model from the HF Hub, then it is cached).
- Alternative: `EMBEDDING=ollama:nomic-embed-text` with
  `OLLAMA_BASE_URL=http://127.0.0.1:11434` if an Ollama daemon is running.

## Retriever: keyless options

Default is `tavily`, which needs `TAVILY_API_KEY`. Keyless choices (validated
against the directory names under `gpt_researcher/retrievers/`):

- `RETRIEVER=duckduckgo` — general web search via the `ddgs` package (already
  in `requirements.txt`). No key. Occasional rate limiting; the retriever
  degrades to an empty result set rather than crashing.
- `RETRIEVER=arxiv` — academic papers only, no key.
- Combinable: `RETRIEVER=duckduckgo,arxiv`.
- `searx` also works keyless but needs a `SEARX_URL` pointing at an instance.

## Smoke test

```bash
python -m llm_runtimes.server --port 8399 &   # once per machine
export OPENAI_BASE_URL=http://127.0.0.1:8399/v1 OPENAI_API_KEY=llm-runtimes \
       FAST_LLM=openai:claudecli-haiku SMART_LLM=openai:claudecli-sonnet \
       STRATEGIC_LLM=openai:claudecli-sonnet \
       EMBEDDING=huggingface:sentence-transformers/all-MiniLM-L6-v2 \
       RETRIEVER=duckduckgo
python cli.py "What is retrieval augmented generation?" --report_type research_report
# report is written to outputs/
```

Verified end-to-end on 2026-08-12 with recipe B (claudecli-haiku for all three
LLM roles): duckduckgo retrieval, local MiniLM embeddings, and report writing
all completed, producing a ~25k-character cited report.

## Code changes on this branch

One unrelated pre-existing bug blocked any use of the package:
`gpt_researcher/actions/query_processing.py` referenced `Any`/`List` in
function annotations before the `typing` import (the imports sit below the
function), so `import gpt_researcher` raised `NameError` at module load.
Fixed additively with a single `from __future__ import annotations` line,
which defers annotation evaluation without reordering anything.

## Limitations

- **Fake streaming**: the server buffers the whole completion and emits it as
  one SSE chunk, so "streamed" report output appears all at once.
- **Usage/cost tracking is meaningless** for runtime models: the server
  returns zero token usage, and gpt-researcher's cost fallback estimates with
  tiktoken at OpenAI prices. Ignore the reported costs.
- `temperature`, `max_tokens`, and `REASONING_EFFORT` are ignored by the
  `claudecli-*` backend (accepted on the wire, not forwarded to `claude -p`).
  `local-*` honors temperature/max_tokens.
- Tool calling is limited to the server's single-tool prompt-flattening shim;
  the default web research path does not use it, but the MCP retriever's
  `bind_tools` usage may be unreliable through runtime models.
- `claudecli-*` latency is high (one `claude -p` subprocess per request) and
  concurrent sub-queries fan out into parallel CLI invocations; deep/detailed
  report types will be slow.
- Do not point `OPENAI_BASE_URL` at the runtime server while also relying on
  real OpenAI models or embeddings in the same process; the base URL is
  global to the `openai` provider.

## Test run (2026-08-12, overnight)

- Command: `python cli.py "How reliable is LLM-as-a-judge evaluation, and what are the main approaches to quantifying and improving its uncertainty and calibration (2024-2026)?" --report_type research_report`
- Backends: SMART/STRATEGIC=claudecli-sonnet, FAST=claudecli-haiku, embeddings=local MiniLM, retriever=duckduckgo+arxiv. No API keys.
- Simulated human input: the research question above (chosen to match the maintainer's research area).
- Output: `runs/2026-08-12-llm-judge-calibration/` (34 sources, ~21k chars, PDF included). Wall time ~13 min.
