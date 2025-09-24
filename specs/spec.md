# Objective
This specification outlines the minimum requirements and features needed to create a Jupyter Notebook based test to exercise different models in Azure AI Foundry. 
---
## Requirements
1. This project will use uv as the Python project management tool. All dependencies will be added to the pyproject.toml file in the correct format
2. This project will use Jupyter Notebooks as the primary coding and execution environment.
  - The first notebook should be openai-gpt5-notebook.ipynb
  - The second notebook should be model-comparison-notebook.ipynb
  - This should exist in a src folder under the project root directory
3. Interactions with Azure AI Foundry and the models in it will be done via Entra ID. Be sure to use the azure-identity libraray
4. The openai-gpt5-notebook.ipynb notebook will need to test model interactions using at least three mechanisms
  - The Azure Foundry SDK (or whatever package can call Foundry and invoke Responses APIs)
  - The OpenAI SDK (must be able to call a Responses API)
  - Raw APIs - so aiohttp most likely
5. The model-comparison-notebook.ipynb notebook will be used to compare token consumption
  - The same prompt will be sent against numerous models
  - The models will be enumerated in the .env file using the convention AZURE_{MODEL_NAME}_MODEL
  - The notebook will begin with a cell that has a collection of MODEL_NAME values coded in by the notebook user
  - All tokens (input, output, and reasoning if applicable) will be tracked and kept in a data structure that is output in an ASCII formatted report (with colors and any emoji applicable)
  - Use the RAW APIs for all of the models (assume OpenAI APIs running in Azure Foundry. Reference [this API spec](https://learn.microsoft.com/en-us/azure/ai-foundry/openai/latest) for the API secification you should use.
6. Environment variables should be loaded from the project or .env and contain - Azure OpenAI endpoint, Azure OpenAI API version, and Azure OpenAI model name - THESE SHOULD USE STANDARD ENVIRONMENT VARIABLE NAMING CONVENTIONS
---
## Features
# Feature: GPT-5 Access and Model Types
# Feature: GPT-5 Access and Model Types
- [x] Create `openai-gpt5-notebook.ipynb` in the `src` folder
- [x] Create a cell that describes in Markdown what imports are to be made, what environment variables are picked up, and how we will get identity to drive the notebook
- [x] Create a cell that does the imports, populates any needed environment variables from .env or similar, and gets a Default Credential from Azure (assume the platform running the code will be logged into Azure so DefaultCredential() should work)
- [x] Create cells (Markdown to describe, Python implementation) to test the API with raw API calls (Responses API)
- [x] Create cells (Markdown to describe, Python implementation) to test the models using the OpenAI SDK but using an Azure OpenAI model deployed in Foundry (get the endpoint and model name from environment variables)
- [ ] (Deferred) Create the cells (Markdown to describe, and Python implementation) to test the models with the Azure Foundry SDK
  - Deferred rationale: Current iteration focuses on raw Responses API and OpenAI SDK streaming paths. Foundry SDK cell removed intentionally pending future need.

# Feature: Model Comparison

## Plan: `model-comparison-notebook.ipynb`

### 1. Goal
Create a reusable Jupyter notebook that compares token consumption, latency, and (optionally later) qualitative response traits across multiple Azure OpenAI (Foundry) model deployments using only RAW Responses API calls authenticated with Entra ID (DefaultAzureCredential). Output a colorized ASCII summary table for quick at-a-glance efficiency assessment.

### 2. Assumptions
1. All compared models are Azure OpenAI deployments accessible via the same endpoint base (single Foundry endpoint) and API version.
2. Each model deployment name is exposed via environment variable: `AZURE_<MODEL_NAME>_MODEL` (e.g., `AZURE_GPT5_MODEL`, `AZURE_GPT4o_MODEL`).
3. Shared environment variables:
   - `AZURE_OPENAI_ENDPOINT` (ex: `https://my-foundry-endpoint.openai.azure.com`)
   - `AZURE_OPENAI_API_VERSION` (ex: `2024-10-01-preview`) – must match a Responses API-supported version.
4. Access token scope: `https://cognitiveservices.azure.com/.default`.
5. Notebook runner is already logged into Azure (so `DefaultAzureCredential()` works without interactive prompts).
6. The Responses API returns usage metadata including `usage.input_tokens`, `usage.output_tokens`, optional `usage.reasoning_tokens` or nested details (will defensively probe several possible fields per evolving spec).
7. Pricing or cost metrics are out-of-scope for first iteration (may add later).

### 3. Environment Variables & Validation
On startup, validate presence of:
* `AZURE_OPENAI_ENDPOINT`
* `AZURE_OPENAI_API_VERSION`
* For each `MODEL_NAME` in user list: `AZURE_<MODEL_NAME>_MODEL`
Failure mode: collect missing vars and raise a concise error cell early (fail-fast) with remediation hints.

### 4. User-Editable Configuration (First Code Cell)
Variables the notebook user can adjust:
* `MODEL_NAMES = ["GPT5", "GPT4o", "GPT4o_MINI"]` (example placeholders)
* `PROMPT = "Explain the principle of least action in 3 concise bullet points."`
* `MAX_OUTPUT_TOKENS = 512` (or `None` for default)
* `TEMPERATURE = 0.2`
* `REQUEST_TIMEOUT_SECONDS = 60`
* `RETRIES = 2` (for transient HTTP 429 / 5xx)
* `PARALLEL = True` (whether to issue calls concurrently with asyncio)

### 5. Architecture & Flow
1. Load config & env vars.
2. Acquire access token once per run (refresh if expired for longer sessions; initial iteration can fetch per call if simpler, but plan for caching object with expiration field).
3. For each model:
   - Resolve deployment name from env var.
   - Construct POST URL: `{AZURE_OPENAI_ENDPOINT}/openai/deployments/{deployment}/responses?api-version={version}`
   - Build JSON body:
     ```json (structure reference only, not code)
     {
       "input": [ { "role": "user", "content": [ { "type": "text", "text": PROMPT } ] } ],
       "temperature": TEMPERATURE,
       "max_output_tokens": MAX_OUTPUT_TOKENS
     }
     ```
   - Send request (aiohttp) with headers: `Authorization: Bearer <token>`, `Content-Type: application/json`, `Accept: application/json`.
   - Capture timings: start/end monotonic for latency ms.
   - Parse response JSON for:
     * Text output (first candidate or aggregated depending on shape: often `output[0].content[0].text` or `choices[0].message.content`).
     * Token usage fields (see Section 7).
   - Handle & record errors (HTTP status, message) without aborting entire batch unless all fail.
4. Aggregate results into an in-memory list of dicts.
5. Derive metrics (efficiency ranking, relative % vs best total tokens).
6. Render ASCII table with colors & emoji.

### 6. Data Structures
`ResultRecord` (conceptual):
```
{
  "model_name": str,              # Friendly name from MODEL_NAMES list
  "deployment": str,              # Env-resolved deployment id
  "status": str,                  # "ok" | "error"
  "http_status": int | None,
  "latency_ms": float | None,
  "input_tokens": int | None,
  "output_tokens": int | None,
  "reasoning_tokens": int | 0 | None,
  "total_tokens": int | None,     # computed if not provided
  "response_excerpt": str | None, # first ~120 chars for context
  "error": str | None
}
```
Stored in `results: List[dict]` then optionally converted to pandas DataFrame for sorting & formatting (pandas only if we decide the dependency is worthwhile; can start with pure Python).

### 7. Token Usage Extraction Strategy
Probe in order (first match wins):
1. `usage` root keys: `input_tokens`, `output_tokens`, `total_tokens`, `reasoning_tokens`.
2. Nested forms (defensive): `usage.prompt_tokens`, `usage.completion_tokens` (legacy naming), `usage.input_tokens_details.reasoning_tokens`.
3. If only input/output found: compute total = input + output + reasoning (if any).
4. If no usage object: mark tokens as None and flag in `error` or `notes`.

### 8. Concurrency Model
If `PARALLEL=True`, use `asyncio.gather` with a semaphore limiting simultaneous requests (default permit = len(MODEL_NAMES) or a configurable `MAX_IN_FLIGHT`). Provide sequential fallback for easier debugging.

### 9. Retry & Error Handling
Retry on transient statuses: 429, 500, 502, 503, 504 with exponential backoff: base 1s * (2 ** attempt). Do not retry on 4xx (except 429). Capture final failure reason. Timeouts produce `status="error"` and `http_status=None`.

### 10. Reporting (ASCII + Colors + Emoji)
Library preference: `rich` (color, table). If not available, degrade gracefully to plain text.
Columns (final order):
`Rank | Model | Input | Output | Reasoning | Total | Latency(ms) | ΔTotal% | Status | Notes`
Formatting rules:
* Lowest total: green bold ✅
* Highest total: red bold ❌
* Reasoning tokens > 0: add 🧠 in Reasoning column.
* Errors: entire row dim + Status = ERROR (red).
* ΔTotal% = (model_total - best_total)/best_total * 100 (omit if tokens unavailable).
Provide summary footer: best model, average latency, any models missing usage data.

### 11. Notebook Cell Outline
1. Markdown: Purpose & high-level description.
2. Markdown: Environment variable expectations & naming pattern.
3. Code: User config (MODEL_NAMES, PROMPT, etc.).
4. Code: Imports & optional dependency check (rich, aiohttp, azure-identity, asyncio, time, json, os, dataclasses (if used)).
5. Code: Load & validate env vars (fail-fast listing missing ones).
6. Code: Credential acquisition (DefaultAzureCredential test token fetch) + helper to get bearer token.
7. Code: Helper functions
   - `build_url(deployment)`
   - `extract_usage(json)`
   - `truncate(text)`
   - `request_model(model_name, deployment, prompt)` (async)
8. Code: Orchestrator (async) to run all models (parallel or sequential), produce `results`.
9. Code: Post-processing (compute ranks, relative percentages).
10. Code: Reporting (rich table or fallback).
11. Code: (Optional) Export results to JSON file with timestamp.
12. Markdown: Future enhancements & pricing integration placeholder.

### 12. Edge Cases
* Missing env variable for one model: exclude that model with warning but continue others.
* All models missing: abort early.
* Partial usage object: log note, compute what is possible.
* API version mismatch (HTTP 404/400): surface clearly.
* Rate limiting bursts: retries then final failure.
* Large output truncated for summary but full retained if we later export.

### 13. Dependencies (planned additions to `pyproject.toml` later — not added now)
* `azure-identity`
* `aiohttp`
* (Optional) `rich`
* (Optional) `pandas` (defer unless needed; initial ASCII can be pure Python)

### 14. Acceptance Criteria Mapping
Requirement: Same prompt to multiple models -> Implemented via shared `PROMPT` and iteration.
Requirement: Models enumerated via `AZURE_<MODEL_NAME>_MODEL` -> Resolution & validation logic.
Requirement: User-provided MODEL_NAME list in first cell -> Provided.
Requirement: Track input/output/reasoning tokens -> Extraction strategy Section 7.
Requirement: ASCII colored report -> Reporting strategy Section 10.
Requirement: Use RAW APIs with Azure Foundry spec -> URL & body design Section 5.
Requirement: Load env vars (endpoint, api version, model names) -> Sections 3 & 4.

### 15. Future Enhancements (Not in current scope)
* Cost estimation (USD) via loaded pricing table.
* Support for streaming responses & partial token accrual.
* Statistical sampling (multiple runs, variance & stability metrics).
* Prompt templating & batch scenarios.
* Persisting historical comparisons for trend analysis.

---
End of Model Comparison Plan.

