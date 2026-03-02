# Lab 13: Observability & Telemetry — Python Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../lab-guide.md)

## Step 1: Create the Project

```bash
mkdir lab13_observability
cd lab13_observability
python -m venv .venv

# Windows PowerShell
.venv\Scripts\Activate.ps1
# Bash / macOS / Linux
# source .venv/bin/activate
```

## Step 2: Install Packages

```bash
pip install agent-framework --pre azure-identity opentelemetry-sdk opentelemetry-exporter-console
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

## Step 4: Add Telemetry to an Agent

Create a file named `main.py`:

```python
import asyncio
import random

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, SimpleSpanProcessor
from opentelemetry.sdk.resources import Resource

from agent_framework import tool
from agent_framework.azure import AzureOpenAIChatClient
from azure.identity import AzureCliCredential

# ── Configure OpenTelemetry to export to console ──────────────────────────────
resource = Resource.create({"service.name": "maf-lab13"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(SimpleSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)


# ── Tool ──────────────────────────────────────────────────────────────────────
@tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    return f"Weather in {city}: sunny, {random.randint(15, 30)}°C."


async def main():
    client = AzureOpenAIChatClient(credential=AzureCliCredential())
    agent = client.as_agent(
        name="TelemetryAgent",
        instructions="You are a helpful weather assistant.",
        tools=[get_weather],
    )

    # ── Run the agent and observe telemetry ───────────────────────────────────
    print("═══ Running agent with OpenTelemetry ═══\n")

    print("Q: What's the weather in Amsterdam?")
    result = await agent.run("What's the weather in Amsterdam?")
    print(f"A: {result}\n")

    print("Q: How about Tokyo?")
    result = await agent.run("How about Tokyo?")
    print(f"A: {result}\n")

    print("═══ Check console output above for trace spans! ═══\n")
    print("Look for:")
    print("  - invoke_agent spans with agent name and instructions")
    print("  - gen_ai.usage.input_tokens and output_tokens")
    print("  - execute_tool spans with function name and duration")


asyncio.run(main())
```

## Step 5: Run It

```bash
python main.py
```

**Observe in the console output:**
- Trace spans emitted to the console for each agent invocation
- Token usage attributes (`gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`)
- Tool call spans showing which functions were called
- Trace IDs for correlating related operations

### Exercise B: Add Metrics Export

```python
from opentelemetry import metrics
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import ConsoleMetricExporter, PeriodicExportingMetricReader

reader = PeriodicExportingMetricReader(ConsoleMetricExporter())
meter_provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(meter_provider)
```

### Exercise C: Export to OTLP (e.g., Aspire Dashboard)

```bash
pip install opentelemetry-exporter-otlp
```

```python
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter

provider.add_span_processor(
    SimpleSpanProcessor(OTLPSpanExporter(endpoint="http://localhost:4317"))
)
```
