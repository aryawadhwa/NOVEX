# Design Document: Novex

## Introduction

Novex reimagines collaborative software development by creating a "collective brain" for development teams. Unlike traditional version control systems that operate on a commit-push-pull model, Novex enables real-time, collision-free collaboration where multiple developers can work simultaneously on the same codebase without creating conflicts. When conflicts do arise, AI automatically resolves 70-85% of them by understanding code semantics, not just text differences.

Built with a local-first philosophy, Novex runs entirely on developer laptops without requiring constant internet connectivity. AWS services enhance the experience with cloud backup, relay servers for remote teams, and powerful AI models for complex conflict resolution - but the core functionality remains fully operational offline. This makes Novex ideal for distributed teams across India, where connectivity can be variable.

The system targets 2-20 person development teams working with Python, JavaScript/TypeScript, Rust, and Go, running efficiently on typical developer hardware (Intel i5/Ryzen 5, 8-16 GB RAM).

## Overview

Novex is a lightweight, local-first collaborative coding memory hub built with Rust and Tauri 2.x. The system combines CRDT-based synchronization, P2P networking, AI-powered conflict resolution, and optional AWS cloud services to create a seamless team collaboration experience that goes beyond traditional version control.

The architecture prioritizes local-first operation with cloud enhancement, ensuring the application remains functional offline while providing optional cloud backup, relay services, and enhanced AI capabilities when connectivity is available.

### Design Philosophy

1. **Local-First**: All core functionality works offline; cloud is optional enhancement
2. **Lightweight**: Strict resource constraints for typical developer laptops
3. **Real-Time**: Sub-second synchronization and conflict detection
4. **Intelligent**: AI-powered conflict resolution with 70-85% automation rate
5. **Secure**: End-to-end encryption for all team communication

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Tauri Desktop App                        │
├─────────────────────────────────────────────────────────────┤
│  Frontend (React 19/Svelte 5)                               │
│  ├─ Code Editor View                                        │
│  ├─ Orbital 3D Visualization (Three.js)                     │
│  ├─ Activity Feed & Presence                                │
│  └─ AI Chat Interface                                       │
├─────────────────────────────────────────────────────────────┤
│  Rust Backend                                               │
│  ├─ CRDT Engine (Automerge 2.0)                            │
│  ├─ Sync Engine (libp2p-rs)                                │
│  ├─ Conflict Resolver                                       │
│  ├─ Tree-sitter Parser                                      │
│  ├─ AI Agent (Ollama + Phi-3 Mini)                         │
│  ├─ Memory Box (Knowledge Graph)                           │
│  └─ SQLite + CRDT Persistence                              │
├─────────────────────────────────────────────────────────────┤
│  Optional AWS Integration Layer                             │
│  ├─ S3 Backup Client                                        │
│  ├─ DynamoDB Metadata Client                               │
│  ├─ Bedrock/SageMaker AI Client                           │
│  ├─ Cognito Auth Client                                    │
│  └─ CloudWatch Telemetry Client                            │
└─────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
    Local Peers          Relay Server         AWS Services
   (P2P libp2p)      (Lambda/EC2)         (S3/DynamoDB/etc)
```

### Component Architecture

#### 1. CRDT Engine

**Technology**: Automerge 2.0 (Rust bindings)

**Responsibilities**:
- Maintain conflict-free replicated data structures for code and metadata
- Generate and apply CRDT operations for local edits
- Merge remote operations maintaining causal consistency
- Provide transaction boundaries for atomic changes

**Data Structures**:
- `AutomergeDoc`: Root CRDT document containing project state
- `FileMap`: CRDT Map of file paths to file content
- `FileContent`: CRDT Text for each file's content
- `MetadataMap`: CRDT Map for comments, annotations, decisions
- `PresenceMap`: CRDT Map for real-time team member presence

**Key Operations**:
- `create_transaction()`: Start a new local transaction
- `apply_local_change(change)`: Apply local edit to CRDT
- `merge_remote_changes(changes)`: Merge incoming peer changes
- `get_snapshot()`: Get current state snapshot for persistence

#### 2. Sync Engine

**Technology**: libp2p-rs with custom protocols

**Responsibilities**:
- Discover peers on LAN (mDNS) and WAN (relay servers)
- Establish encrypted P2P connections using libsodium
- Broadcast CRDT operations to connected peers
- Handle NAT traversal and connection resilience
- Manage presence and heartbeat messages

**Network Protocols**:
- `/novex/sync/1.0.0`: CRDT operation synchronization
- `/novex/presence/1.0.0`: Real-time presence updates
- `/novex/discovery/1.0.0`: Team member discovery

**Connection Strategy**:
1. Attempt direct LAN connection via mDNS
2. If LAN fails, attempt hole-punching for WAN
3. If hole-punching fails, use relay server (Lambda/EC2)
4. If all P2P fails, fall back to AWS sync (S3 + DynamoDB)

**Bandwidth Optimization**:
- Delta compression for CRDT operations
- Batching of small operations (max 100ms window)
- Presence updates throttled to 1 per second per peer
- Binary encoding using MessagePack

#### 3. Tree-sitter Parser

**Technology**: tree-sitter with Rust bindings

**Responsibilities**:
- Parse source code into abstract syntax trees
- Detect syntax errors and structural issues
- Provide semantic information for conflict detection
- Support incremental parsing for performance

**Supported Languages**:
- Python (tree-sitter-python)
- JavaScript/TypeScript (tree-sitter-javascript, tree-sitter-typescript)
- Rust (tree-sitter-rust)
- Go (tree-sitter-go)

**Parser Pipeline**:
```
Source Code → Tree-sitter → AST → Semantic Analyzer → Conflict Detector
```

**Key Functions**:
- `parse_file(path, content)`: Parse file and return AST
- `incremental_parse(old_tree, edits)`: Update AST incrementally
- `extract_symbols(ast)`: Extract functions, classes, variables
- `detect_syntax_errors(ast)`: Find syntax issues

#### 4. Conflict Resolver

**Responsibilities**:
- Detect compile-time, type, and semantic conflicts
- Classify conflicts by severity and type
- Coordinate with AI Agent for resolution generation
- Apply automatic resolutions or present to user

**Conflict Detection Pipeline**:
```
CRDT Merge → Tree-sitter Parse → Syntax Check → Type Check → Semantic Analysis → Conflict Classification
```

**Conflict Types**:
1. **Compile-time**: Syntax errors, missing imports, undefined symbols
2. **Type-level**: Type mismatches, incompatible signatures
3. **Semantic**: Logic conflicts, race conditions, invariant violations

**Resolution Strategy**:
```rust
match conflict.confidence_score {
    0.85..=1.0 => auto_apply_resolution(),
    0.70..=0.85 => suggest_resolution_for_approval(),
    0.0..=0.70 => present_conflict_with_ai_context(),
}
```

#### 5. AI Agent

**Technology**: Ollama with Phi-3 Mini (3.8B parameters)

**Responsibilities**:
- Analyze conflict context and generate resolutions
- Provide AI-enhanced annotations and comments
- Facilitate negotiation dialogues for complex conflicts
- Generate embeddings for research document search

**Model Configuration**:
- Local inference using Ollama runtime
- Context window: 4096 tokens
- Temperature: 0.3 for conflict resolution (deterministic)
- Temperature: 0.7 for negotiation chat (creative)

**Prompt Templates**:
- Conflict resolution: "Given this code conflict... suggest a resolution"
- Annotation enhancement: "Explain this code section in context"
- Negotiation: "Present pros and cons for these approaches"

**AWS Enhancement**:
- Optional fallback to AWS Bedrock (Claude 3 Sonnet) for complex conflicts
- Fine-tuned models on SageMaker for team-specific patterns
- Graceful degradation to local model if AWS unavailable

#### 6. Memory Box (Knowledge Graph)

**Data Model**:
```rust
struct MemoryBox {
    code_regions: HashMap<CodeRegionId, CodeRegion>,
    comments: HashMap<CommentId, Comment>,
    decisions: HashMap<DecisionId, Decision>,
    research_docs: HashMap<DocId, ResearchDoc>,
    associations: Vec<Association>,
}

struct Association {
    source: EntityId,
    target: EntityId,
    relation_type: RelationType,
    metadata: HashMap<String, Value>,
}
```

**CRDT Synchronization**:
- All Memory Box entities stored in CRDT Map
- Associations stored as CRDT Set
- Full-text search index rebuilt locally from CRDT state

**Search Capabilities**:
- Full-text search using Tantivy (Rust search library)
- Semantic search using local embeddings (sentence-transformers)
- Graph traversal for related entities

#### 7. SQLite Persistence Layer

**Schema Design**:
```sql
-- CRDT state snapshots
CREATE TABLE crdt_snapshots (
    id INTEGER PRIMARY KEY,
    project_id TEXT NOT NULL,
    snapshot_data BLOB NOT NULL,
    created_at INTEGER NOT NULL
);

-- Incremental CRDT operations
CREATE TABLE crdt_operations (
    id INTEGER PRIMARY KEY,
    project_id TEXT NOT NULL,
    operation_data BLOB NOT NULL,
    actor_id TEXT NOT NULL,
    sequence_num INTEGER NOT NULL,
    created_at INTEGER NOT NULL
);

-- Project metadata
CREATE TABLE projects (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    root_path TEXT NOT NULL,
    languages TEXT NOT NULL,
    created_at INTEGER NOT NULL,
    last_synced_at INTEGER
);

-- Team members and presence
CREATE TABLE team_members (
    id TEXT PRIMARY KEY,
    project_id TEXT NOT NULL,
    display_name TEXT NOT NULL,
    public_key TEXT NOT NULL,
    last_seen_at INTEGER,
    FOREIGN KEY (project_id) REFERENCES projects(id)
);

-- Conflict history
CREATE TABLE conflicts (
    id TEXT PRIMARY KEY,
    project_id TEXT NOT NULL,
    conflict_type TEXT NOT NULL,
    file_path TEXT NOT NULL,
    resolution_status TEXT NOT NULL,
    confidence_score REAL,
    created_at INTEGER NOT NULL,
    resolved_at INTEGER,
    FOREIGN KEY (project_id) REFERENCES projects(id)
);
```

**Persistence Strategy**:
- Write-ahead logging for durability
- Periodic snapshots every 1000 operations
- Incremental backups to prevent data loss
- Encryption at rest using SQLCipher

## Components and Interfaces

### Frontend-Backend Interface (Tauri Commands)

```rust
// Project management
#[tauri::command]
async fn init_project(path: String) -> Result<ProjectInfo, Error>;

#[tauri::command]
async fn get_project_state(project_id: String) -> Result<ProjectState, Error>;

// File operations
#[tauri::command]
async fn get_file_content(project_id: String, path: String) -> Result<String, Error>;

#[tauri::command]
async fn apply_edit(project_id: String, path: String, edit: Edit) -> Result<(), Error>;

// Sync and presence
#[tauri::command]
async fn get_connected_peers(project_id: String) -> Result<Vec<PeerInfo>, Error>;

#[tauri::command]
async fn get_presence_info(project_id: String) -> Result<Vec<PresenceInfo>, Error>;

// Conflict management
#[tauri::command]
async fn get_conflicts(project_id: String) -> Result<Vec<Conflict>, Error>;

#[tauri::command]
async fn resolve_conflict(conflict_id: String, resolution: Resolution) -> Result<(), Error>;

// AI interactions
#[tauri::command]
async fn start_negotiation_chat(conflict_id: String) -> Result<ChatSession, Error>;

#[tauri::command]
async fn send_chat_message(session_id: String, message: String) -> Result<String, Error>;

// Memory Box
#[tauri::command]
async fn add_annotation(project_id: String, annotation: Annotation) -> Result<(), Error>;

#[tauri::command]
async fn search_knowledge(project_id: String, query: String) -> Result<Vec<SearchResult>, Error>;

#[tauri::command]
async fn import_research_doc(project_id: String, pdf_path: String) -> Result<DocId, Error>;

// AWS integration
#[tauri::command]
async fn enable_aws_backup(project_id: String, config: AwsConfig) -> Result<(), Error>;

#[tauri::command]
async fn sync_to_cloud(project_id: String) -> Result<(), Error>;
```

### P2P Protocol Messages

```rust
// Sync protocol messages
enum SyncMessage {
    // Initial handshake
    Hello {
        peer_id: PeerId,
        project_id: ProjectId,
        protocol_version: String,
    },
    
    // CRDT synchronization
    SyncRequest {
        last_known_heads: Vec<ChangeHash>,
    },
    SyncResponse {
        missing_changes: Vec<Change>,
    },
    
    // Real-time operations
    Operation {
        change: Change,
        actor: ActorId,
        sequence: u64,
    },
    
    // Presence updates
    PresenceUpdate {
        actor: ActorId,
        file_path: Option<String>,
        cursor_position: Option<Position>,
        selection: Option<Range>,
    },
}
```

### AWS Integration Interfaces

```rust
// S3 backup client
trait BackupClient {
    async fn upload_snapshot(&self, project_id: &str, data: &[u8]) -> Result<String>;
    async fn download_snapshot(&self, project_id: &str, snapshot_id: &str) -> Result<Vec<u8>>;
    async fn list_snapshots(&self, project_id: &str) -> Result<Vec<SnapshotInfo>>;
}

// DynamoDB metadata client
trait MetadataClient {
    async fn store_project_metadata(&self, metadata: &ProjectMetadata) -> Result<()>;
    async fn get_project_metadata(&self, project_id: &str) -> Result<ProjectMetadata>;
    async fn update_sync_state(&self, project_id: &str, state: &SyncState) -> Result<()>;
}

// Bedrock AI client
trait CloudAIClient {
    async fn resolve_conflict(&self, context: &ConflictContext) -> Result<Resolution>;
    async fn enhance_annotation(&self, code: &str, annotation: &str) -> Result<String>;
}

// Cognito auth client
trait AuthClient {
    async fn authenticate(&self, credentials: &Credentials) -> Result<AuthToken>;
    async fn verify_team_member(&self, token: &AuthToken, team_id: &str) -> Result<bool>;
    async fn get_team_members(&self, team_id: &str) -> Result<Vec<TeamMember>>;
}

// CloudWatch telemetry client
trait TelemetryClient {
    async fn send_metrics(&self, metrics: &[Metric]) -> Result<()>;
    async fn log_event(&self, event: &Event) -> Result<()>;
}
```

## Data Models

### Core Data Structures

```rust
// Project representation
struct Project {
    id: ProjectId,
    name: String,
    root_path: PathBuf,
    languages: Vec<Language>,
    crdt_doc: AutomergeDoc,
    team_members: Vec<TeamMember>,
    encryption_key: EncryptionKey,
}

// File content with CRDT
struct FileState {
    path: String,
    content: CRDTText,
    ast: Option<Tree>,
    last_modified: Timestamp,
    last_modified_by: ActorId,
}

// Conflict representation
struct Conflict {
    id: ConflictId,
    conflict_type: ConflictType,
    file_path: String,
    region: CodeRegion,
    local_version: String,
    remote_version: String,
    ai_resolution: Option<Resolution>,
    confidence_score: f32,
    status: ConflictStatus,
}

enum ConflictType {
    CompileTime { error: String },
    TypeLevel { type_error: String },
    Semantic { description: String },
}

enum ConflictStatus {
    Detected,
    Analyzing,
    ResolutionProposed { resolution: Resolution },
    AutoResolved,
    ManuallyResolved,
}

// Resolution proposal
struct Resolution {
    patch: String,
    explanation: String,
    confidence: f32,
    alternative_approaches: Vec<String>,
}

// Team member presence
struct PresenceInfo {
    actor_id: ActorId,
    display_name: String,
    current_file: Option<String>,
    cursor_position: Option<Position>,
    selection: Option<Range>,
    last_activity: Timestamp,
    status: PresenceStatus,
}

enum PresenceStatus {
    Active,
    Idle,
    Offline,
}

// Memory Box entities
struct Annotation {
    id: AnnotationId,
    code_region: CodeRegion,
    author: ActorId,
    content: String,
    ai_context: Option<String>,
    created_at: Timestamp,
    thread: Vec<Comment>,
}

struct Decision {
    id: DecisionId,
    title: String,
    description: String,
    rationale: String,
    related_code: Vec<CodeRegion>,
    related_docs: Vec<DocId>,
    decided_by: ActorId,
    decided_at: Timestamp,
}

struct ResearchDoc {
    id: DocId,
    title: String,
    authors: Vec<String>,
    file_path: PathBuf,
    extracted_text: String,
    embeddings: Vec<f32>,
    related_code: Vec<CodeRegion>,
}

// Code region reference
struct CodeRegion {
    file_path: String,
    start_line: u32,
    end_line: u32,
    start_col: u32,
    end_col: u32,
    content_hash: String, // For tracking across refactors
}
```

### AWS Data Models

```rust
// S3 snapshot format
struct CloudSnapshot {
    project_id: ProjectId,
    snapshot_id: String,
    crdt_data: Vec<u8>, // Encrypted CRDT state
    metadata: SnapshotMetadata,
    created_at: Timestamp,
}

// DynamoDB project metadata
struct ProjectMetadata {
    project_id: ProjectId,
    team_id: TeamId,
    team_members: Vec<TeamMemberId>,
    last_sync_timestamp: Timestamp,
    sync_heads: Vec<ChangeHash>,
    encryption_key_id: String,
}

// CloudWatch metrics
struct UsageMetrics {
    project_id: ProjectId,
    conflicts_detected: u32,
    conflicts_auto_resolved: u32,
    conflicts_manual: u32,
    sync_operations: u32,
    ai_invocations: u32,
    timestamp: Timestamp,
}
```

## Data Models (continued)

### Orbital View Data Model

```rust
// 3D visualization state
struct OrbitalViewState {
    nodes: Vec<ProjectNode>,
    edges: Vec<ProjectEdge>,
    conflicts: Vec<ConflictMarker>,
    active_members: Vec<MemberMarker>,
}

struct ProjectNode {
    id: NodeId,
    node_type: NodeType,
    position: Vector3,
    label: String,
    metadata: HashMap<String, Value>,
}

enum NodeType {
    Directory,
    File { language: Language },
    Function { signature: String },
    Class { name: String },
}

struct ProjectEdge {
    source: NodeId,
    target: NodeId,
    edge_type: EdgeType,
    weight: f32,
}

enum EdgeType {
    Contains,
    Imports,
    Calls,
    Inherits,
}

struct ConflictMarker {
    node_id: NodeId,
    conflict_id: ConflictId,
    severity: ConflictSeverity,
    pulse_animation: bool,
}

struct MemberMarker {
    actor_id: ActorId,
    node_id: NodeId,
    color: Color,
    label: String,
}
```


## Correctness Properties

A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.

### Property Reflection Analysis

After analyzing all acceptance criteria, I identified several areas where properties can be consolidated:

1. **CRDT Operations**: Multiple criteria (3.1, 3.3, 3.4, 7.2, 9.3) all relate to CRDT synchronization. These can be combined into comprehensive CRDT properties.
2. **Conflict Classification**: Criteria 5.2, 5.3, 5.4 all test conflict type classification and can be unified.
3. **AI Confidence Thresholds**: Criteria 6.3, 6.4, 6.5 test different confidence ranges and can be combined into one property.
4. **Encryption**: Criteria 13.4, 19.1, 19.2, 19.3 all test encryption and can be unified.
5. **Persistence Round-trips**: Criteria 18.1 and 18.2 form a natural round-trip property.

### Core CRDT Properties

**Property 1: CRDT Transaction Creation**
*For any* local edit operation, the CRDT Engine should create a transaction that can be serialized and broadcast to peers.
**Validates: Requirements 3.1, 3.3**

**Property 2: Non-overlapping Edit Convergence**
*For any* two non-overlapping edits to the same file by different team members, merging their CRDT transactions should produce a consistent result across all peers without conflicts.
**Validates: Requirements 3.2, 3.5**

**Property 3: Remote Change Application**
*For any* valid CRDT transaction received from a peer, applying it locally should succeed and update the document state.
**Validates: Requirements 3.4**

**Property 4: Offline Change Queueing**
*For any* changes made while offline, they should be queued and successfully synchronized when connectivity is restored.
**Validates: Requirements 4.6**


### Project Initialization Properties

**Property 5: Supported File Discovery**
*For any* project directory containing source files in supported languages (Python, JS, TS, Rust, Go), the scanner should discover all files with those extensions.
**Validates: Requirements 2.1**

**Property 6: Valid Code Parsing**
*For any* syntactically valid source file in a supported language, Tree-sitter parsing should produce a valid AST without errors.
**Validates: Requirements 2.2, 17.1**

**Property 7: Project Persistence Round-trip**
*For any* project initialized and stored in SQLite, retrieving the project metadata should return equivalent data to what was stored.
**Validates: Requirements 2.3, 18.1, 18.2**

**Property 8: Language Detection**
*For any* project directory with standard language markers (package.json, Cargo.toml, requirements.txt, go.mod), the system should correctly identify the primary language(s).
**Validates: Requirements 2.4, 17.2**

**Property 9: Initialization Error Reporting**
*For any* invalid project setup (missing files, unsupported structure), initialization should fail with a descriptive error message.
**Validates: Requirements 2.5**

### Conflict Detection and Resolution Properties

**Property 10: Post-merge Parsing**
*For any* CRDT merge operation, the resulting code should be parsed with Tree-sitter to detect syntax issues.
**Validates: Requirements 5.1**

**Property 11: Conflict Type Classification**
*For any* detected conflict, it should be correctly classified as compile-time (syntax errors), type-level (type mismatches), or semantic (logical inconsistencies) based on the error type.
**Validates: Requirements 5.2, 5.3, 5.4**

**Property 12: Conflict Annotation Completeness**
*For any* detected conflict, the annotation should include location (file, line, column), conflict type, and affected code regions.
**Validates: Requirements 5.5**


### AI-Powered Resolution Properties

**Property 13: AI Analysis Invocation**
*For any* detected conflict, the AI Agent should analyze the conflict context and generate resolution suggestions.
**Validates: Requirements 6.1, 6.2**

**Property 14: Confidence-based Resolution Strategy**
*For any* AI-generated resolution, the system should apply it automatically if confidence > 85%, suggest it for approval if 70-85%, or present with context if < 70%.
**Validates: Requirements 6.3, 6.4, 6.5**

**Property 15: Offline AI Operation**
*For any* AI operation (conflict resolution, annotation enhancement, negotiation), no network requests should be made when using local models.
**Validates: Requirements 1.5, 6.6**

**Property 16: AI Fallback on Cloud Failure**
*For any* AWS AI service call that fails or times out, the system should fall back to the local AI model and complete the operation.
**Validates: Requirements 14.3**

### Memory Box and Knowledge Graph Properties

**Property 17: Association Storage and Retrieval**
*For any* association created between code regions, comments, decisions, or documents, it should be stored in the Memory Box and retrievable via query.
**Validates: Requirements 7.1, 7.2**

**Property 18: PDF Metadata Extraction**
*For any* valid PDF file imported, the system should extract text content and metadata (title, authors) locally.
**Validates: Requirements 7.3, 12.1**

**Property 19: Knowledge Search Completeness**
*For any* text stored in the Memory Box (comments, decisions, research docs), it should be findable via full-text search using relevant keywords.
**Validates: Requirements 7.4, 12.3**

**Property 20: Association Persistence Across Refactoring**
*For any* code region with associations, if the code is moved or renamed (but content hash remains similar), the associations should be maintained.
**Validates: Requirements 7.5**


### Presence and Collaboration Properties

**Property 21: Presence Broadcasting**
*For any* editing activity by a team member, presence information (file, cursor position) should be broadcast to all connected peers.
**Validates: Requirements 8.1, 8.3**

**Property 22: Presence Display**
*For any* presence update received from a peer, the system should display the teammate's current file and cursor position in the UI.
**Validates: Requirements 8.2**

### Annotation Properties

**Property 23: Annotation Attachment**
*For any* annotation created, it should be attachable to a specific code region (file, line range) and stored with that association.
**Validates: Requirements 9.1**

**Property 24: AI Context Generation**
*For any* annotation where AI enhancement is requested, contextual information about the code should be generated and attached.
**Validates: Requirements 9.2**

**Property 25: Annotation Synchronization**
*For any* annotation created by a team member, it should be synchronized via CRDT to all connected peers.
**Validates: Requirements 9.3**

**Property 26: Annotation Metadata Completeness**
*For any* annotation, it should include author ID, timestamp, content, and optional AI context when retrieved.
**Validates: Requirements 9.4**

**Property 27: Threaded Discussion Support**
*For any* annotation, team members should be able to add replies, forming a threaded discussion that is synchronized.
**Validates: Requirements 9.5**

### AI Negotiation Properties

**Property 28: Negotiation Chat Availability**
*For any* conflict requiring manual resolution (confidence < 70%), the system should offer to start an AI negotiation chat.
**Validates: Requirements 11.1**

**Property 29: Multi-perspective Generation**
*For any* negotiation chat session, the AI should generate multiple perspectives (pros and cons) for different resolution approaches.
**Validates: Requirements 11.2**

**Property 30: Negotiation Question Answering**
*For any* question asked during negotiation, the AI should provide an answer based on conflict context.
**Validates: Requirements 11.3**


**Property 31: Resolution Code Generation**
*For any* resolution approach chosen in negotiation, the AI should generate implementation code for that resolution.
**Validates: Requirements 11.4**

**Property 32: Negotiation Logging**
*For any* negotiation chat session, the conversation should be logged and retrievable for future reference.
**Validates: Requirements 11.5**

### Research Integration Properties

**Property 33: Document Embedding Generation**
*For any* research document imported, the system should generate embeddings using local models for semantic search.
**Validates: Requirements 12.2**

**Property 34: Document-Code Linking**
*For any* link created between a research document and code region, it should be stored in the Memory Box and retrievable.
**Validates: Requirements 12.4**

**Property 35: Citation Generation**
*For any* research document linked to code, the system should be able to generate a properly formatted citation.
**Validates: Requirements 12.5**

### AWS Integration Properties

**Property 36: Cloud Backup Encryption**
*For any* data uploaded to AWS services (S3, DynamoDB), it should be encrypted client-side before transmission.
**Validates: Requirements 13.4, 19.3**

**Property 37: Conditional Cloud Backup**
*For any* CRDT snapshot, when cloud backup is enabled, it should be uploaded to S3; when disabled, no upload should occur.
**Validates: Requirements 13.1, 13.5**

**Property 38: Cloud Metadata Storage**
*For any* project with cloud backup enabled, metadata should be stored in DynamoDB and retrievable.
**Validates: Requirements 13.2**

**Property 39: Cloud AI Service Usage**
*For any* AI operation when AWS integration is enabled and cloud AI is selected, the system should use AWS Bedrock/SageMaker.
**Validates: Requirements 14.1**


**Property 40: AI Cost Tracking**
*For any* AWS AI service invocation, the cost should be tracked and included in usage reports.
**Validates: Requirements 14.5**

### Authentication and Security Properties

**Property 41: Identity Verification**
*For any* team member attempting to join a project, their identity should be verified (via Cognito or local keys) before granting access.
**Validates: Requirements 15.2**

**Property 42: Team-specific Encryption**
*For any* P2P message sent between team members, it should be encrypted using the team-specific encryption key.
**Validates: Requirements 15.3, 19.1**

**Property 43: Access Control Enforcement**
*For any* team, an access control list should exist and be enforced for all operations.
**Validates: Requirements 15.5**

**Property 44: Local Database Encryption**
*For any* SQLite database file created, it should be encrypted using AES-256.
**Validates: Requirements 19.2**

**Property 45: Secure Key Derivation**
*For any* team encryption key generated, it should be derived using a secure key derivation function (e.g., Argon2, PBKDF2).
**Validates: Requirements 19.4**

**Property 46: Key Rotation Without Data Loss**
*For any* project undergoing key rotation, all encrypted data should remain accessible after re-encryption with the new key.
**Validates: Requirements 19.5**

### Telemetry Properties

**Property 47: Conditional Telemetry Transmission**
*For any* usage metric, when telemetry is enabled, it should be sent to CloudWatch; when disabled, it should be stored locally only.
**Validates: Requirements 16.1, 16.4**

**Property 48: Conflict Metrics Tracking**
*For any* conflict detected and resolved, metrics (type, resolution method, success) should be tracked.
**Validates: Requirements 16.2**


**Property 49: Telemetry Privacy**
*For any* telemetry data sent, it should not contain source code, file contents, or other sensitive project data.
**Validates: Requirements 16.5**

### Multi-Language Support Properties

**Property 50: Language-specific Parsing**
*For any* file in a supported language, the appropriate Tree-sitter parser should be used based on file extension and content.
**Validates: Requirements 17.1, 17.4**

**Property 51: Language-specific Conflict Resolution**
*For any* conflict in a specific language, the Conflict Resolver should apply language-appropriate resolution strategies.
**Validates: Requirements 17.3**

### Data Persistence Properties

**Property 52: Incremental Backup Availability**
*For any* state change, an incremental backup should be created and available for recovery.
**Validates: Requirements 18.3**

**Property 53: Corruption Recovery**
*For any* detected database corruption, the system should attempt recovery from the most recent valid backup.
**Validates: Requirements 18.4**

**Property 54: State Export Validity**
*For any* project state exported, the output should be in a valid standard format (JSON, Git bundle, etc.) that can be imported elsewhere.
**Validates: Requirements 18.5**

## Error Handling

### Error Categories

1. **Network Errors**: Connection failures, timeouts, peer disconnections
2. **Parse Errors**: Invalid syntax, unsupported language constructs
3. **CRDT Errors**: Invalid operations, merge conflicts, state corruption
4. **AI Errors**: Model loading failures, inference timeouts, invalid responses
5. **Storage Errors**: Database corruption, disk full, permission denied
6. **AWS Errors**: Service unavailable, authentication failures, quota exceeded


### Error Handling Strategies

**Network Errors**:
- Retry with exponential backoff (max 3 attempts)
- Queue operations for later synchronization
- Fall back to relay servers or cloud sync
- Notify user of connectivity issues

**Parse Errors**:
- Report syntax errors with line/column information
- Attempt partial parsing for incremental updates
- Provide AI-suggested fixes for common errors
- Allow manual conflict resolution

**CRDT Errors**:
- Validate operations before applying
- Maintain operation history for debugging
- Attempt automatic state repair from snapshots
- Provide manual merge tools as last resort

**AI Errors**:
- Fall back to local model if cloud AI fails
- Use cached responses for repeated queries
- Provide degraded functionality without AI
- Log errors for debugging

**Storage Errors**:
- Attempt recovery from backups
- Provide export functionality before corruption spreads
- Alert user to critical storage issues
- Gracefully degrade to read-only mode

**AWS Errors**:
- Fall back to local-only operation
- Cache cloud data for offline access
- Retry with exponential backoff
- Provide clear error messages with remediation steps

### Error Recovery Patterns

```rust
// Retry with exponential backoff
async fn retry_with_backoff<F, T, E>(
    operation: F,
    max_attempts: u32,
) -> Result<T, E>
where
    F: Fn() -> Future<Output = Result<T, E>>,
{
    let mut attempt = 0;
    loop {
        match operation().await {
            Ok(result) => return Ok(result),
            Err(e) if attempt < max_attempts => {
                let delay = Duration::from_millis(100 * 2_u64.pow(attempt));
                tokio::time::sleep(delay).await;
                attempt += 1;
            }
            Err(e) => return Err(e),
        }
    }
}

// Graceful degradation
async fn resolve_conflict_with_fallback(
    conflict: &Conflict,
    use_cloud_ai: bool,
) -> Result<Resolution> {
    if use_cloud_ai {
        match cloud_ai_resolve(conflict).await {
            Ok(resolution) => return Ok(resolution),
            Err(e) => {
                log::warn!("Cloud AI failed, falling back to local: {}", e);
            }
        }
    }
    
    local_ai_resolve(conflict).await
}
```


## Testing Strategy

### Dual Testing Approach

Novex requires both unit testing and property-based testing for comprehensive coverage:

- **Unit tests**: Verify specific examples, edge cases, and error conditions
- **Property tests**: Verify universal properties across all inputs

Both approaches are complementary and necessary. Unit tests catch concrete bugs in specific scenarios, while property tests verify general correctness across a wide input space.

### Property-Based Testing Configuration

**Library Selection**: 
- Rust: `proptest` or `quickcheck`
- TypeScript/JavaScript: `fast-check`

**Test Configuration**:
- Minimum 100 iterations per property test (due to randomization)
- Each property test must reference its design document property
- Tag format: `// Feature: novex, Property {number}: {property_text}`

**Example Property Test**:
```rust
use proptest::prelude::*;

// Feature: novex, Property 2: Non-overlapping Edit Convergence
proptest! {
    #[test]
    fn test_non_overlapping_edits_converge(
        content in ".*",
        edit1_pos in 0..100usize,
        edit1_text in ".*",
        edit2_pos in 100..200usize,
        edit2_text in ".*",
    ) {
        // Create two peers with same initial content
        let mut peer1 = create_peer(&content);
        let mut peer2 = create_peer(&content);
        
        // Apply non-overlapping edits
        let change1 = peer1.apply_edit(edit1_pos, &edit1_text);
        let change2 = peer2.apply_edit(edit2_pos, &edit2_text);
        
        // Merge changes
        peer1.merge(change2);
        peer2.merge(change1);
        
        // Both peers should have identical state
        assert_eq!(peer1.get_content(), peer2.get_content());
    }
}
```

### Unit Testing Strategy

Unit tests should focus on:
- Specific examples demonstrating correct behavior
- Edge cases (empty inputs, boundary conditions)
- Error conditions and exception handling
- Integration points between components

**Avoid writing too many unit tests** - property-based tests handle covering lots of inputs. Focus unit tests on:
- Concrete examples that illustrate requirements
- Edge cases that property tests might miss
- Error scenarios with specific expected outcomes


### Test Coverage by Component

**CRDT Engine**:
- Property tests: Convergence, causal consistency, transaction creation
- Unit tests: Specific merge scenarios, edge cases (empty documents, large edits)

**Sync Engine**:
- Property tests: Message delivery, encryption, presence broadcasting
- Unit tests: Connection establishment, peer discovery, error handling

**Tree-sitter Parser**:
- Property tests: Valid code parsing, language detection
- Unit tests: Specific syntax errors, multi-language projects

**Conflict Resolver**:
- Property tests: Conflict classification, annotation completeness
- Unit tests: Specific conflict scenarios, resolution strategies

**AI Agent**:
- Property tests: Offline operation, confidence-based strategies
- Unit tests: Specific prompts, fallback behavior

**Memory Box**:
- Property tests: Association storage/retrieval, search completeness
- Unit tests: Specific queries, graph traversal

**AWS Integration**:
- Property tests: Encryption, conditional behavior, fallback
- Unit tests: Specific AWS service interactions, error scenarios

### Integration Testing

Integration tests verify end-to-end workflows:
1. Project initialization → file scanning → AST generation → persistence
2. Local edit → CRDT transaction → P2P broadcast → remote merge
3. Conflict detection → AI analysis → resolution generation → application
4. Annotation creation → CRDT sync → display on remote peer
5. Cloud backup → encryption → S3 upload → download → decryption

### Performance Testing

While not property-based, performance tests ensure resource constraints:
- Startup time < 1.5 seconds
- Memory usage: idle < 150 MB, active < 400 MB
- Binary size: 10-40 MB compressed
- Sync latency < 100ms for local peers

### AWS Hackathon Testing Strategy

For the hackathon demo, prioritize:
1. **Core CRDT properties** (Properties 1-4): Demonstrate collision-free collaboration
2. **Conflict resolution properties** (Properties 10-14): Show AI-powered conflict handling
3. **AWS integration properties** (Properties 36-40): Highlight cloud enhancement
4. **Security properties** (Properties 41-46): Demonstrate encryption and auth

Create a demo script that exercises these properties with realistic scenarios.


## AWS Integration Strategy

### Architecture Principles
1. **Local-First with Cloud Enhancement**: Core functionality works offline; AWS provides optional enhancements
2. **Graceful Degradation**: System remains functional if AWS services are unavailable
3. **Cost Optimization**: Minimize AWS usage for free tier compatibility
4. **Security**: Client-side encryption before any cloud transmission

### AWS Services Integration

#### 1. Amazon S3 - Project State Backup

**Purpose**: Disaster recovery and cross-device synchronization

**Implementation**:
```rust
struct S3BackupClient {
    client: aws_sdk_s3::Client,
    bucket: String,
    encryption_key: EncryptionKey,
}

impl S3BackupClient {
    async fn upload_snapshot(&self, project_id: &str, snapshot: &[u8]) -> Result<String> {
        // Encrypt snapshot client-side
        let encrypted = self.encrypt(snapshot)?;
        
        // Upload to S3 with server-side encryption
        let key = format!("projects/{}/snapshots/{}.enc", project_id, Uuid::new_v4());
        self.client
            .put_object()
            .bucket(&self.bucket)
            .key(&key)
            .body(encrypted.into())
            .server_side_encryption(ServerSideEncryption::Aes256)
            .send()
            .await?;
        
        Ok(key)
    }
}
```

**Cost Optimization**:
- Compress snapshots before upload (gzip)
- Use S3 Intelligent-Tiering for automatic cost optimization
- Implement snapshot retention policy (keep last 10, delete older)
- Delta uploads: only upload changed CRDT operations

**Hackathon Demo**: Show backup/restore across devices

#### 2. Amazon DynamoDB - Project Metadata and Sync State

**Purpose**: Fast metadata queries and sync coordination

**Schema**:
```
Table: novex-projects
- PK: project_id (String)
- SK: "metadata" (String)
- team_id (String)
- team_members (List<String>)
- last_sync_timestamp (Number)
- sync_heads (List<String>)
- encryption_key_id (String)
- created_at (Number)

Table: novex-sync-state
- PK: project_id (String)
- SK: peer_id (String)
- last_seen (Number)
- sync_head (String)
- pending_operations (Number)
```

**Implementation**:
```rust
async fn update_sync_state(
    &self,
    project_id: &str,
    peer_id: &str,
    sync_head: &str,
) -> Result<()> {
    self.client
        .put_item()
        .table_name("novex-sync-state")
        .item("project_id", AttributeValue::S(project_id.to_string()))
        .item("peer_id", AttributeValue::S(peer_id.to_string()))
        .item("last_seen", AttributeValue::N(Utc::now().timestamp().to_string()))
        .item("sync_head", AttributeValue::S(sync_head.to_string()))
        .send()
        .await?;
    Ok(())
}
```

**Cost Optimization**:
- Use on-demand billing for variable workload
- Batch write operations
- Cache frequently accessed metadata locally

**Hackathon Demo**: Show team member discovery and sync coordination


#### 3. AWS Lambda - Relay Server and Sync Coordination

**Purpose**: NAT traversal relay and sync coordination when P2P fails

**Functions**:

1. **Relay Function**: Forward CRDT operations between peers
```python
# lambda/relay.py
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('novex-relay-messages')

def lambda_handler(event, context):
    """Relay CRDT operations between peers"""
    body = json.loads(event['body'])
    
    # Store message for recipient
    table.put_item(Item={
        'recipient_id': body['to_peer_id'],
        'sender_id': body['from_peer_id'],
        'message': body['crdt_operation'],
        'timestamp': int(time.time()),
        'ttl': int(time.time()) + 3600  # 1 hour TTL
    })
    
    return {'statusCode': 200, 'body': json.dumps({'status': 'relayed'})}
```

2. **Discovery Function**: Help peers find each other
```python
# lambda/discovery.py
def lambda_handler(event, context):
    """Return list of online peers for a project"""
    project_id = event['queryStringParameters']['project_id']
    
    # Query DynamoDB for active peers
    response = table.query(
        KeyConditionExpression='project_id = :pid',
        FilterExpression='last_seen > :threshold',
        ExpressionAttributeValues={
            ':pid': project_id,
            ':threshold': int(time.time()) - 300  # 5 minutes
        }
    )
    
    peers = [item['peer_id'] for item in response['Items']]
    return {'statusCode': 200, 'body': json.dumps({'peers': peers})}
```

**Cost Optimization**:
- Use Lambda free tier (1M requests/month)
- Set short timeout (3 seconds)
- Use DynamoDB TTL for automatic message cleanup

**Hackathon Demo**: Show relay fallback when direct P2P fails

#### 4. Amazon Bedrock - Enhanced AI Conflict Resolution

**Purpose**: More powerful AI for complex conflicts

**Implementation**:
```rust
struct BedrockAIClient {
    client: aws_sdk_bedrockruntime::Client,
    model_id: String,
}

impl BedrockAIClient {
    async fn resolve_conflict(&self, context: &ConflictContext) -> Result<Resolution> {
        let prompt = format!(
            "Analyze this code conflict and suggest a resolution:\n\
             Local version:\n{}\n\
             Remote version:\n{}\n\
             Context: {}\n\
             Provide a resolution with explanation.",
            context.local_code,
            context.remote_code,
            context.surrounding_code
        );
        
        let response = self.client
            .invoke_model()
            .model_id(&self.model_id)
            .body(json!({
                "prompt": prompt,
                "max_tokens": 1000,
                "temperature": 0.3
            }).to_string())
            .send()
            .await?;
        
        // Parse response and create Resolution
        parse_ai_response(response.body())
    }
}
```

**Model Selection**:
- Primary: Claude 3 Sonnet (balanced cost/performance)
- Fallback: Claude 3 Haiku (faster, cheaper)
- Local fallback: Phi-3 Mini via Ollama

**Cost Optimization**:
- Cache resolutions for similar conflicts
- Use local model for simple conflicts
- Only invoke Bedrock for confidence < 70% cases

**Hackathon Demo**: Show side-by-side comparison of local vs. cloud AI resolution quality


#### 5. Amazon Cognito - Team Authentication

**Purpose**: Secure team member authentication and authorization

**User Pool Configuration**:
- Email-based authentication
- MFA optional for enhanced security
- Custom attributes: team_id, role (admin/member)

**Implementation**:
```rust
struct CognitoAuthClient {
    client: aws_sdk_cognitoidentityprovider::Client,
    user_pool_id: String,
    client_id: String,
}

impl CognitoAuthClient {
    async fn authenticate(&self, email: &str, password: &str) -> Result<AuthToken> {
        let response = self.client
            .initiate_auth()
            .auth_flow(AuthFlowType::UserPasswordAuth)
            .client_id(&self.client_id)
            .auth_parameters("USERNAME", email)
            .auth_parameters("PASSWORD", password)
            .send()
            .await?;
        
        let token = response
            .authentication_result()
            .ok_or(Error::AuthFailed)?
            .id_token()
            .ok_or(Error::NoToken)?;
        
        Ok(AuthToken::from(token))
    }
    
    async fn verify_team_member(&self, token: &AuthToken, team_id: &str) -> Result<bool> {
        let user = self.client
            .get_user()
            .access_token(token.as_str())
            .send()
            .await?;
        
        // Check custom:team_id attribute
        let user_team_id = user
            .user_attributes()
            .iter()
            .find(|attr| attr.name() == Some("custom:team_id"))
            .and_then(|attr| attr.value())
            .ok_or(Error::NoTeamId)?;
        
        Ok(user_team_id == team_id)
    }
}
```

**Cost Optimization**:
- Use Cognito free tier (50,000 MAUs)
- Cache tokens locally with refresh
- Support local key-based auth as alternative

**Hackathon Demo**: Show secure team onboarding flow

#### 6. Amazon CloudWatch - Telemetry and Monitoring

**Purpose**: Usage analytics and system health monitoring

**Metrics**:
- Conflict detection rate
- AI resolution success rate
- Sync latency
- Active users per project
- AWS service usage

**Implementation**:
```rust
struct CloudWatchTelemetry {
    client: aws_sdk_cloudwatch::Client,
    namespace: String,
}

impl CloudWatchTelemetry {
    async fn send_metrics(&self, metrics: &[Metric]) -> Result<()> {
        let metric_data: Vec<_> = metrics
            .iter()
            .map(|m| {
                MetricDatum::builder()
                    .metric_name(&m.name)
                    .value(m.value)
                    .unit(StandardUnit::Count)
                    .timestamp(DateTime::from(m.timestamp))
                    .build()
            })
            .collect();
        
        self.client
            .put_metric_data()
            .namespace(&self.namespace)
            .set_metric_data(Some(metric_data))
            .send()
            .await?;
        
        Ok(())
    }
}
```

**Privacy**:
- Only send anonymized metrics
- No source code or file contents
- Aggregate data at project level
- User opt-in required

**Cost Optimization**:
- Batch metrics (max 20 per request)
- Send every 5 minutes, not real-time
- Use CloudWatch free tier (10 custom metrics)

**Hackathon Demo**: Show real-time dashboard of team collaboration metrics


### AWS Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Novex Desktop App                         │
│                   (Local-First Core)                         │
└────────────┬────────────────────────────────────────────────┘
             │
             │ Optional Cloud Enhancement
             │
    ┌────────┴────────┐
    │                 │
    ▼                 ▼
┌─────────┐      ┌─────────┐
│ Cognito │      │ Bedrock │
│  Auth   │      │   AI    │
└────┬────┘      └────┬────┘
     │                │
     │    ┌───────────┴──────────┐
     │    │                      │
     ▼    ▼                      ▼
┌──────────────┐         ┌──────────────┐
│  DynamoDB    │         │      S3      │
│  Metadata    │         │   Backups    │
└──────────────┘         └──────────────┘
     │                           │
     │    ┌──────────────────────┘
     │    │
     ▼    ▼
┌──────────────┐         ┌──────────────┐
│    Lambda    │         │  CloudWatch  │
│    Relay     │         │  Telemetry   │
└──────────────┘         └──────────────┘
```

### Cost Estimation (AWS Free Tier)

**Monthly Usage Assumptions** (10-person team, 20 projects):
- S3: 5 GB storage, 1000 PUT/GET requests → $0.12
- DynamoDB: 1M reads, 500K writes → Free tier
- Lambda: 100K invocations, 30s avg duration → Free tier
- Bedrock: 1M input tokens, 500K output tokens → $3.00
- Cognito: 50 active users → Free tier
- CloudWatch: 10 custom metrics → Free tier

**Total Monthly Cost**: ~$3.15 (mostly Bedrock AI)

**Cost Reduction Strategies**:
- Use local AI for 80% of conflicts → $0.60/month
- Implement aggressive caching → $0.40/month
- Batch operations → $0.30/month

**Hackathon Budget**: Easily within free tier for demo

### Deployment Strategy

**Infrastructure as Code** (AWS CDK):
```typescript
// cdk/lib/novex-stack.ts
export class NovexStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // S3 bucket for backups
    const backupBucket = new s3.Bucket(this, 'NovexBackups', {
      encryption: s3.BucketEncryption.S3_MANAGED,
      versioned: true,
      lifecycleRules: [{
        expiration: cdk.Duration.days(90),
        noncurrentVersionExpiration: cdk.Duration.days(30),
      }],
    });

    // DynamoDB tables
    const projectsTable = new dynamodb.Table(this, 'NovexProjects', {
      partitionKey: { name: 'project_id', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'sk', type: dynamodb.AttributeType.STRING },
      billingMode: dynamodb.BillingMode.PAY_PER_REQUEST,
    });

    // Lambda relay function
    const relayFunction = new lambda.Function(this, 'RelayFunction', {
      runtime: lambda.Runtime.PYTHON_3_11,
      handler: 'relay.lambda_handler',
      code: lambda.Code.fromAsset('lambda'),
      timeout: cdk.Duration.seconds(3),
      environment: {
        PROJECTS_TABLE: projectsTable.tableName,
      },
    });

    // Cognito user pool
    const userPool = new cognito.UserPool(this, 'NovexUsers', {
      selfSignUpEnabled: true,
      signInAliases: { email: true },
      customAttributes: {
        team_id: new cognito.StringAttribute({ mutable: true }),
      },
    });
  }
}
```

**Deployment Commands**:
```bash
cd cdk
npm install
cdk bootstrap
cdk deploy
```

### Hackathon Demo Strategy

**Demo Flow** (10 minutes):
1. **Setup** (1 min): Show Novex installation and project initialization
2. **Local Collaboration** (2 min): Two developers editing simultaneously, CRDT merge
3. **Conflict Detection** (2 min): Introduce semantic conflict, show AI analysis
4. **Cloud Enhancement** (2 min): Enable AWS, show Bedrock resolution vs. local
5. **Team Features** (2 min): Annotations, presence, Memory Box search
6. **Backup/Recovery** (1 min): Simulate crash, restore from S3

**Key Metrics to Highlight**:
- Startup time: < 1.5s
- Memory usage: < 400 MB
- Conflict auto-resolution: 85% success rate
- Sync latency: < 100ms local, < 500ms cloud relay

**Innovation Points for Judges**:
1. **Beyond Git**: Semantic conflict detection, not just text diffs
2. **AI-Powered**: 85% automatic resolution with explainable AI
3. **Local-First**: Works offline, cloud enhances
4. **Lightweight**: Runs on any laptop, not just high-end machines
5. **Real-Time**: Sub-second collaboration, not commit-based

