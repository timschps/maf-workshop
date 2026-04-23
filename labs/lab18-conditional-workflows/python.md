# Lab 18: Handoff Workflows — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab18_handoff_workflows
cd lab18_handoff_workflows
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
$env:AZURE_OPENAI_MODEL = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_MODEL="gpt-4o-mini"
```

## Step 4: Write the Code

Create a file named `main.py`:

```python
import asyncio
from agent_framework import Agent, Message
from agent_framework.openai import OpenAIChatCompletionClient
from agent_framework.orchestrations import HandoffBuilder
from azure.identity import AzureCliCredential

async def main():
    # ── Create the chat client ────────────────────────────────────────────────
    client = OpenAIChatCompletionClient(credential=AzureCliCredential())

    # ── Create specialist agents ──────────────────────────────────────────────
    triage = Agent(
        name="TriageAgent",
        instructions=(
            "You are a customer service triage agent. Analyze the customer message "
            "and route to the right specialist.\n"
            "- Billing/payment/invoice -> hand off to BillingSpecialist\n"
            "- Technical issues/bugs/how-to -> hand off to TechSupport\n"
            "- General -> answer directly"
        ),
        client=client,
    )

    billing = Agent(
        name="BillingSpecialist",
        instructions="You are a billing specialist. Help with payment, invoice, refund questions. Be concise (under 80 words).",
        client=client,
    )

    tech = Agent(
        name="TechSupport",
        instructions="You are tech support. Help troubleshoot issues and provide solutions. Be concise (under 80 words).",
        client=client,
    )

    # ── Build the handoff workflow ────────────────────────────────────────────
    workflow = (
        HandoffBuilder(participants=[triage, billing, tech])
        .with_start_agent(triage)
        .build()
    )

    # ── Test Scenario 1: Billing Query ────────────────────────────────────────
    print("=== Billing Query ===")
    message1 = "I was charged twice for my subscription last month. Can I get a refund?"
    print(f"Customer: {message1}\n")

    result1 = await workflow.run([Message(role="user", contents=[message1])])
    print(f"Response: {result1}\n")

    # ── Test Scenario 2: Tech Support ─────────────────────────────────────────
    # Rebuild workflow (handoff workflows are single-use)
    workflow = (
        HandoffBuilder(participants=[triage, billing, tech])
        .with_start_agent(triage)
        .build()
    )

    print("=== Tech Support ===")
    message2 = "My app keeps crashing when I try to upload files. I'm on version 3.2."
    print(f"Customer: {message2}\n")

    result2 = await workflow.run([Message(role="user", contents=[message2])])
    print(f"Response: {result2}\n")

    print("Handoff workflow complete!")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

You should see the triage agent route the billing question to BillingSpecialist and the tech question to TechSupport.

> **Note:** Handoff workflows are single-use — you must rebuild the workflow for each new conversation. That's why we create a new `HandoffBuilder` for each test scenario.
