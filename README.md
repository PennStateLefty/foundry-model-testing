## Foundry Model Testing

Minimal test project for exercising an Azure OpenAI (e.g., GPT-5) deployment through three access paths:
1. Raw Responses API via `aiohttp`
2. OpenAI Python SDK against an Azure endpoint (Responses API)
3. Azure Foundry / Inference SDK (`azure-ai-inference`)

Primary artifact: `src/openai-gpt5-notebook.ipynb`.

---
## Prerequisites
- Python >= 3.11
- [uv](https://github.com/astral-sh/uv) (Python project & dependency manager)
- Azure account access with permission to use the target Azure OpenAI / Foundry resource
- An Azure OpenAI deployment (model name referenced in `AZURE_OPENAI_MODEL`)

Install uv (macOS / Linux):
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```
Ensure `~/.local/bin` (installer output path) is on your `PATH` if needed.

---
## Environment Variables
Copy the example file and edit real values:
```bash
cp .env.example .env
```
Required entries:
- `AZURE_OPENAI_ENDPOINT` (no trailing slash, e.g. https://my-resource.openai.azure.com)
- `AZURE_OPENAI_API_VERSION` (e.g. 2024-06-01)
- `AZURE_OPENAI_MODEL` (deployment/model name)

These are loaded automatically in the notebook via `python-dotenv`.

---
## First-Time Project Initialization
From the repository root:
```bash
# Install / resolve dependencies into an isolated environment
uv sync

# (Optional) Upgrade a dependency later
uv add <package>
```
`uv sync` will create / reuse a `.venv` (or uv-managed environment) and install all packages listed in `pyproject.toml`.

> If you prefer an explicit virtual environment name/location you can create one first: `uv venv` (then run `uv sync`).

---
## Restoring on a New Machine
Clone and install:
```bash
git clone <your-fork-or-repo-url>.git
cd foundry-model-testing
cp .env.example .env  # then edit values
uv sync
```
Open the notebook in VS Code or run a Jupyter server:
```bash
uv run jupyter notebook  # or: uv run jupyter lab
```

---
## Running the Notebook in VS Code
1. Open the folder in VS Code.
2. Ensure the Python extension is installed.
3. Run `uv sync` (once) in the integrated terminal.
4. Open `src/openai-gpt5-notebook.ipynb` and select the environment (it should auto-detect the uv/venv Python).
5. Execute cells top-to-bottom.

---
## Updating Dependencies
Add a new package:
```bash
uv add package-name
```
Remove a package:
```bash
uv remove package-name
```
Upgrade all (respecting constraints):
```bash
uv lock --upgrade
uv sync
```

---
## Notes
- Authentication uses `DefaultAzureCredential()`; ensure you're logged in (e.g. `az login`) or have environment-based credentials.
- Token scope used: `https://cognitiveservices.azure.com/.default`.
- `.env` is git-ignored; never commit real values.

---
## Troubleshooting
| Issue | Check |
|-------|-------|
| Missing env vars error | Confirm `.env` exists & names match exactly |
| Auth failures | Run `az login` or configure managed identity / VS Code Azure sign-in |
| ImportError (jupyter) | Install Jupyter: `uv add notebook` (if not already available) |
| 401 from raw API | Ensure Azure role assignment for your identity and correct endpoint/api version |

---
## License
Add a license file if you intend to publish (none included yet).

