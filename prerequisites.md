# Workshop Prerequisites

Participants should have the following set up **before** the workshop day.

---

## Required Accounts & Access

- [ ] **Azure subscription** with access to Azure OpenAI Service
- [ ] **Azure OpenAI resource** deployed with a `gpt-4o-mini` (or `gpt-4o`) model
  - Note the endpoint URL and deployment name
- [ ] **Azure CLI** installed and logged in (`az login`)

## Development Environment

### Option A: C# / .NET (Primary)

- [ ] [.NET 9 SDK](https://dotnet.microsoft.com/download/dotnet/9.0) or later
- [ ] IDE: Visual Studio 2022 (17.12+) or Visual Studio Code with C# Dev Kit
- [ ] Git installed

### Option B: Python

- [ ] Python 3.11 or later
- [ ] `pip` package manager
- [ ] IDE: Visual Studio Code with Python extension
- [ ] Git installed

## Verify Your Setup

### .NET

```bash
# Verify .NET
dotnet --version          # Should be 9.0+

# Verify Azure CLI
az --version
az login
az account show           # Confirm correct subscription

# Quick test — create and run a test project
dotnet new console -n TestSetup
cd TestSetup
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI --prerelease
dotnet run                # Should build without errors
```

### Python

```bash
# Verify Python
python --version          # Should be 3.11+

# Verify Azure CLI
az --version
az login

# Install MAF
pip install agent-framework --pre
pip install azure-identity
```

## Environment Variables

Set these before the workshop (or create a `.env` file):

### C# (.NET)

```bash
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Python

```bash
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

> **⚠️ Important:** C# and Python use **different** environment variable names for the deployment. C# uses `AZURE_OPENAI_DEPLOYMENT_NAME` while Python uses `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME`.

> **Note:** MAF does **not** auto-load `.env` files. Use `load_dotenv()` in Python or set variables in your shell/IDE.

## Optional but Recommended

- [ ] Docker Desktop (for MCP server labs)
- [ ] Postman or REST Client extension (for testing hosted agents)
- [ ] Familiarity with basic AI/LLM concepts (prompts, tokens, completions)

## Hackathon-Specific Prerequisites

- [ ] Access to the **partner application repository** (link will be shared)
- [ ] Ability to clone and build the partner app locally
- [ ] Read through the partner app README before the workshop
