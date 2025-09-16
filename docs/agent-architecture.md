# Agent Architecture

This document details the internal architecture of the memory agent, including the specialized LLM, tool system, and execution engine.

## Agent Core Architecture

```mermaid
graph TB
    subgraph "Agent Controller"
        AC[Agent Class<br/>agent.py]
        MSG[Message Management<br/>Conversation History]
        TOOL_ORCH[Tool Orchestration<br/>Call Management]
    end
    
    subgraph "Model Interface"
        MC[Model Client<br/>model.py]
        VLLM[vLLM Client<br/>Linux/GPU]
        MLX[MLX Client<br/>macOS/Metal]
        OR[OpenRouter Client<br/>Cloud Fallback]
    end
    
    subgraph "Tool System"
        TR[Tool Registry<br/>tools.py]
        MEM_TOOLS[Memory Tools<br/>read_file, search_files]
        EXEC_TOOLS[Execution Tools<br/>execute_python]
        UTIL_TOOLS[Utility Tools<br/>list_directory]
    end
    
    subgraph "Execution Engine"
        EE[Execution Engine<br/>engine.py]
        SANDBOX[Python Sandbox<br/>Restricted Environment]
        VALIDATOR[Code Validator<br/>Security Checks]
    end
    
    subgraph "Utilities"
        UTILS[Agent Utils<br/>utils.py]
        PARSERS[Response Parsers<br/>Extract Tools/Reply]
        FORMATTERS[Response Formatters<br/>Structure Output]
    end
    
    AC --> MSG
    AC --> TOOL_ORCH
    AC --> MC
    
    MC --> VLLM
    MC --> MLX
    MC --> OR
    
    TOOL_ORCH --> TR
    TR --> MEM_TOOLS
    TR --> EXEC_TOOLS
    TR --> UTIL_TOOLS
    
    EXEC_TOOLS --> EE
    EE --> SANDBOX
    EE --> VALIDATOR
    
    AC --> UTILS
    UTILS --> PARSERS
    UTILS --> FORMATTERS
    
    %% Styling
    classDef controller fill:#e8f5e8
    classDef model fill:#f1f8e9
    classDef tool fill:#fff3e0
    classDef execution fill:#fce4ec
    classDef utility fill:#f3e5f5
    
    class AC,MSG,TOOL_ORCH controller
    class MC,VLLM,MLX,OR model
    class TR,MEM_TOOLS,EXEC_TOOLS,UTIL_TOOLS tool
    class EE,SANDBOX,VALIDATOR execution
    class UTILS,PARSERS,FORMATTERS utility
```

## Memory Agent Specialization

### Dria Fine-tuned Model
The memory agent is based on Dria's specialized fine-tuning for memory management:

```mermaid
graph LR
    subgraph "Base Model"
        LLM[Large Language Model<br/>General Purpose]
    end
    
    subgraph "Dria Fine-tuning"
        MEMORY_DATA[Memory Task Data<br/>Obsidian-style Patterns]
        TOOL_DATA[Tool Usage Data<br/>File Operations]
        SEARCH_DATA[Search Pattern Data<br/>Query Understanding]
    end
    
    subgraph "Specialized Agent"
        MEM_AGENT[Memory Agent<br/>driaforall/mem-agent]
        CAPABILITIES[Enhanced Capabilities<br/>- Memory Navigation<br/>- Tool Selection<br/>- Context Assembly]
    end
    
    LLM --> MEMORY_DATA
    LLM --> TOOL_DATA
    LLM --> SEARCH_DATA
    
    MEMORY_DATA --> MEM_AGENT
    TOOL_DATA --> MEM_AGENT
    SEARCH_DATA --> MEM_AGENT
    
    MEM_AGENT --> CAPABILITIES
```

**Model Variants:**
- **4-bit Quantization**: `mem-agent-mlx-4bit` - High efficiency for macOS
- **8-bit Quantization**: `mem-agent-mlx-8bit` - Balanced performance/quality  
- **bf16 Precision**: `driaforall/mem-agent-mlx-bf16` - Full precision for accuracy

### System Prompt Architecture
```mermaid
graph TD
    subgraph "System Prompt Components"
        ROLE[Role Definition<br/>Memory Agent Identity]
        MEMORY[Memory System Rules<br/>File Structure, Wikilinks]
        TOOLS[Tool Documentation<br/>Available Functions]
        BEHAVIOR[Behavioral Guidelines<br/>Response Patterns]
        EXAMPLES[Usage Examples<br/>Query Patterns]
    end
    
    subgraph "Dynamic Context"
        MEM_PATH[Memory Path<br/>Current Location]
        FILTERS[Active Filters<br/>Privacy Controls]
        USER_INFO[User Profile<br/>From user.md]
    end
    
    ROLE --> MEMORY
    MEMORY --> TOOLS
    TOOLS --> BEHAVIOR
    BEHAVIOR --> EXAMPLES
    
    MEM_PATH --> ROLE
    FILTERS --> BEHAVIOR
    USER_INFO --> EXAMPLES
```

## Tool System Architecture

### Tool Registry Implementation
```mermaid
classDiagram
    class ToolRegistry {
        +dict available_tools
        +register_tool(func, description)
        +get_tool_schema(name)
        +execute_tool(name, args)
        +list_tools()
    }
    
    class MemoryTool {
        +read_file(file_path: str) str
        +search_files(query: str, filters: list) str
        +list_directory(path: str) str
    }
    
    class ExecutionTool {
        +execute_python(code: str) str
    }
    
    class Tool {
        +str name
        +str description  
        +dict parameters
        +function callable
    }
    
    ToolRegistry --> Tool
    Tool <|-- MemoryTool
    Tool <|-- ExecutionTool
```

### Memory Tools Detail

#### `read_file` Tool
```mermaid
sequenceDiagram
    participant Agent
    participant ReadTool
    participant FileSystem
    participant FilterSystem
    
    Agent->>ReadTool: read_file(path)
    ReadTool->>ReadTool: Validate path security
    ReadTool->>FileSystem: Read file content
    FileSystem-->>ReadTool: Raw content
    ReadTool->>FilterSystem: Apply privacy filters
    FilterSystem-->>ReadTool: Filtered content
    ReadTool-->>Agent: Processed content
```

**Security Features:**
- Path traversal prevention
- File extension validation
- Size limits for large files
- Privacy filter application

#### `search_files` Tool  
```mermaid
flowchart TD
    subgraph "Search Input"
        QUERY[Search Query]
        FILTERS[Search Filters]
        PATH[Search Path]
    end
    
    subgraph "File Discovery"
        SCAN[Recursive Directory Scan]
        VALIDATE[Validate File Types]
        BLACKLIST[Apply Blacklist]
    end
    
    subgraph "Content Search"
        READ[Read File Contents]
        PATTERN[Pattern Matching]
        CONTEXT[Extract Context]
        RANK[Rank Results]
    end
    
    subgraph "Result Processing"
        FILTER_APPLY[Apply Privacy Filters]
        LIMIT[Apply Result Limits]
        FORMAT[Format Output]
    end
    
    QUERY --> SCAN
    FILTERS --> VALIDATE
    PATH --> SCAN
    
    SCAN --> VALIDATE
    VALIDATE --> BLACKLIST
    BLACKLIST --> READ
    
    READ --> PATTERN
    PATTERN --> CONTEXT
    CONTEXT --> RANK
    
    RANK --> FILTER_APPLY
    FILTER_APPLY --> LIMIT
    LIMIT --> FORMAT
```

**Search Features:**
- Regular expression support
- Case-insensitive matching
- Context window extraction
- Relevance scoring
- Result limits and pagination

#### `execute_python` Tool
```mermaid
sequenceDiagram
    participant Agent
    participant ExecuteTool
    participant CodeValidator
    participant SandboxEngine
    participant FileSystem
    
    Agent->>ExecuteTool: execute_python(code)
    ExecuteTool->>CodeValidator: validate_code(code)
    
    alt Code validation fails
        CodeValidator-->>ExecuteTool: Validation error
        ExecuteTool-->>Agent: Error message
    else Code validation succeeds
        ExecuteTool->>SandboxEngine: execute_in_sandbox(code)
        SandboxEngine->>SandboxEngine: Create restricted environment
        SandboxEngine->>FileSystem: Limited file access
        FileSystem-->>SandboxEngine: File operations
        SandboxEngine-->>ExecuteTool: Execution result
        ExecuteTool-->>Agent: Formatted result
    end
```

## Execution Engine Architecture

### Sandboxed Code Execution
```mermaid
graph TB
    subgraph "Security Layers"
        INPUT[Code Input]
        PARSE[AST Parsing<br/>Syntax Validation]
        RESTRICT[Import Restrictions<br/>Dangerous Module Blocking]
        TIMEOUT[Execution Timeout<br/>Resource Limits]
    end
    
    subgraph "Execution Environment"
        NAMESPACE[Restricted Namespace<br/>Limited Builtins]
        FILESYSTEM[File Access Control<br/>Memory Directory Only]
        STDOUT[Output Capture<br/>Result Collection]
    end
    
    subgraph "Result Processing"
        OUTPUT[Execution Output]
        ERROR[Error Handling<br/>Exception Capture]
        FORMAT[Output Formatting<br/>Clean Response]
    end
    
    INPUT --> PARSE
    PARSE --> RESTRICT
    RESTRICT --> TIMEOUT
    
    TIMEOUT --> NAMESPACE
    NAMESPACE --> FILESYSTEM
    FILESYSTEM --> STDOUT
    
    STDOUT --> OUTPUT
    OUTPUT --> ERROR
    ERROR --> FORMAT
```

### Security Restrictions
```mermaid
graph LR
    subgraph "Blocked Operations"
        IMPORTS[Dangerous Imports<br/>os, sys, subprocess]
        NETWORK[Network Access<br/>urllib, requests]
        FILESYSTEM_WRITE[Unrestricted Writes<br/>Outside memory dir]
        EXEC[Code Execution<br/>exec, eval]
    end
    
    subgraph "Allowed Operations"
        READ[Memory File Reading<br/>Restricted paths]
        PROCESS[Data Processing<br/>json, re, string]
        MATH[Mathematical Operations<br/>math, statistics]
        TIME[Time Operations<br/>datetime]
    end
    
    subgraph "Execution Limits"
        TIMEOUT[30 Second Timeout]
        MEMORY[Limited Memory Usage]
        RECURSION[Recursion Depth Limit]
        OUTPUT[Output Size Limit]
    end
    
    IMPORTS -.-> READ
    NETWORK -.-> PROCESS  
    FILESYSTEM_WRITE -.-> MATH
    EXEC -.-> TIME
    
    READ --> TIMEOUT
    PROCESS --> MEMORY
    MATH --> RECURSION
    TIME --> OUTPUT
```

## Conversation Management

### Message Flow Architecture
```mermaid
sequenceDiagram
    participant Client
    participant Agent
    participant MessageQueue
    participant ModelClient
    participant ToolSystem
    
    Client->>Agent: User Query
    Agent->>MessageQueue: Add user message
    Agent->>MessageQueue: Add system prompt
    
    loop Conversation Turn
        Agent->>ModelClient: Send message history
        ModelClient-->>Agent: Model response
        Agent->>MessageQueue: Add assistant message
        
        alt Response contains tool calls
            Agent->>ToolSystem: Execute tools
            ToolSystem-->>Agent: Tool results
            Agent->>MessageQueue: Add tool results
        else Response is final
            break
        end
    end
    
    Agent->>Agent: Extract final response
    Agent-->>Client: Formatted response
```

### Conversation State Management
```mermaid
classDiagram
    class ChatMessage {
        +Role role
        +str content
        +str tool_calls
        +str tool_call_id
        +str name
    }
    
    class ConversationManager {
        +list~ChatMessage~ messages
        +int max_turns
        +add_message(message)
        +get_conversation()
        +clear_conversation()
        +export_conversation()
    }
    
    class Role {
        <<enumeration>>
        SYSTEM
        USER
        ASSISTANT
        TOOL
    }
    
    ConversationManager --> ChatMessage
    ChatMessage --> Role
```

## Response Processing Architecture

### Response Extraction Pipeline
```mermaid
flowchart TD
    subgraph "Raw Model Output"
        RAW[Raw LLM Response]
        PARSE[Parse Structure]
        VALIDATE[Validate Format]
    end
    
    subgraph "Component Extraction"
        THOUGHTS[Extract Thoughts<br/>Internal reasoning]
        REPLY[Extract Reply<br/>User-facing response]
        TOOLS[Extract Tool Calls<br/>Function calls]
    end
    
    subgraph "Response Assembly"
        FORMAT[Format Components]
        STRUCTURE[Create Response Object]
        VALIDATE_OUT[Final Validation]
    end
    
    RAW --> PARSE
    PARSE --> VALIDATE
    VALIDATE --> THOUGHTS
    VALIDATE --> REPLY
    VALIDATE --> TOOLS
    
    THOUGHTS --> FORMAT
    REPLY --> FORMAT
    TOOLS --> FORMAT
    FORMAT --> STRUCTURE
    STRUCTURE --> VALIDATE_OUT
```

### Response Format Structure
```mermaid
classDiagram
    class AgentResponse {
        +str thoughts
        +str reply
        +str formatted_response
        +bool has_tool_calls
        +list tool_results
    }
    
    class ThoughtExtraction {
        +extract_thoughts(content: str) str
        +clean_thoughts(thoughts: str) str
    }
    
    class ReplyExtraction {
        +extract_reply(content: str) str
        +format_reply(reply: str) str
    }
    
    class ResponseFormatter {
        +format_response(response: AgentResponse) str
        +add_context(response: str) str
    }
    
    AgentResponse --> ThoughtExtraction
    AgentResponse --> ReplyExtraction
    AgentResponse --> ResponseFormatter
```

## Performance and Optimization

### Model Selection Logic
```mermaid
graph TD
    START[Agent Initialization]
    
    CHECK_VLLM{vLLM Available?<br/>Linux + GPU}
    CHECK_MLX{MLX Available?<br/>macOS + Metal}
    CHECK_OPENROUTER{OpenRouter<br/>API Key?}
    
    VLLM_CLIENT[vLLM Client<br/>High Performance]
    MLX_CLIENT[MLX Client<br/>Apple Silicon]
    OR_CLIENT[OpenRouter Client<br/>Cloud Fallback]
    ERROR[No Backend Available]
    
    START --> CHECK_VLLM
    CHECK_VLLM -->|Yes| VLLM_CLIENT
    CHECK_VLLM -->|No| CHECK_MLX
    CHECK_MLX -->|Yes| MLX_CLIENT
    CHECK_MLX -->|No| CHECK_OPENROUTER
    CHECK_OPENROUTER -->|Yes| OR_CLIENT
    CHECK_OPENROUTER -->|No| ERROR
```

### Memory and Performance Characteristics

#### Tool Execution Performance
- **File Reading**: O(1) for individual files, cached content
- **Directory Scanning**: O(n) linear with file count
- **Search Operations**: O(n*m) with content size and pattern complexity
- **Code Execution**: Limited by sandbox overhead and timeout

#### Memory Usage Patterns
- **Conversation History**: Grows linearly with turns, managed by max_turns
- **File Content Caching**: LRU cache for frequently accessed files
- **Model Context**: Limited by model context window (typically 4K-32K tokens)
- **Tool Results**: Temporary storage, cleared after response

## Next Steps

For related architecture documentation:
- [MCP Server Architecture](./mcp-server-architecture.md) - MCP protocol implementation
- [Memory System Architecture](./memory-system-architecture.md) - Memory storage and organization
- [Deployment Architecture](./deployment-architecture.md) - Deployment patterns and configuration