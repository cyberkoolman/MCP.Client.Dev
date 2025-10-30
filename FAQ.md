# MCP Client Development FAQ

Frequently asked questions about developing MCP clients.

## General Questions

### Q: What is the Model Context Protocol (MCP)?

**A:** MCP is an open standard that enables seamless integration between AI systems (LLMs) and various data sources and tools. It standardizes how applications provide context to LLMs, similar to how USB-C standardizes device connections.

### Q: Why should I use MCP instead of direct API integrations?

**A:** MCP provides:
- **Standardization**: One protocol for all integrations instead of N×M custom implementations
- **Flexibility**: Switch between LLM providers without changing data integrations
- **Growing ecosystem**: Access to pre-built servers for common services
- **Best practices**: Built-in patterns for security and human approval

### Q: Is MCP only for Anthropic's Claude?

**A:** No! While Anthropic created MCP, it's an open standard. Any LLM or AI application can implement MCP clients to connect to MCP servers.

## Architecture Questions

### Q: What's the difference between a client, server, and host?

**A:**
- **Host**: Your application (e.g., Claude Desktop, your custom AI app)
- **Client**: Component within the host that connects to ONE server
- **Server**: Program that exposes tools, resources, or prompts via MCP

### Q: Can one client connect to multiple servers?

**A:** No. Each MCP client maintains a **1:1 connection** with one server. To connect to multiple servers, you need multiple client instances within your host application.

### Q: What's the difference between stdio and HTTP transports?

**A:**
- **stdio**: Uses standard input/output, good for local servers on the same machine
- **HTTP/SSE**: Uses HTTP with Server-Sent Events, good for remote servers
- **Streamable HTTP**: Newer bidirectional HTTP transport

Choose stdio for local development and HTTP for production/remote scenarios.

## C# SDK Questions

### Q: Does the C# SDK handle everything I need?

**A:** The SDK handles:
- ✅ MCP protocol implementation
- ✅ Connection management
- ✅ Tool/resource discovery
- ✅ Tool execution
- ✅ Integration with Microsoft.Extensions.AI

You must implement:
- ❌ LLM orchestration logic
- ❌ Conversation management
- ❌ Tool approval/security
- ❌ Multi-server routing

### Q: How do MCP tools integrate with LLMs?

**A:** `McpClientTool` inherits from `AIFunction`, so tools are automatically in the right format:

```csharp
IList<McpClientTool> tools = await client.ListToolsAsync();
var response = await chatClient.GetResponseAsync(
    "your prompt",
    new() { Tools = [.. tools] } // Direct usage!
);
```

### Q: Do I need Microsoft.Extensions.AI?

**A:** It's highly recommended but not strictly required. The SDK is designed to work seamlessly with Microsoft.Extensions.AI, making LLM integration much easier. Without it, you'll need to manually convert tool schemas and handle function calling.

### Q: How do I handle tool calls from the LLM?

**A:** You need to:
1. Send tools to the LLM via `ChatOptions.Tools`
2. Parse the LLM response for function calls
3. Extract tool name and arguments
4. Call `CallToolAsync()` on the MCP client
5. Add results back to conversation context

Example:
```csharp
foreach (var update in llmResponse)
{
    if (update is FunctionCallUpdate functionCall)
    {
        var result = await mcpClient.CallToolAsync(
            functionCall.Name,
            functionCall.Arguments,
            ct);
        // Add result to conversation
    }
}
```

## NPM/Package Questions

### Q: What does `@modelcontextprotocol/server-everything` mean?

**A:** It's an NPM scoped package:
- `@modelcontextprotocol` = organization/scope
- `server-everything` = package name

This is standard NPM syntax for namespaced packages.

### Q: What is `npx -y`?

**A:** 
- `npx`: Node Package Runner, executes npm packages
- `-y`: Automatically accepts prompts (auto-installs if not present)

Example: `npx -y @modelcontextprotocol/server-everything`

### Q: Do I need Node.js for C# MCP clients?

**A:** Only if you're connecting to Node.js-based MCP servers. Many reference servers are written in Node.js (like `server-everything`), but you can also connect to servers written in any language.

## Development Questions

### Q: How do I test my MCP client?

**A:** 
1. Use `@modelcontextprotocol/server-everything` for comprehensive testing
2. Use `@modelcontextprotocol/inspector` for visual debugging
3. Write unit tests that mock `IMcpClient`
4. Test with real servers gradually

### Q: How do I debug MCP communication?

**A:**
```csharp
// Enable logging
var factory = LoggerFactory.Create(builder => 
    builder.AddConsole().SetMinimumLevel(LogLevel.Debug));

var client = await McpClientFactory.CreateAsync(
    config, 
    options, 
    loggerFactory: factory);
```

Or use the MCP Inspector:
```bash
npx @modelcontextprotocol/inspector
```

### Q: How do I handle errors?

**A:**
```csharp
try
{
    var result = await client.CallToolAsync(name, args, ct);
}
catch (McpException ex)
{
    // MCP-specific error
    Console.WriteLine($"MCP Error: {ex.ErrorCode}");
}
catch (Exception ex)
{
    // General error
    Console.WriteLine($"Error: {ex.Message}");
}
```

### Q: How should I implement security/approval?

**A:** Implement a wrapper:
```csharp
async Task<CallToolResult?> CallWithApprovalAsync(
    string toolName, 
    object args)
{
    // Show user what will be executed
    Console.WriteLine($"Tool: {toolName}");
    Console.WriteLine($"Args: {JsonSerializer.Serialize(args)}");
    Console.Write("Approve? (y/n): ");
    
    if (Console.ReadLine() == "y")
    {
        return await client.CallToolAsync(toolName, args, ct);
    }
    return null;
}
```

## Protocol Questions

### Q: What are the three core primitives?

**A:**
1. **Tools**: Actions the AI can perform (model-controlled)
2. **Resources**: Data the AI can read (app-controlled)
3. **Prompts**: Templates for interactions (user-controlled)

### Q: What's the difference between tools and resources?

**A:**
- **Tools**: Active operations with potential side effects (like calling an API, writing a file)
- **Resources**: Read-only data access with no side effects (like reading a file, querying a database)

### Q: What is "sampling" in MCP?

**A:** Sampling allows an MCP server to request the client to prompt the LLM on its behalf. This enables servers to use AI capabilities while maintaining the client's control over LLM access.

### Q: What's the initialization handshake?

**A:** When a client connects to a server:
1. Client sends `initialize` request with capabilities
2. Server responds with its capabilities
3. Client confirms with `initialized` notification
4. Communication proceeds normally

The SDK handles this automatically.

## Best Practices Questions

### Q: Should I create one client or multiple?

**A:** It depends:
- **One server**: One client
- **Multiple servers**: Multiple clients (one per server)
- **Same server, different instances**: Multiple clients if needed

### Q: How do I manage conversation context?

**A:**
```csharp
// Maintain conversation history
List<ChatMessage> history = [];

// Add user message
history.Add(new(ChatRole.User, input));

// Get response
var response = await chatClient.GetResponseAsync(
    history, 
    new() { Tools = tools });

// Add assistant response
history.Add(response.Message);

// Continue conversation...
```

### Q: Should I implement caching?

**A:** Consider caching:
- Tool/resource lists (refresh periodically)
- Resource contents (if they don't change frequently)
- LLM responses (if appropriate)

But don't cache:
- Tool execution results (may have side effects)
- Real-time data

### Q: How do I handle multiple concurrent tool calls?

**A:** If your LLM requests multiple tool calls in parallel:
```csharp
var tasks = toolCalls.Select(async call => 
    await client.CallToolAsync(call.Name, call.Args, ct));
    
var results = await Task.WhenAll(tasks);
```

## Troubleshooting

### Q: My client can't connect to the server

**Checklist:**
1. Is the server process running?
2. Is the transport type correct (stdio vs HTTP)?
3. Check logs with `LogLevel.Debug`
4. Try with MCP Inspector first
5. Verify command/arguments are correct

### Q: Tools aren't being called by the LLM

**Checklist:**
1. Are tools being passed in `ChatOptions.Tools`?
2. Is `.UseFunctionInvocation()` called on the builder?
3. Are tool descriptions clear and specific?
4. Does the LLM support function calling?
5. Try with a simpler prompt

### Q: I'm getting "Unknown tool" errors

**Causes:**
- Tool name mismatch between LLM and MCP
- Server doesn't expose that tool
- Refresh tool list: `await client.ListToolsAsync()`

## Integration Questions

### Q: Can I use MCP with Azure OpenAI?

**A:** Yes! The C# SDK integrates seamlessly:
```csharp
IChatClient client = new ChatClientBuilder(
    new AzureOpenAIClient(endpoint, credential)
        .GetChatClient("gpt-4o")
        .AsIChatClient())
    .UseFunctionInvocation()
    .Build();
```

### Q: Can I use MCP with other LLM providers?

**A:** Yes, as long as they:
1. Support function/tool calling
2. Have a Microsoft.Extensions.AI adapter

Currently supported: Azure OpenAI, OpenAI, Ollama, and others.

### Q: Can I create my own MCP server?

**A:** Absolutely! The C# SDK supports both clients and servers:
```csharp
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();
```

## Resources

### Q: Where can I find pre-built MCP servers?

**A:** 
- [Awesome MCP Servers](https://mcpservers.org/)
- [Official GitHub Org](https://github.com/modelcontextprotocol)
- Search npm for `@modelcontextprotocol/server-*`

### Q: Where can I get help?

**A:**
- [GitHub Discussions](https://github.com/orgs/modelcontextprotocol/discussions)
- [C# SDK Issues](https://github.com/modelcontextprotocol/csharp-sdk/issues)
- [Microsoft Learn Docs](https://learn.microsoft.com/en-us/dotnet/ai/)

---

**Have a question not answered here?** Open an issue on GitHub!
