# Lab 14: Hosting & A2A Protocol — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab14_hosting_a2a
cd lab14_hosting_a2a
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework --pre agent-framework-a2a --pre azure-identity uvicorn
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

## Step 4: Build a Hosted Agent API

In Python, we host an agent via the A2A protocol by creating an **A2A server** using the `a2a-sdk` package. This involves:

1. Creating an `AgentExecutor` that wraps your agent
2. Configuring an `AgentCard` (the A2A discovery document)
3. Setting up a `DefaultRequestHandler` and `A2AStarletteApplication`
4. Running the app with `uvicorn`

Create a file named `main.py`:

```python
import random
import uuid
from datetime import datetime, timezone

import uvicorn
from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

# A2A server imports
from a2a.server.apps import A2AStarletteApplication
from a2a.server.request_handlers import DefaultRequestHandler
from a2a.server.agent_execution import AgentExecutor
from a2a.server.agent_execution.context import RequestContext
from a2a.server.events.event_queue import EventQueue
from a2a.server.tasks import InMemoryTaskStore
from a2a.types import (
    AgentCard,
    AgentSkill,
    AgentCapabilities,
    Message as A2AMessage,
    TextPart,
    Role as A2ARole,
)


# ── Define tools ──────────────────────────────────────────────────────────────
@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"Weather in {city}: partly cloudy, {random.randint(15, 30)}°C"


@tool
def get_time(tz: str) -> str:
    """Get the current time in a timezone, e.g. UTC, EST, CET."""
    return f"Current time in {tz}: {datetime.now(timezone.utc).strftime('%H:%M:%S')} UTC"


# ── Create the agent ─────────────────────────────────────────────────────────
client = AzureOpenAIChatClient(credential=AzureCliCredential())
agent = client.as_agent(
    name="TravelAssistant",
    instructions=(
        "You are a helpful travel assistant. You can provide weather information "
        "and current time for destinations around the world. Be concise and "
        "friendly. When asked about a destination, proactively share both weather "
        "and time information."
    ),
    tools=[get_weather, get_time],
)


# ── A2A Executor: bridges agent-framework → A2A protocol ─────────────────────
class MAFAgentExecutor(AgentExecutor):
    """Wraps a Microsoft Agent Framework agent as an A2A AgentExecutor."""

    def __init__(self, agent):
        self._agent = agent

    async def execute(self, context: RequestContext, event_queue: EventQueue):
        # Extract the user's text from the A2A message
        user_msg = ""
        if context.message and context.message.parts:
            for part in context.message.parts:
                if hasattr(part, "root") and hasattr(part.root, "text"):
                    user_msg += part.root.text
                elif hasattr(part, "text"):
                    user_msg += part.text

        # Run the agent
        result = await self._agent.run(user_msg)

        # Send the response back as an A2A message
        response_msg = A2AMessage(
            role=A2ARole.agent,
            parts=[TextPart(text=result.text)],
            messageId=str(uuid.uuid4()),
        )
        await event_queue.enqueue_event(response_msg)

    async def cancel(self, context: RequestContext, event_queue: EventQueue):
        pass


# ── Configure the A2A server ─────────────────────────────────────────────────
agent_card = AgentCard(
    name="TravelAssistant",
    description="A helpful travel assistant that provides weather and time info",
    url="http://localhost:8088",
    version="1.0.0",
    skills=[
        AgentSkill(
            id="travel",
            name="Travel Advice",
            description="Provides weather and time information for destinations",
            tags=["travel", "weather"],
        )
    ],
    capabilities=AgentCapabilities(streaming=False),
    default_input_modes=["text"],
    default_output_modes=["text"],
)

task_store = InMemoryTaskStore()
handler = DefaultRequestHandler(
    agent_executor=MAFAgentExecutor(agent),
    task_store=task_store,
)
app = A2AStarletteApplication(agent_card=agent_card, http_handler=handler)

# ── Start the server ─────────────────────────────────────────────────────────
if __name__ == "__main__":
    print("╔══════════════════════════════════════════════════════════════╗")
    print("║  Travel Assistant API is running!                           ║")
    print("║                                                             ║")
    print("║  A2A endpoint: http://localhost:8088                        ║")
    print("║  Agent card:   http://localhost:8088/.well-known/agent.json ║")
    print("╚══════════════════════════════════════════════════════════════╝")
    uvicorn.run(app.build(), host="0.0.0.0", port=8088)
```

## Step 5: Run the Server

```bash
python main.py
```

## Step 6: Test the Agent with HTTP Requests

Open a **new terminal** and test:

### Health Check / A2A Discovery

```bash
curl http://localhost:8088/.well-known/agent.json
```

### Send a Message via A2A

```bash
curl -X POST http://localhost:8088 \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "method": "message/send",
    "id": "1",
    "params": {
        "message": {
            "role": "user",
            "parts": [
                {
                    "kind": "text",
                    "text": "What is the weather like in Paris right now?"
                }
            ],
            "messageId": "msg-001"
        }
    }
}'
```

**Observe:** The response follows the A2A JSON-RPC protocol with structured parts.

### Exercise C: Agent-to-Agent Communication

Create a separate file `client.py` that uses the agent-framework A2A client:

```python
import asyncio
from agent_framework.a2a import A2AAgent


async def main():
    async with A2AAgent(name="TravelClient", url="http://localhost:8088") as a2a_agent:
        result = await a2a_agent.run("What is the weather in London?")
        print(f"Response: {result.text}")


asyncio.run(main())
```

Run it while the server is still running:

```bash
python client.py
```

This demonstrates the core A2A pattern: agents communicating over HTTP using a standard protocol.
