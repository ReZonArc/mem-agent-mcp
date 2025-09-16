# MCP Server Architecture

This document details the Model Context Protocol (MCP) server implementation, transport layers, and client integration patterns.

## MCP Server Overview

```mermaid
graph TB
    subgraph "MCP Server Core"
        FASTMCP[FastMCP Framework<br/>Protocol Implementation]
        SERVER[MCP Server<br/>server.py]
        TOOLS[MCP Tools<br/>Tool Registry]
    end
    
    subgraph "Transport Layer"
        STDIO[STDIO Transport<br/>Standard I/O]
        HTTP[HTTP Transport<br/>Web Integration]
        SSE[SSE Transport<br/>Server-Sent Events]
    end
    
    subgraph "Client Integrations"
        CLAUDE_DESKTOP[Claude Desktop<br/>STDIO]
        CLAUDE_CODE[Claude Code<br/>STDIO]
        CHATGPT[ChatGPT<br/>HTTP]
        LM_STUDIO[LM Studio<br/>STDIO]
    end
    
    subgraph "Backend Services"
        AGENT[Memory Agent<br/>Core Processing]
        MEMORY[Memory System<br/>File Operations]
        CONFIG[Configuration<br/>Settings Management]
    end
    
    SERVER --> FASTMCP
    SERVER --> TOOLS
    SERVER --> AGENT
    
    FASTMCP --> STDIO
    FASTMCP --> HTTP
    FASTMCP --> SSE
    
    STDIO --> CLAUDE_DESKTOP
    STDIO --> CLAUDE_CODE
    STDIO --> LM_STUDIO
    HTTP --> CHATGPT
    
    AGENT --> MEMORY
    AGENT --> CONFIG
    
    %% Styling
    classDef core fill:#e8f5e8
    classDef transport fill:#f3e5f5
    classDef client fill:#e1f5fe
    classDef backend fill:#fff3e0
    
    class FASTMCP,SERVER,TOOLS core
    class STDIO,HTTP,SSE transport
    class CLAUDE_DESKTOP,CLAUDE_CODE,CHATGPT,LM_STUDIO client
    class AGENT,MEMORY,CONFIG backend
```

## MCP Protocol Implementation

### FastMCP Framework Integration
```mermaid
classDiagram
    class FastMCP {
        +str name
        +dict tools
        +register_tool(func, schema)
        +run_stdio()
        +run_http(port)
        +handle_request(request)
    }
    
    class MCPServer {
        +FastMCP mcp
        +Agent agent_instance
        +str memory_path
        +str model_name
        +initialize_server()
        +register_tools()
        +start_server()
    }
    
    class MCPTool {
        +str name
        +str description
        +dict parameters
        +function handler
        +execute(args)
    }
    
    FastMCP --> MCPServer
    MCPServer --> MCPTool
```

### Tool Registration System
```mermaid
sequenceDiagram
    participant Server
    participant FastMCP
    participant ToolRegistry
    participant Agent
    
    Note over Server,Agent: Server Initialization
    
    Server->>FastMCP: Initialize MCP instance
    Server->>ToolRegistry: Register read_memory_files
    Server->>ToolRegistry: Register search_files
    Server->>ToolRegistry: Register get_available_tools
    
    ToolRegistry->>FastMCP: Register tool schemas
    FastMCP->>FastMCP: Validate tool definitions
    
    Server->>Agent: Initialize agent instance
    Server->>FastMCP: Start transport layer
    
    Note over Server,Agent: Tool Execution Flow
    
    FastMCP->>Server: Tool execution request
    Server->>Agent: Forward request to agent
    Agent-->>Server: Agent response
    Server-->>FastMCP: Return tool result
```

## Transport Layer Architecture

### STDIO Transport (Primary)
```mermaid
sequenceDiagram
    participant Client
    participant MCPServer
    participant Agent
    participant Memory
    
    Note over Client,Memory: STDIO MCP Communication
    
    Client->>MCPServer: JSON-RPC over stdin
    MCPServer->>MCPServer: Parse MCP request
    MCPServer->>Agent: Execute tool
    Agent->>Memory: Memory operations
    Memory-->>Agent: Memory content
    Agent-->>MCPServer: Tool result
    MCPServer->>MCPServer: Format MCP response
    MCPServer-->>Client: JSON-RPC over stdout
```

**STDIO Transport Features:**
- JSON-RPC 2.0 protocol compliance
- Bidirectional communication over stdin/stdout
- Native support in Claude Desktop, Claude Code, LM Studio
- Low latency, direct process communication
- Automatic process lifecycle management

### HTTP Transport (ChatGPT Integration)
```mermaid
graph TB
    subgraph "HTTP MCP Server"
        HTTP_SERVER[HTTP Server<br/>mcp_http_server.py]
        MCP_ENDPOINT[/mcp Endpoint<br/>MCP Protocol Handler]
        CORS[CORS Handler<br/>Cross-Origin Support]
    end
    
    subgraph "Request Processing"
        REQ_PARSER[Request Parser<br/>HTTP to MCP]
        MCP_PROCESSOR[MCP Processor<br/>Tool Execution]
        RESP_FORMATTER[Response Formatter<br/>MCP to HTTP]
    end
    
    subgraph "External Access"
        NGROK[ngrok Tunnel<br/>Public Access]
        CHATGPT_CLIENT[ChatGPT Client<br/>Developer Mode]
    end
    
    HTTP_SERVER --> MCP_ENDPOINT
    HTTP_SERVER --> CORS
    MCP_ENDPOINT --> REQ_PARSER
    REQ_PARSER --> MCP_PROCESSOR
    MCP_PROCESSOR --> RESP_FORMATTER
    
    NGROK --> HTTP_SERVER
    CHATGPT_CLIENT --> NGROK
```

**HTTP Transport Configuration:**
```mermaid
sequenceDiagram
    participant Developer
    participant MCPHTTPServer
    participant Ngrok
    participant ChatGPT
    
    Developer->>MCPHTTPServer: make serve-mcp-http
    MCPHTTPServer->>MCPHTTPServer: Start on localhost:8081
    
    Developer->>Ngrok: ngrok http 8081
    Ngrok-->>Developer: Public HTTPS URL
    
    Developer->>ChatGPT: Configure MCP Server
    Note right of ChatGPT: URL: https://abc123.ngrok.io/mcp<br/>Protocol: HTTP<br/>Auth: None
    
    ChatGPT->>Ngrok: MCP Requests
    Ngrok->>MCPHTTPServer: Forward requests
    MCPHTTPServer-->>Ngrok: MCP Responses
    Ngrok-->>ChatGPT: Forward responses
```

## MCP Tool Implementations

### Tool Schema Definition
```mermaid
classDiagram
    class MCPToolSchema {
        +str name
        +str description
        +dict parameters
        +list required
        +validate_input(args)
    }
    
    class ReadMemoryTool {
        +name: "read_memory_files"
        +description: "Read memory markdown files"
        +parameters: {files: array}
        +handler: read_memory_handler
    }
    
    class SearchFilesTool {
        +name: "search_files"
        +description: "Search memory files"
        +parameters: {query: str, max_results: int}
        +handler: search_files_handler
    }
    
    class AvailableToolsTool {
        +name: "get_available_tools" 
        +description: "List available tools"
        +parameters: {}
        +handler: available_tools_handler
    }
    
    MCPToolSchema <|-- ReadMemoryTool
    MCPToolSchema <|-- SearchFilesTool  
    MCPToolSchema <|-- AvailableToolsTool
```

### Tool Execution Pipeline
```mermaid
flowchart TD
    subgraph "Request Processing"
        MCP_REQ[MCP Tool Request]
        VALIDATE[Validate Parameters]
        ROUTE[Route to Handler]
    end
    
    subgraph "Tool Execution"
        HANDLER[Tool Handler]
        AGENT_CALL[Agent Method Call]
        AGENT_EXEC[Agent Execution]
    end
    
    subgraph "Response Processing"
        RESULT[Tool Result]
        FORMAT[Format Response]
        MCP_RESP[MCP Response]
    end
    
    subgraph "Error Handling"
        ERROR[Error Occurred]
        LOG_ERROR[Log Error]
        ERROR_RESP[Error Response]
    end
    
    MCP_REQ --> VALIDATE
    VALIDATE --> ROUTE
    ROUTE --> HANDLER
    
    HANDLER --> AGENT_CALL
    AGENT_CALL --> AGENT_EXEC
    AGENT_EXEC --> RESULT
    
    RESULT --> FORMAT
    FORMAT --> MCP_RESP
    
    VALIDATE -.-> ERROR
    AGENT_EXEC -.-> ERROR
    ERROR --> LOG_ERROR
    LOG_ERROR --> ERROR_RESP
```

### Read Memory Files Tool
```mermaid
sequenceDiagram
    participant MCP
    participant ReadTool
    participant Agent
    participant FileSystem
    
    MCP->>ReadTool: read_memory_files([file1, file2])
    ReadTool->>ReadTool: Validate file parameters
    ReadTool->>Agent: Initialize with memory query
    Agent->>Agent: Process file read request
    Agent->>FileSystem: Read specified files
    FileSystem-->>Agent: File contents
    Agent->>Agent: Apply privacy filters
    Agent-->>ReadTool: Formatted response
    ReadTool-->>MCP: Tool execution result
```

### Search Files Tool
```mermaid
sequenceDiagram
    participant MCP
    participant SearchTool  
    participant Agent
    participant SearchEngine
    participant FilterSystem
    
    MCP->>SearchTool: search_files(query, max_results)
    SearchTool->>SearchTool: Validate search parameters
    SearchTool->>Agent: Initialize with search query
    Agent->>SearchEngine: Execute search
    SearchEngine->>FilterSystem: Apply privacy filters
    FilterSystem-->>SearchEngine: Filtered results
    SearchEngine-->>Agent: Search results
    Agent->>Agent: Format response
    Agent-->>SearchTool: Formatted response
    SearchTool-->>MCP: Tool execution result
```

## Configuration Management

### Server Configuration
```mermaid
graph TB
    subgraph "Configuration Sources"
        ENV_VARS[Environment Variables<br/>MEMORY_DIR, MCP_TRANSPORT]
        CONFIG_FILES[Configuration Files<br/>.memory_path, .filters]
        CLI_ARGS[CLI Arguments<br/>--memory-path, --model]
    end
    
    subgraph "Configuration Processing"
        LOADER[Config Loader<br/>settings.py]
        VALIDATOR[Config Validator<br/>Path validation]
        DEFAULTS[Default Values<br/>Fallback settings]
    end
    
    subgraph "Runtime Configuration"
        MCP_CONFIG[MCP Server Config<br/>Transport, Tools]
        AGENT_CONFIG[Agent Config<br/>Model, Memory Path]
        LOGGING_CONFIG[Logging Config<br/>Level, Output]
    end
    
    ENV_VARS --> LOADER
    CONFIG_FILES --> LOADER
    CLI_ARGS --> LOADER
    
    LOADER --> VALIDATOR
    VALIDATOR --> DEFAULTS
    
    DEFAULTS --> MCP_CONFIG
    DEFAULTS --> AGENT_CONFIG
    DEFAULTS --> LOGGING_CONFIG
```

### Memory Path Resolution
```mermaid
flowchart TD
    START[Server Start]
    
    CHECK_ENV{MEMORY_DIR<br/>env var set?}
    READ_FILE{.memory_path<br/>file exists?}
    VALIDATE_PATH{Path exists<br/>and valid?}
    
    ENV_PATH[Use Environment Path]
    FILE_PATH[Use File Path]
    DEFAULT_PATH[Use Default Path<br/>memory/mcp-server]
    
    EXPAND[Expand Variables<br/>~, $HOME, etc.]
    ABSOLUTE[Convert to Absolute Path]
    CREATE_DIR[Create Directory if Missing]
    
    FINAL[Final Memory Path]
    
    START --> CHECK_ENV
    CHECK_ENV -->|Yes| ENV_PATH
    CHECK_ENV -->|No| READ_FILE
    READ_FILE -->|Yes| FILE_PATH
    READ_FILE -->|No| DEFAULT_PATH
    
    ENV_PATH --> VALIDATE_PATH
    FILE_PATH --> VALIDATE_PATH
    DEFAULT_PATH --> VALIDATE_PATH
    
    VALIDATE_PATH -->|Valid| EXPAND
    VALIDATE_PATH -->|Invalid| DEFAULT_PATH
    
    EXPAND --> ABSOLUTE
    ABSOLUTE --> CREATE_DIR
    CREATE_DIR --> FINAL
```

## Client Integration Patterns

### Claude Desktop Integration
```mermaid
graph LR
    subgraph "Claude Desktop"
        CD_APP[Claude Desktop App]
        MCP_CLIENT[MCP Client<br/>Built-in]
        CONFIG[claude_desktop.json]
    end
    
    subgraph "Configuration"
        MCP_JSON[mcp.json<br/>Generated Config]
        SERVER_CMD[Server Command<br/>bash + uv run]
        ENV_VARS[Environment Variables<br/>LOG_LEVEL, TIMEOUT]
    end
    
    subgraph "MCP Server"
        STDIO_SERVER[MCP Server<br/>STDIO Mode]
        AGENT_BACKEND[Agent Backend]
    end
    
    CD_APP --> MCP_CLIENT
    MCP_CLIENT --> CONFIG
    CONFIG --> MCP_JSON
    MCP_JSON --> SERVER_CMD
    SERVER_CMD --> STDIO_SERVER
    STDIO_SERVER --> AGENT_BACKEND
```

**Claude Desktop Configuration Example:**
```json
{
  "mcpServers": {
    "memory-agent-stdio": {
      "command": "bash",
      "args": ["-lc", "cd /path/to/repo && uv run python mcp_server/server.py"],
      "env": {
        "FASTMCP_LOG_LEVEL": "INFO",
        "MCP_TRANSPORT": "stdio"
      },
      "timeout": 600000
    }
  }
}
```

### ChatGPT Integration Pattern
```mermaid
sequenceDiagram
    participant User
    participant ChatGPT
    participant Ngrok
    participant MCPHTTPServer
    participant Agent
    
    Note over User,Agent: ChatGPT Integration Setup
    
    User->>MCPHTTPServer: Start HTTP server
    User->>Ngrok: Expose local server
    User->>ChatGPT: Configure MCP connector
    
    Note over User,Agent: Query Processing
    
    User->>ChatGPT: "Search my memory for X"
    ChatGPT->>Ngrok: HTTP MCP request
    Ngrok->>MCPHTTPServer: Forward request
    MCPHTTPServer->>Agent: Execute search
    Agent-->>MCPHTTPServer: Search results
    MCPHTTPServer-->>Ngrok: MCP response
    Ngrok-->>ChatGPT: Forward response
    ChatGPT-->>User: Formatted answer
```

## Error Handling and Logging

### Error Handling Strategy
```mermaid
graph TB
    subgraph "Error Types"
        CONFIG_ERR[Configuration Errors<br/>Invalid paths, missing files]
        TOOL_ERR[Tool Execution Errors<br/>Agent failures, timeouts]
        TRANSPORT_ERR[Transport Errors<br/>Protocol, connection issues]
        SYSTEM_ERR[System Errors<br/>Memory, permissions]
    end
    
    subgraph "Error Handling"
        CATCH[Error Catching<br/>Try-catch blocks]
        LOG[Error Logging<br/>Structured logging]
        RECOVER[Error Recovery<br/>Fallback strategies]
    end
    
    subgraph "Error Response"
        MCP_ERROR[MCP Error Response<br/>Structured error format]
        USER_MSG[User-friendly Message<br/>Actionable information]
        DEBUG_INFO[Debug Information<br/>For troubleshooting]
    end
    
    CONFIG_ERR --> CATCH
    TOOL_ERR --> CATCH
    TRANSPORT_ERR --> CATCH
    SYSTEM_ERR --> CATCH
    
    CATCH --> LOG
    LOG --> RECOVER
    RECOVER --> MCP_ERROR
    
    MCP_ERROR --> USER_MSG
    MCP_ERROR --> DEBUG_INFO
```

### Logging Architecture
```mermaid
graph TD
    subgraph "Logging Sources"
        MCP_LOGS[MCP Server Logs<br/>Protocol events]
        AGENT_LOGS[Agent Logs<br/>Tool execution]
        TOOL_LOGS[Tool Logs<br/>Memory operations]
        HTTP_LOGS[HTTP Logs<br/>Request/response]
    end
    
    subgraph "Log Processing"
        FORMATTER[Log Formatter<br/>Structured format]
        FILTER[Log Filter<br/>Level-based filtering]
        HANDLER[Log Handler<br/>Output routing]
    end
    
    subgraph "Log Outputs"
        STDOUT[Standard Output<br/>Console logging]
        FILES[Log Files<br/>Persistent storage]
        DEBUG[Debug Output<br/>Troubleshooting]
    end
    
    MCP_LOGS --> FORMATTER
    AGENT_LOGS --> FORMATTER
    TOOL_LOGS --> FORMATTER
    HTTP_LOGS --> FORMATTER
    
    FORMATTER --> FILTER
    FILTER --> HANDLER
    
    HANDLER --> STDOUT
    HANDLER --> FILES
    HANDLER --> DEBUG
```

## Performance and Scalability

### Request Processing Performance
```mermaid
graph LR
    subgraph "Performance Characteristics"
        LATENCY[Request Latency<br/>~100-500ms typical]
        THROUGHPUT[Throughput<br/>Sequential processing]
        MEMORY[Memory Usage<br/>Conversation + cache]
        CPU[CPU Usage<br/>Model inference bound]
    end
    
    subgraph "Optimization Strategies"
        CACHE[Response Caching<br/>Common queries]
        PARALLEL[Parallel Tools<br/>Independent operations]
        LAZY[Lazy Loading<br/>On-demand files]
        LIMITS[Request Limits<br/>Timeout, size limits]
    end
    
    LATENCY --> CACHE
    THROUGHPUT --> PARALLEL
    MEMORY --> LAZY
    CPU --> LIMITS
```

### Scaling Considerations
- **Single Process**: Current design is single-process, single-threaded
- **Memory Limits**: Bounded by model context window and conversation history
- **Concurrent Clients**: Multiple clients can connect to same server instance
- **File System**: Performance scales with memory directory size and structure

## Security Considerations

### Security Model
```mermaid
graph TB
    subgraph "Attack Surface"
        MCP_PROTOCOL[MCP Protocol<br/>JSON-RPC inputs]
        TOOL_PARAMS[Tool Parameters<br/>User-controlled data]
        FILE_ACCESS[File Access<br/>Path traversal risks]
        CODE_EXEC[Code Execution<br/>Python sandbox]
    end
    
    subgraph "Security Controls"
        INPUT_VALID[Input Validation<br/>Parameter checking]
        PATH_RESTRICT[Path Restrictions<br/>Memory dir only]
        SANDBOX[Code Sandboxing<br/>Restricted execution]
        FILTER[Content Filtering<br/>Privacy controls]
    end
    
    subgraph "Security Boundaries"
        PROCESS[Process Isolation<br/>Separate agent process]
        FILESYSTEM[File System<br/>Limited access scope]
        NETWORK[Network Isolation<br/>Local-only by default]
    end
    
    MCP_PROTOCOL --> INPUT_VALID
    TOOL_PARAMS --> INPUT_VALID
    FILE_ACCESS --> PATH_RESTRICT
    CODE_EXEC --> SANDBOX
    
    INPUT_VALID --> PROCESS
    PATH_RESTRICT --> FILESYSTEM
    SANDBOX --> PROCESS
    FILTER --> FILESYSTEM
```

## Next Steps

For related architecture documentation:
- [Agent Architecture](./agent-architecture.md) - Agent implementation details
- [Memory System Architecture](./memory-system-architecture.md) - Memory organization
- [API Architecture](./api-architecture.md) - API design patterns
- [Deployment Architecture](./deployment-architecture.md) - Deployment configurations