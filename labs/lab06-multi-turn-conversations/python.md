# Lab 6: Multi-Turn Conversations & Sessions — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab6_multi_turn
cd lab6_multi_turn
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

## Step 4: Write a Multi-Turn Agent

Create a file named `main.py`:

```python
import asyncio
from typing import Annotated
from random import randint
from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential
from pydantic import Field

# ── Tool from Lab 3 ──────────────────────────────────────────────────────────
@tool(approval_mode="never_require")
def get_weather(
    location: Annotated[str, Field(description="The city name.")],
) -> str:
    """Get the current weather for a given location."""
    conditions = ["sunny", "cloudy", "rainy", "partly cloudy"]
    return f"Weather in {location}: {conditions[randint(0, 3)]}, {randint(5, 30)}°C."

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ── Create the agent ──────────────────────────────────────────────────────
    agent = client.as_agent(
        instructions="You are a friendly travel assistant. You can check weather for cities. Remember the user's name and preferences throughout the conversation.",
        name="TravelAssistant",
        tools=[get_weather],
    )

    # ── Create a session for multi-turn conversation ──────────────────────────
    session = agent.create_session()

    # ── Turn 1: Introduce yourself ────────────────────────────────────────────
    print("--- Turn 1 ---")
    r1 = await agent.run("Hi! My name is Alice and I love warm weather destinations.", session=session)
    print(r1.text)
    print()

    # ── Turn 2: Ask for a recommendation (agent should remember the preference)
    print("--- Turn 2 ---")
    r2 = await agent.run("Can you check the weather in Barcelona for me?", session=session)
    print(r2.text)
    print()

    # ── Turn 3: Test memory — does the agent remember the name? ───────────────
    print("--- Turn 3 ---")
    r3 = await agent.run("Based on my preferences, would you recommend Barcelona for me? Also, what's my name?", session=session)
    print(r3.text)
    print()

    # ── Turn 4: Without a session — agent should NOT remember ─────────────────
    print("--- Turn 4 (NO SESSION — new context!) ---")
    r4 = await agent.run("What's my name?")
    print(r4.text)
    print()

asyncio.run(main())
```

## Step 5: Run and Observe

```bash
python main.py
```

**Key observations:**
- Turns 1–3 use the same `session` — the agent remembers Alice's name and preference
- Turn 4 has NO session — the agent doesn't know the user's name
- The weather tool is called automatically in Turn 2

---

## 🏋️ Exercises

### Exercise A: Interactive Multi-Turn Chat

Build an interactive console loop with session:

```python
import asyncio
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions="You are a friendly travel assistant. Remember the user's name and preferences.",
        name="TravelAssistant",
    )

    session = agent.create_session()
    print("Chat with the travel assistant (type 'exit' to quit):\n")

    while True:
        user_input = input("You: ")
        if not user_input or user_input.lower() == "exit":
            break

        print("Agent: ", end="", flush=True)
        async for chunk in agent.run(user_input, session=session, stream=True):
            if chunk.text:
                print(chunk.text, end="", flush=True)
        print("\n")

asyncio.run(main())
```

Have a multi-turn conversation: introduce yourself, ask about weather, then test if it remembers your name.

### Exercise B: Session Persistence

Show how session state preserves context:

```python
async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions="You are a helpful assistant. Remember everything the user tells you.",
        name="MemoryAgent",
    )

    session = agent.create_session()

    # Build up context
    await agent.run("My favorite color is blue.", session=session)
    await agent.run("I work as a data engineer.", session=session)

    # Test recall
    result = await agent.run("What's my favorite color and what do I do for work?", session=session)
    print(f"Agent: {result.text}")

asyncio.run(main())
```

### Exercise C (Stretch): Parallel Sessions

Create two separate sessions and show they're independent:

```python
async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions="You are a helpful assistant. Remember the user's name.",
        name="NameAgent",
    )

    session1 = agent.create_session()
    session2 = agent.create_session()

    await agent.run("My name is Alice.", session=session1)
    await agent.run("My name is Bob.", session=session2)

    r1 = await agent.run("What's my name?", session=session1)
    r2 = await agent.run("What's my name?", session=session2)

    print(f"Session 1: {r1.text}")  # Alice
    print(f"Session 2: {r2.text}")  # Bob

asyncio.run(main())
```
