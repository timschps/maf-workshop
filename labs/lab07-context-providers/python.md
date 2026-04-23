# Lab 7: Context Providers & Memory — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab7_context_providers
cd lab7_context_providers
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

## Step 4: Agent with Custom Context Injection

Create a file named `main.py`:

```python
import asyncio
from agent_framework.openai import OpenAIChatCompletionClient
from azure.identity import AzureCliCredential

# ── Simulate a user profile store ────────────────────────────────────────────
user_profiles = {
    "user-alice": {
        "name": "Alice",
        "role": "Senior Developer",
        "interests": "cloud architecture, AI agents, hiking",
        "preferred_language": "C#",
    },
    "user-bob": {
        "name": "Bob",
        "role": "Product Manager",
        "interests": "roadmap planning, user research",
        "preferred_language": "Python",
    },
}

async def chat_as_user(client, user_id):
    profile = user_profiles[user_id]

    # Build personalized instructions that include user context
    personalized_instructions = f"""
You are a helpful assistant. You know the following about the current user:
- Name: {profile["name"]}
- Role: {profile["role"]}
- Interests: {profile["interests"]}
- Preferred programming language: {profile["preferred_language"]}

Always address the user by name. Tailor your responses to their role and
interests. When showing code examples, use their preferred language.
"""

    agent = client.as_agent(
        instructions=personalized_instructions,
        name=f"Assistant-{profile['name']}",
    )

    session = agent.create_session()

    print(f"═══ Chatting as {profile['name']} ({profile['role']}) ═══\n")

    # The agent should already know who the user is
    print("Q: Who am I and what do I do?")
    print("A: ", end="", flush=True)
    async for chunk in agent.run("Who am I and what do I do?", session=session, stream=True):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print("\n")

    # The agent should use their preferred language
    print("Q: Show me a quick example of an HTTP request.")
    print("A: ", end="", flush=True)
    async for chunk in agent.run("Show me a quick example of an HTTP request.", session=session, stream=True):
        if chunk.text:
            print(chunk.text, end="", flush=True)
    print("\n")

async def main():
    client = OpenAIChatCompletionClient(credential=AzureCliCredential())

    await chat_as_user(client, "user-alice")
    print("─" * 70 + "\n")
    await chat_as_user(client, "user-bob")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:**
- Alice gets C# code examples; Bob gets Python
- Both agents address users by name without being told
- The "context" is injected at agent creation time via instructions

---

## 🏋️ Exercises

### Exercise A: Dynamic Context Injection

Instead of hardcoding context into instructions, build a helper that combines a base prompt with runtime context:

```python
def build_instructions(base_instructions: str, context: dict[str, str]) -> str:
    context_block = "\n".join(f"- {k}: {v}" for k, v in context.items())
    return f"{base_instructions}\n\nCurrent user context:\n{context_block}"

instructions = build_instructions(
    "You are a helpful assistant.",
    user_profiles["user-alice"],
)
```

### Exercise B: Custom Context Provider

Create a custom `BaseContextProvider` that injects context into every agent run:

```python
from agent_framework import BaseContextProvider, Message

class UserProfileContextProvider(BaseContextProvider):
    def __init__(self, profile: dict[str, str]):
        super().__init__(source_id='user-profile')
        self.profile = profile

    async def before_run(self, *, agent, session, context, state):
        context_text = "\n".join(f"- {k}: {v}" for k, v in self.profile.items())
        context.extend_instructions(self.source_id, f"Current user context:\n{context_text}")

agent = client.as_agent(
    instructions="You are a helpful assistant.",
    context_providers=[UserProfileContextProvider(user_profiles["user-alice"])],
)
```

### Exercise C (Stretch): Knowledge Base Context

Create a "knowledge base" context provider that injects FAQ entries:

```python
from agent_framework import BaseContextProvider

class FAQContextProvider(BaseContextProvider):
    def __init__(self, faqs: list[str]):
        super().__init__(source_id='faq-provider')
        self.faqs = faqs

    async def before_run(self, *, agent, session, context, state):
        faq_text = "\n".join(self.faqs)
        context.extend_instructions(self.source_id, f"Answer based on these FAQs:\n{faq_text}\n\nIf the answer is not in the FAQs, say 'I'll escalate this to a human agent.'")

faqs = [
    "Q: What are your hours? A: We are open Mon-Fri 9am-5pm CET.",
    "Q: How do I reset my password? A: Go to Settings > Security > Reset.",
    "Q: What's the refund policy? A: Full refund within 30 days.",
]

agent = client.as_agent(
    instructions="You are a customer support agent.",
    context_providers=[FAQContextProvider(faqs)],
)
```
