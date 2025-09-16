# Memory System Architecture

This document details the memory storage, organization, and retrieval system that forms the core of the mem-agent-mcp platform.

## Memory System Overview

```mermaid
graph TB
    subgraph "Memory Organization"
        USER[user.md<br/>Central Profile]
        ENTITIES[entities/<br/>Entity Directory]
        TOPICS[topics/<br/>Auto-generated Topics]
        CONVS[conversations/<br/>Imported Conversations]
    end
    
    subgraph "Memory Structure"
        WIKILINKS[Wikilink Navigation<br/>[[entity/file.md]]]
        RELATIONSHIPS[Relationship Mapping<br/>Bidirectional Links]
        METADATA[Metadata Management<br/>Attributes & Properties]
    end
    
    subgraph "Memory Operations"
        READ[Read Operations<br/>File Content Access]
        SEARCH[Search Operations<br/>Pattern Matching]
        INDEX[Index Operations<br/>Relationship Queries]
        FILTER[Filter Operations<br/>Privacy Controls]
    end
    
    subgraph "Storage Layer"
        FILESYSTEM[File System<br/>Markdown Files]
        CACHE[Content Cache<br/>LRU Cache]
        BACKUP[Backup System<br/>Version Control]
    end
    
    USER --> ENTITIES
    ENTITIES --> TOPICS
    ENTITIES --> CONVS
    
    USER --> WIKILINKS
    WIKILINKS --> RELATIONSHIPS
    RELATIONSHIPS --> METADATA
    
    READ --> FILESYSTEM
    SEARCH --> FILESYSTEM
    INDEX --> CACHE
    FILTER --> CACHE
    
    FILESYSTEM --> BACKUP
    
    %% Styling
    classDef organization fill:#e8f5e8
    classDef structure fill:#f3e5f5
    classDef operations fill:#fff3e0
    classDef storage fill:#e1f5fe
    
    class USER,ENTITIES,TOPICS,CONVS organization
    class WIKILINKS,RELATIONSHIPS,METADATA structure
    class READ,SEARCH,INDEX,FILTER operations
    class FILESYSTEM,CACHE,BACKUP storage
```

## Memory Organization Patterns

### Obsidian-Style Structure
```mermaid
graph TD
    subgraph "Root Memory Directory"
        ROOT[memory/mcp-server/]
    end
    
    subgraph "Core Files"
        USER_FILE[user.md<br/>Central Profile]
        INDEX_FILE[index.md<br/>Navigation Hub]
    end
    
    subgraph "Entity Organization"
        ENT_DIR[entities/<br/>Directory]
        PERSON1[person1.md]
        COMPANY1[company1.md] 
        PROJECT1[project1.md]
        CONCEPT1[concept1.md]
    end
    
    subgraph "Generated Content"
        TOPIC_DIR[topics/<br/>Auto-generated]
        TOPIC1[ai-research.md]
        TOPIC2[programming.md]
        TOPIC3[product-strategy.md]
    end
    
    subgraph "Import Data"
        CONV_DIR[conversations/<br/>Imported Data]
        CONV1[conv_001.md]
        CONV2[conv_002.md]
        GITHUB_DIR[github/<br/>Repository Data]
    end
    
    ROOT --> USER_FILE
    ROOT --> INDEX_FILE
    ROOT --> ENT_DIR
    ROOT --> TOPIC_DIR
    ROOT --> CONV_DIR
    
    ENT_DIR --> PERSON1
    ENT_DIR --> COMPANY1
    ENT_DIR --> PROJECT1
    ENT_DIR --> CONCEPT1
    
    TOPIC_DIR --> TOPIC1
    TOPIC_DIR --> TOPIC2
    TOPIC_DIR --> TOPIC3
    
    CONV_DIR --> CONV1
    CONV_DIR --> CONV2
    CONV_DIR --> GITHUB_DIR
```

### File Structure Specifications

#### User Profile Structure (`user.md`)
```mermaid
classDiagram
    class UserProfile {
        +string user_name
        +date birth_date
        +string birth_location
        +string living_location
        +string zodiac_sign
        +list relationships
        +generate_profile()
        +update_relationships()
    }
    
    class Relationship {
        +string type
        +string entity_link
        +string description
        +validate_link()
    }
    
    class WikiLink {
        +string target_path
        +string display_text
        +bool is_valid
        +resolve_target()
    }
    
    UserProfile --> Relationship
    Relationship --> WikiLink
```

**Example user.md Structure:**
```markdown
# User Information
- user_name: John Doe
- birth_date: 1990-01-01
- birth_location: New York, USA
- living_location: San Francisco, CA
- zodiac_sign: Capricorn

## User Relationships
- company: [[entities/acme_corp.md]]
- manager: [[entities/jane_smith.md]]
- project: [[entities/ai_project.md]]
- hobby: [[entities/photography.md]]

## Recent Activity
- Last conversation: 2024-01-15
- Active projects: 3
- Memory entries: 47
```

#### Entity File Structure
```mermaid
classDiagram
    class EntityFile {
        +string name
        +string entity_type
        +dict attributes
        +list relationships
        +string content
        +list backlinks
        +generate_entity()
        +update_backlinks()
    }
    
    class EntityType {
        <<enumeration>>
        PERSON
        COMPANY
        PROJECT
        CONCEPT
        EVENT
        LOCATION
    }
    
    class Attribute {
        +string key
        +string value
        +string type
        +validate()
    }
    
    EntityFile --> EntityType
    EntityFile --> Attribute
```

**Example Entity File Structure:**
```markdown
# Jane Smith
- relationship: Manager
- department: Engineering
- start_date: 2019-03-15
- email: jane.smith@company.com
- skills: ["Python", "Machine Learning", "Team Leadership"]

## Background
Jane has been leading the AI research team since 2019. She has extensive experience in machine learning and has published several papers on neural networks.

## Projects
- [[entities/ai_project.md]] - Current project lead
- [[entities/research_initiative.md]] - Technical advisor

## Related Conversations
- [[conversations/weekly_standup_2024_01_15.md]]
- [[conversations/project_planning_2024_01_10.md]]
```

## Wikilink System Architecture

### Link Resolution System
```mermaid
sequenceDiagram
    participant Agent
    participant LinkResolver
    participant FileSystem
    participant Cache
    
    Agent->>LinkResolver: Resolve [[entities/person.md]]
    LinkResolver->>LinkResolver: Parse wikilink syntax
    LinkResolver->>Cache: Check link cache
    
    alt Cache hit
        Cache-->>LinkResolver: Cached link data
    else Cache miss
        LinkResolver->>FileSystem: Validate target file exists
        FileSystem-->>LinkResolver: File existence status
        LinkResolver->>FileSystem: Read target file
        FileSystem-->>LinkResolver: File content
        LinkResolver->>Cache: Update cache
    end
    
    LinkResolver-->>Agent: Resolved link with content
```

### Bidirectional Link Management
```mermaid
graph TB
    subgraph "Link Creation"
        DETECT[Detect New Wikilink<br/>[[entities/file.md]]]
        VALIDATE[Validate Target Exists<br/>File system check]
        CREATE[Create Forward Link<br/>Source → Target]
    end
    
    subgraph "Backlink Generation"
        SCAN[Scan All Files<br/>Find references]
        EXTRACT[Extract Wikilinks<br/>Parse markdown]
        BACKLINK[Generate Backlinks<br/>Target ← Sources]
    end
    
    subgraph "Link Maintenance"
        VALIDATE_LINKS[Validate All Links<br/>Check file existence]
        UPDATE_INDEX[Update Link Index<br/>Relationship mapping]
        REPAIR[Repair Broken Links<br/>Suggest alternatives]
    end
    
    DETECT --> VALIDATE
    VALIDATE --> CREATE
    CREATE --> SCAN
    
    SCAN --> EXTRACT
    EXTRACT --> BACKLINK
    BACKLINK --> VALIDATE_LINKS
    
    VALIDATE_LINKS --> UPDATE_INDEX
    UPDATE_INDEX --> REPAIR
```

### Link Types and Semantics
```mermaid
classDiagram
    class WikiLink {
        +string source_file
        +string target_file
        +string link_text
        +LinkType type
        +string context
    }
    
    class LinkType {
        <<enumeration>>
        ENTITY_REFERENCE
        CONVERSATION_LINK
        TOPIC_REFERENCE
        CROSS_REFERENCE
        PARENT_CHILD
        RELATED_TO
    }
    
    class LinkContext {
        +string surrounding_text
        +int line_number
        +string section_header
        +LinkSemantic semantic
    }
    
    class LinkSemantic {
        <<enumeration>>
        RELATIONSHIP
        MENTION
        DETAILED_REFERENCE
        NAVIGATION
        CATEGORIZATION
    }
    
    WikiLink --> LinkType
    WikiLink --> LinkContext
    LinkContext --> LinkSemantic
```

## Memory Search Architecture

### Search Engine Implementation
```mermaid
graph TB
    subgraph "Search Input Processing"
        QUERY[User Search Query]
        TOKENIZE[Tokenize Query<br/>Split terms, operators]
        PARSE[Parse Search Syntax<br/>Filters, modifiers]
    end
    
    subgraph "File Discovery"
        SCAN[Directory Scan<br/>Recursive traversal]
        FILTER_TYPE[Filter by Type<br/>.md files only]
        BLACKLIST[Apply Blacklist<br/>Exclude patterns]
    end
    
    subgraph "Content Search"
        CONTENT[Read File Content<br/>Parallel processing]
        MATCH[Pattern Matching<br/>Regex, fuzzy, exact]
        CONTEXT[Extract Context<br/>Surrounding lines]
        SCORE[Relevance Scoring<br/>TF-IDF, position]
    end
    
    subgraph "Result Processing"
        RANK[Rank Results<br/>Score-based ordering]
        PRIVACY[Apply Privacy Filters<br/>Content filtering]
        LIMIT[Apply Result Limits<br/>Max results, pagination]
        FORMAT[Format Output<br/>Structured response]
    end
    
    QUERY --> TOKENIZE
    TOKENIZE --> PARSE
    PARSE --> SCAN
    
    SCAN --> FILTER_TYPE
    FILTER_TYPE --> BLACKLIST
    BLACKLIST --> CONTENT
    
    CONTENT --> MATCH
    MATCH --> CONTEXT
    CONTEXT --> SCORE
    
    SCORE --> RANK
    RANK --> PRIVACY
    PRIVACY --> LIMIT
    LIMIT --> FORMAT
```

### Search Query Types
```mermaid
classDiagram
    class SearchQuery {
        +string raw_query
        +QueryType type
        +list filters
        +SearchOptions options
        +parse_query()
        +execute()
    }
    
    class QueryType {
        <<enumeration>>
        EXACT_MATCH
        FUZZY_SEARCH
        REGEX_PATTERN
        SEMANTIC_SEARCH
        ENTITY_LOOKUP
    }
    
    class SearchFilter {
        +string field
        +string operator
        +string value
        +apply_filter()
    }
    
    class SearchOptions {
        +int max_results
        +bool case_sensitive
        +bool include_content
        +int context_lines
        +list file_patterns
    }
    
    SearchQuery --> QueryType
    SearchQuery --> SearchFilter
    SearchQuery --> SearchOptions
```

**Search Query Examples:**
```bash
# Exact phrase search
"machine learning project"

# Entity-specific search
entity:person manager

# File type filtering
type:conversation topic:ai

# Regex pattern search
pattern:"\b\d{4}-\d{2}-\d{2}\b"  # Date patterns

# Combined filters
"project planning" entity:company created:2024-01
```

### Search Result Ranking
```mermaid
graph TD
    subgraph "Ranking Factors"
        RELEVANCE[Content Relevance<br/>TF-IDF scoring]
        RECENCY[File Recency<br/>Last modified date]
        ENTITY[Entity Importance<br/>Link count, centrality]
        CONTEXT[Context Quality<br/>Complete sentences, headers]
    end
    
    subgraph "Scoring Algorithm"
        WEIGHTED[Weighted Combination<br/>Factor × Weight]
        NORMALIZE[Score Normalization<br/>0.0 - 1.0 range]
        BOOST[Boost Factors<br/>Title match, exact phrase]
    end
    
    subgraph "Final Ranking"
        SORT[Sort by Score<br/>Descending order]
        GROUP[Group by Type<br/>Entity, conversation, topic]
        DIVERSITY[Promote Diversity<br/>Avoid duplicate sources]
    end
    
    RELEVANCE --> WEIGHTED
    RECENCY --> WEIGHTED  
    ENTITY --> WEIGHTED
    CONTEXT --> WEIGHTED
    
    WEIGHTED --> NORMALIZE
    NORMALIZE --> BOOST
    BOOST --> SORT
    
    SORT --> GROUP
    GROUP --> DIVERSITY
```

## Privacy and Filtering System

### Content Filter Architecture
```mermaid
graph TB
    subgraph "Filter Configuration"
        FILTER_FILE[.filters File<br/>User-defined rules]
        DEFAULT[Default Filters<br/>Built-in patterns]
        DYNAMIC[Dynamic Filters<br/>Context-sensitive]
    end
    
    subgraph "Filter Types"
        CONTENT_FILTER[Content Filters<br/>Text patterns, keywords]
        ENTITY_FILTER[Entity Filters<br/>Specific entities]
        SECTION_FILTER[Section Filters<br/>Markdown sections]
        METADATA_FILTER[Metadata Filters<br/>File attributes]
    end
    
    subgraph "Filter Application"
        PRE_PROCESS[Pre-processing<br/>Before search/read]
        POST_PROCESS[Post-processing<br/>Before response]
        REDACTION[Content Redaction<br/>Replace with placeholders]
    end
    
    FILTER_FILE --> CONTENT_FILTER
    DEFAULT --> ENTITY_FILTER
    DYNAMIC --> SECTION_FILTER
    
    CONTENT_FILTER --> PRE_PROCESS
    ENTITY_FILTER --> POST_PROCESS
    SECTION_FILTER --> REDACTION
    METADATA_FILTER --> REDACTION
```

### Filter Rule System
```mermaid
classDiagram
    class FilterRule {
        +string pattern
        +FilterType type
        +FilterAction action
        +int priority
        +bool enabled
        +match(content)
        +apply(content)
    }
    
    class FilterType {
        <<enumeration>>
        REGEX_PATTERN
        EXACT_MATCH
        ENTITY_NAME
        EMAIL_ADDRESS
        PHONE_NUMBER
        DATE_PATTERN
    }
    
    class FilterAction {
        <<enumeration>>
        REMOVE_CONTENT
        REDACT_WITH_PLACEHOLDER
        SKIP_FILE
        TRUNCATE_SECTION
        TRANSFORM_CONTENT
    }
    
    class FilterEngine {
        +list~FilterRule~ rules
        +load_filters()
        +apply_filters(content)
        +get_active_filters()
    }
    
    FilterRule --> FilterType
    FilterRule --> FilterAction
    FilterEngine --> FilterRule
```

**Example Filter Configuration:**
```yaml
# Content privacy filters
filters:
  - pattern: "\b\d{3}-\d{2}-\d{4}\b"  # SSN
    type: regex_pattern
    action: redact_with_placeholder
    placeholder: "[SSN]"
    
  - pattern: "email"
    type: exact_match
    action: remove_content
    
  - entity: "confidential_project"
    type: entity_name
    action: skip_file
    
  - section: "## Private Notes"
    type: section_filter
    action: truncate_section
```

## Memory Connector Integration

### Data Import Pipeline
```mermaid
sequenceDiagram
    participant Connector
    participant Extractor
    participant Organizer
    participant MemorySystem
    participant FileSystem
    
    Note over Connector,FileSystem: Data Import Process
    
    Connector->>Extractor: Extract source data
    Extractor->>Extractor: Parse format (ZIP, JSON, API)
    Extractor-->>Organizer: Raw extracted data
    
    Organizer->>Organizer: Categorize by topics
    Organizer->>Organizer: Extract entities
    Organizer->>Organizer: Build relationships
    Organizer-->>MemorySystem: Organized data
    
    MemorySystem->>MemorySystem: Generate user.md updates
    MemorySystem->>MemorySystem: Create entity files
    MemorySystem->>FileSystem: Write memory files
    MemorySystem->>MemorySystem: Update wikilinks
    MemorySystem->>MemorySystem: Build index
    
    MemorySystem-->>Connector: Import complete
```

### Entity Extraction and Relationship Mapping
```mermaid
graph TB
    subgraph "Data Processing"
        RAW[Raw Import Data<br/>Conversations, Documents]
        NLP[NLP Processing<br/>Entity recognition]
        EXTRACT[Entity Extraction<br/>People, companies, concepts]
    end
    
    subgraph "Entity Classification"
        CLASSIFY[Entity Classification<br/>Type determination]
        DEDUPE[Deduplication<br/>Merge similar entities]
        VALIDATE[Validation<br/>Confirm accuracy]
    end
    
    subgraph "Relationship Mapping"
        CONNECT[Connection Discovery<br/>Co-occurrence analysis]
        WEIGHT[Relationship Weighting<br/>Strength calculation]
        HIERARCHY[Hierarchy Building<br/>Parent-child relationships]
    end
    
    subgraph "Memory Integration"
        FILES[Generate Entity Files<br/>Markdown creation]
        LINKS[Create Wikilinks<br/>Navigation structure]
        INDEX[Update Index<br/>Master relationship map]
    end
    
    RAW --> NLP
    NLP --> EXTRACT
    EXTRACT --> CLASSIFY
    
    CLASSIFY --> DEDUPE
    DEDUPE --> VALIDATE
    VALIDATE --> CONNECT
    
    CONNECT --> WEIGHT
    WEIGHT --> HIERARCHY
    HIERARCHY --> FILES
    
    FILES --> LINKS
    LINKS --> INDEX
```

## Performance and Optimization

### Caching Strategy
```mermaid
graph TB
    subgraph "Cache Layers"
        FILE_CACHE[File Content Cache<br/>LRU cache, 100MB limit]
        SEARCH_CACHE[Search Result Cache<br/>Query → Results mapping]
        LINK_CACHE[Link Resolution Cache<br/>Wikilink → Target mapping]
        METADATA_CACHE[File Metadata Cache<br/>Stats, modification times]
    end
    
    subgraph "Cache Management"
        EXPIRY[Cache Expiry<br/>Time-based invalidation]
        INVALIDATE[Cache Invalidation<br/>File modification events]
        PRELOAD[Cache Preloading<br/>Common access patterns]
    end
    
    subgraph "Cache Storage"
        MEMORY[In-Memory<br/>Python dictionaries]
        PERSISTENT[Persistent Cache<br/>Optional disk storage]
    end
    
    FILE_CACHE --> EXPIRY
    SEARCH_CACHE --> EXPIRY
    LINK_CACHE --> INVALIDATE
    METADATA_CACHE --> PRELOAD
    
    EXPIRY --> MEMORY
    INVALIDATE --> MEMORY
    PRELOAD --> PERSISTENT
```

### File System Optimization
```mermaid
graph TD
    subgraph "Directory Organization"
        FLAT[Flat Structure<br/>Entities in single directory]
        HIERARCHICAL[Hierarchical Structure<br/>Type-based subdirectories]
        HYBRID[Hybrid Structure<br/>Generated + manual organization]
    end
    
    subgraph "File Management"
        NAMING[Consistent Naming<br/>slug-based filenames]
        METADATA[File Metadata<br/>YAML frontmatter]
        VERSIONING[Version Control<br/>Git integration optional]
    end
    
    subgraph "Performance Considerations"
        SIZE[File Size Limits<br/>Large files handling]
        COUNT[File Count Limits<br/>Directory performance]
        INDEXING[File Indexing<br/>Pre-built search indexes]
    end
    
    FLAT --> NAMING
    HIERARCHICAL --> METADATA
    HYBRID --> VERSIONING
    
    NAMING --> SIZE
    METADATA --> COUNT
    VERSIONING --> INDEXING
```

## Data Consistency and Integrity

### Consistency Guarantees
```mermaid
graph TB
    subgraph "Consistency Models"
        EVENTUAL[Eventual Consistency<br/>File system updates]
        IMMEDIATE[Immediate Consistency<br/>In-memory operations]
        TRANSACTIONAL[Transactional Updates<br/>Atomic operations]
    end
    
    subgraph "Integrity Checks"
        LINK_VALID[Link Validation<br/>Wikilink target verification]
        FILE_VALID[File Validation<br/>Markdown syntax checking]
        SCHEMA_VALID[Schema Validation<br/>Entity structure verification]
    end
    
    subgraph "Conflict Resolution"
        LAST_WRITE[Last Write Wins<br/>File modification conflicts]
        MERGE[Content Merging<br/>Non-conflicting changes]
        BACKUP[Backup Strategy<br/>Recovery mechanisms]
    end
    
    EVENTUAL --> LINK_VALID
    IMMEDIATE --> FILE_VALID
    TRANSACTIONAL --> SCHEMA_VALID
    
    LINK_VALID --> LAST_WRITE
    FILE_VALID --> MERGE
    SCHEMA_VALID --> BACKUP
```

### Backup and Recovery
```mermaid
sequenceDiagram
    participant User
    participant MemorySystem
    participant FileSystem
    participant BackupSystem
    
    Note over User,BackupSystem: Backup Strategy
    
    User->>MemorySystem: Import new data
    MemorySystem->>FileSystem: Write memory files
    MemorySystem->>BackupSystem: Create backup
    BackupSystem->>BackupSystem: Archive existing files
    
    Note over User,BackupSystem: Recovery Process
    
    User->>BackupSystem: Request recovery
    BackupSystem->>BackupSystem: List available backups
    BackupSystem-->>User: Backup selection
    User->>BackupSystem: Select backup
    BackupSystem->>FileSystem: Restore files
    FileSystem-->>MemorySystem: Notify restoration
    MemorySystem->>MemorySystem: Rebuild indexes
```

## Next Steps

For related architecture documentation:
- [Agent Architecture](./agent-architecture.md) - Agent processing and tools
- [Memory Connectors Architecture](./memory-connectors-architecture.md) - Data import systems
- [Data Flow Architecture](./data-flow-architecture.md) - Information processing flows
- [API Architecture](./api-architecture.md) - External integration patterns