# Lab 19: Group Chat — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab19_group_chat
cd lab19_group_chat
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
from agent_framework import Agent, Message
from agent_framework.azure import AzureOpenAIChatClient
from agent_framework.orchestrations import GroupChatBuilder, GroupChatState
from azure.identity import AzureCliCredential


def round_robin_selector(state: GroupChatState) -> str:
    """Round-robin speaker selection."""
    participant_ids = list(state.participants.keys())
    if not state.conversation:
        return participant_ids[0]
    last_author = state.conversation[-1].author_name
    for i, p_id in enumerate(participant_ids):
        name = state.participants[p_id]
        if last_author == name:
            return participant_ids[(i + 1) % len(participant_ids)]
    return participant_ids[0]

async def main():
    # ── Create the chat client ────────────────────────────────────────────────
    client = AzureOpenAIChatClient(credential=AzureCliCredential())

    # ══════════════════════════════════════════════════════════════════════════
    #  Scenario 1: Round-Robin Group Chat
    # ══════════════════════════════════════════════════════════════════════════
    print("========================================")
    print("  Scenario 1: Round-Robin Group Chat")
    print("========================================\n")

    writer = Agent(
        name="CopyWriter",
        instructions=(
            "You are a creative copywriter. Generate catchy slogans and marketing copy. "
            "Be concise and impactful. Keep responses under 60 words."
        ),
        client=client,
    )

    reviewer = Agent(
        name="Reviewer",
        instructions=(
            "You are a marketing reviewer. Evaluate slogans for clarity, impact, and brand alignment. "
            "Provide constructive feedback or say APPROVED if the slogan is ready. "
            "Keep responses under 60 words."
        ),
        client=client,
    )

    workflow = GroupChatBuilder(
        participants=[writer, reviewer],
        selection_func=round_robin_selector,
        termination_condition=lambda conv: len(conv) >= 4,
        max_rounds=6,
    ).build()

    result = await workflow.run(
        [Message(role="user", contents=["Create a slogan for an eco-friendly electric vehicle."])]
    )
    print(f"Result:\n{result}\n")

    # ══════════════════════════════════════════════════════════════════════════
    #  Scenario 2: Approval-Based Termination
    # ══════════════════════════════════════════════════════════════════════════
    print("========================================")
    print("  Scenario 2: Approval-Based Termination")
    print("========================================\n")

    writer2 = Agent(
        name="Writer",
        instructions=(
            "You are a creative copywriter. Write short product descriptions (under 50 words). "
            "Revise based on feedback."
        ),
        client=client,
    )

    editor = Agent(
        name="Editor",
        instructions=(
            "You are a senior editor. Review product descriptions for quality. "
            "If the description is good, respond with exactly 'APPROVED: ' followed by the final text. "
            "Otherwise, provide specific feedback for improvement. Keep responses under 60 words."
        ),
        client=client,
    )

    def approval_termination(conversation):
        """Terminate when the Editor says APPROVED."""
        if not conversation:
            return False
        last = conversation[-1]
        last_text = str(last)
        return "APPROVED" in last_text.upper()

    workflow2 = GroupChatBuilder(
        participants=[writer2, editor],
        selection_func=round_robin_selector,
        termination_condition=approval_termination,
        max_rounds=6,
    ).build()

    result2 = await workflow2.run(
        [Message(role="user", contents=["Write a product description for a smart water bottle that tracks hydration."])]
    )
    print(f"Result:\n{result2}\n")

    print("Group chat complete!")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Scenario 1** shows a round-robin group chat where the CopyWriter and Reviewer take turns — the writer creates a slogan, the reviewer provides feedback, and they iterate until the conversation reaches 4 messages.

**Scenario 2** demonstrates an approval-based termination that ends the conversation early when the Editor responds with "APPROVED". The `termination_condition` lambda inspects the conversation to decide when to stop.
