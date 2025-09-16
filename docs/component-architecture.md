# Component Architecture

This document details the internal architecture and interactions between components in the mem-agent-mcp system.

## Component Interaction Overview

```mermaid
graph TB
    subgraph "Client Layer"
        CLIENT[Client Application]
    end
    
    subgraph "MCP Server Layer"
        MCPSERVER[MCP Server<br/>server.py]
        MCPTOOLS[MCP Tools<br/>read_memory, search_files]
        MCPCONFIG[Configuration<br/>memory_path, filters]
    end
    
    subgraph "Agent Layer"  
        AGENT[Agent Controller<br/>agent.py]
        AGENTTOOLS[Agent Tools<br/>tools.py]
        AGENTENG[Execution Engine<br/>engine.py]
        AGENTUTILS[Agent Utils<br/>utils.py]
    end
    
    subgraph "Model Layer"
        MODELCLIENT[Model Client<br/>model.py]
        VLLMCLIENT[vLLM Client]
        MLXCLIENT[MLX Client]
        OPENROUTER[OpenRouter Client]
    end
    
    subgraph "Memory Layer"
        MEMFILES[Memory Files<br/>.md format]
        MEMINDEX[Memory Index<br/>user.md, entities/]
        MEMFILTER[Memory Filters<br/>.filters]
    end
    
    subgraph "Connector Layer"
        MEMCONN[Memory Connectors<br/>memory_connect.py]
        CONNECTORS[Individual Connectors<br/>chatgpt, github, etc.]
    end
    
    %% Interactions
    CLIENT --> MCPSERVER
    MCPSERVER --> MCPTOOLS
    MCPTOOLS --> AGENT
    MCPCONFIG --> AGENT
    
    AGENT --> AGENTTOOLS
    AGENT --> AGENTENG
    AGENT --> AGENTUTILS
    AGENT --> MODELCLIENT
    
    MODELCLIENT --> VLLMCLIENT
    MODELCLIENT --> MLXCLIENT  
    MODELCLIENT --> OPENROUTER
    
    AGENTTOOLS --> MEMFILES
    AGENTTOOLS --> MEMINDEX
    AGENTTOOLS --> MEMFILTER
    
    MEMCONN --> CONNECTORS
    CONNECTORS --> MEMFILES
    
    %% Styling
    classDef client fill:#e1f5fe
    classDef mcp fill:#f3e5f5
    classDef agent fill:#e8f5e8
    classDef model fill:#f1f8e9
    classDef memory fill:#fff3e0
    classDef connector fill:#fce4ec
    
    class CLIENT client
    class MCPSERVER,MCPTOOLS,MCPCONFIG mcp
    class AGENT,AGENTTOOLS,AGENTENG,AGENTUTILS agent
    class MODELCLIENT,VLLMCLIENT,MLXCLIENT,OPENROUTER model
    class MEMFILES,MEMINDEX,MEMFILTER memory
    class MEMCONN,CONNECTORS connector
```

## Detailed Component Specifications

### MCP Server Components

#### MCP Server (`mcp_server/server.py`)
```mermaid
classDiagram
    class MCPServer {
        +FastMCP mcp
        +str memory_path
        +str model_name
        +Agent agent_instance
        +initialize()
        +read_memory_files()
        +search_files()
        +get_available_tools()
    }
    
    class FastMCP {
        +register_tool()
        +run_stdio()
        +run_http()
    }
    
    class Agent {
        +process_query()
        +get_response()
        +execute_tools()
    }
    
    MCPServer --> FastMCP
    MCPServer --> Agent
```

**Key Responsibilities:**
- MCP protocol implementation using FastMCP
- Tool registration and execution
- Client request routing to agent
- Memory path configuration management

#### MCP Tools
- `read_memory_files`: Read and parse memory markdown files
- `search_files`: Search across memory files with filters
- Configuration management for memory path and filters

### Agent Components  

#### Agent Controller (`agent/agent.py`)
```mermaid
classDiagram
    class Agent {
        +list~ChatMessage~ messages
        +int max_tool_turns
        +bool use_vllm
        +str model
        +str memory_path
        +process_query(query: str) AgentResponse
        +get_model_response() str
        +execute_tool_call() str
        +format_response() AgentResponse
    }
    
    class ChatMessage {
        +Role role
        +str content
        +str tool_calls
        +str tool_call_id
    }
    
    class AgentResponse {
        +str thoughts
        +str reply
        +str formatted_response
    }
    
    Agent --> ChatMessage
    Agent --> AgentResponse
```

**Key Responsibilities:**
- Query processing and conversation management
- Tool orchestration and execution
- Model interaction coordination
- Response formatting and extraction

#### Agent Tools (`agent/tools.py`)
```mermaid
classDiagram
    class ToolRegistry {
        +dict tools
        +register_tool()
        +get_tool()
        +execute_tool()
    }
    
    class MemoryTool {
        +read_file()
        +search_files()
        +list_directory()
    }
    
    class ExecutionTool {
        +execute_python()
        +validate_code()
    }
    
    ToolRegistry --> MemoryTool
    ToolRegistry --> ExecutionTool
```

**Available Tools:**
- `read_file`: Read memory markdown files
- `search_files`: Search with filters and patterns
- `list_directory`: Navigate memory structure
- `execute_python`: Sandboxed code execution

#### Execution Engine (`agent/engine.py`)
```mermaid
sequenceDiagram
    participant Agent
    participant Engine
    participant Sandbox
    participant FileSystem
    
    Agent->>Engine: execute_sandboxed_code(code)
    Engine->>Engine: validate_code()
    Engine->>Sandbox: create_sandbox_env()
    Engine->>Sandbox: execute_in_sandbox()
    Sandbox->>FileSystem: restricted_file_access()
    FileSystem-->>Sandbox: file_contents
    Sandbox-->>Engine: execution_result
    Engine-->>Agent: formatted_result
```

**Security Features:**
- Sandboxed Python code execution
- Restricted file system access
- Import restrictions and validation
- Output capturing and formatting

### Model Layer Components

#### Model Client (`agent/model.py`)
```mermaid
classDiagram
    class ModelClient {
        +create_openai_client()
        +create_vllm_client()
        +get_model_response()
        +handle_tool_calls()
    }
    
    class VLLMClient {
        +str host
        +int port
        +generate_response()
    }
    
    class MLXClient {
        +str model_name
        +generate_response()
    }
    
    class OpenRouterClient {
        +str api_key
        +str model
        +generate_response()
    }
    
    ModelClient --> VLLMClient
    ModelClient --> MLXClient
    ModelClient --> OpenRouterClient
```

**Backend Selection Logic:**
1. Check for vLLM configuration (Linux/GPU)
2. Check for MLX model availability (macOS)
3. Fallback to OpenRouter (cloud inference)

### Memory System Components

#### Memory Organization
```mermaid
graph TD
    subgraph "Memory Directory"
        USER[user.md<br/>Main Profile]
        ENTITIES[entities/<br/>Directory]
    end
    
    subgraph "Entity Files"
        ENT1[person1.md]
        ENT2[company1.md]
        ENT3[project1.md]
    end
    
    subgraph "Generated Content"
        TOPICS[topics/<br/>Auto-generated]
        CONVS[conversations/<br/>Import data]
        INDEX[index.md<br/>Navigation]
    end
    
    USER --> ENTITIES
    ENTITIES --> ENT1
    ENTITIES --> ENT2
    ENTITIES --> ENT3
    
    ENTITIES --> TOPICS
    ENTITIES --> CONVS
    ENTITIES --> INDEX
```

**Memory Structure:**
- `user.md`: Central profile with relationship links
- `entities/`: Individual entity files with relationships
- Wikilink format: `[[entities/entity_name.md]]`
- Auto-generated topic organization
- Cross-referenced navigation

### Memory Connector Components

#### Connector Architecture
```mermaid
classDiagram
    class BaseMemoryConnector {
        <<abstract>>
        +str output_path
        +dict config
        +extract_data(source_path)* 
        +organize_data(data)*
        +generate_memory_files(data)*
    }
    
    class ChatGPTConnector {
        +extract_conversations()
        +categorize_by_topic()
        +generate_entity_files()
    }
    
    class GitHubConnector {
        +fetch_repository_data()
        +process_issues_prs()
        +create_project_entities()
    }
    
    class NotionConnector {
        +parse_notion_export()
        +extract_page_hierarchy()
        +convert_to_markdown()
    }
    
    BaseMemoryConnector <|-- ChatGPTConnector
    BaseMemoryConnector <|-- GitHubConnector
    BaseMemoryConnector <|-- NotionConnector
```

**Connector Types:**
- **Export-based**: Process ZIP/JSON exports (ChatGPT, Notion, Nuclino)
- **Live API**: Real-time data sync (GitHub, Google Docs)
- **Categorization**: AI-powered topic organization
- **Entity Extraction**: Automatic relationship mapping

## Component Communication Patterns

### Request Flow
```mermaid
sequenceDiagram
    participant Client
    participant MCPServer
    participant Agent
    participant ModelClient
    participant MemorySystem
    
    Client->>MCPServer: MCP Request
    MCPServer->>Agent: Process Query
    Agent->>MemorySystem: Search/Read Files
    MemorySystem-->>Agent: Memory Content
    Agent->>ModelClient: Generate Response
    ModelClient-->>Agent: LLM Response
    Agent->>Agent: Extract Tools/Reply
    Agent-->>MCPServer: Formatted Response
    MCPServer-->>Client: MCP Response
```

### Tool Execution Flow
```mermaid
sequenceDiagram
    participant Agent
    participant Tools
    participant ExecutionEngine
    participant MemoryFiles
    
    Agent->>Tools: Execute Tool Call
    Tools->>ExecutionEngine: Sandboxed Execution
    ExecutionEngine->>MemoryFiles: File Operations
    MemoryFiles-->>ExecutionEngine: File Contents
    ExecutionEngine-->>Tools: Execution Result
    Tools-->>Agent: Tool Response
```

## Configuration and Settings

### Environment Variables
- `MEMORY_DIR`: Memory directory path
- `FASTMCP_LOG_LEVEL`: Logging level
- `MCP_TRANSPORT`: Transport protocol (stdio/http)
- `VLLM_HOST`, `VLLM_PORT`: vLLM configuration
- `OPENROUTER_API_KEY`: Cloud inference fallback

### Configuration Files
- `.memory_path`: Memory directory location
- `.filters`: Privacy and content filters  
- `.mlx_model_name`: MLX model selection
- `mcp.json`: MCP client configuration

## Next Steps

For more detailed documentation:
- [Data Flow Architecture](./data-flow-architecture.md) - Request processing flows
- [Agent Architecture](./agent-architecture.md) - Agent implementation details
- [Memory System Architecture](./memory-system-architecture.md) - Memory organization