# API Architecture

This document details the API design, protocol specifications, and integration patterns for the mem-agent-mcp system.

## API Overview

```mermaid
graph TB
    subgraph "External APIs"
        MCP_API[Model Context Protocol<br/>JSON-RPC 2.0]
        HTTP_API[HTTP API<br/>RESTful endpoints]
        OPENAI_API[OpenAI-compatible API<br/>Model inference]
    end
    
    subgraph "Internal APIs"
        AGENT_API[Agent API<br/>Memory operations]
        TOOL_API[Tool API<br/>Function execution]
        MEMORY_API[Memory API<br/>File operations]
    end
    
    subgraph "Protocol Layers"
        TRANSPORT[Transport Layer<br/>STDIO, HTTP, WebSocket]
        SERIALIZATION[Serialization Layer<br/>JSON, MessagePack]
        VALIDATION[Validation Layer<br/>Schema validation]
    end
    
    subgraph "Client Integration"
        CLAUDE[Claude Desktop/Code<br/>MCP STDIO]
        CHATGPT[ChatGPT<br/>MCP HTTP]
        CUSTOM[Custom Clients<br/>API integrations]
    end
    
    MCP_API --> TRANSPORT
    HTTP_API --> TRANSPORT
    OPENAI_API --> SERIALIZATION
    
    AGENT_API --> TOOL_API
    TOOL_API --> MEMORY_API
    
    TRANSPORT --> VALIDATION
    SERIALIZATION --> VALIDATION
    
    CLAUDE --> MCP_API
    CHATGPT --> HTTP_API
    CUSTOM --> OPENAI_API
    
    %% Styling
    classDef external fill:#e3f2fd
    classDef internal fill:#f3e5f5
    classDef protocol fill:#e8f5e8
    classDef client fill:#fff3e0
    
    class MCP_API,HTTP_API,OPENAI_API external
    class AGENT_API,TOOL_API,MEMORY_API internal
    class TRANSPORT,SERIALIZATION,VALIDATION protocol
    class CLAUDE,CHATGPT,CUSTOM client
```

## Model Context Protocol (MCP) API

### MCP Protocol Specification
```mermaid
sequenceDiagram
    participant Client
    participant MCPServer
    participant Tool
    participant Agent
    
    Note over Client,Agent: MCP Protocol Flow
    
    Client->>MCPServer: Initialize Request
    MCPServer-->>Client: Server Info + Capabilities
    
    Client->>MCPServer: List Tools Request
    MCPServer-->>Client: Available Tools Schema
    
    Client->>MCPServer: Tool Call Request
    MCPServer->>Tool: Execute Tool
    Tool->>Agent: Process with agent
    Agent-->>Tool: Agent response
    Tool-->>MCPServer: Tool result
    MCPServer-->>Client: Tool Response
    
    Client->>MCPServer: Shutdown Request
    MCPServer-->>Client: Shutdown Confirmation
```

### MCP Tool Schema
```mermaid
classDiagram
    class MCPTool {
        +string name
        +string description
        +object parameters
        +array required
        +validate_parameters()
        +execute()
    }
    
    class ToolParameters {
        +string type "object"
        +object properties
        +array required
        +boolean additionalProperties
    }
    
    class ParameterProperty {
        +string type
        +string description
        +array enum
        +any default
        +validate_value()
    }
    
    MCPTool --> ToolParameters
    ToolParameters --> ParameterProperty
```

**MCP Tool Definitions:**

#### Read Memory Files Tool
```json
{
  "name": "read_memory_files",
  "description": "Read content from memory markdown files",
  "parameters": {
    "type": "object",
    "properties": {
      "files": {
        "type": "array",
        "items": {"type": "string"},
        "description": "List of memory file paths to read",
        "minItems": 1,
        "maxItems": 10
      }
    },
    "required": ["files"]
  }
}
```

#### Search Files Tool
```json
{
  "name": "search_files", 
  "description": "Search through memory files with optional filters",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query or pattern",
        "minLength": 1,
        "maxLength": 500
      },
      "max_results": {
        "type": "integer",
        "description": "Maximum number of results to return",
        "minimum": 1,
        "maximum": 50,
        "default": 10
      },
      "file_pattern": {
        "type": "string",
        "description": "File pattern filter (glob)",
        "default": "*.md"
      }
    },
    "required": ["query"]
  }
}
```

### MCP Message Format
```mermaid
classDiagram
    class MCPRequest {
        +string jsonrpc "2.0"
        +string method
        +object params
        +string id
        +validate_request()
    }
    
    class MCPResponse {
        +string jsonrpc "2.0"
        +any result
        +object error
        +string id
        +format_response()
    }
    
    class MCPError {
        +integer code
        +string message
        +any data
        +format_error()
    }
    
    MCPRequest --> MCPResponse
    MCPResponse --> MCPError
```

**Standard MCP Error Codes:**
```json
{
  "PARSE_ERROR": -32700,
  "INVALID_REQUEST": -32600,
  "METHOD_NOT_FOUND": -32601,
  "INVALID_PARAMS": -32602,
  "INTERNAL_ERROR": -32603,
  "TOOL_EXECUTION_ERROR": -32000,
  "MEMORY_ACCESS_ERROR": -32001,
  "AGENT_ERROR": -32002
}
```

## HTTP API Specification

### HTTP API Endpoints
```mermaid
graph TB
    subgraph "MCP HTTP Endpoints"
        MCP_ROOT[POST /mcp<br/>MCP-compliant HTTP endpoint]
        MCP_HEALTH[GET /health<br/>Health check endpoint]
        MCP_TOOLS[GET /tools<br/>Tool discovery endpoint]
    end
    
    subgraph "Memory API Endpoints"
        MEM_READ[POST /api/memory/read<br/>Read memory files]
        MEM_SEARCH[POST /api/memory/search<br/>Search memory content]
        MEM_LIST[GET /api/memory/list<br/>List memory files]
    end
    
    subgraph "Agent API Endpoints"
        AGENT_QUERY[POST /api/agent/query<br/>Direct agent queries]
        AGENT_TOOLS[GET /api/agent/tools<br/>Available agent tools]
        AGENT_STATUS[GET /api/agent/status<br/>Agent status]
    end
    
    MCP_ROOT --> MEM_READ
    MCP_TOOLS --> MEM_SEARCH
    MCP_HEALTH --> MEM_LIST
    
    MEM_READ --> AGENT_QUERY
    MEM_SEARCH --> AGENT_TOOLS
    MEM_LIST --> AGENT_STATUS
```

### HTTP API Request/Response Flow
```mermaid
sequenceDiagram
    participant Client
    participant HTTPServer
    participant MCPAdapter
    participant Agent
    
    Note over Client,Agent: HTTP to MCP Adaptation
    
    Client->>HTTPServer: POST /mcp (MCP over HTTP)
    HTTPServer->>HTTPServer: Parse HTTP request
    HTTPServer->>MCPAdapter: Convert to MCP format
    MCPAdapter->>Agent: Execute MCP request
    Agent-->>MCPAdapter: MCP response
    MCPAdapter->>HTTPServer: Convert to HTTP format
    HTTPServer-->>Client: HTTP response
    
    Note over Client,Agent: Direct API Access
    
    Client->>HTTPServer: POST /api/memory/search
    HTTPServer->>HTTPServer: Validate request
    HTTPServer->>Agent: Direct agent call
    Agent-->>HTTPServer: Agent response
    HTTPServer-->>Client: JSON response
```

### API Authentication and Security
```mermaid
graph TB
    subgraph "Authentication Methods"
        NO_AUTH[No Authentication<br/>Local development]
        API_KEY[API Key<br/>Header-based auth]
        JWT[JWT Tokens<br/>Stateless auth]
        OAUTH[OAuth 2.0<br/>Third-party auth]
    end
    
    subgraph "Security Headers"
        CORS[CORS Headers<br/>Cross-origin requests]
        SECURITY[Security Headers<br/>HSTS, CSP, etc.]
        RATE_LIMIT[Rate Limit Headers<br/>Usage tracking]
    end
    
    subgraph "Request Validation"
        INPUT_VALID[Input Validation<br/>Parameter checking]
        SCHEMA_VALID[Schema Validation<br/>JSON schema]
        PATH_VALID[Path Validation<br/>Security checks]
    end
    
    NO_AUTH --> CORS
    API_KEY --> SECURITY
    JWT --> RATE_LIMIT
    OAUTH --> RATE_LIMIT
    
    CORS --> INPUT_VALID
    SECURITY --> SCHEMA_VALID
    RATE_LIMIT --> PATH_VALID
```

### OpenAPI Specification
```yaml
openapi: 3.0.3
info:
  title: mem-agent-mcp API
  description: Memory Agent MCP Server HTTP API
  version: 1.0.0
  
servers:
  - url: http://localhost:8081
    description: Local development server
    
paths:
  /mcp:
    post:
      summary: MCP over HTTP endpoint
      description: Model Context Protocol requests over HTTP
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MCPRequest'
      responses:
        200:
          description: MCP response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MCPResponse'
        400:
          description: Invalid request
        500:
          description: Server error
          
  /api/memory/search:
    post:
      summary: Search memory files
      description: Search through memory files with filters
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                query:
                  type: string
                  description: Search query
                max_results:
                  type: integer
                  minimum: 1
                  maximum: 50
                  default: 10
                filters:
                  type: array
                  items:
                    type: string
              required: [query]
      responses:
        200:
          description: Search results
          content:
            application/json:
              schema:
                type: object
                properties:
                  results:
                    type: array
                    items:
                      $ref: '#/components/schemas/SearchResult'
                  total:
                    type: integer
                  query_time:
                    type: number
                    
components:
  schemas:
    MCPRequest:
      type: object
      properties:
        jsonrpc:
          type: string
          enum: ["2.0"]
        method:
          type: string
        params:
          type: object
        id:
          type: string
      required: [jsonrpc, method, id]
      
    MCPResponse:
      type: object
      properties:
        jsonrpc:
          type: string
          enum: ["2.0"]
        result:
          type: object
        error:
          $ref: '#/components/schemas/MCPError'
        id:
          type: string
      required: [jsonrpc, id]
      
    MCPError:
      type: object
      properties:
        code:
          type: integer
        message:
          type: string
        data:
          type: object
      required: [code, message]
      
    SearchResult:
      type: object
      properties:
        file_path:
          type: string
        content:
          type: string
        score:
          type: number
        context:
          type: string
      required: [file_path, content, score]
```

## Internal API Architecture

### Agent API
```mermaid
classDiagram
    class AgentAPI {
        +process_query(query: str) AgentResponse
        +execute_tool(tool: str, params: dict) ToolResult
        +get_conversation_history() list
        +clear_conversation() void
        +get_agent_status() AgentStatus
    }
    
    class AgentResponse {
        +str thoughts
        +str reply
        +str formatted_response
        +bool has_tool_calls
        +list tool_results
    }
    
    class ToolResult {
        +str tool_name
        +dict parameters
        +any result
        +bool success
        +str error_message
    }
    
    class AgentStatus {
        +bool is_ready
        +str model_backend
        +str memory_path
        +int conversation_turns
        +float last_response_time
    }
    
    AgentAPI --> AgentResponse
    AgentAPI --> ToolResult
    AgentAPI --> AgentStatus
```

### Tool API Specification
```mermaid
sequenceDiagram
    participant Agent
    participant ToolRegistry
    participant MemoryTool
    participant FileSystem
    
    Note over Agent,FileSystem: Tool Execution Flow
    
    Agent->>ToolRegistry: execute_tool("read_file", params)
    ToolRegistry->>ToolRegistry: Validate tool exists
    ToolRegistry->>ToolRegistry: Validate parameters
    ToolRegistry->>MemoryTool: call_tool(params)
    MemoryTool->>FileSystem: perform_operation(params)
    FileSystem-->>MemoryTool: operation_result
    MemoryTool->>MemoryTool: format_result()
    MemoryTool-->>ToolRegistry: formatted_result
    ToolRegistry-->>Agent: tool_response
```

### Memory API Operations
```mermaid
classDiagram
    class MemoryAPI {
        +read_file(path: str) FileContent
        +search_files(query: str, filters: list) SearchResults
        +list_directory(path: str) DirectoryListing
        +get_file_metadata(path: str) FileMetadata
        +validate_path(path: str) bool
    }
    
    class FileContent {
        +str file_path
        +str content
        +dict metadata
        +datetime last_modified
        +int size_bytes
    }
    
    class SearchResults {
        +list results
        +int total_matches
        +float query_time
        +str query_string
        +dict filters_applied
    }
    
    class DirectoryListing {
        +str directory_path
        +list files
        +list subdirectories
        +int total_items
    }
    
    MemoryAPI --> FileContent
    MemoryAPI --> SearchResults
    MemoryAPI --> DirectoryListing
```

## Protocol Implementations

### STDIO Transport
```mermaid
sequenceDiagram
    participant Client
    participant STDIOServer
    participant MessageHandler
    participant ResponseFormatter
    
    Note over Client,ResponseFormatter: STDIO Communication
    
    Client->>STDIOServer: JSON message via stdin
    STDIOServer->>MessageHandler: parse_message()
    MessageHandler->>MessageHandler: validate_json_rpc()
    MessageHandler->>MessageHandler: route_to_handler()
    MessageHandler-->>ResponseFormatter: handler_result
    ResponseFormatter->>ResponseFormatter: format_response()
    ResponseFormatter-->>STDIOServer: json_response
    STDIOServer-->>Client: JSON response via stdout
```

### HTTP Transport  
```mermaid
graph TB
    subgraph "HTTP Server"
        FASTAPI[FastAPI Application<br/>ASGI framework]
        MIDDLEWARE[Middleware Stack<br/>CORS, logging, auth]
        ROUTING[Request Routing<br/>Path-based routing]
    end
    
    subgraph "Request Processing"
        PARSER[Request Parser<br/>JSON, form data]
        VALIDATOR[Request Validator<br/>Schema validation]
        CONVERTER[Format Converter<br/>HTTP ↔ MCP]
    end
    
    subgraph "Response Handling"
        FORMATTER[Response Formatter<br/>JSON serialization]
        ERROR_HANDLER[Error Handler<br/>Exception mapping]
        HEADERS[Header Management<br/>CORS, security]
    end
    
    FASTAPI --> MIDDLEWARE
    MIDDLEWARE --> ROUTING
    ROUTING --> PARSER
    
    PARSER --> VALIDATOR
    VALIDATOR --> CONVERTER
    CONVERTER --> FORMATTER
    
    FORMATTER --> ERROR_HANDLER
    ERROR_HANDLER --> HEADERS
```

### WebSocket Transport (Future)
```mermaid
sequenceDiagram
    participant Client
    participant WSServer
    participant ConnectionManager
    participant MessageRouter
    
    Note over Client,MessageRouter: WebSocket Lifecycle
    
    Client->>WSServer: WebSocket connection request
    WSServer->>ConnectionManager: register_connection()
    ConnectionManager-->>Client: Connection established
    
    loop Message Exchange
        Client->>WSServer: JSON message
        WSServer->>MessageRouter: route_message()
        MessageRouter-->>WSServer: response
        WSServer-->>Client: JSON response
    end
    
    Client->>WSServer: Close connection
    WSServer->>ConnectionManager: unregister_connection()
    ConnectionManager-->>Client: Connection closed
```

## Data Serialization and Validation

### JSON Schema Validation
```mermaid
graph TB
    subgraph "Schema Definitions"
        MCP_SCHEMA[MCP Protocol Schema<br/>JSON-RPC 2.0 spec]
        TOOL_SCHEMA[Tool Parameter Schema<br/>Parameter validation]
        RESPONSE_SCHEMA[Response Schema<br/>Output validation]
    end
    
    subgraph "Validation Pipeline"
        INPUT_VALID[Input Validation<br/>Request parsing]
        PARAM_VALID[Parameter Validation<br/>Tool parameters]
        OUTPUT_VALID[Output Validation<br/>Response format]
    end
    
    subgraph "Error Handling"
        SCHEMA_ERROR[Schema Error<br/>Validation failures]
        ERROR_FORMAT[Error Formatting<br/>User-friendly messages]
        ERROR_LOG[Error Logging<br/>Debug information]
    end
    
    MCP_SCHEMA --> INPUT_VALID
    TOOL_SCHEMA --> PARAM_VALID
    RESPONSE_SCHEMA --> OUTPUT_VALID
    
    INPUT_VALID --> SCHEMA_ERROR
    PARAM_VALID --> SCHEMA_ERROR
    OUTPUT_VALID --> SCHEMA_ERROR
    
    SCHEMA_ERROR --> ERROR_FORMAT
    ERROR_FORMAT --> ERROR_LOG
```

### Message Serialization
```mermaid
classDiagram
    class MessageSerializer {
        +serialize(obj: any) bytes
        +deserialize(data: bytes) any
        +validate_schema(data: dict) bool
        +compress(data: bytes) bytes
    }
    
    class JSONSerializer {
        +json_encode(obj: any) str
        +json_decode(data: str) any
        +handle_datetime(dt: datetime) str
        +handle_custom_types() any
    }
    
    class MessagePackSerializer {
        +msgpack_encode(obj: any) bytes
        +msgpack_decode(data: bytes) any
        +handle_binary_data() bytes
    }
    
    MessageSerializer <|-- JSONSerializer
    MessageSerializer <|-- MessagePackSerializer
```

## Error Handling and Status Codes

### API Error Classification
```mermaid
graph TB
    subgraph "Client Errors (4xx)"
        BAD_REQUEST[400 Bad Request<br/>Invalid parameters]
        UNAUTHORIZED[401 Unauthorized<br/>Missing authentication]
        FORBIDDEN[403 Forbidden<br/>Access denied]
        NOT_FOUND[404 Not Found<br/>Resource missing]
        TOO_MANY[429 Too Many Requests<br/>Rate limit exceeded]
    end
    
    subgraph "Server Errors (5xx)"
        INTERNAL_ERROR[500 Internal Server Error<br/>Unexpected failure]
        SERVICE_UNAVAIL[503 Service Unavailable<br/>Backend down]
        TIMEOUT[504 Gateway Timeout<br/>Operation timeout]
    end
    
    subgraph "Custom Errors (MCP)"
        TOOL_ERROR[-32000 Tool Execution Error<br/>Tool-specific failures]
        MEMORY_ERROR[-32001 Memory Access Error<br/>File system issues]
        AGENT_ERROR[-32002 Agent Error<br/>Agent processing failures]
    end
    
    BAD_REQUEST --> TOOL_ERROR
    NOT_FOUND --> MEMORY_ERROR
    INTERNAL_ERROR --> AGENT_ERROR
```

### Error Response Format
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32001,
    "message": "Memory access error",
    "data": {
      "error_type": "FileNotFoundError",
      "error_details": "Memory file not found: entities/missing.md",
      "suggested_action": "Check file path and ensure file exists",
      "timestamp": "2024-01-15T10:30:00Z",
      "request_id": "req_123456"
    }
  },
  "id": "call_123"
}
```

## Rate Limiting and Throttling

### Rate Limiting Strategy
```mermaid
graph TB
    subgraph "Rate Limiting Algorithms"
        TOKEN_BUCKET[Token Bucket<br/>Burst allowance]
        SLIDING_WINDOW[Sliding Window<br/>Time-based limits]
        FIXED_WINDOW[Fixed Window<br/>Period-based limits]
    end
    
    subgraph "Limit Categories"
        PER_CLIENT[Per Client<br/>IP-based limits]
        PER_TOOL[Per Tool<br/>Function-specific limits]
        GLOBAL[Global<br/>System-wide limits]
    end
    
    subgraph "Enforcement"
        MEMORY_STORE[In-Memory Store<br/>Redis/Local cache]
        COUNTER[Request Counter<br/>Increment operations]
        DECISION[Allow/Deny Logic<br/>Limit checking]
    end
    
    TOKEN_BUCKET --> PER_CLIENT
    SLIDING_WINDOW --> PER_TOOL
    FIXED_WINDOW --> GLOBAL
    
    PER_CLIENT --> MEMORY_STORE
    PER_TOOL --> COUNTER
    GLOBAL --> DECISION
```

### API Usage Metrics
```mermaid
classDiagram
    class UsageMetrics {
        +int requests_per_minute
        +int requests_per_hour
        +int requests_per_day
        +float avg_response_time
        +int error_count
        +track_request()
        +get_usage_stats()
    }
    
    class RateLimiter {
        +dict limits
        +dict counters
        +bool is_allowed(client_id: str)
        +reset_counters()
        +get_remaining_quota(client_id: str)
    }
    
    class MetricsCollector {
        +record_request(endpoint: str)
        +record_response_time(duration: float)
        +record_error(error_type: str)
        +export_metrics()
    }
    
    UsageMetrics --> RateLimiter
    UsageMetrics --> MetricsCollector
```

## Client SDK and Integration

### Python Client SDK
```python
class MemAgentMCPClient:
    def __init__(self, transport: str = "stdio", 
                 host: str = "localhost", 
                 port: int = 8081):
        self.transport = transport
        self.host = host  
        self.port = port
        self._session = None
    
    async def read_memory_files(self, files: List[str]) -> Dict:
        """Read content from memory files"""
        return await self._call_tool("read_memory_files", {"files": files})
    
    async def search_files(self, query: str, max_results: int = 10) -> Dict:
        """Search through memory files"""
        params = {"query": query, "max_results": max_results}
        return await self._call_tool("search_files", params)
    
    async def _call_tool(self, method: str, params: Dict) -> Dict:
        """Generic tool call method"""
        if self.transport == "stdio":
            return await self._stdio_call(method, params)
        elif self.transport == "http":
            return await self._http_call(method, params)
        else:
            raise ValueError(f"Unsupported transport: {self.transport}")
```

### JavaScript Client SDK
```javascript
class MemAgentMCPClient {
  constructor(options = {}) {
    this.transport = options.transport || 'http';
    this.baseURL = options.baseURL || 'http://localhost:8081';
    this.timeout = options.timeout || 30000;
  }
  
  async readMemoryFiles(files) {
    return this.callTool('read_memory_files', { files });
  }
  
  async searchFiles(query, maxResults = 10) {
    return this.callTool('search_files', { 
      query, 
      max_results: maxResults 
    });
  }
  
  async callTool(method, params) {
    const response = await fetch(`${this.baseURL}/mcp`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method: method,
        params: params,
        id: this.generateId()
      })
    });
    
    const result = await response.json();
    if (result.error) {
      throw new Error(result.error.message);
    }
    return result.result;
  }
}
```

## API Versioning and Evolution

### API Versioning Strategy
```mermaid
graph TB
    subgraph "Version Management"
        SEMVER[Semantic Versioning<br/>MAJOR.MINOR.PATCH]
        BACKWARD_COMPAT[Backward Compatibility<br/>Non-breaking changes]
        DEPRECATION[Deprecation Policy<br/>Phased removal]
    end
    
    subgraph "Version Discovery"
        VERSION_HEADER[Version Header<br/>API-Version: v1]
        CONTENT_TYPE[Content Type<br/>application/vnd.api+json]
        CAPABILITY[Capability Negotiation<br/>Feature detection]
    end
    
    subgraph "Migration Support"
        DUAL_SUPPORT[Dual Version Support<br/>Parallel implementations]
        MIGRATION_GUIDE[Migration Guide<br/>Documentation]
        COMPATIBILITY_LAYER[Compatibility Layer<br/>Translation middleware]
    end
    
    SEMVER --> VERSION_HEADER
    BACKWARD_COMPAT --> CONTENT_TYPE
    DEPRECATION --> CAPABILITY
    
    VERSION_HEADER --> DUAL_SUPPORT
    CONTENT_TYPE --> MIGRATION_GUIDE
    CAPABILITY --> COMPATIBILITY_LAYER
```

## Next Steps

For implementation details:
- [MCP Server Architecture](./mcp-server-architecture.md) - MCP protocol implementation
- [Agent Architecture](./agent-architecture.md) - Agent processing details  
- [Memory System Architecture](./memory-system-architecture.md) - Memory operations
- [Memory Connectors Architecture](./memory-connectors-architecture.md) - Data integration APIs