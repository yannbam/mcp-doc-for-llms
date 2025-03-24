# MCP Documentation (Condensed)

> This is a condensed version of the official Anthropic documentation for the Model Context Protocol (MCP) version 2024-11-05.

# Model Context Protocol (MCP) Documentation

## Introduction

MCP is an open protocol standardizing how applications provide context to LLMs, acting like a USB-C port for AI applications, connecting AI models to different data sources and tools.

### Why MCP?
- Build agents and workflows on top of LLMs
- Access pre-built integrations
- Switch between LLM providers flexibly
- Secure data within your infrastructure

### Architecture
```
Host (Claude, IDEs, Tools) <--> MCP Clients <--> MCP Servers <--> Data Sources (local/remote)
```
- **MCP Hosts**: Programs wanting to access data through MCP
- **MCP Clients**: Maintain 1:1 connections with servers
- **MCP Servers**: Expose specific capabilities through protocol
- **Data Sources**: Files, databases, services, APIs

## Core Concepts

### Architecture
- Client-server architecture allowing hosts to connect to multiple servers
- Clients maintain 1:1 connections with servers inside the host application
- Servers provide context, tools, and prompts to clients

### Protocol Layer
- Handles message framing, request/response linking, and communication patterns
- Key classes: `Protocol`, `Client`, `Server`

### Transport Layer
- **Stdio transport**: Standard input/output for local processes
- **HTTP with SSE transport**: Server-Sent Events for server-to-client, HTTP POST for client-to-server
- All transports use JSON-RPC 2.0

### Message Types
1. **Requests**: Expect responses
   ```typescript
   interface Request {
     method: string;
     params?: { ... };
   }
   ```
2. **Results**: Successful responses
   ```typescript
   interface Result {
     [key: string]: unknown;
   }
   ```
3. **Errors**: Request failures
   ```typescript
   interface Error {
     code: number;
     message: string;
     data?: unknown;
   }
   ```
4. **Notifications**: One-way messages
   ```typescript
   interface Notification {
     method: string;
     params?: { ... };
   }
   ```

### Connection Lifecycle
1. **Initialization**:
   - Client sends `initialize` request with version and capabilities
   - Server responds with supported capabilities
   - Client sends `initialized` notification
2. **Message Exchange**: Requests, responses, notifications
3. **Termination**: Clean shutdown via transport disconnection

### Error Handling
Standard error codes:
```typescript
enum ErrorCode {
  ParseError = -32700,
  InvalidRequest = -32600,
  MethodNotFound = -32601,
  InvalidParams = -32602,
  InternalError = -32603
}
```

## Key MCP Capabilities

### Resources
- File-like data read by clients
- Application-controlled (client decides usage)
- Identified by URIs in format: `[protocol]://[host]/[path]`
- Content types: Text (UTF-8) or Binary (base64)

#### Resource Discovery:
- Direct resources via `resources/list` endpoint
- Resource templates (URI templates) for dynamic resources

#### Reading Resources:
- `resources/read` request with resource URI
- Server responds with resource contents (text or blob)

#### Resource Updates:
- List changes via `notifications/resources/list_changed`
- Content changes via subscription:
  - `resources/subscribe` → resource URI
  - `notifications/resources/updated` → content changed
  - `resources/read` → fetch updated content
  - `resources/unsubscribe` → stop notifications

### Tools
- Functions exposed to LLMs (model-controlled)
- Support automatic invocation with human approval
- Defined with name, description, and input schema

#### Tool Definition:
```typescript
{
  name: string;
  description?: string;
  inputSchema: {
    type: "object",
    properties: { ... }
  }
}
```

#### Tool Patterns:
- System operations (shell commands, system functions)
- API integrations (external services, web APIs)
- Data processing (transformations, analysis)

#### Error Handling:
- Tool errors reported in result object (`isError: true`)
- Protocol errors for method/invocation issues

### Prompts
- Reusable templates and workflows
- User-controlled (explicitly selected by users)
- Can accept dynamic arguments and include context

#### Prompt Structure:
```typescript
{
  name: string;
  description?: string;
  arguments?: [
    {
      name: string;
      description?: string;
      required?: boolean;
    }
  ]
}
```

#### Usage:
- Discover via `prompts/list`
- Retrieve via `prompts/get` with arguments
- Can include embedded resources and context
- Support for updates via `notifications/prompts/list_changed`

### Sampling
- Servers request LLM completions via client
- Human-in-the-loop design for security and control
- Standardized message format with model preferences

#### Message Format:
```typescript
{
  messages: [...],
  modelPreferences?: {
    hints?: [{name?: string}],
    costPriority?: number,
    speedPriority?: number,
    intelligencePriority?: number
  },
  systemPrompt?: string,
  includeContext?: "none" | "thisServer" | "allServers",
  temperature?: number,
  maxTokens: number,
  stopSequences?: string[],
  metadata?: Record<string, unknown>
}
```

#### Request Parameters:
- Messages: Conversation history with role and content
- Model preferences: Hints and priorities for model selection
- System prompt: Optional instruction context
- Context inclusion: What MCP context to include
- Sampling parameters: Control generation behavior

### Roots
- Define boundaries where servers operate
- Inform servers about relevant resources and locations
- Primarily used for filesystem paths (can be any URI)

#### Usage:
- Clients declare `roots` capability
- Provide list of suggested roots to server
- Notify when roots change
- Guide servers on resource access boundaries

## SDKs and Implementation

### Available SDKs
1. **Python SDK**: [modelcontextprotocol/python-sdk](https://github.com/modelcontextprotocol/python-sdk)
   - FastMCP API for simplified server development
2. **TypeScript SDK**: [modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk)
   - Express-like API with TypeScript safety
3. **Java SDK**: [modelcontextprotocol/java-sdk](https://github.com/modelcontextprotocol/java-sdk)
   - Spring AI integration
4. **Kotlin SDK**: [modelcontextprotocol/kotlin-sdk](https://github.com/modelcontextprotocol/kotlin-sdk)
   - Idiomatic Kotlin interface

### Implementation Example - Server

**Python:**
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("example-server")

@mcp.tool()
def calculate_sum(a: int, b: int) -> int:
    """Add two numbers and return the result.
    Args:
        a: First number
        b: Second number
    """
    return a + b

@mcp.tool()
def get_weather(location: str) -> str:
    """Get the weather for a location.
    Args:
        location: City name or zip code
    """
    return f"The weather in {location} is sunny and 72°F"

if __name__ == "__main__":
    mcp.run(transport='stdio')
```

**TypeScript:**
```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { z } from "zod";

const server = new McpServer({
  name: "example-server",
  version: "1.0.0",
});

server.tool(
  "calculate_sum",
  "Add two numbers together",
  {
    a: z.number().describe("First number"),
    b: z.number().describe("Second number"),
  },
  async ({ a, b }) => {
    return {
      content: [
        {
          type: "text",
          text: String(a + b),
        },
      ],
    };
  },
);

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Server running on stdio");
}

main().catch((error) => {
  console.error("Fatal error:", error);
  process.exit(1);
});
```

### Implementation Example - Client

**Python:**
```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["path/to/server.py"],
    )
    
    async with stdio_client(server_params) as streams:
        async with ClientSession(streams[0], streams[1]) as session:
            await session.initialize()
            
            tools_result = await session.list_tools()
            print(f"Available tools: {[tool.name for tool in tools_result.tools]}")
            
            result = await session.call_tool("calculate_sum", {"a": 5, "b": 7})
            print(f"5 + 7 = {result.content[0].text}")

if __name__ == "__main__":
    asyncio.run(main())
```

**TypeScript:**
```typescript
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

async function main() {
  const client = new Client({
    name: "example-client",
    version: "1.0.0",
  });
  
  const transport = new StdioClientTransport({
    command: "node",
    args: ["path/to/server.js"],
  });
  
  await client.connect(transport);
  
  try {
    const toolsResult = await client.listTools();
    console.log("Available tools:", toolsResult.tools.map(t => t.name));
    
    const result = await client.callTool({
      name: "calculate_sum",
      arguments: { a: 5, b: 7 },
    });
    
    console.log("5 + 7 =", result.content[0].text);
  } finally {
    await client.close();
  }
}

main().catch(console.error);
```

## Debugging and Tools

### MCP Inspector
Interactive tool for testing and debugging MCP servers:
```bash
npx @modelcontextprotocol/inspector <command>
```

#### Inspecting NPM/PyPI Servers:
```bash
# NPM package
npx -y @modelcontextprotocol/inspector npx <package-name> <args>

# PyPI package
npx @modelcontextprotocol/inspector uvx <package-name> <args>
```

#### Inspecting Local Servers:
```bash
# TypeScript
npx @modelcontextprotocol/inspector node path/to/server/index.js args...

# Python
npx @modelcontextprotocol/inspector \
  uv \
  --directory path/to/server \
  run \
  package-name \
  args...
```

#### Features:
- Server connection pane
- Resources tab for content inspection
- Prompts tab for testing templates
- Tools tab for execution testing
- Notifications pane for server logs

### Debugging in Claude Desktop

#### Checking Server Status
- Click MCP plug icon for connected servers, available prompts/resources
- Click MCP hammer icon for available tools

#### Viewing Logs
```bash
tail -n 20 -F ~/Library/Logs/Claude/mcp*.log
```

#### Common Issues:
- **Working Directory**: Use absolute paths in configuration
- **Environment Variables**: Configure in `claude_desktop_config.json`
- **Server Initialization**: Check paths, permissions, configuration
- **Connection Problems**: Verify protocol compatibility

## Example Servers and Implementations

### Reference Implementations
- **Data Systems**: Filesystem, PostgreSQL, SQLite, Google Drive
- **Development Tools**: Git, GitHub, GitLab, Sentry
- **Web Automation**: Brave Search, Fetch, Puppeteer
- **Productivity**: Slack, Google Maps, Memory (knowledge graph)
- **AI Tools**: EverArt, Sequential Thinking, AWS KB Retrieval

### Official Integrations
Companies with MCP servers:
- Axiom, Browserbase, Cloudflare, E2B, Neon, Obsidian, Qdrant
- Raygun, Search1API, Stripe, Tinybird, Weaviate

### Community Highlights
- Docker, Kubernetes, Linear, Snowflake, Spotify, Todoist

## Building MCP with LLMs

### Preparation
1. Gather documentation
2. Review SDK repositories
3. Describe server requirements clearly

### Implementation Steps
1. Start with core functionality
2. Iterate to add features
3. Test thoroughly
4. Document clearly
5. Follow protocol specifications

## Example Clients

### Client Feature Support Matrix
- Claude Desktop: Resources ✅, Prompts ✅, Tools ✅, Sampling ❌, Roots ❌
- Continue: Resources ✅, Prompts ✅, Tools ✅, Sampling ❌, Roots ❌
- Cursor: Resources ❌, Prompts ❌, Tools ✅, Sampling ❌, Roots ❌
- Zed: Resources ❌, Prompts ✅, Tools ❌, Sampling ❌, Roots ❌

Note: Claude.ai web app doesn't support MCP, only desktop version does.

## Roadmap and Future Direction
Priorities for 2025:
1. **Remote MCP**: Authentication, authorization, service discovery
2. **Reference Implementations**: Client examples, protocol drafting
3. **Distribution & Discovery**: Package management, installation tools
4. **Agent Support**: Hierarchical systems, interactive workflows
5. **Ecosystem Growth**: Community-led standards, additional modalities

## Resources
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [MCP Website](https://modelcontextprotocol.io/)
- [GitHub Organization](https://github.com/modelcontextprotocol)
- SDK repos: [Python](https://github.com/modelcontextprotocol/python-sdk), [TypeScript](https://github.com/modelcontextprotocol/typescript-sdk), [Java](https://github.com/modelcontextprotocol/java-sdk), [Kotlin](https://github.com/modelcontextprotocol/kotlin-sdk)
- [MCP Servers Repository](https://github.com/modelcontextprotocol/servers)