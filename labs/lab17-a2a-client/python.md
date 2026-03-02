# Lab 17: A2A Client — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Part A: Create the A2A Server

### Step 1: Create the Server Project

```bash
mkdir lab17_a2a_server
cd lab17_a2a_server
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

### Step 2: Install Server Packages

```bash
pip install agent-framework --pre agent-framework-a2a --pre azure-identity uvicorn
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

### Step 4: Write the Server Code

Create a file named `server.py`:

```python
import uuid

import uvicorn
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

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

# ── Create the agent ──────────────────────────────────────────────────────────
client = AzureOpenAIChatClient(credential=AzureCliCredential())
travel_agent = client.as_agent(
    name="TravelAssistant",
    instructions=(
        "You are a travel assistant. Provide helpful travel advice, "
        "destination recommendations, and trip planning tips. Keep responses concise."
    ),
)


# ── A2A Executor wrapping the agent ──────────────────────────────────────────
class MAFAgentExecutor(AgentExecutor):
    def __init__(self, agent):
        self._agent = agent

    async def execute(self, context: RequestContext, event_queue: EventQueue):
        user_msg = ""
        if context.message and context.message.parts:
            for part in context.message.parts:
                if hasattr(part, "root") and hasattr(part.root, "text"):
                    user_msg += part.root.text
                elif hasattr(part, "text"):
                    user_msg += part.text

        result = await self._agent.run(user_msg)
        response_msg = A2AMessage(
            role=A2ARole.agent,
            parts=[TextPart(text=result.text)],
            messageId=str(uuid.uuid4()),
        )
        await event_queue.enqueue_event(response_msg)

    async def cancel(self, context: RequestContext, event_queue: EventQueue):
        pass


# ── Configure and run the A2A server ─────────────────────────────────────────
agent_card = AgentCard(
    name="TravelAssistant",
    description="A travel assistant that provides destination advice",
    url="http://localhost:8088",
    version="1.0.0",
    skills=[
        AgentSkill(
            id="travel",
            name="Travel Advice",
            description="Provides travel recommendations",
            tags=["travel"],
        )
    ],
    capabilities=AgentCapabilities(streaming=False),
    default_input_modes=["text"],
    default_output_modes=["text"],
)

task_store = InMemoryTaskStore()
handler = DefaultRequestHandler(
    agent_executor=MAFAgentExecutor(travel_agent),
    task_store=task_store,
)
app = A2AStarletteApplication(agent_card=agent_card, http_handler=handler)

if __name__ == "__main__":
    print("🌍 Travel Assistant A2A Server running on http://localhost:8088")
    uvicorn.run(app.build(), host="0.0.0.0", port=8088)
```

### Step 5: Start the Server

```bash
python server.py
```

Leave this running and open a **new terminal** for the client.

---

## Part B: Create the A2A Client

### Step 6: Create the Client Project (in a new terminal)

```bash
mkdir lab17_a2a_client
cd lab17_a2a_client
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

### Step 7: Install Client Packages

```bash
pip install agent-framework --pre agent-framework-a2a --pre
```

### Step 8: Write the Client Code

Create a file named `main.py`:

```python
import asyncio
from agent_framework.a2a import A2AAgent


async def main():
    server_url = "http://localhost:8088"

    print("🌍 A2A Client — Calling Remote Travel Assistant\n")

    # ── Create an A2A agent that connects to the remote server ────────────────
    async with A2AAgent(name="TravelClient", url=server_url) as a2a_agent:
        # ── Call the remote agent ─────────────────────────────────────────────
        print("📤 Sending: 'What are the top 3 must-visit places in Japan?'\n")
        result = await a2a_agent.run("What are the top 3 must-visit places in Japan?")
        print(f"📥 Response:\n{result.text}\n")

        # ── Another question ──────────────────────────────────────────────────
        print("📤 Sending: 'What should I pack for a winter trip to Iceland?'\n")
        result2 = await a2a_agent.run("What should I pack for a winter trip to Iceland?")
        print(f"📥 Response:\n{result2.text}\n")

    print("✅ A2A client communication complete!")


asyncio.run(main())
```

### Step 9: Run the Client

```bash
python main.py
```

You should see the client communicate with the remote Travel Assistant agent and receive travel advice!

> **Note:** The `A2AAgent` from `agent_framework.a2a` is the standard way to connect to any A2A-compliant server. It wraps the A2A protocol and presents a familiar agent interface (`run()`, `run(..., stream=True)`).
