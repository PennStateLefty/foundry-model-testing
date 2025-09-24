Decisions / Implementation Notes (Iteration 1)
--------------------------------------------
1. Added dependencies to `pyproject.toml` per spec: azure-identity, azure-ai-inference (for Foundry / inference SDK), openai, aiohttp, python-dotenv.
2. Notebook name and location: `src/openai-gpt5-notebook.ipynb` exactly as required.
3. Environment variables expected (standard naming):
	- AZURE_OPENAI_ENDPOINT
	- AZURE_OPENAI_API_VERSION
	- AZURE_OPENAI_MODEL
4. Will load `.env` using python-dotenv if present; otherwise rely on existing environment.
5. Authentication strategy: `DefaultAzureCredential()` (no extra parameters) assuming environment / developer login set up.
6. Raw API test: use `aiohttp` POST to `{AZURE_OPENAI_ENDPOINT}/openai/responses?api-version={AZURE_OPENAI_API_VERSION}` with bearer token from `get_token("https://cognitiveservices.azure.com/.default")`.
7. OpenAI SDK test: configure `openai.AzureOpenAI` client (from new openai >=1.x) with `api_key` acquired via Azure AD token (set temporary via callback) or directly pass `azure_ad_token` if supported; simplest: supply `azure_ad_token` using credential token value and base_url = endpoint.
8. Azure Foundry SDK: using `azure.ai.inference` (ResponsesClient) with `AzureAIInferenceCredential` or fallback using provided `DefaultAzureCredential` if library supports; minimal example created with `from azure.ai.inference import Client` (beta interface) — if import path differs in future, will adjust in later iteration.
9. Scope for token: `https://cognitiveservices.azure.com/.default` reused for all calls.
10. Minimal prompt used: "Say hello briefly." to keep cost low.
11. No extra error handling added beyond simple prints to keep implementation minimal per spec.
12. Added `.env` template file at project root with placeholder values for required variables.
13. Added `.env` to `.gitignore` and created `.env.example` for safe publication to GitHub.
14. Populated `README.md` with project description, uv init/restore instructions, environment variable guidance, and troubleshooting.
15. Added `rich` dependency for colored terminal output used by rprint in notebook.
16. Modified raw Responses API cell to use SSE streaming with incremental delta handling and synthesized final result.
17. Converted OpenAI SDK responses cell to SSE streaming using client.responses.stream with delta accumulation and final usage summary.
18. Deferred Azure Foundry SDK code cells; requirement marked deferred in spec with rationale (focus on raw + OpenAI SDK streaming first).

Iteration 2 - Model Comparison Notebook
--------------------------------------
19. Implemented `src/model-comparison-notebook.ipynb` per plan focusing exclusively on RAW Responses API calls (no OpenAI SDK wrapper) to ensure uniform surface across models.
20. Added user configuration cell with: MODEL_NAMES, PROMPT, MAX_OUTPUT_TOKENS, TEMPERATURE, retries, timeout, parallel toggle, export toggle, ranking key, concurrency cap.
21. Environment variable resolution pattern: for each logical model name `X`, expect `AZURE_<X>_MODEL`; fail-fast if any missing along with shared `AZURE_OPENAI_ENDPOINT` and `AZURE_OPENAI_API_VERSION`.
22. Token acquisition: Cached bearer token via `DefaultAzureCredential` against scope `https://cognitiveservices.azure.com/.default` with 60s pre-expiration refresh window.
23. Asynchronous execution: `asyncio` + `aiohttp` with optional parallel execution guarded by a semaphore sized to model count (or explicit cap).
24. Retry logic: Exponential backoff (2**attempt seconds) on 429 and >=500 statuses up to `RETRIES` additional attempts; surfaces final error with status & payload text.
25. Usage extraction: Defensive function supporting both canonical `input_tokens`, `output_tokens`, `total_tokens`, `reasoning_tokens` and legacy `prompt_tokens`, `completion_tokens` plus nested `input_tokens_details.reasoning_tokens`.
26. Text extraction: Handles both `output[0].content[0].text` style and potential `choices[0].message.content` fallback for spec evolution resilience.
27. Ranking: Computed by `total_tokens` (configurable) among successful responses only; relative delta percentage vs best total included when calculable.
28. Reporting: Rich table with styling (✅ for best, ❌ for worst, 🧠 for reasoning tokens) and plain-text fallback if `rich` unavailable.
29. Export: Optional timestamped JSON artifact of raw result records for external analysis / future trend tracking.
30. Truncation: Response excerpts limited to 120 chars; full text omission flagged via config to reduce notebook clutter.
31. Edge cases: Missing usage leads to warning row without halting; timeouts and repeated failures create error rows preserving latency window.
32. Deferred features (explicit): pricing, streaming partial deltas, multi-run statistical sampling, historical persistence beyond single JSON export, cost normalization.
33. No additional dependencies required beyond those already declared in `pyproject.toml`; pandas intentionally omitted (current pure-Python summarization sufficient).
34. Maintained spec alignment: All acceptance criteria for Model Comparison feature addressed with traceable implementation sections.
35. Future adaptation note: If Responses API schema shifts to streaming-first or alters token field names, only `extract_usage` and `extract_text` helpers should need modification.

Iteration 3 - Dual Chat & Reasoning Support Plan
-----------------------------------------------
36. Introduce per-model configuration via `MODEL_PROFILES` dict mapping logical name -> profile (mode, effort, max_reasoning_tokens).
37. Backward compatibility: if only `MODEL_NAMES` defined, auto-generate profiles with `mode='chat'`.
38. Request construction refactored into `build_body(profile)` injecting `reasoning` object only when `mode == 'reasoning'`.
39. Reasoning object fields: `effort` (default 'medium'), optional `max_reasoning_tokens` when provided.
40. Response parsing already tolerant of reasoning tokens; will add additional fallback keys (e.g., `output_reasoning_tokens`).
41. Extend result records with: `mode`, `reasoning_effort`, `reasoning_enabled`, `notes` list, plus warning note if reasoning requested but no reasoning tokens returned.
42. Reporting table expanded: new columns `Mode` and `Effort`; reasoning tokens column shows `0 ⚠️` if missing when enabled.
43. Ranking remains by `total_tokens` by default; unchanged logic skipping rows with missing totals.
44. Validation: pre-validate reasoning `effort` in {low, medium, high}; fallback to 'medium' with note if invalid.
45. Edge cases: Non-reasoning model with reasoning request -> treat as normal error if API 400; record error row.
46. Memory and plan updated before applying notebook code modifications to maintain traceability.
47. Deferred still: pricing, streaming partial reasoning traces, cost metrics, multi-run statistical variance.
48. Risk mitigation: All reasoning-related logic isolated to config + `build_body` + reporting; minimal impact on existing token parsing infrastructure.
49. Added event-loop safe orchestration: detect running IPython/Jupyter loop; if active, use `await run_all()` (with a guard) instead of `asyncio.run()` to avoid RuntimeError. Included optional `nest_asyncio` application for broader compatibility.
50. Iteration 4: Migrated notebook to unified Responses API endpoint (`POST /openai/v1/responses`) with `model` in request body; retained `USE_LEGACY_DEPLOYMENT_PATH` flag for backward compatibility.
51. Updated `build_url` to normalize various endpoint base forms and synthesize `/openai/v1/responses` when not using legacy deployment path.
52. Extended request body builder to always include `model` (deployment name) making legacy path tolerant to the extra field.
53. Enhanced usage extraction to probe `usage.output_tokens_details.reasoning_tokens` per latest documentation while preserving previous fallbacks.
54. Added clearer markdown guidance distinguishing unified vs legacy modes and when to flip the flag.

