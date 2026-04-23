# Lab 1: Hello Agent — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab1_hello_agent
cd lab1_hello_agent
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
$env:AZURE_OPENAI_CHAT_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_CHAT_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 4: Write the Code

Create a file named `main.py`:

```python
import asyncio
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

async def main():
    # ── Create the agent ──────────────────────────────────────────────────────
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        name="HelloAgent",
        instructions="You are a friendly assistant. Keep your answers brief.",
    )

    # ── Run 1: Non-streaming (complete response at once) ──────────────────────
    print("=== Non-Streaming ===")
    result = await agent.run("What is the largest city in France?")
    print(f"Agent: {result.text}\n")

    # ── Run 2: Streaming (token-by-token) ─────────────────────────────────────
    print("=== Streaming ===")
    print("Agent: ", end="", flush=True)
    async for chunk in agent.run(
        "Tell me a one-sentence fun fact about the Eiffel Tower.", stream=True
    ):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print()

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

You should see the agent's response appear — first as a complete block, then token-by-token.

### Exercise C (Stretch): Interactive Console Agent

```python
import asyncio
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        name="HelloAgent",
        instructions="You are a friendly assistant. Keep your answers brief.",
    )

    print("Chat with the agent (type 'exit' to quit):")
    while True:
        user_input = input("\nYou: ")
        if not user_input or user_input.lower() == "exit":
            break

        print("Agent: ", end="", flush=True)
        async for chunk in agent.run(user_input, stream=True):
            if chunk.text:
                print(chunk.text, end="", flush=True)
        print()

asyncio.run(main())
```

> ⚠️ Note: This loop does NOT maintain conversation history between turns yet —
> each call is independent. We'll fix that in Lab 6!
