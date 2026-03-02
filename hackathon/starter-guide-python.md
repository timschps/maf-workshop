# Hackathon Starter Guide (Python)

This guide provides scaffolding and integration patterns for adding agentic experiences to an existing application using the Microsoft Agent Framework.

---

## Quick Setup

### 1. Add MAF to an Existing Python Project

```bash
cd your_partner_app
pip install agent-framework --pre
pip install openai
pip install azure-identity
```

### 2. Create a Shared Agent Factory

Add a helper module that creates and configures agents. This keeps agent setup centralized and consistent across the app.

```python
# services/agent_factory.py
import os
from openai import AzureOpenAI
from azure.identity import AzureCliCredential
from agent_framework import Agent

class AgentFactory:
    def __init__(self):
        self._endpoint = os.environ.get("AZURE_OPENAI_ENDPOINT")
        if not self._endpoint:
            raise ValueError("Set AZURE_OPENAI_ENDPOINT")
        self._deployment = os.environ.get("AZURE_OPENAI_CHAT_DEPLOYMENT_NAME", "gpt-4o-mini")

    def _get_client(self):
        credential = AzureCliCredential()
        token = credential.get_token("https://cognitiveservices.azure.com/.default")
        return AzureOpenAI(
            azure_endpoint=self._endpoint,
            azure_ad_token=token.token,
            api_version="2024-12-01-preview",
        )

    def create_agent(self, name: str, instructions: str, tools: list = None) -> Agent:
        """Create a basic agent with the given instructions and optional tools."""
        client = self._get_client()
        return Agent(
            name=name,
            model=self._deployment,
            client=client,
            instructions=instructions,
            tools=tools or [],
        )
```

### 3. Register in Your App (FastAPI Example)

```python
# main.py — add to your FastAPI app
from services.agent_factory import AgentFactory

factory = AgentFactory()
```

---

## Scenario Scaffolding

### Scenario A: Conversational Assistant

Add a chat agent that helps users navigate the application.

**Integration point:** Add a `/api/chat` endpoint that accepts user messages and returns agent responses.

```python
# routes/chat.py
from fastapi import APIRouter
from pydantic import BaseModel
from agent_framework import Agent, tool
from services.agent_factory import AgentFactory

router = APIRouter()
factory = AgentFactory()

# Store sessions in memory (use Redis or a database in production)
sessions: dict[str, list] = {}


class ChatRequest(BaseModel):
    session_id: str
    message: str


class ChatResponse(BaseModel):
    reply: str


# ── Example tools — replace with your actual app services ────────────────

@tool
def search_products(query: str) -> str:
    """Search for products by name or category."""
    # TODO: Replace with actual service call
    # e.g., return await product_service.search(query)
    return f"Found 3 products matching '{query}': Widget A ($10), Widget B ($15), Widget C ($20)."


@tool
def get_order_status(order_id: str) -> str:
    """Get the status of an order by order ID."""
    # TODO: Replace with actual service call
    return f"Order {order_id}: Shipped on Feb 25, expected delivery Feb 28."


@router.post("/api/chat", response_model=ChatResponse)
async def chat(request: ChatRequest):
    # ── Create the agent with tools that wrap your existing services ─────
    agent = factory.create_agent(
        name="AppAssistant",
        instructions="""You are a helpful assistant for [App Name].
You can help users find information, navigate features, and answer questions.
Always be concise and helpful.
If you don't know something, say so — don't make things up.""",
        tools=[search_products, get_order_status],
    )

    # ── Get or create session for this user ─────────────────────────────
    if request.session_id not in sessions:
        sessions[request.session_id] = []
    history = sessions[request.session_id]

    # ── Run the agent ───────────────────────────────────────────────────
    result = await agent.run(request.message, history=history)

    return ChatResponse(reply=str(result))
```

---

### Scenario B: Process Automation (Workflow)

Automate a multi-step business process using a MAF workflow.

**Example:** Order processing pipeline

```python
# workflows/order_processing.py
from agent_framework import Agent, tool
from agent_framework.orchestrations import SequentialWorkflow
from services.agent_factory import AgentFactory


def create_order_workflow(factory: AgentFactory):
    """Build a sequential order-processing pipeline."""

    # ── Step 1: Agent validates the order ─────────────────────────────────
    validate_agent = factory.create_agent(
        name="ValidateAgent",
        instructions="""You validate orders. Check that the order JSON contains:
- A valid product ID
- A positive quantity
- A shipping address
Report any issues found, or confirm the order is valid.""",
    )

    # ── Step 2: Agent enriches the order (e.g., suggests upsells) ────────
    enrich_agent = factory.create_agent(
        name="EnrichAgent",
        instructions="You analyze orders and suggest one relevant upsell product. Be brief.",
    )

    # ── Step 3: Agent generates confirmation email ───────────────────────
    confirm_agent = factory.create_agent(
        name="ConfirmAgent",
        instructions="You write friendly, concise order confirmation emails.",
    )

    # ── Build the sequential workflow ────────────────────────────────────
    workflow = SequentialWorkflow(
        agents=[validate_agent, enrich_agent, confirm_agent]
    )
    return workflow


# ── Usage ────────────────────────────────────────────────────────────────
async def process_order(order_json: str):
    factory = AgentFactory()
    workflow = create_order_workflow(factory)

    result = await workflow.run(
        f"Process this order: {order_json}"
    )
    print("Final result:", result)
```

---

### Scenario C: Smart Search & Insights

Add an agent that searches application data and provides intelligent summaries.

```python
# services/insights_agent.py
from agent_framework import Agent, tool
from services.agent_factory import AgentFactory


@tool
def search_database(query: str) -> str:
    """Search the application database with a natural language query. Returns matching records."""
    # TODO: Replace with actual database search
    # e.g., use SQLAlchemy, Azure AI Search, etc.
    return f"Found 42 records matching '{query}'. Top results: [Record 1], [Record 2], [Record 3]."


@tool
def get_sales_metrics(period: str) -> str:
    """Get sales metrics for a given time period (e.g., 'last month', 'Q4 2025')."""
    # TODO: Replace with actual metrics query
    return f"Sales for {period}: Revenue $125K, Orders 340, Avg order $367, Growth +12%."


@tool
def get_customer_feedback(topic: str) -> str:
    """Get recent customer feedback and reviews for a product or category."""
    # TODO: Replace with actual feedback query
    return f"Recent feedback for '{topic}': 4.2/5 avg rating, 89% positive sentiment. Top complaint: shipping delays."


class InsightsAgentService:
    def __init__(self, factory: AgentFactory):
        self._agent = factory.create_agent(
            name="InsightsAgent",
            instructions="""You are a data insights assistant for [App Name].
When users ask questions about their data, use the available tools to search
and then provide a clear, concise summary with key takeaways.
Always cite which data sources you used.
Format responses with bullet points for readability.""",
            tools=[search_database, get_sales_metrics, get_customer_feedback],
        )

    async def ask(self, question: str) -> str:
        result = await self._agent.run(question)
        return str(result)
```

---

### Scenario D: Multi-Agent Collaboration

Build specialist agents that work together via Agent-as-a-Tool.

```python
# services/multi_agent.py
from agent_framework import Agent, tool
from services.agent_factory import AgentFactory


@tool
def query_data(query: str) -> str:
    """Query application data with a SQL-like natural language request."""
    # TODO: Replace with actual data query
    return f"Query results for '{query}': [sample data rows]"


@tool
def get_user_history(user_id: str) -> str:
    """Get a user's purchase history and preferences."""
    # TODO: Replace with actual user data lookup
    return f"User {user_id}: 15 purchases, prefers electronics, avg spend $85, last active 2 days ago."


def create_collaborative_agent(factory: AgentFactory) -> Agent:
    # ── Specialist: Data Analyst ─────────────────────────────────────────
    analyst_agent = factory.create_agent(
        name="DataAnalyst",
        instructions="""You are a data analyst. When given data or a question about data,
provide statistical analysis, identify trends, and highlight anomalies. Be precise.""",
        tools=[query_data],
    )

    # ── Specialist: Recommendation Engine ────────────────────────────────
    recommender_agent = factory.create_agent(
        name="Recommender",
        instructions="""You are a recommendation specialist. Based on user data and preferences,
suggest relevant products, actions, or next steps. Be specific and actionable.""",
        tools=[get_user_history],
    )

    # ── Main orchestrator agent ──────────────────────────────────────────
    orchestrator = factory.create_agent(
        name="Orchestrator",
        instructions="""You are the main assistant for [App Name].
You coordinate with specialist agents to answer complex questions.
- Use the DataAnalyst for data-related questions
- Use the Recommender for personalized suggestions
Always synthesize the specialists' responses into a cohesive answer.""",
        tools=[analyst_agent.as_tool(), recommender_agent.as_tool()],
    )
    return orchestrator
```

---

## Wiring Tools to Your Existing Services

The key to the hackathon is connecting MAF tools to the partner app's **existing services**. Here's the pattern:

```python
# ── Pattern: Wrap an existing service as a tool ──────────────────────────────
import json
from agent_framework import Agent, tool
from services.product_service import ProductService
from services.order_service import OrderService


class AgentToolBridge:
    """Bridge between MAF tools and your existing services."""

    def __init__(self, product_service: ProductService, order_service: OrderService):
        self._product_service = product_service
        self._order_service = order_service

    @tool
    async def search_products(self, query: str) -> str:
        """Search products in the catalog by name, category, or keyword."""
        results = await self._product_service.search(query)
        return json.dumps(results[:5], default=str)

    @tool
    async def get_product_details(self, product_id: str) -> str:
        """Get detailed information about a specific product by ID."""
        product = await self._product_service.get_by_id(product_id)
        return json.dumps(product, default=str) if product else f"Product {product_id} not found."

    @tool
    async def place_order(self, customer_id: str, product_id: str, quantity: int) -> str:
        """Place a new order for a customer."""
        order = await self._order_service.create(customer_id, product_id, quantity)
        return f"Order {order.id} created successfully. Total: ${order.total:.2f}."


# ── Usage: register tools from the bridge ────────────────────────────────────
bridge = AgentToolBridge(product_service, order_service)
agent = factory.create_agent(
    "AppAssistant",
    "You help users find and order products.",
    tools=[bridge.search_products, bridge.get_product_details, bridge.place_order],
)
```

---

## Adding a Simple Chat UI

If the partner app has a web frontend, here's a minimal chat component:

```html
<!-- chat-widget.html — add to your existing page -->
<div id="agent-chat" style="position:fixed; bottom:20px; right:20px; width:380px; border:1px solid #ccc; border-radius:12px; background:#fff; box-shadow:0 4px 12px rgba(0,0,0,0.15); font-family:system-ui;">
  <div style="background:#0078d4; color:white; padding:12px 16px; border-radius:12px 12px 0 0; font-weight:600;">
    🤖 AI Assistant
  </div>
  <div id="chat-messages" style="height:300px; overflow-y:auto; padding:12px;"></div>
  <div style="display:flex; border-top:1px solid #eee; padding:8px;">
    <input id="chat-input" type="text" placeholder="Ask me anything..."
           style="flex:1; border:1px solid #ddd; border-radius:8px; padding:8px 12px; outline:none;"
           onkeypress="if(event.key==='Enter')sendMessage()">
    <button onclick="sendMessage()"
            style="margin-left:8px; background:#0078d4; color:white; border:none; border-radius:8px; padding:8px 16px; cursor:pointer;">
      Send
    </button>
  </div>
</div>

<script>
const sessionId = crypto.randomUUID();
const messagesDiv = document.getElementById('chat-messages');

function addMessage(text, isUser) {
  const div = document.createElement('div');
  div.style.cssText = `margin:8px 0; padding:8px 12px; border-radius:8px; max-width:80%; ${
    isUser ? 'margin-left:auto; background:#e3f2fd;' : 'background:#f5f5f5;'
  }`;
  div.textContent = text;
  messagesDiv.appendChild(div);
  messagesDiv.scrollTop = messagesDiv.scrollHeight;
}

async function sendMessage() {
  const input = document.getElementById('chat-input');
  const message = input.value.trim();
  if (!message) return;

  addMessage(message, true);
  input.value = '';

  try {
    const res = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ sessionId, message })
    });
    const data = await res.json();
    addMessage(data.reply, false);
  } catch (err) {
    addMessage('Sorry, something went wrong. Please try again.', false);
  }
}
</script>
```

---

## Hackathon Tips

1. **Start simple:** Get a basic agent responding to one question before adding tools
2. **Tools are your bridge:** Every existing service can become a tool with the `@tool` decorator
3. **Don't over-engineer:** A working demo of 1 scenario beats a broken demo of 3
4. **Use sessions:** Multi-turn conversations make the demo much more impressive
5. **Add middleware last:** Logging/security middleware is polish, not core functionality
6. **Test with real questions:** Think about what a real user would actually ask

---

## Key Differences from C\#

| Aspect | C# | Python |
|--------|-----|--------|
| Package | `Microsoft.Agents.AI.OpenAI` | `agent-framework` |
| Tool decorator | `[Description("...")]` attribute | `@tool` decorator |
| Agent-as-Tool | `agent.AsAIFunction()` | `agent.as_tool()` |
| Workflow | `AgentWorkflowBuilder` | `SequentialWorkflow(agents=[...])` |
| Env var for model | `AZURE_OPENAI_DEPLOYMENT_NAME` | `AZURE_OPENAI_CHAT_DEPLOYMENT_NAME` |
| Session | `agent.CreateSessionAsync()` | `agent.run(msg, history=history)` |
| DI framework | `builder.Services.AddSingleton<>()` | Manual / FastAPI `Depends()` |
| Async | `async Task<T>` | `async def` / `await` |
