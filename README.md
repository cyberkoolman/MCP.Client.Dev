# MCP Client Development Guide

A comprehensive guide to understanding and developing Model Context Protocol (MCP) clients.

## Table of Contents

- [What is MCP?](#what-is-mcp)
- [Core Architecture](#core-architecture)
- [Key Concepts for Client Developers](#key-concepts-for-client-developers)
- [NPM Package Syntax Explained](#npm-package-syntax-explained)
- [C# SDK Coverage Analysis](#c-sdk-coverage-analysis)
- [Getting Started](#getting-started)
- [Resources](#resources)

## What is MCP?

The **Model Context Protocol (MCP)** is an open standard that enables seamless integration between AI systems (like LLMs) and various data sources and tools. Think of it as **USB-C for AI applications** - a universal connector that standardizes how LLMs access context.

### Key Benefits

- 🔌 **Single Integration Point**: Connect to any MCP server through one protocol
- 🔄 **LLM Independence**: Switch between LLM providers without changing integrations
- 🔒 **Built-in Security**: Human approval patterns and security checks
- 📦 **Extensibility**: Easy integration of new tools and data sources
- 🎯 **Modularity**: Separation of LLM logic from data access logic

## Core Architecture

MCP follows a **client-server architecture** with these components:

```
┌─────────────────────────────────────────────────┐
│  MCP Host (Your AI Application)                 │
│  ┌──────────────────────────────────────────┐  │
│  │  MCP Client 1 ←→ MCP Server (Weather)   │  │
│  │  MCP Client 2 ←→ MCP Server (Database)  │  │
│  │  MCP Client 3 ←→ MCP Server (Files)     │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Components

1. **MCP Hosts**: Applications users interact with (Claude Desktop, IDEs, custom apps)
2. **MCP Clients**: Live within host applications, maintain **1:1 connections** with servers
3. **MCP Servers**: Lightweight programs exposing capabilities via standard API

> **Important**: Each MCP client maintains a 1:1 connection with ONE server. For multiple servers, run multiple client instances.

## Key Concepts for Client Developers

### 1. Three Core Primitives

Your client must handle three types of capabilities:

#### 🛠️ Tools (Model-Controlled)
Functions that LLMs can invoke to perform actions.

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "inputSchema": {
    "type": "object",
    "properties": {
      "location": { "type": "string" }
    }
  }
}
```

#### 📄 Resources (Application-Controlled)
Data sources the LLM can read (similar to REST GET endpoints).

```json
{
  "uri": "file:///project/README.md",
  "name": "Project README",
  "mimeType": "text/markdown"
}
```

#### 💬 Prompts (User-Controlled)
Pre-defined templates that guide LLM interactions.

```json
{
  "name": "code_review",
  "description": "Review code for best practices",
  "arguments": [
    {
      "name": "code",
      "description": "Code to review",
      "required": true
    }
  ]
}
```

### 2. Transport Mechanisms

Choose how your client communicates:

- **stdio** (Standard Input/Output): For local servers on the same machine
- **HTTP with SSE**: For remote servers (Server-Sent Events)
- **Streamable HTTP**: Newer transport method

### 3. Connection Flow

```
┌──────────┐                           ┌──────────┐
│  Client  │                           │  Server  │
└────┬─────┘                           └────┬─────┘
     │                                      │
     │  1. Initialize (Handshake)           │
     │ ────────────────────────────────────>│
     │  <────────────────────────────────── │
     │                                      │
     │  2. Discover (List Tools/Resources)  │
     │ ────────────────────────────────────>│
     │  <────────────────────────────────── │
     │                                      │
     │  3. Invoke (Call Tool)               │
     │ ────────────────────────────────────>│
     │  <────────────────────────────────── │
     │                                      │
```

### 4. Client Responsibilities

When building an MCP client, you must handle:

✅ **Initialize** connections to MCP servers  
✅ **Discover** available capabilities  
✅ **Translate** LLM function calls → MCP requests  
✅ **Execute** tool calls and fetch resources  
✅ **Handle** results and feed back to LLM  
✅ **Manage** security and user approval  

## NPM Package Syntax Explained

When you see `@modelcontextprotocol/server-everything`, this is **NPM scoped package** syntax:

```
@modelcontextprotocol/server-everything
│                   │
│                   └─ Package name
└─ Scope (organization namespace)
```

### Common MCP Packages

- `@modelcontextprotocol/sdk` - TypeScript/JavaScript SDK
- `@modelcontextprotocol/server-everything` - Test server with all features
- `@modelcontextprotocol/inspector` - Debugging tool

### Example Usage

```bash
# Run with npx (no installation)
npx -y @modelcontextprotocol/server-everything

# Install globally
npm install -g @modelcontextprotocol/server-everything@latest

# Use in C# client
Command = "npx",
Arguments = ["-y", "@modelcontextprotocol/server-everything"]
```

## C# SDK Coverage Analysis

### ✅ What the C# SDK Handles

#### 1. Connection Management
```csharp
var client = await McpClient.CreateAsync(transport);
```
- ✅ Connection establishment
- ✅ Protocol handshake
- ✅ Transport management (stdio, HTTP)

#### 2. Capability Discovery
```csharp
// Tools
IList<McpClientTool> tools = await client.ListToolsAsync();

// Resources
var resources = await client.ListResourcesAsync();

// Prompts
var prompts = await client.ListPromptsAsync();
```

#### 3. Tool Execution
```csharp
var result = await client.CallToolAsync(
    "toolName",
    new Dictionary<string, object?>() { ["arg"] = "value" },
    cancellationToken
);
```

#### 4. Protocol Details
- ✅ JSON-RPC message formatting
- ✅ Request/response serialization
- ✅ Protocol version negotiation
- ✅ Error handling (McpException)

#### 5. Microsoft.Extensions.AI Integration
```csharp
// McpClientTool inherits from AIFunction
IList<McpClientTool> tools = await client.ListToolsAsync();
IChatClient chatClient = ...;

// Tools are ALREADY in LLM-compatible format!
var response = await chatClient.GetResponseAsync(
    "your prompt",
    new() { Tools = [.. tools] }
);
```

### ❌ What You Must Implement

#### 1. LLM Function Call Translation

```csharp
// YOU parse LLM responses
var llmResponse = await chatClient.GetResponseAsync(...);

foreach (var update in llmResponse)
{
    if (update is FunctionCallUpdate functionCall)
    {
        // Extract tool name and arguments
        string toolName = functionCall.Name;
        var args = functionCall.Arguments;
        
        // Call MCP
        var result = await client.CallToolAsync(toolName, args, ct);
    }
}
```

#### 2. Conversation Context Management

```csharp
List<ChatMessage> conversationHistory = [];

// Add user message
conversationHistory.Add(new ChatMessage(ChatRole.User, userInput));

// Get LLM response with tool calls
var response = await chatClient.GetResponseAsync(
    conversationHistory, 
    new() { Tools = tools }
);

// YOU must manage:
// - Assistant's tool call messages
// - Tool execution results  
// - Conversation continuation
conversationHistory.Add(new ChatMessage(ChatRole.Assistant, response));
conversationHistory.Add(new ChatMessage(ChatRole.Tool, toolResult));
```

#### 3. Security & Approval

```csharp
async Task<bool> GetUserApproval(string toolName, object args)
{
    Console.WriteLine($"⚠️  Allow execution of {toolName}?");
    Console.WriteLine($"   Arguments: {JsonSerializer.Serialize(args)}");
    Console.Write("   Approve? (y/n): ");
    
    return Console.ReadLine()?.ToLower() == "y";
}

if (await GetUserApproval(toolName, args))
{
    await client.CallToolAsync(toolName, args, ct);
}
```

#### 4. Orchestration Loop

```csharp
// YOU implement the agentic conversation loop
var messages = new List<ChatMessage>();

while (!done)
{
    // 1. Get user input
    var userInput = Console.ReadLine();
    messages.Add(new(ChatRole.User, userInput));
    
    // 2. Send to LLM with tools
    var updates = new List<ChatResponseUpdate>();
    await foreach (var update in chatClient.GetStreamingResponseAsync(
        messages, new() { Tools = tools }))
    {
        // 3. Check for tool calls in updates
        // 4. Execute via MCP client
        // 5. Add results to messages
        updates.Add(update);
    }
    
    // 6. Continue conversation
    messages.AddMessages(updates);
}
```

#### 5. Multiple Server Management

```csharp
// Each client = ONE server
// YOU route tool calls to the correct client

var weatherClient = await McpClient.CreateAsync(weatherTransport);
var dbClient = await McpClient.CreateAsync(dbTransport);
var fileClient = await McpClient.CreateAsync(fileTransport);

// Map tool names to clients
var toolRouter = new Dictionary<string, IMcpClient>
{
    ["get_weather"] = weatherClient,
    ["query_database"] = dbClient,
    ["read_file"] = fileClient
};

// Route tool calls
var targetClient = toolRouter[toolName];
await targetClient.CallToolAsync(toolName, args, ct);
```

## Getting Started

### Prerequisites

```bash
# For C# development
dotnet add package ModelContextProtocol --prerelease
dotnet add package Microsoft.Extensions.AI --prerelease
```

### Basic Client Example

```csharp
using ModelContextProtocol.Client;
using Microsoft.Extensions.AI;

// 1. Create transport (connects to MCP server)
var transport = new StdioClientTransport(new StdioClientTransportOptions
{
    Name = "Everything",
    Command = "npx",
    Arguments = ["-y", "@modelcontextprotocol/server-everything"],
});

// 2. Create and connect client
var mcpClient = await McpClient.CreateAsync(transport);

// 3. Discover available tools
IList<McpClientTool> tools = await mcpClient.ListToolsAsync();
Console.WriteLine("Available tools:");
foreach (var tool in tools)
{
    Console.WriteLine($"  - {tool.Name}: {tool.Description}");
}

// 4. Setup LLM client (example with Azure OpenAI)
IChatClient chatClient = new ChatClientBuilder(
    new AzureOpenAIClient(endpoint, credential)
        .GetChatClient("gpt-4o")
        .AsIChatClient())
    .UseFunctionInvocation()
    .Build();

// 5. Conversational loop
var messages = new List<ChatMessage>();
while (true)
{
    Console.Write("You: ");
    messages.Add(new(ChatRole.User, Console.ReadLine()));
    
    var updates = new List<ChatResponseUpdate>();
    await foreach (var update in chatClient.GetStreamingResponseAsync(
        messages, new() { Tools = [.. tools] }))
    {
        Console.Write(update);
        updates.Add(update);
    }
    Console.WriteLine();
    
    messages.AddMessages(updates);
}
```

### Testing Your Client

Use the MCP Inspector for debugging:

```bash
npx @modelcontextprotocol/inspector
```

Or test with the "everything" server:

```bash
npx -y @modelcontextprotocol/server-everything
```

## SDK Options

| Language | Package | Description |
|----------|---------|-------------|
| C# | `ModelContextProtocol` | Official SDK maintained by Microsoft |
| TypeScript/JavaScript | `@modelcontextprotocol/sdk` | Official TypeScript SDK |
| Python | `mcp` | Official Python SDK |
| Java | Community | Community-maintained |
| Rust | Community | Community-maintained |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Your Application                      │
│                          (MCP Host)                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Your Code                           │ │
│  │  • Conversation loop                                   │ │
│  │  • Security/approval logic                             │ │
│  │  • Tool call routing                                   │ │
│  │  • Context management                                  │ │
│  └───────────────┬────────────────────────────────────────┘ │
│                  │                                           │
│  ┌───────────────▼────────────────────────────────────────┐ │
│  │            Microsoft.Extensions.AI                     │ │
│  │  • IChatClient abstraction                             │ │
│  │  • Function invocation handling                        │ │
│  └───────────────┬────────────────────────────────────────┘ │
│                  │                                           │
│  ┌───────────────▼────────────────────────────────────────┐ │
│  │           MCP C# SDK (McpClient)                       │ │
│  │  ✅ Connection management                              │ │
│  │  ✅ Capability discovery                               │ │
│  │  ✅ Tool execution                                      │ │
│  │  ✅ Protocol handling                                   │ │
│  └───────────────┬────────────────────────────────────────┘ │
└──────────────────┼──────────────────────────────────────────┘
                   │
         ┌─────────┴─────────┐
         │  stdio/HTTP/SSE   │  (Transport Layer)
         └─────────┬─────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
┌────────┐    ┌────────┐    ┌────────┐
│ Server │    │ Server │    │ Server │
│   1    │    │   2    │    │   3    │
└────────┘    └────────┘    └────────┘
```

## Key Takeaways

1. **MCP is the protocol**, not the implementation
2. **Each client connects to ONE server** (1:1 relationship)
3. **The SDK handles protocol details**, you build the orchestration
4. **Tools integrate seamlessly** with Microsoft.Extensions.AI
5. **You control security, routing, and conversation flow**

## Resources

### Official Documentation
- [MCP Official Docs](https://modelcontextprotocol.io/)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [C# SDK GitHub](https://github.com/modelcontextprotocol/csharp-sdk)
- [Microsoft Learn - MCP Quickstart](https://learn.microsoft.com/en-us/dotnet/ai/quickstarts/build-mcp-client)

### Tools & Utilities
- [MCP Inspector](https://www.npmjs.com/package/@modelcontextprotocol/inspector) - Debugging tool
- [Server Everything](https://www.npmjs.com/package/@modelcontextprotocol/server-everything) - Test server
- [Awesome MCP Servers](https://mcpservers.org/) - Community servers directory

