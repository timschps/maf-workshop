# Lab 9: Human-in-the-Loop Tool Approval — Python Implementation

[← Back to Lab Overview](./README.md)

## Step 1: Create the Project

```bash
mkdir lab9_tool_approval
cd lab9_tool_approval
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

from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential


# ── Safe tool: no approval needed ─────────────────────────────────────────────
@tool
def get_weather(city: str) -> str:
    """Get the current weather for a given city. This is a read-only operation."""
    return f"Weather in {city}: sunny, {random.randint(15, 30)}°C."


# ── Sensitive tools: require approval ─────────────────────────────────────────
@tool(approval_mode="always_require")
def send_email(to: str, subject: str, body: str) -> str:
    """Send an email to a recipient with a subject and body. This sends a real email."""
    return f"✅ Email sent to {to} with subject '{subject}'."


@tool(approval_mode="always_require")
def place_order(product: str, quantity: int) -> str:
    """Place an order for a product. This charges the customer's account."""
    return f"✅ Order placed: {quantity}x {product}."


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions=(
            "You are a helpful assistant that can check weather, send emails, "
            "and place orders. Always confirm the details before taking action."
        ),
        tools=[get_weather, send_email, place_order],
    )

    # ── Test scenarios ────────────────────────────────────────────────────────
    print("═══ Scenario 1: Safe tool (no approval needed) ═══")
    result = await agent.run("What's the weather in Amsterdam?")
    print(f"💬 {result}\n")

    print("═══ Scenario 2: Sensitive tool (needs approval) ═══")
    result = await agent.run(
        "Send an email to alice@example.com with subject 'Meeting tomorrow' "
        "and body 'Hi Alice, can we meet at 2pm?'"
    )
    print(f"💬 {result}\n")

    print("═══ Scenario 3: Another sensitive tool ═══")
    result = await agent.run("Order 3 units of Widget Pro.")
    print(f"💬 {result}\n")


asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:**
- Weather queries execute immediately (no approval needed)
- Email and order tools trigger the framework's approval flow before execution
- The `approval_mode="always_require"` decorator handles the approval prompt automatically

### Exercise B: Approval with Details

Use `FunctionMiddleware` to customize the approval prompt with richer details:

```python
from collections.abc import Awaitable, Callable
from agent_framework import FunctionMiddleware, FunctionInvocationContext


class DetailedApprovalMiddleware(FunctionMiddleware):
    async def process(
        self,
        context: FunctionInvocationContext,
        call_next: Callable[[], Awaitable[None]],
    ) -> None:
        if "send_email" in context.function.name:
            print(f"  📧 To: {context.arguments.get('to')}")
            print(f"  📝 Subject: {context.arguments.get('subject')}")
            approval = input("  Send this email? (y/n): ").strip().lower()
            if approval not in ("y", "yes"):
                context.result = "Action rejected by the user."
                return
        await call_next()
```

### Exercise C: Audit Trail

```python
from datetime import datetime

audit_log: list[dict] = []

class AuditMiddleware(FunctionMiddleware):
    async def process(
        self,
        context: FunctionInvocationContext,
        call_next: Callable[[], Awaitable[None]],
    ) -> None:
        approved = True
        # ... add approval logic ...
        audit_log.append({
            "tool": context.function.name,
            "approved": approved,
            "time": datetime.now().strftime("%H:%M:%S"),
        })
        await call_next()

# Print audit trail at the end
print("\n═══ Audit Trail ═══")
for entry in audit_log:
    status = "✅" if entry["approved"] else "❌"
    print(f"  {entry['time']} | {entry['tool']} | {status}")
```
