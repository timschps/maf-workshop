# Lab 11: Simple Workflows — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Part A: Simple Function Workflow (10 min)

### Step 1: Create the Project

```bash
mkdir lab11_workflows
cd lab11_workflows
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

### Step 2: Install Packages

```bash
pip install agent-framework --pre azure-identity
```

### Step 3: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_CHAT_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

### Step 4: Build a Two-Step Workflow

Create a file named `main.py`:

```python
import asyncio

from agent_framework import Agent, WorkflowBuilder
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ── Step 1: Agent to convert text to uppercase ────────────────────────────
    uppercase_agent = Agent(
        name="UpperCaseAgent",
        instructions="Convert the user's input text to UPPERCASE. Output only the uppercase text, nothing else.",
        client=client,
    )

    # ── Step 2: Agent to reverse the string ───────────────────────────────────
    reverse_agent = Agent(
        name="ReverseAgent",
        instructions="Reverse the user's input text character by character. Output only the reversed text, nothing else.",
        client=client,
    )

    # ── Build and run the workflow ────────────────────────────────────────────
    workflow = (
        WorkflowBuilder(start_executor=uppercase_agent)
        .add_edge(uppercase_agent, reverse_agent)
        .build()
    )

    print("=== Running Workflow: UpperCase → Reverse ===\n")
    result = await workflow.run("hello world")
    print(f"\nFinal output: {result}")
    # Expected: DLROW OLLEH


asyncio.run(main())
```

### Step 5: Run It

```bash
python main.py
```

**Observe:** Data flows through the sequential workflow: `"hello world"` → `UpperCaseAgent` → `"HELLO WORLD"` → `ReverseAgent` → `"DLROW OLLEH"`.

---

## Part B: Agent Workflow — Research & Write (15 min)

### Step 6: Add an Agent-Powered Workflow

Replace `main.py` with the expanded version:

```python
import asyncio

from agent_framework import Agent, WorkflowBuilder
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ══════════════════════════════════════════════════════════════════════════
    # WORKFLOW: Topic → Research → Write Summary → Output
    # ══════════════════════════════════════════════════════════════════════════

    research_agent = Agent(
        name="ResearchAgent",
        instructions="You are a research assistant. When given a topic, provide clear, factual bullet points. Be concise and accurate.",
        client=client,
    )

    writer_agent = Agent(
        name="WriterAgent",
        instructions="You are a professional writer. Transform research notes into polished, engaging prose. Keep it concise — 3 sentences maximum.",
        client=client,
    )

    workflow = (
        WorkflowBuilder(start_executor=research_agent)
        .add_edge(research_agent, writer_agent)
        .build()
    )

    print("═══════════════════════════════════════════════════════")
    print("  WORKFLOW: Research → Write Summary")
    print("═══════════════════════════════════════════════════════\n")

    topic = "The impact of artificial intelligence on healthcare"
    print(f"Topic: {topic}\n")

    result = await workflow.run(topic)

    print("\n═══════════════════════════════════════════════════════")
    print("  FINAL OUTPUT:")
    print("═══════════════════════════════════════════════════════\n")
    print(result)


asyncio.run(main())
```

### Step 7: Run the Agent Workflow

```bash
python main.py
```

**Observe:** The research agent generates facts, those facts flow sequentially to the writer agent, which produces a polished summary.
