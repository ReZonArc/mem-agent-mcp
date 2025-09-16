# Data Flow Architecture

This document describes the data flow patterns, processing pipelines, and information movement throughout the mem-agent-mcp system.

## System Data Flow Overview

```mermaid
flowchart TD
    subgraph "Data Ingestion"
        EXT[External Sources<br/>ChatGPT, GitHub, etc.]
        CONN[Memory Connectors]
        PROC[Data Processing<br/>Categorization, Entity Extraction]
    end
    
    subgraph "Memory Storage"
        MD[Markdown Files<br/>user.md, entities/*.md]
        IDX[Memory Index<br/>Wikilinks, Relationships]
        FLT[Filters<br/>Privacy Controls]
    end
    
    subgraph "Query Processing"
        QRY[User Query]
        MCP[MCP Server]
        AGENT[Memory Agent]
        TOOLS[Tool Execution]
    end
    
    subgraph "Response Generation"
        SEARCH[Memory Search]
        CONTEXT[Context Assembly]
        LLM[LLM Processing]
        RESP[Response Formatting]
    end
    
    %% Data ingestion flow
    EXT --> CONN
    CONN --> PROC
    PROC --> MD
    PROC --> IDX
    
    %% Query processing flow
    QRY --> MCP
    MCP --> AGENT
    AGENT --> TOOLS
    TOOLS --> SEARCH
    
    %% Response generation flow
    SEARCH --> MD
    MD --> CONTEXT
    IDX --> CONTEXT
    FLT --> CONTEXT
    CONTEXT --> LLM
    LLM --> RESP
    RESP --> AGENT
    AGENT --> MCP
    MCP --> QRY
    
    %% Styling
    classDef ingestion fill:#e3f2fd
    classDef storage fill:#f3e5f5  
    classDef query fill:#e8f5e8
    classDef response fill:#fff8e1
    
    class EXT,CONN,PROC ingestion
    class MD,IDX,FLT storage
    class QRY,MCP,AGENT,TOOLS query
    class SEARCH,CONTEXT,LLM,RESP response
```

## Detailed Data Flow Patterns

### 1. Data Ingestion Pipeline

#### Memory Connector Data Flow
```mermaid
sequenceDiagram
    participant User
    participant Connector
    participant Extractor
    participant Organizer
    participant Generator
    participant FileSystem
    
    User->>Connector: Source Path + Config
    Connector->>Extractor: extract_data(source)
    
    alt Export-based (ZIP/JSON)
        Extractor->>Extractor: Parse archive
        Extractor->>Extractor: Extract conversations/pages
    else Live API
        Extractor->>Extractor: API authentication  
        Extractor->>Extractor: Fetch live data
    end
    
    Extractor-->>Organizer: Raw extracted data
    Organizer->>Organizer: Categorize by topics
    Organizer->>Organizer: Extract entities
    Organizer->>Organizer: Build relationships
    Organizer-->>Generator: Organized data
    
    Generator->>FileSystem: Create user.md
    Generator->>FileSystem: Create entities/*.md
    Generator->>FileSystem: Create topic files
    Generator-->>User: Import complete
```

#### ChatGPT History Processing
```mermaid
flowchart TD
    subgraph "Input Processing"
        ZIP[ChatGPT Export ZIP]
        JSON[conversations.json]
        PARSE[JSON Parser]
    end
    
    subgraph "Content Processing" 
        CONV[Individual Conversations]
        FILTER[Content Filtering]
        CLEAN[Text Cleaning]
        TITLE[Title Generation]
    end
    
    subgraph "Categorization"
        KWORD[Keyword Method]
        TFIDF[TF-IDF Clustering]
        LLM_CAT[LLM Categorization]
        TOPICS[Topic Assignment]
    end
    
    subgraph "Entity Extraction"
        ENT_DETECT[Entity Detection]
        REL_MAP[Relationship Mapping]
        CROSS_REF[Cross References]
    end
    
    subgraph "Output Generation"
        USER_MD[user.md Generation]
        ENT_FILES[Entity Files]
        TOPIC_FILES[Topic Files]
        INDEX_GEN[Index Generation]
    end
    
    ZIP --> JSON
    JSON --> PARSE
    PARSE --> CONV
    CONV --> FILTER
    FILTER --> CLEAN
    CLEAN --> TITLE
    
    TITLE --> KWORD
    TITLE --> TFIDF  
    TITLE --> LLM_CAT
    KWORD --> TOPICS
    TFIDF --> TOPICS
    LLM_CAT --> TOPICS
    
    TOPICS --> ENT_DETECT
    ENT_DETECT --> REL_MAP
    REL_MAP --> CROSS_REF
    
    CROSS_REF --> USER_MD
    CROSS_REF --> ENT_FILES
    TOPICS --> TOPIC_FILES
    ENT_FILES --> INDEX_GEN
```

### 2. Query Processing Pipeline

#### MCP Request Processing
```mermaid
sequenceDiagram
    participant Client
    participant MCPServer
    participant Agent
    participant Tools
    participant Memory
    participant ModelBackend
    
    Note over Client,ModelBackend: Query Processing Flow
    
    Client->>MCPServer: MCP Request (read_memory/search)
    MCPServer->>MCPServer: Parse MCP protocol
    MCPServer->>Agent: Initialize with query
    
    Agent->>Agent: Load system prompt
    Agent->>Agent: Add query to conversation
    
    loop Tool Execution Loop (max_tool_turns)
        Agent->>ModelBackend: Generate response
        ModelBackend-->>Agent: Response with tool calls
        
        alt Tool calls present
            Agent->>Tools: Execute tool calls
            Tools->>Memory: File operations
            Memory-->>Tools: Memory content
            Tools-->>Agent: Tool results
            Agent->>Agent: Add to conversation
        else No tool calls
            break
        end
    end
    
    Agent->>Agent: Extract final response
    Agent-->>MCPServer: Formatted response
    MCPServer-->>Client: MCP Response
```

#### Memory Search Data Flow
```mermaid
flowchart TD
    subgraph "Search Input"
        QUERY[Search Query]
        FILTERS[Active Filters]
        PATH[Memory Path]
    end
    
    subgraph "File Discovery"
        SCAN[Directory Scan]
        FILTER_EXT[Filter by Extension]
        BLACKLIST[Apply Blacklist]
        CANDIDATES[Candidate Files]
    end
    
    subgraph "Content Processing"
        READ[Read File Content]
        APPLY_FILTERS[Apply Privacy Filters]
        SEARCH_MATCH[Pattern Matching]
        SCORE[Relevance Scoring]
    end
    
    subgraph "Result Assembly"
        RANK[Rank Results]
        LIMIT[Apply Limits]
        FORMAT[Format Output]
        CONTEXT[Add Context]
    end
    
    QUERY --> SCAN
    FILTERS --> APPLY_FILTERS
    PATH --> SCAN
    
    SCAN --> FILTER_EXT
    FILTER_EXT --> BLACKLIST
    BLACKLIST --> CANDIDATES
    
    CANDIDATES --> READ
    READ --> APPLY_FILTERS
    APPLY_FILTERS --> SEARCH_MATCH
    SEARCH_MATCH --> SCORE
    
    SCORE --> RANK
    RANK --> LIMIT
    LIMIT --> FORMAT
    FORMAT --> CONTEXT
```

### 3. Memory Organization Data Structures

#### Memory File Relationships
```mermaid
erDiagram
    USER_PROFILE {
        string name
        string birth_date
        string location
        array relationships
    }
    
    ENTITY {
        string name
        string type
        string description
        array attributes
        array relationships
    }
    
    TOPIC {
        string name
        string description
        array conversations
        array related_entities
    }
    
    CONVERSATION {
        string id
        string title
        datetime timestamp
        string content
        array participants
    }
    
    WIKILINK {
        string source_file
        string target_file
        string link_text
        string context
    }
    
    USER_PROFILE ||--o{ ENTITY : "has relationships"
    ENTITY ||--o{ ENTITY : "cross-references"
    TOPIC ||--o{ CONVERSATION : "contains"
    ENTITY ||--o{ CONVERSATION : "mentioned in"
    ENTITY ||--o{ WIKILINK : "linked via"
```

#### Memory Storage Format
```mermaid
graph TD
    subgraph "user.md Structure"
        USER_HEADER[# User Information]
        USER_ATTRS[- user_name<br/>- birth_date<br/>- location]
        USER_REL_HEADER[## User Relationships]
        USER_LINKS[- company: [[entities/company.md]]<br/>- friend: [[entities/person.md]]]
    end
    
    subgraph "Entity File Structure"
        ENT_HEADER[# Entity Name]
        ENT_ATTRS[- relationship: type<br/>- attr1: value<br/>- attr2: value]
        ENT_CONTENT[## Content/Description]
        ENT_LINKS[## Related Links<br/>[[entities/other.md]]]
    end
    
    subgraph "Topic File Structure"
        TOPIC_HEADER[# Topic Name]
        TOPIC_DESC[Brief description of topic]
        TOPIC_CONVS[## Conversations<br/>- [[conversations/conv1.md]]<br/>- [[conversations/conv2.md]]]
        TOPIC_ENTITIES[## Related Entities<br/>- [[entities/person1.md]]<br/>- [[entities/project1.md]]]
    end
    
    USER_HEADER --> USER_ATTRS
    USER_ATTRS --> USER_REL_HEADER
    USER_REL_HEADER --> USER_LINKS
    
    ENT_HEADER --> ENT_ATTRS
    ENT_ATTRS --> ENT_CONTENT
    ENT_CONTENT --> ENT_LINKS
    
    TOPIC_HEADER --> TOPIC_DESC
    TOPIC_DESC --> TOPIC_CONVS
    TOPIC_CONVS --> TOPIC_ENTITIES
```

### 4. Response Generation Pipeline

#### Agent Response Processing
```mermaid
sequenceDiagram
    participant Agent
    participant ModelClient
    participant ToolRegistry
    participant ResponseFormatter
    
    Note over Agent,ResponseFormatter: Response Generation Flow
    
    Agent->>ModelClient: Send conversation history
    ModelClient->>ModelClient: Model inference
    ModelClient-->>Agent: Raw model response
    
    Agent->>Agent: Parse response structure
    
    alt Contains tool calls
        Agent->>ToolRegistry: Execute tool calls
        ToolRegistry-->>Agent: Tool results
        Agent->>Agent: Add results to conversation
        Agent->>Agent: Continue conversation loop
    else Final response
        Agent->>ResponseFormatter: Extract thoughts
        Agent->>ResponseFormatter: Extract reply
        ResponseFormatter-->>Agent: Structured response
    end
    
    Agent->>Agent: Format final response
    Agent-->>Agent: Return AgentResponse
```

#### Memory Content Assembly
```mermaid
flowchart TD
    subgraph "Content Sources"
        USER[user.md]
        ENTITIES[entities/*.md]
        TOPICS[topics/*.md]
        CONVS[conversations/*.md]
    end
    
    subgraph "Context Assembly"
        RELEVANT[Relevant Content]
        FILTER_APPLY[Apply Filters]
        CONTEXT_BUILD[Build Context]
        RELATIONSHIP[Add Relationships]
    end
    
    subgraph "Response Integration"
        CONTEXT_INJECT[Inject into Prompt]
        MODEL_PROCESS[Model Processing]
        RESPONSE_GEN[Generate Response]
        FORMAT_OUT[Format Output]
    end
    
    USER --> RELEVANT
    ENTITIES --> RELEVANT
    TOPICS --> RELEVANT
    CONVS --> RELEVANT
    
    RELEVANT --> FILTER_APPLY
    FILTER_APPLY --> CONTEXT_BUILD
    CONTEXT_BUILD --> RELATIONSHIP
    
    RELATIONSHIP --> CONTEXT_INJECT
    CONTEXT_INJECT --> MODEL_PROCESS
    MODEL_PROCESS --> RESPONSE_GEN
    RESPONSE_GEN --> FORMAT_OUT
```

### 5. Error Handling and Data Validation

#### Error Flow Management
```mermaid
flowchart TD
    subgraph "Input Validation"
        VALIDATE[Validate Input]
        SANITIZE[Sanitize Content]
        CHECK_PERMS[Check Permissions]
    end
    
    subgraph "Processing Errors"
        PROC_ERR[Processing Error]
        RETRY[Retry Logic]
        FALLBACK[Fallback Strategy]
        LOG_ERR[Log Error]
    end
    
    subgraph "Output Validation"
        VALIDATE_OUT[Validate Output]
        FILTER_OUT[Filter Content]
        FORMAT_ERR[Format Errors]
    end
    
    VALIDATE --> SANITIZE
    SANITIZE --> CHECK_PERMS
    CHECK_PERMS --> PROC_ERR
    
    PROC_ERR --> RETRY
    RETRY --> FALLBACK
    FALLBACK --> LOG_ERR
    
    LOG_ERR --> VALIDATE_OUT
    VALIDATE_OUT --> FILTER_OUT
    FILTER_OUT --> FORMAT_ERR
```

## Data Consistency and Integrity

### Memory Synchronization
- **Real-time Updates**: Memory files can be modified without server restart
- **Index Regeneration**: Automatic relationship updating on file changes
- **Conflict Resolution**: Last-write-wins for concurrent modifications
- **Backup Strategy**: Original data preservation during connector imports

### Data Validation Rules
- **Wikilink Integrity**: Validate all `[[entity/file.md]]` references
- **Schema Validation**: Ensure proper markdown structure
- **Content Filtering**: Apply privacy filters before processing
- **Relationship Consistency**: Bidirectional relationship validation

## Performance Characteristics

### Throughput Patterns
- **Memory Search**: O(n) file scanning with content filtering
- **Context Assembly**: Limited by model context window
- **Connector Processing**: Batch processing with progress tracking
- **Response Generation**: Model inference latency dominates

### Caching Strategies
- **File Content Caching**: In-memory caching of frequently accessed files
- **Search Result Caching**: Cache search patterns and results
- **Model Response Caching**: Optional response caching for repeated queries

## Next Steps

For implementation details:
- [Agent Architecture](./agent-architecture.md) - Agent processing implementation
- [Memory System Architecture](./memory-system-architecture.md) - Memory storage details
- [API Architecture](./api-architecture.md) - API and protocol specifications