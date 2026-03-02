# Lab 8: Middleware Pipeline — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab8_middleware
cd lab8_middleware
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

## Step 4: Write the Code

Create a file named `main.py`:

```python
import asyncio
import random
import time
from collections.abc import Awaitable, Callable

from agent_framework import (
    AgentMiddleware,
    AgentContext,
    FunctionMiddleware,
    FunctionInvocationContext,
    tool,
)
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


# ── Tool ──────────────────────────────────────────────────────────────────────
@tool
def get_weather(city: str) -> str:
    """Get the current weather for a given city."""
    return f"Weather in {city}: sunny, {random.randint(15, 30)}°C."


# ── Security Middleware ───────────────────────────────────────────────────────
class SecurityMiddleware(AgentMiddleware):
    async def process(
        self, context: AgentContext, call_next: Callable[[], Awaitable[None]]
    ) -> None:
        last_message = context.messages[-1].text if context.messages else ""
        blocked = ["password", "secret", "credit card", "ssn", "api key"]

        if any(kw in last_message.lower() for kw in blocked):
            print("  🛡️ [Security] BLOCKED — sensitive content detected")
            context.result = "I can't process requests containing sensitive information."
            return

        print("  🛡️ [Security] Passed")
        await call_next()


# ── Logging Middleware ────────────────────────────────────────────────────────
class LoggingMiddleware(AgentMiddleware):
    def __init__(self):
        self.turn_count = 0

    async def process(
        self, context: AgentContext, call_next: Callable[[], Awaitable[None]]
    ) -> None:
        self.turn_count += 1
        start = time.time()
        print(
            f"  📊 [Logging] Turn #{self.turn_count} | Input messages: {len(context.messages)}"
        )

        await call_next()

        elapsed = (time.time() - start) * 1000
        print(f"  📊 [Logging] Turn #{self.turn_count} | Duration: {elapsed:.0f}ms")


# ── Function Logging Middleware ───────────────────────────────────────────────
class FunctionLoggingMiddleware(FunctionMiddleware):
    async def process(
        self,
        context: FunctionInvocationContext,
        call_next: Callable[[], Awaitable[None]],
    ) -> None:
        print(f"  🔧 [FuncLog] → {context.function.name}(...)")
        start = time.time()

        await call_next()

        elapsed = (time.time() - start) * 1000
        print(
            f"  🔧 [FuncLog] ← {context.function.name} returned in {elapsed:.0f}ms"
        )


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        name="WeatherAgent",
        instructions="You are a helpful weather assistant.",
        tools=[get_weather],
        middleware=[
            SecurityMiddleware(),
            LoggingMiddleware(),
            FunctionLoggingMiddleware(),
        ],
    )

    print("═══ Test 1: Normal question (triggers tool) ═══\n")
    result = await agent.run("What's the weather in Amsterdam?")
    print(f"{result}\n")

    print("═══ Test 2: Follow-up question ═══\n")
    result = await agent.run("How about in Tokyo?")
    print(f"{result}\n")

    print("═══ Test 3: Blocked by security middleware ═══\n")
    result = await agent.run("What's the api key for the weather service?")
    print(f"{result}\n")

    print("═══ Test 4: No tool needed ═══\n")
    result = await agent.run("What's the capital of France?")
    print(f"{result}\n")


asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe the pipeline in action:**
- Test 1: Security ✅ → Logging → Function call logged → Response
- Test 3: Security ❌ → BLOCKED before reaching the agent

### Exercise A: Add Rate Limiting Middleware

```python
class RateLimitMiddleware(AgentMiddleware):
    def __init__(self, max_calls: int = 5):
        self.call_count = 0
        self.max_calls = max_calls

    async def process(
        self, context: AgentContext, call_next: Callable[[], Awaitable[None]]
    ) -> None:
        self.call_count += 1
        if self.call_count > self.max_calls:
            context.result = (
                f"Rate limit exceeded. Max {self.max_calls} calls per session."
            )
            return
        await call_next()
```

### Exercise B: Result Modifier Middleware

```python
class DisclaimerMiddleware(FunctionMiddleware):
    async def process(
        self,
        context: FunctionInvocationContext,
        call_next: Callable[[], Awaitable[None]],
    ) -> None:
        await call_next()
        context.result = f"{context.result} (Data is simulated for demonstration.)"
```
