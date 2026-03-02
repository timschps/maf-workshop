# Lab 4: Multi-Tool Agents — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab4_multi_tool
cd lab4_multi_tool
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework --pre azure-identity
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

## Step 4: Define Multiple Tools

Create a file named `main.py`:

```python
import asyncio
from typing import Annotated
from random import randint
from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential
from pydantic import Field

# ── Tool 1: Weather ──────────────────────────────────────────────────────────
@tool(approval_mode="never_require")
def get_weather(
    city: Annotated[str, Field(description="The city name, e.g. 'Amsterdam' or 'Tokyo'.")],
) -> str:
    """Get the current weather for a given city."""
    conditions = ["sunny", "cloudy", "rainy", "partly cloudy"]
    return f"Weather in {city}: {conditions[randint(0, 3)]}, {randint(5, 30)}°C."

# ── Tool 2: Time ─────────────────────────────────────────────────────────────
@tool(approval_mode="never_require")
def get_time(
    timezone: Annotated[str, Field(description="Timezone abbreviation, e.g. 'CET', 'EST', 'JST', 'UTC'.")],
) -> str:
    """Get the current local time in a given timezone."""
    from datetime import datetime, timezone as tz, timedelta
    offsets = {"UTC": 0, "CET": 1, "EST": -5, "PST": -8, "JST": 9, "AEST": 11}
    if timezone.upper() in offsets:
        now = datetime.now(tz(timedelta(hours=offsets[timezone.upper()])))
        return f"Current time in {timezone}: {now.strftime('%H:%M')}."
    return f"Unknown timezone '{timezone}'. Supported: {', '.join(offsets.keys())}."

# ── Tool 3: Currency ─────────────────────────────────────────────────────────
@tool(approval_mode="never_require")
def convert_currency(
    amount: Annotated[float, Field(description="The amount to convert.")],
    from_currency: Annotated[str, Field(description="The source currency code, e.g. 'USD', 'EUR'.")],
    to_currency: Annotated[str, Field(description="The target currency code, e.g. 'JPY', 'GBP'.")],
) -> str:
    """Convert an amount from one currency to another."""
    rates = {"USD": 1.0, "EUR": 0.92, "GBP": 0.79, "JPY": 149.5, "CHF": 0.88}
    if from_currency.upper() in rates and to_currency.upper() in rates:
        result = amount / rates[from_currency.upper()] * rates[to_currency.upper()]
        return f"{amount} {from_currency} = {result:.2f} {to_currency}."
    return f"Unsupported currency. Supported: {', '.join(rates.keys())}."

# ── Tool 4: Translation ─────────────────────────────────────────────────────
@tool(approval_mode="never_require")
def translate(
    text: Annotated[str, Field(description="The text to translate.")],
    target_language: Annotated[str, Field(description="The target language, e.g. 'French', 'Spanish', 'Japanese'.")],
) -> str:
    """Translate a short phrase into another language. Returns the translated text."""
    # In a real app, call a translation API — here we fake it
    return f"[Translated to {target_language}]: «{text}» (simulated translation)"

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ── Create the multi-tool agent ───────────────────────────────────────────
    agent = client.as_agent(
        instructions=(
            "You are a helpful travel assistant. You have tools for weather, time, "
            "currency conversion, and translation. Use the appropriate tools to "
            "answer the user's questions. If a question requires multiple tools, "
            "call all of them."
        ),
        tools=[get_weather, get_time, convert_currency, translate],
    )

    # ── Test with various questions ───────────────────────────────────────────
    questions = [
        # Single-tool questions
        "What's the weather in Tokyo?",
        "What time is it in New York (EST)?",
        "Convert 100 USD to EUR.",
        # Multi-tool questions — the agent should call multiple tools!
        "I'm planning a trip to Tokyo. What's the weather, what time is it (JST), and how much is 500 USD in JPY?",
        # Ambiguous question — which tool does it pick?
        "How do I say 'good morning' in Japanese?",
        # No tool needed
        "What's the capital of France?",
    ]

    for q in questions:
        print(f"❓ {q}")
        print("💬 ", end="", flush=True)
        async for chunk in agent.run(q, stream=True):
            if chunk.text:
                print(chunk.text, end="", flush=True)
        print("\n")

asyncio.run(main())
```

## Step 5: Run and Observe

```bash
python main.py
```

**Key observations:**
- Single-tool questions → agent calls exactly one tool
- Multi-tool questions → agent calls multiple tools and combines results
- Questions that don't need tools → agent answers from its own knowledge
- The LLM uses tool **descriptions** to make the selection

---

## 🏋️ Exercises

### Exercise C (Stretch): Tool Selection Logging

Add logging around each tool function to track which tools are called per question. Create a table showing which questions trigger which tools.
