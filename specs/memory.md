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
