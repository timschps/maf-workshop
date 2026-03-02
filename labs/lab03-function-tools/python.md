# Lab 3: Agent with Tools — Function Calling — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab3_agent_tools
cd lab3_agent_tools
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

## Step 4: Define Your First Function Tool

A function tool uses the `@tool` decorator with type annotations. The docstring and `Field` descriptions tell the LLM **what** the tool does and **when** to call it.

Create a file named `main.py`:

```python
import asyncio
from typing import Annotated
from random import randint
from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential
from pydantic import Field

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The city name, e.g. 'Amsterdam' or 'Seattle'.")],
) -> str:
    """Get the current weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "partly cloudy"]
    temp = randint(5, 30)
    condition = conditions[randint(0, len(conditions) - 1)]
    return f"The weather in {location} is {condition} with a temperature of {temp}°C."

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions="You are a helpful weather assistant. Use the get_weather tool to answer weather questions. If the user asks about something else, politely redirect to weather topics.",
        tools=[get_weather],
    )

    print("Question: What is the weather like in Amsterdam?\n")
    result = await agent.run("What is the weather like in Amsterdam?")
    print(result.text)

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:** The agent automatically decided to call `get_weather("Amsterdam")`, received the result, and wove it into a natural-language response. You didn't tell it to call the function — the LLM figured that out from the tool description!

---

## Step 6: Add a Second Tool

Add this function above `main()`:

```python
@tool(approval_mode="never_require")
def get_time(
    timezone: Annotated[str, Field(description="The timezone, e.g. 'CET', 'EST', 'PST', 'UTC'.")],
) -> str:
    """Get the current time in a given timezone."""
    from datetime import datetime, timezone as tz, timedelta
    offsets = {"UTC": 0, "CET": 1, "EST": -5, "PST": -8, "JST": 9, "AEST": 11}
    if timezone.upper() in offsets:
        now = datetime.now(tz(timedelta(hours=offsets[timezone.upper()])))
        return f"The current time in {timezone} is {now.strftime('%H:%M')}."
    return f"Unknown timezone: {timezone}. Try UTC, CET, EST, PST, JST, or AEST."
```

Update the agent to include both tools:

```python
agent = client.as_agent(
    instructions="You are a helpful assistant that can check weather and time around the world.",
    tools=[get_weather, get_time],
)
```

Now test with a question that requires **both** tools:

```python
result = await agent.run(
    "I'm planning a call with someone in Tokyo. What's the time there and what's the weather like?"
)
print(result.text)
```

**Observe:** The agent calls **both** tools and combines the results!

---

## 🏋️ Exercises

### Exercise B: Bad Descriptions Experiment

Try giving a tool a vague or misleading description:

```python
@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="A parameter.")],
) -> str:
    """Does something."""
    ...
```

Ask the agent about the weather. Does it still call the tool? Now restore the good description. **Lesson:** Tool descriptions are critical for correct function calling.

### Exercise C (Stretch): Agent-as-a-Tool

Create specialist agents and use them as tools for a main "router" agent:

```python
import asyncio
from typing import Annotated
from random import randint
from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential
from pydantic import Field

@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The city name.")],
) -> str:
    """Get the current weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "partly cloudy"]
    return f"Weather in {location}: {conditions[randint(0,3)]}, {randint(5,30)}°C."

@tool(approval_mode="never_require")
def get_time(
    timezone: Annotated[str, Field(description="The timezone, e.g. 'CET', 'EST'.")],
) -> str:
    """Get the current time in a given timezone."""
    from datetime import datetime, timezone as tz, timedelta
    offsets = {"UTC": 0, "CET": 1, "EST": -5, "PST": -8, "JST": 9}
    if timezone.upper() in offsets:
        now = datetime.now(tz(timedelta(hours=offsets[timezone.upper()])))
        return f"Time in {timezone}: {now.strftime('%H:%M')}"
    return f"Unknown timezone: {timezone}"

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # Specialist weather agent
    weather_agent = client.as_agent(
        name="WeatherAgent",
        description="An agent that answers questions about the weather.",
        instructions="You answer questions about the weather.",
        tools=[get_weather],
    )

    # Specialist time agent
    time_agent = client.as_agent(
        name="TimeAgent",
        description="An agent that answers questions about time around the world.",
        instructions="You answer questions about time in different timezones.",
        tools=[get_time],
    )

    # Main "router" agent that delegates to specialists
    main_agent = client.as_agent(
        instructions="You are a helpful assistant. Delegate weather questions to the WeatherAgent and time questions to the TimeAgent.",
        tools=[weather_agent.as_tool(), time_agent.as_tool()],
    )

    result = await main_agent.run(
        "What's the weather in Paris and what time is it in Tokyo?"
    )
    print(result.text)

asyncio.run(main())
```

**Observe:** The main agent delegates to specialist agents, which in turn call their own tools!
