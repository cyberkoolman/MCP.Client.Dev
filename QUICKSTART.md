# Quick Start Guide

Get started with MCP client development in 5 minutes.

## Prerequisites

```bash
# Install .NET 8 or later
dotnet --version

# Install Node.js (for test servers)
node --version
```

## Step 1: Create a New Project

```bash
mkdir MyMcpClient
cd MyMcpClient
dotnet new console
```

## Step 2: Install Packages

```bash
dotnet add package ModelContextProtocol --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
dotnet add package Azure.AI.OpenAI --prerelease
dotnet add package Azure.Identity
```

## Step 3: Write Your Client

Replace `Program.cs` with:

```csharp
using ModelContextProtocol.Client;
using Microsoft.Extensions.AI;
using Azure.AI.OpenAI;
using Azure.Identity;

// 1. Create MCP client
var transport = new StdioClientTransport(new()
{
    Command = "npx",
    Arguments = ["-y", "@modelcontextprotocol/server-everything"],
    Name = "Everything Server"
});

var mcpClient = await McpClient.CreateAsync(transport);
Console.WriteLine("✅ Connected to MCP server");

// 2. Get tools
var tools = await mcpClient.ListToolsAsync();
Console.WriteLine($"📋 Found {tools.Count} tools\n");

// 3. Create LLM client (replace with your endpoint)
var chatClient = new ChatClientBuilder(
    new AzureOpenAIClient(
        new Uri("YOUR-ENDPOINT-HERE"),
        new DefaultAzureCredential())
    .GetChatClient("gpt-4o")
    .AsIChatClient())
    .UseFunctionInvocation()
    .Build();

// 4. Chat loop
var messages = new List<ChatMessage>();
Console.WriteLine("💬 Chat started (type 'exit' to quit)\n");

while (true)
{
    Console.Write("You: ");
    var input = Console.ReadLine();
    if (input?.ToLower() == "exit") break;
    
    messages.Add(new(ChatRole.User, input));
    
    var updates = new List<ChatResponseUpdate>();
    await foreach (var update in chatClient.GetStreamingResponseAsync(
        messages, new() { Tools = [.. tools] }))
    {
        Console.Write(update);
        updates.Add(update);
    }
    Console.WriteLine("\n");
    
    messages.AddMessages(updates);
}
```

## Step 4: Run

```bash
dotnet run
```

## What Just Happened?

1. **Connected** to an MCP server (the "everything" test server)
2. **Discovered** available tools from the server
3. **Created** an LLM client that can use those tools
4. **Started** a conversation where the LLM can call tools automatically

## Try These Prompts

- "Use the echo tool to say hello"
- "Add 5 and 3"
- "Tell me a joke and then echo it back"

## Next Steps

### Test Different Servers

```bash
# Weather server
Command = "npx",
Arguments = ["-y", "@modelcontextprotocol/server-weather"]

# File system server
Command = "npx",
Arguments = ["-y", "@modelcontextprotocol/server-filesystem"]
```

### Add Security

```csharp
// Before calling tools
Console.WriteLine($"⚠️  Execute {toolName}? (y/n)");
if (Console.ReadLine() != "y") return;
```

### Use Multiple Servers

```csharp
var weatherClient = await McpClient.CreateAsync(weatherTransport);
var dbClient = await McpClient.CreateAsync(dbTransport);

// Combine tools
var allTools = new List<McpClientTool>();
allTools.AddRange(await weatherClient.ListToolsAsync());
allTools.AddRange(await dbClient.ListToolsAsync());
```

## Troubleshooting

### "npx command not found"
Install Node.js: https://nodejs.org/

### "Azure.AI.OpenAI error"
Replace with your actual Azure OpenAI endpoint and ensure credentials are configured.

### "Connection refused"
Check that the server command is correct and Node.js is installed.

### "No tools found"
Server might not have started. Check logs with `LogLevel.Debug`.

## Test Without Azure OpenAI

Use local LLM with Ollama:

```bash
# Install Ollama
# Download a model: ollama pull llama2
```

```csharp
// Replace Azure client with Ollama
dotnet add package Microsoft.Extensions.AI.Ollama --prerelease

var chatClient = new OllamaChatClient(
    new Uri("http://localhost:11434"), 
    "llama2");
```

## Debug with MCP Inspector

```bash
npx @modelcontextprotocol/inspector
```

Opens a visual interface to test servers before integrating with your client.

## Resources

- [Full Guide](README.md) - Complete documentation
- [Cheat Sheet](CHEATSHEET.md) - Quick reference
- [FAQ](FAQ.md) - Common questions
- [Example Code](example.cs) - Complete example

## Common Issues

| Issue | Solution |
|-------|----------|
| npm errors | Install Node.js |
| Tool not found | Check server exposes that tool |
| LLM not calling tools | Ensure `.UseFunctionInvocation()` is used |
| Connection timeout | Verify server command is correct |

---

**Got it working?** Check out the [full guide](README.md) for advanced topics!
