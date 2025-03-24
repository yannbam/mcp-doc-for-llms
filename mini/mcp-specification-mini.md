# MCP Specification (Condensed)

> This is a condensed version of the official Anthropic specification for the Model Context Protocol (MCP) version 2024-11-05.

# Model Context Protocol Specification

**Protocol Revision**: 2024-11-05

## Overview

Model Context Protocol (MCP) is an open protocol enabling seamless integration between LLM applications and external data sources and tools. It standardizes how to connect LLMs with the context they need for application development, AI-powered IDEs, and custom AI workflows.

The protocol uses [JSON-RPC](https://www.jsonrpc.org/) 2.0 messages for communication between:
- **Hosts**: LLM applications initiating connections
- **Clients**: Connectors within the host application
- **Servers**: Services providing context and capabilities

The authoritative specification is defined in the [TypeScript schema](https://github.com/modelcontextprotocol/specification/blob/main/schema/2024-11-05/schema.ts).

## Key Details

### Base Protocol
- JSON-RPC message format
- Stateful connections
- Server and client capability negotiation

### Features
**Servers offer**:
- **Resources**: Context and data for user/AI use
- **Prompts**: Templated messages and workflows
- **Tools**: Functions for AI to execute

**Clients may offer**:
- **Sampling**: Server-initiated recursive LLM interactions
- **Roots**: Information about file system locations

## Security and Trust & Safety

### Key Principles
1. **User Consent and Control**
   - Explicit consent for all data access and operations
   - User control over shared data and actions
   - Clear UI for reviewing and authorizing activities

2. **Data Privacy**
   - Explicit consent before exposing user data
   - Protection with appropriate access controls

3. **Tool Safety**
   - Explicit user consent before tool invocation
   - Clear understanding of tool functions

4. **LLM Sampling Controls**
   - User approval for LLM sampling requests
   - Control over prompts and visible results

### Implementation Guidelines
Implementors **SHOULD**:
- Build robust consent flows
- Document security implications
- Implement appropriate access controls
- Follow security best practices
- Consider privacy implications

## Architecture

MCP follows a client-host-server architecture where each host can run multiple client instances, enabling integration while maintaining security boundaries.

### Core Components
```
Host --> Clients --> Servers --> Resources
```

### Host
- Creates and manages multiple clients
- Controls connection permissions and lifecycle
- Enforces security policies and consent
- Coordinates AI/LLM integration
- Manages context aggregation

### Clients
- Establish one stateful session per server
- Handle protocol negotiation
- Route protocol messages
- Manage subscriptions and notifications
- Maintain security boundaries

### Servers
- Expose resources, tools and prompts
- Operate independently with focused responsibilities
- Request sampling through client interfaces
- Respect security constraints

### Design Principles
1. **Servers should be extremely easy to build**
   - Simple interfaces minimize implementation overhead
   - Clear separation enables maintainable code

2. **Servers should be highly composable**
   - Each server provides focused functionality
   - Multiple servers combine seamlessly
   - Shared protocol enables interoperability

3. **Servers should not be able to read the whole conversation**
   - Servers receive only necessary information
   - Full conversation history stays with host
   - Cross-server interactions controlled by host

4. **Features can be added progressively**
   - Core protocol provides minimal functionality
   - Capabilities negotiated as needed
   - Protocol designed for extensibility
   - Backwards compatibility maintained

### Message Types
- **Requests**: Bidirectional messages expecting response
- **Responses**: Successful results or errors
- **Notifications**: One-way messages requiring no response

### Capability Negotiation
- Servers declare capabilities (resources, tools, prompts)
- Clients declare capabilities (sampling, roots)
- Both respect declared capabilities throughout session

## Base Protocol

All messages **MUST** follow [JSON-RPC 2.0](https://www.jsonrpc.org/specification) with these types:

| Type            | Description                            | Requirements                           |
| --------------- | -------------------------------------- | -------------------------------------- |
| `Requests`      | Messages sent to initiate an operation | Must include unique ID and method name |
| `Responses`     | Messages sent in reply to requests     | Must include same ID as request        |
| `Notifications` | One-way messages with no reply         | Must not include an ID                 |

### Protocol Layers
- **Base Protocol**: Core JSON-RPC message types
- **Lifecycle Management**: Connection initialization, negotiation, session control
- **Server Features**: Resources, prompts, and tools
- **Client Features**: Sampling and root directories
- **Utilities**: Logging, pagination, completion

All implementations **MUST** support base protocol and lifecycle management. Other components optional.

### Auth
Authentication and authorization not currently part of core specification. Future consideration.

### Schema
Full specification defined in [TypeScript schema](https://github.com/modelcontextprotocol/specification/tree/main/schema/2024-11-05/schema.ts).

### Messages

#### Requests
```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  method: string;
  params?: {
    [key: string]: unknown;
  };
}
```
- **MUST** include string or integer ID
- ID **MUST NOT** be `null`
- ID **MUST NOT** have been previously used in same session

#### Responses
```typescript
{
  jsonrpc: "2.0";
  id: string | number;
  result?: {
    [key: string]: unknown;
  }
  error?: {
    code: number;
    message: string;
    data?: unknown;
  }
}
```
- **MUST** include same ID as request
- Either `result` or `error` **MUST** be set, not both

#### Notifications
```typescript
{
  jsonrpc: "2.0";
  method: string;
  params?: {
    [key: string]: unknown;
  };
}
```
- **MUST NOT** include an ID

### Lifecycle

1. **Initialization**
   - Client sends `initialize` with capabilities
   - Server responds with supported capabilities
   - Client sends `initialized` notification

2. **Operation**
   - Normal protocol communication

3. **Shutdown**
   - Graceful termination via transport

#### Initialization Phase
Client must send:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "roots": {
        "listChanged": true
      },
      "sampling": {}
    },
    "clientInfo": {
      "name": "ExampleClient",
      "version": "1.0.0"
    }
  }
}
```

Server must respond:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "protocolVersion": "2024-11-05",
    "capabilities": {
      "logging": {},
      "prompts": {
        "listChanged": true
      },
      "resources": {
        "subscribe": true,
        "listChanged": true
      },
      "tools": {
        "listChanged": true
      }
    },
    "serverInfo": {
      "name": "ExampleServer",
      "version": "1.0.0"
    }
  }
}
```

Client must then send:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

#### Version Negotiation
- Client sends version it supports (should be latest)
- Server responds with same version or another it supports
- Client should disconnect if it doesn't support server's version

#### Capability Negotiation
Key capabilities:
- Client: `roots`, `sampling`, `experimental`
- Server: `prompts`, `resources`, `tools`, `logging`, `experimental`

Sub-capabilities:
- `listChanged`: Support for list change notifications
- `subscribe`: Support for subscribing to item changes

#### Operation Phase
- Exchange messages according to negotiated capabilities
- Respect negotiated protocol version

#### Shutdown Phase
- Terminate connection via transport mechanism
- For stdio: close input stream and wait for exit
- For HTTP: close associated connections

### Transports

MCP defines two standard transport mechanisms:

#### stdio
- Client launches server as subprocess
- Server receives on stdin, writes to stdout
- Messages delimited by newlines with no embedded newlines
- Logging allowed on stderr

#### HTTP with SSE
- Server operates as independent process handling multiple connections
- Server provides SSE endpoint for client to receive messages
- Server provides HTTP POST endpoint for client to send messages
- Client connects to SSE for server-to-client messages
- Client sends HTTP POST for client-to-server messages

#### Custom Transports
- Clients and servers may implement additional transports
- Must preserve JSON-RPC message format and lifecycle requirements

### Versioning
- Format: `YYYY-MM-DD` (date of last breaking change)
- Current: **2024-11-05**
- Version negotiated during initialization
- Only incremented for backwards incompatible changes

### Basic Protocol Utilities

#### Ping
Simple request to verify connection:
```json
{
  "jsonrpc": "2.0",
  "id": "123",
  "method": "ping"
}
```
Response:
```json
{
  "jsonrpc": "2.0",
  "id": "123",
  "result": {}
}
```

#### Cancellation
Notification to cancel in-progress request:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/cancelled",
  "params": {
    "requestId": "123",
    "reason": "User requested cancellation"
  }
}
```

#### Progress
Updates for long-running operations:
- Request includes `progressToken` in metadata
- Server sends progress notifications with token, progress, optional total

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/progress",
  "params": {
    "progressToken": "abc123",
    "progress": 50,
    "total": 100
  }
}
```

## Server Features

### Prompts

Standardized way for servers to expose prompt templates. Prompts are **user-controlled**, typically triggered through UI commands.

#### Capabilities
```json
{
  "capabilities": {
    "prompts": {
      "listChanged": true
    }
  }
}
```

#### Protocol Messages

##### Listing Prompts
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "prompts/list"
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "prompts": [
      {
        "name": "code_review",
        "description": "Analyze code quality",
        "arguments": [
          {
            "name": "code",
            "description": "The code to review",
            "required": true
          }
        ]
      }
    ]
  }
}
```

##### Getting a Prompt
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "prompts/get",
  "params": {
    "name": "code_review",
    "arguments": {
      "code": "def hello():\n    print('world')"
    }
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "description": "Code review prompt",
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "Please review this Python code:\ndef hello():\n    print('world')"
        }
      }
    ]
  }
}
```

##### List Changed Notification
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/prompts/list_changed"
}
```

#### Content Types
- **Text**: `{ "type": "text", "text": "content" }`
- **Image**: `{ "type": "image", "data": "base64...", "mimeType": "image/png" }`
- **Embedded Resources**: `{ "type": "resource", "resource": { "uri": "...", "text": "..." } }`

### Resources

File-like data that servers expose to clients, identified by URIs. Resources are **application-driven**, with hosts determining context inclusion.

#### Capabilities
```json
{
  "capabilities": {
    "resources": {
      "subscribe": true,
      "listChanged": true
    }
  }
}
```

#### Protocol Messages

##### Listing Resources
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "resources/list"
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "resources": [
      {
        "uri": "file:///project/src/main.rs",
        "name": "main.rs",
        "description": "Application entry point",
        "mimeType": "text/x-rust"
      }
    ]
  }
}
```

##### Reading Resources
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "resources/read",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "contents": [
      {
        "uri": "file:///project/src/main.rs",
        "mimeType": "text/x-rust",
        "text": "fn main() {\n    println!(\"Hello world!\");\n}"
      }
    ]
  }
}
```

##### Resource Templates
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "resources/templates/list"
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "resourceTemplates": [
      {
        "uriTemplate": "file:///{path}",
        "name": "Project Files",
        "description": "Access files in the project directory"
      }
    ]
  }
}
```

##### Subscriptions
Subscribe:
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "method": "resources/subscribe",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

Update Notification:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/resources/updated",
  "params": {
    "uri": "file:///project/src/main.rs"
  }
}
```

#### URI Schemes
- `https://`: Web resources (client can fetch directly)
- `file://`: Filesystem-like resources
- `git://`: Git version control integration

### Tools

Functions that can be invoked by language models. Tools are **model-controlled**, allowing AI to discover and invoke automatically (with human approval).

#### Capabilities
```json
{
  "capabilities": {
    "tools": {
      "listChanged": true
    }
  }
}
```

#### Protocol Messages

##### Listing Tools
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/list"
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get weather information",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name or zip code"
            }
          },
          "required": ["location"]
        }
      }
    ]
  }
}
```

##### Calling Tools
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": {
      "location": "New York"
    }
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Current weather in New York:\nTemperature: 72°F\nConditions: Partly cloudy"
      }
    ],
    "isError": false
  }
}
```

##### Tool Error Example
```json
{
  "jsonrpc": "2.0",
  "id": 4,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Failed to fetch weather: API rate limit exceeded"
      }
    ],
    "isError": true
  }
}
```

### Server Utilities

#### Completion
Provides suggestions for prompt arguments and resource URIs:

Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "completion/complete",
  "params": {
    "ref": {
      "type": "ref/prompt",
      "name": "code_review"
    },
    "argument": {
      "name": "language",
      "value": "py"
    }
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "completion": {
      "values": ["python", "pytorch", "pyside"],
      "total": 10,
      "hasMore": true
    }
  }
}
```

#### Logging
Structured log messages from server to client:

Setting Level:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "logging/setLevel",
  "params": {
    "level": "info"
  }
}
```

Log Message:
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/message",
  "params": {
    "level": "error",
    "logger": "database",
    "data": {
      "error": "Connection failed",
      "details": {
        "host": "localhost",
        "port": 5432
      }
    }
  }
}
```

Levels: `debug`, `info`, `notice`, `warning`, `error`, `critical`, `alert`, `emergency`

#### Pagination
Cursor-based pagination for list operations:

Response with cursor:
```json
{
  "jsonrpc": "2.0",
  "id": "123",
  "result": {
    "resources": [...],
    "nextCursor": "eyJwYWdlIjogM30="
  }
}
```

Follow-up request:
```json
{
  "jsonrpc": "2.0",
  "method": "resources/list",
  "params": {
    "cursor": "eyJwYWdlIjogMn0="
  }
}
```

Supported operations:
- `resources/list`
- `resources/templates/list`
- `prompts/list`
- `tools/list`

## Client Features

### Roots

File system paths where servers can operate. Clients expose these to guide server operations.

#### Capabilities
```json
{
  "capabilities": {
    "roots": {
      "listChanged": true
    }
  }
}
```

#### Protocol Messages

##### Listing Roots
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "roots/list"
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "roots": [
      {
        "uri": "file:///home/user/projects/myproject",
        "name": "My Project"
      }
    ]
  }
}
```

##### Root List Changes
```json
{
  "jsonrpc": "2.0",
  "method": "notifications/roots/list_changed"
}
```

### Sampling

Allows servers to request LLM completions via client. This enables agentic behaviors while maintaining security.

#### Capabilities
```json
{
  "capabilities": {
    "sampling": {}
  }
}
```

#### Protocol Messages

##### Creating Messages
Request:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "sampling/createMessage",
  "params": {
    "messages": [
      {
        "role": "user",
        "content": {
          "type": "text",
          "text": "What is the capital of France?"
        }
      }
    ],
    "modelPreferences": {
      "hints": [
        {
          "name": "claude-3-sonnet"
        }
      ],
      "intelligencePriority": 0.8,
      "speedPriority": 0.5
    },
    "systemPrompt": "You are a helpful assistant.",
    "maxTokens": 100
  }
}
```

Response:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "role": "assistant",
    "content": {
      "type": "text",
      "text": "The capital of France is Paris."
    },
    "model": "claude-3-sonnet-20240307",
    "stopReason": "endTurn"
  }
}
```

#### Model Preferences
- `costPriority`: Importance of minimizing costs (0-1)
- `speedPriority`: Importance of low latency (0-1)
- `intelligencePriority`: Importance of advanced capabilities (0-1)
- `hints`: Suggested model names/families

## Revisions

- **2024-11-05** (Current): Latest specification, may receive backwards compatible changes

## Contributing

Contributions welcome via [GitHub](https://github.com/modelcontextprotocol/specification).