# Lab 5: Structured Output — C# Implementation

[← Back to Lab Overview](./README.md) | [📋 Lab Guide](../../lab-guide.md)

## Step 1: Create the Project

```bash
dotnet new console -n Lab5_StructuredOutput
cd Lab5_StructuredOutput
dotnet add package Azure.AI.OpenAI
dotnet add package Azure.Identity
dotnet add package Microsoft.Agents.AI.OpenAI
```

## Step 2: Set Environment Variables

```bash
# Windows PowerShell
$env:AZURE_OPENAI_ENDPOINT = "https://your-resource.openai.azure.com/"
$env:AZURE_OPENAI_DEPLOYMENT_NAME = "gpt-4o-mini"

# Bash / macOS / Linux
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"
export AZURE_OPENAI_DEPLOYMENT_NAME="gpt-4o-mini"
```

## Step 3: Define Output Types and Extract Data

Replace `Program.cs`:

```csharp
using System;
using System.Text.Json;
using Azure.AI.OpenAI;
using Azure.Identity;
using Microsoft.Agents.AI;
using OpenAI.Chat;
using Microsoft.Extensions.AI;

var endpoint = Environment.GetEnvironmentVariable("AZURE_OPENAI_ENDPOINT")
    ?? throw new InvalidOperationException("Set AZURE_OPENAI_ENDPOINT");
var deploymentName = Environment.GetEnvironmentVariable("AZURE_OPENAI_DEPLOYMENT_NAME") ?? "gpt-4o-mini";

AIAgent agent = new AzureOpenAIClient(new Uri(endpoint), new AzureCliCredential())
    .GetChatClient(deploymentName)
    .AsAIAgent(instructions: "You extract structured data from text accurately.", name: "Extractor");

// ── Define a structured output type ──────────────────────────────────────────
public class PersonInfo
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public string Occupation { get; set; } = "";
    public string City { get; set; } = "";
}

// ── Extract structured data with RunAsync<T> ─────────────────────────────────
Console.WriteLine("=== Example 1: RunAsync<T> ===\n");

var text = "Meet Sarah Chen, a 28-year-old data scientist living in Amsterdam. She specializes in NLP.";
Console.WriteLine($"Input: {text}\n");

AgentResponse<PersonInfo> response = await agent.RunAsync<PersonInfo>(text);
PersonInfo person = response.Result;

Console.WriteLine($"  Name:       {person.Name}");
Console.WriteLine($"  Age:        {person.Age}");
Console.WriteLine($"  Occupation: {person.Occupation}");
Console.WriteLine($"  City:       {person.City}");

// ── Extract a list via a wrapper type ────────────────────────────────────────
Console.WriteLine("\n=== Example 2: Extract Multiple Items ===\n");

public class MeetingInfo
{
    public string Title { get; set; } = "";
    public string Date { get; set; } = "";
    public string Time { get; set; } = "";
    public List<string> Attendees { get; set; } = [];
    public string ActionItem { get; set; } = "";
}

public class MeetingList
{
    public List<MeetingInfo> Meetings { get; set; } = [];
}

var meetingNotes = """
    Team sync on March 5th at 2pm with Alice, Bob, and Carol.
    Action: Alice to send the Q1 report by Friday.

    Design review on March 7th at 10am with Dave and Eve.
    Action: Dave to update the wireframes.
    """;

Console.WriteLine($"Input:\n{meetingNotes}");

var meetings = await agent.RunAsync<MeetingList>(meetingNotes);
foreach (var m in meetings.Result.Meetings)
{
    Console.WriteLine($"  📅 {m.Title} — {m.Date} at {m.Time}");
    Console.WriteLine($"     Attendees: {string.Join(", ", m.Attendees)}");
    Console.WriteLine($"     Action: {m.ActionItem}\n");
}

// ── Using ResponseFormat with AgentRunOptions ────────────────────────────────
Console.WriteLine("=== Example 3: ResponseFormat (runtime schema) ===\n");

var runOptions = new AgentRunOptions
{
    ResponseFormat = ChatResponseFormat.ForJsonSchema<PersonInfo>()
};

var response2 = await agent.RunAsync(
    "Tell me about John Smith, 35, software engineer in Seattle.", options: runOptions);

var parsed = JsonSerializer.Deserialize<PersonInfo>(response2.Text, JsonSerializerOptions.Web)!;
Console.WriteLine($"  Name: {parsed.Name}, Age: {parsed.Age}, Occupation: {parsed.Occupation}, City: {parsed.City}");
```

## Step 4: Run It

```bash
dotnet run
```

**Observe:** Instead of free-form text, you get strongly-typed C# objects you can immediately use in your application logic.

---

## 🏋️ Exercises

### Exercise A: Build a Receipt Parser

Create a `ReceiptInfo` type and extract data from receipt text:

```csharp
public class ReceiptItem
{
    public string Name { get; set; } = "";
    public double Price { get; set; }
    public int Quantity { get; set; }
}

public class ReceiptInfo
{
    public string StoreName { get; set; } = "";
    public string Date { get; set; } = "";
    public List<ReceiptItem> Items { get; set; } = [];
    public double Total { get; set; }
}
```

Test with: `"Grocery Mart, Feb 25 2026. Apples x3 $4.50, Bread x1 $3.00, Milk x2 $7.00. Total: $14.50"`

### Exercise B: Sentiment Analysis

Create a structured output that classifies text sentiment:

```csharp
public class SentimentResult
{
    public string Sentiment { get; set; } = ""; // positive, negative, neutral
    public double Confidence { get; set; }       // 0.0 to 1.0
    public string Summary { get; set; } = "";
}
```

### Exercise C (Stretch): Structured + Streaming

Use `RunStreamingAsync` with structured output — stream the tokens, then parse the final result:

```csharp
var updates = new List<AgentResponseUpdate>();
await foreach (var update in agent.RunStreamingAsync(text, options: runOptions))
{
    Console.Write(update);
    updates.Add(update);
}
// Assemble and deserialize the final response
var fullText = string.Join("", updates.Select(u => u.ToString()));
var result = JsonSerializer.Deserialize<PersonInfo>(fullText, JsonSerializerOptions.Web);
```
