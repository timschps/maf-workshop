# Lab 21: Hosted Multi-Agent Workflow — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab21_hosted_multi_agent
cd lab21_hosted_multi_agent
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

## Step 4: Write the Code

Create a file named `main.py`:

```python
import uuid

import uvicorn
from agent_framework import Agent
from agent_framework.azure import AzureOpenAIChatClient
from agent_framework.orchestrations import SequentialBuilder
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

# ── Create the chat client ────────────────────────────────────────────────────
client = AzureOpenAIChatClient(credential=AzureCliCredential())

# ── Register individual agents ────────────────────────────────────────────────
researcher = Agent(
    name="Researcher",
    instructions=(
        "You are a research specialist. When given a topic, provide 3-5 key facts "
        "and data points. Be factual and concise. Output only the research findings."
    ),
    client=client,
)

writer = Agent(
    name="Writer",
    instructions=(
        "You are a professional writer. Take the research provided and write a short, "
        "engaging article (150-200 words). Use a clear structure with an introduction, "
        "body, and conclusion."
    ),
    client=client,
)

editor = Agent(
    name="Editor",
    instructions=(
        "You are an editor. Review the article provided and improve it: fix grammar, "
        "improve clarity, add a catchy title. Output the final polished article with "
        "the title on the first line."
    ),
    client=client,
)

# ── Build the sequential workflow ─────────────────────────────────────────────
workflow = SequentialBuilder(participants=[researcher, writer, editor]).build()

# ── Convert workflow to agent ─────────────────────────────────────────────────
workflow_agent = workflow.as_agent(name="ContentPipeline")


# ── A2A Executor wrapping the workflow agent ──────────────────────────────────
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


# ── Configure A2A server ─────────────────────────────────────────────────────
agent_card = AgentCard(
    name="ContentPipeline",
    description="Multi-agent content generation pipeline (Researcher → Writer → Editor)",
    url="http://localhost:8088",
    version="1.0.0",
    skills=[
        AgentSkill(
            id="content",
            name="Content Generation",
            description="Generates polished articles on any topic",
            tags=["content", "writing"],
        )
    ],
    capabilities=AgentCapabilities(streaming=False),
    default_input_modes=["text"],
    default_output_modes=["text"],
)

task_store = InMemoryTaskStore()
handler = DefaultRequestHandler(
    agent_executor=MAFAgentExecutor(workflow_agent),
    task_store=task_store,
)
app = A2AStarletteApplication(agent_card=agent_card, http_handler=handler)

if __name__ == "__main__":
    print("📝 Content Pipeline Multi-Agent System running on http://localhost:8088")
    print("   Agents: Researcher → Writer → Editor")
    print("   POST / — Submit a topic for article generation")
    uvicorn.run(app.build(), host="0.0.0.0", port=8088)
```

## Step 5: Run It

```bash
python main.py
```

## Step 6: Test with the A2A Client

In a new terminal, create a file named `client.py`:

```python
import asyncio
from agent_framework.a2a import A2AAgent


async def main():
    async with A2AAgent(name="ContentClient", url="http://localhost:8088") as agent:
        result = await agent.run("Write an article about quantum computing")
        print(f"Generated Article:\n{result.text}")


asyncio.run(main())
```

```bash
pip install agent-framework --pre agent-framework-a2a --pre
python client.py
```

You should receive a polished article that went through all three agents (Researcher → Writer → Editor)!

Alternatively, test with **curl** (raw A2A JSON-RPC):

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
        "parts": [{"kind": "text", "text": "Write an article about quantum computing"}],
        "messageId": "msg-1"
      }
    }
  }'
```

> **Note:** The Python A2A server uses `uvicorn` + Starlette on port 8088. The C# version uses ASP.NET Core on port 5200.
