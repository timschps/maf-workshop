# Lab 10: Agent-as-a-Tool — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab10_agent_as_tool
cd lab10_agent_as_tool
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

## Step 4: Build Specialist Agents

Create a file named `main.py`:

```python
import asyncio
import random
from datetime import datetime, timezone, timedelta

from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


# ── Tool functions for specialists ────────────────────────────────────────────
@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    conditions = ["sunny", "cloudy", "rainy"]
    return f"Weather in {city}: {random.choice(conditions)}, {random.randint(10, 30)}°C."


@tool
def get_time(tz: str) -> str:
    """Get the current time in a timezone, e.g. CET, EST, JST."""
    offsets = {"UTC": 0, "CET": 1, "EST": -5, "JST": 9}
    upper = tz.upper()
    if upper in offsets:
        dt = datetime.now(timezone(timedelta(hours=offsets[upper])))
        return f"Time in {tz}: {dt.strftime('%H:%M')}."
    return f"Unknown timezone: {tz}."


@tool
def convert_currency(amount: float, from_currency: str, to_currency: str) -> str:
    """Convert currency between two currency codes."""
    rates = {"USD": 1.0, "EUR": 0.92, "JPY": 149.5, "GBP": 0.79}
    f = rates.get(from_currency.upper())
    t = rates.get(to_currency.upper())
    if f and t:
        return f"{amount} {from_currency} = {amount / f * t:.2f} {to_currency}."
    return "Unsupported currency pair."


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ══════════════════════════════════════════════════════════════════════════
    # SPECIALIST AGENTS
    # ══════════════════════════════════════════════════════════════════════════
    weather_agent = client.as_agent(
        name="WeatherAgent",
        instructions="You are a weather specialist. Answer weather questions concisely.",
        tools=[get_weather],
    )

    time_agent = client.as_agent(
        name="TimeAgent",
        instructions="You are a timezone specialist. Answer time-related questions.",
        tools=[get_time],
    )

    finance_agent = client.as_agent(
        name="FinanceAgent",
        instructions="You are a currency specialist. Help with currency conversions.",
        tools=[convert_currency],
    )

    # ══════════════════════════════════════════════════════════════════════════
    # MANAGER AGENT — delegates to specialists via agent-as-tool
    # ══════════════════════════════════════════════════════════════════════════
    manager_agent = client.as_agent(
        name="TravelPlanner",
        instructions=(
            "You are a travel planning assistant. You coordinate with specialist agents:\n"
            "- WeatherAgent: for weather information\n"
            "- TimeAgent: for time zone queries\n"
            "- FinanceAgent: for currency conversions\n\n"
            "When a user asks a complex question, delegate to the appropriate specialists "
            "and synthesize their responses into a helpful answer. "
            "Always be concise and well-organized."
        ),
        tools=[
            weather_agent.as_tool(),
            time_agent.as_tool(),
            finance_agent.as_tool(),
        ],
    )

    # ── Test with various complexity levels ───────────────────────────────────
    questions = [
        "What's the weather like in Tokyo right now?",
        "What time is it in CET?",
        "I'm traveling from New York to Tokyo next week. What's the weather there, what time is it, and how much is 1000 USD in JPY?",
    ]

    for q in questions:
        print(f"❓ {q}\n")
        print("💬 ", end="", flush=True)
        async for chunk in manager_agent.run(q, stream=True):
            if chunk.text:
                print(chunk.text, end="", flush=True)
        print("\n" + "─" * 70 + "\n")


asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:**
- Simple questions: manager delegates to one specialist
- Complex questions: manager delegates to **multiple** specialists and combines results
- Each specialist has its own tools — the manager doesn't know about `get_weather` directly

### Exercise C: Named Tool Customization

```python
# The as_tool() method uses the agent's name and instructions as the tool description
# The manager sees "WeatherAgent" as a callable tool
custom_tool = weather_agent.as_tool()
```
