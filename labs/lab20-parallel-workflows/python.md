# Lab 20: Concurrent Workflows — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab20_concurrent_workflows
cd lab20_concurrent_workflows
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
$env:AZURE_OPENAI_MODEL = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_MODEL="gpt-4o-mini"
```

## Step 4: Write the Code

Create a file named `main.py`:

```python
import asyncio
from agent_framework import Agent, Message
from agent_framework.openai import OpenAIChatCompletionClient
from agent_framework.orchestrations import ConcurrentBuilder
from azure.identity import AzureCliCredential

async def main():
    # ── Create the chat client ────────────────────────────────────────────────
    client = OpenAIChatCompletionClient(credential=AzureCliCredential())

    # ── Create specialist researcher agents ───────────────────────────────────
    history = Agent(
        name="HistoryResearcher",
        instructions=(
            "You are a history researcher. When given a country, provide 3-4 key historical facts. "
            "Be concise (under 80 words). Start with 'HISTORY:'"
        ),
        client=client,
    )

    culture = Agent(
        name="CultureResearcher",
        instructions=(
            "You are a culture researcher. When given a country, describe 3-4 cultural highlights "
            "(food, customs, art). Be concise (under 80 words). Start with 'CULTURE:'"
        ),
        client=client,
    )

    travel = Agent(
        name="TravelTips",
        instructions=(
            "You are a travel tips expert. When given a country, provide 3-4 practical travel tips. "
            "Be concise (under 80 words). Start with 'TRAVEL TIPS:'"
        ),
        client=client,
    )

    # ── Build the concurrent workflow ─────────────────────────────────────────
    workflow = ConcurrentBuilder(participants=[history, culture, travel]).build()

    topic = "Japan"
    print(f"Researching '{topic}' with 3 parallel agents...\n")

    # ── Run all agents in parallel ────────────────────────────────────────────
    result = await workflow.run([Message(role="user", contents=[f"Tell me about {topic}"])])

    print("=== COMBINED RESEARCH REPORT ===\n")
    print(result)
    print("\nConcurrent workflow complete!")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

You should see all three researchers working simultaneously on the topic, with their results combined into a report. Notice how the total execution time is roughly equal to the slowest single agent — not the sum of all three.
