# Lab 5: Structured Output — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab5_structured_output
cd lab5_structured_output
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

## Step 4: Define Output Types and Extract Data

Create a file named `main.py`:

```python
import asyncio
from pydantic import BaseModel
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

# ── Define structured output types ────────────────────────────────────────────
class PersonInfo(BaseModel):
    name: str
    age: int
    occupation: str
    city: str

class MeetingInfo(BaseModel):
    title: str
    date: str
    time: str
    attendees: list[str]
    action_item: str

class MeetingList(BaseModel):
    meetings: list[MeetingInfo]

async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        instructions="You extract structured data from text accurately.",
        name="Extractor",
    )

    # ── Extract structured data ───────────────────────────────────────────────
    print("=== Example 1: PersonInfo ===\n")

    text = "Meet Sarah Chen, a 28-year-old data scientist living in Amsterdam. She specializes in NLP."
    print(f"Input: {text}\n")

    result = await agent.run(text, options={"response_format": PersonInfo})
    if result.value:
        person = result.value
        print(f"  Name:       {person.name}")
        print(f"  Age:        {person.age}")
        print(f"  Occupation: {person.occupation}")
        print(f"  City:       {person.city}")

    # ── Extract a list ────────────────────────────────────────────────────────
    print("\n=== Example 2: Extract Multiple Items ===\n")

    meeting_notes = """
    Team sync on March 5th at 2pm with Alice, Bob, and Carol.
    Action: Alice to send the Q1 report by Friday.

    Design review on March 7th at 10am with Dave and Eve.
    Action: Dave to update the wireframes.
    """

    print(f"Input:\n{meeting_notes}")

    result = await agent.run(meeting_notes, options={"response_format": MeetingList})
    if result.value:
        for m in result.value.meetings:
            print(f"  📅 {m.title} — {m.date} at {m.time}")
            print(f"     Attendees: {', '.join(m.attendees)}")
            print(f"     Action: {m.action_item}\n")

    # ── Runtime schema extraction ─────────────────────────────────────────────
    print("=== Example 3: Runtime Schema ===\n")

    result = await agent.run(
        "Tell me about John Smith, 35, software engineer in Seattle.",
        options={"response_format": PersonInfo},
    )
    if result.value:
        p = result.value
        print(f"  Name: {p.name}, Age: {p.age}, Occupation: {p.occupation}, City: {p.city}")

asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe:** Instead of free-form text, you get strongly-typed Pydantic objects you can immediately use in your application logic.

---

## 🏋️ Exercises

### Exercise A: Build a Receipt Parser

Create Pydantic models and extract data from receipt text:

```python
class ReceiptItem(BaseModel):
    name: str
    price: float
    quantity: int

class ReceiptInfo(BaseModel):
    store_name: str
    date: str
    items: list[ReceiptItem]
    total: float
```

Test with: `"Grocery Mart, Feb 25 2026. Apples x3 $4.50, Bread x1 $3.00, Milk x2 $7.00. Total: $14.50"`

### Exercise B: Sentiment Analysis

Create a structured output that classifies text sentiment:

```python
class SentimentResult(BaseModel):
    sentiment: str   # positive, negative, neutral
    confidence: float  # 0.0 to 1.0
    summary: str
```

### Exercise C (Stretch): Structured + Streaming

Use streaming with structured output — stream the tokens, then parse the final result:

```python
import json

full_text = ""
async for chunk in agent.run(text, stream=True):
    if chunk.text:
        print(chunk.text, end="", flush=True)
        full_text += chunk.text
print()

# Parse the final response
result = PersonInfo.model_validate_json(full_text)
print(f"  Name: {result.name}, Age: {result.age}")
```
