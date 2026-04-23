# Lab 12: Agent Workflows — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab12_agent_workflows
cd lab12_agent_workflows
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework azure-identity
```

## Step 3: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_CHAT_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 4: Build a Research → Writer → Translator Pipeline

Create a file named `main.py`:

```python
import asyncio

from agent_framework import Agent
from agent_framework.orchestrations import SequentialBuilder
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ── Create the agents ─────────────────────────────────────────────────────
    research_agent = Agent(
        name="Researcher",
        instructions="You are a research assistant. Provide clear, factual bullet points. Be concise.",
        client=client,
    )

    writer_agent = Agent(
        name="Writer",
        instructions=(
            "You are a professional blog writer. Transform notes into engaging, "
            "polished prose. 3 sentences max."
        ),
        client=client,
    )

    translator_agent = Agent(
        name="Translator",
        instructions=(
            "You are a professional translator. Translate the given text to French. "
            "Provide accurate, natural translations. Output only the translated text."
        ),
        client=client,
    )

    # ── Build a sequential workflow: Research → Write → Translate ──────────────
    workflow = SequentialBuilder(
        participants=[research_agent, writer_agent, translator_agent]
    ).build()
    workflow_agent = workflow.as_agent(name="ContentPipeline")

    # ── Run the pipeline ──────────────────────────────────────────────────────
    topic = "The impact of artificial intelligence on healthcare"

    print("╔══════════════════════════════════════════════════════════╗")
    print("║  WORKFLOW: Research → Write → Translate                 ║")
    print("╚══════════════════════════════════════════════════════════╝")
    print(f"\n📎 Topic: {topic}\n")

    session = workflow_agent.create_session()
    async for update in workflow_agent.run(topic, session=session, stream=True):
        if update.text:
            print(update.text, end="", flush=True)

    print("\n\n╔══════════════════════════════════════════════════════════╗")
    print("║  FINAL OUTPUT (French):                                 ║")
    print("╚══════════════════════════════════════════════════════════╝\n")


asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:** Three LLM agents work in a sequential workflow — each one transforms the data and passes it along. The researcher gathers facts, the writer polishes them, and the translator produces the final French version.
