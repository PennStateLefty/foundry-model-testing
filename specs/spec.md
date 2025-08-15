# Objective
This specification outlines the minimum requirements and features needed to create a Jupyter Notebook based test to exercise different models in Azure AI Foundry. 
---
## Requirements
1. This project will use uv as the Python project management tool. All dependencies will be added to the pyproject.toml file in the correct format
2. This project will use Jupyter Notebooks as the primary coding and execution environment.
  - The first notebook should be openai-gpt5-notebook.ipynb
  - This should exist in a src folder under the project root directory
3. Interactions with Azure AI Foundry and the models in it will be done via Entra ID. Be sure to use the azure-identity libraray
4. The notebook will need to test model interactions using at least three mechanisms
  - The Azure Foundry SDK (or whatever package can call Foundry and invoke Responses APIs)
  - The OpenAI SDK (must be able to call a Responses API)
  - Raw APIs - so aiohttp most likely
5. Environment variables should be loaded from the project or .env and contain - Azure OpenAI endpoint, Azure OpenAI API version, and Azure OpenAI model name - THESE SHOULD USE STANDARD ENVIRONMENT VARIABLE NAMING CONVENTIONS
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