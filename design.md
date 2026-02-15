# Design Document: CodeSailor VS Code Extension

## Overview

CodeSailor is a VS Code extension that solves the "Lost at Sea" effect for developers navigating unfamiliar codebases. It combines local vector storage with AWS Bedrock AI to provide instant codebase understanding, predictive impact analysis, and task-to-file navigation.

The design prioritizes fast time-to-value (enable navigation in minutes, not days), privacy (local-first embeddings), and cost efficiency (hash-based caching). The extension operates in a degraded mode when offline, ensuring developers can continue working with cached data.

### Core Value Proposition

**Problem Solved**: New developers spend 70% of their time reading and tracing code, taking 3-6 months to become productive. Documentation is rarely up-to-date.

**Solution**: CodeSailor is a **Knowledge Transfer** tool that indexes your entire repository and provides:

1. **Architecture Overview** - One-click README-style project explanation
2. **Smart File Navigator** - Describe a feature, get relevant files ("Where is payment logic?" → payment.service.ts)
3. **Impact Analysis Graph** - Visualize which modules depend on your current file
4. **Context-Aware Chat** - Ask questions based on project code patterns

**Unique Selling Points**:

- **Predictive Impact Analysis**: Warns you which services might break when you edit a file
- **Task-to-File Navigation**: Describe the task ("Add rate limiting") → Get exact files to modify

### AWS Service Integration

**AWS Bedrock**:

- Models used:
  - **Titan Embeddings V2** (`amazon.titan-embed-text-v2:0`) for code embeddings
  - **Claude Sonnet 4.5** (`anthropic.claude-sonnet-4-5:0`) for Context-Aware Chat
- Request batching: Up to 25 texts per embedding request
- Timeout: 10 seconds per request
- Retry policy: Exponential backoff, max 3 retries

**Authentication**:

- Requires AWS credentials (Access Key ID, Secret Access Key, Region)
- Stored securely using VS Code SecretStorage API
- Supports IAM user with `bedrock:InvokeModel` permission

**Cost Optimization**:

- Local caching reduces embedding API calls by ~90%
- Hash-based change detection prevents re-embedding unchanged files
- Model routing: Cheap Titan for embeddings, Claude only for chat queries
- Estimated cost: $0.10-$0.50 per 1000 files indexed (one-time), $0.01-$0.05 per query

**Network**:

- All AWS communication over HTTPS
- Graceful degradation when offline (local search still works)
- Automatic reconnection on network recovery

### Architecture Layers

```
┌─────────────────────────────────────────────────────────────┐
│                    VS Code Host Layer                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Webview    │  │Impact Graph  │  │Event Listeners│      │
│  │  (3 tabs)    │  │  Visualizer  │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                   Extension Core Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Orchestrator │  │Worker Thread │  │ File Walker  │      │
│  │              │  │              │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│  ┌──────────────┐  ┌──────────────┐                        │
│  │ AST Chunker  │  │ RAG Engine   │                        │
│  │              │  │              │                        │
│  └──────────────┘  └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                     Data Layer                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │Vector Store  │  │Cache Manager │  │Dependency    │      │
│  │  (Faiss )    │  │              │  │Graph         │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            ↕
┌─────────────────────────────────────────────────────────────┐
│                     Cloud Layer                              │
│  ┌──────────────────────────────────────────────────┐       │
│  │           AWS Bedrock Client                     │       │
│  │  ┌────────────────┐  ┌────────────────────┐     │       │
│  │  │ Titan          │  │ Claude 4.5         │     │       │
│  │  │ Embeddings V2  │  │ Sonnet             │     │       │
│  │  └────────────────┘  └────────────────────┘     │       │
│  └──────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────┘
```

**VS Code Host Layer**: UI interactions - Webview with 3 tabs (Chat, Navigator, Impact Analysis), Impact Graph visualizer, and VS Code event listeners.

**Extension Core Layer**: Business logic - Orchestrator coordinates components, Worker Thread handles non-blocking indexing, File Walker traverses projects, AST Chunker processes code, RAG Engine handles queries.

**Data Layer**: Persistent storage - Vector Store (FAISS with vector extension) holds embeddings, Cache Manager implements hash-based caching, Dependency Graph tracks file relationships.

**Cloud Layer**: AWS Bedrock interface for embeddings (Titan) and AI-generated responses (Claude).

## Components and Interfaces

### File Walker

**Purpose**: Recursively traverse project directories while respecting exclusion rules.

**Interface**:

```typescript
interface FileWalker {
  // Scan workspace and return list of files to index
  scanWorkspace(
    rootPath: string,
    gitignorePatterns: string[],
  ): Promise<FileInfo[]>;

  // Check if a file should be indexed
  shouldIndex(filePath: string): boolean;

  // Watch for file system changes
  watchFiles(onChange: (event: FileChangeEvent) => void): Disposable;
}

interface FileInfo {
  path: string;
  size: number;
  lastModified: Date;
  language: string;
}

interface FileChangeEvent {
  type: "created" | "modified" | "deleted";
  path: string;
}
```

**Behavior**:

- Skips directories: node_modules, .git, dist, build, out, .vscode
- Respects .gitignore patterns
- Filters sensitive files: .env, .pem, .key, .cert, secrets._, credentials._
- Prioritizes currently open files
- Emits progress events during scanning

### AST Chunker

**Purpose**: Parse code files into semantic chunks that preserve function and class boundaries.

**Interface**:

```typescript
interface ASTChunker {
  // Chunk a file into semantic segments
  chunkFile(content: string, language: string): CodeChunk[];

  // Extract symbols from a file
  extractSymbols(content: string, language: string): Symbol[];
}

interface CodeChunk {
  id: string;
  content: string;
  startLine: number;
  endLine: number;
  filePath: string;
  symbols: string[]; // Function/class names in this chunk
  metadata: ChunkMetadata;
}

interface ChunkMetadata {
  language: string;
  chunkIndex: number;
  totalChunks: number;
  overlapTokens: number;
}

interface Symbol {
  name: string;
  type: "function" | "class" | "interface" | "variable";
  line: number;
  scope: string;
}
```

**Behavior**:

- Uses language-specific parsers (tree-sitter for TypeScript/JavaScript/Python)
- Target chunk size: 1000 tokens
- Overlap between chunks: 500 tokens
- Falls back to line-based chunking if parsing fails
- Preserves complete function/class definitions
- Attaches symbol metadata to each chunk

### Vector Store

**Purpose**: Store and retrieve code embeddings using local vector database.

**Interface**:

```typescript
interface VectorStore {
  // Add embeddings to the store
  addEmbeddings(chunks: CodeChunk[], embeddings: number[][]): Promise<void>;

  // Search for similar chunks
  search(queryEmbedding: number[], topK: number): Promise<SearchResult[]>;

  // Get all chunks for a specific file
  getFileChunks(filePath: string): Promise<CodeChunk[]>;

  // Remove chunks for a file
  removeFile(filePath: string): Promise<void>;

  // Get statistics
  getStats(): Promise<VectorStoreStats>;
}

interface SearchResult {
  chunk: CodeChunk;
  score: number; // Similarity score 0-1
}

interface VectorStoreStats {
  totalChunks: number;
  totalFiles: number;
  indexSize: number; // bytes
  lastUpdated: Date;
}
```

**Implementation**:

- **MVP**: SQLite with vector extension (sqlite-vss) for simpler deployment and cross-platform compatibility
- Stores embeddings in local file system (~/.vscode/extensions/CodeSailor/vectors)

**Behavior**:

- Uses cosine similarity for search
- Maintains index metadata (file paths, timestamps, hashes)
- Supports incremental updates
- Provides atomic operations for consistency
- Target: <300ms search across 100,000 chunks

### Cache Manager

**Purpose**: Implement hash-based caching to avoid re-embedding unchanged files.

**Interface**:

```typescript
interface CacheManager {
  // Get cached embedding for a file hash
  getCachedEmbedding(fileHash: string): Promise<number[][] | null>;

  // Store embedding with file hash
  setCachedEmbedding(fileHash: string, embeddings: number[][]): Promise<void>;

  // Compute hash for file content
  computeHash(content: string): string;

  // Check if file has changed
  hasFileChanged(filePath: string, currentHash: string): Promise<boolean>;

  // Evict old entries
  evictLRU(targetSize: number): Promise<void>;
}

interface CacheEntry {
  hash: string;
  embeddings: number[][];
  timestamp: Date;
  accessCount: number;
  lastAccessed: Date;
}
```

**Behavior**:

- Uses SHA-256 for file hashing
- Stores cache in SQLite database for persistence
- Implements LRU eviction when cache exceeds 1GB
- Tracks access patterns for eviction decisions
- Provides cache hit/miss statistics

### Bedrock Client

**Purpose**: Interface with AWS Bedrock for embeddings and text generation.

**Interface**:

```typescript
interface BedrockClient {
  // Generate embeddings for text chunks
  generateEmbeddings(texts: string[]): Promise<number[][]>;

  // Generate text using Claude
  generateText(prompt: string, context: string[]): Promise<string>;

  // Check service availability
  healthCheck(): Promise<boolean>;

  // Get usage statistics
  getUsageStats(): UsageStats;
}

interface UsageStats {
  embeddingTokens: number;
  generationTokens: number;
  estimatedCost: number;
  requestCount: number;
}
```

**Configuration**:

- Embedding Model: amazon.titan-embed-text-v2:0
- Generation Model: anthropic.claude-sonnet-4-5:0
- Max tokens per request: 8000 (embeddings), 4096 (generation)
- Timeout: 10 seconds
- Retry policy: Exponential backoff, max 3 retries

**Behavior**:

- Batches embedding requests (up to 25 texts per batch)
- Implements rate limiting to avoid throttling
- Caches credentials securely using VS Code secrets API
- Provides detailed error messages for API failures
- Tracks token usage for cost monitoring

### Dependency Graph

**Purpose**: Track import/require relationships between files.

**Interface**:

```typescript
interface DependencyGraph {
  // Add a file and its dependencies
  addFile(filePath: string, imports: string[]): void;

  // Get files that depend on a given file
  getDependents(filePath: string): string[];

  // Get files that a given file depends on
  getDependencies(filePath: string): string[];

  // Remove a file from the graph
  removeFile(filePath: string): void;

  // Detect circular dependencies
  findCycles(): string[][];

  // Get strongly connected components
  getComponents(): string[][];
}
```

**Behavior**:

- Represents graph as adjacency list
- Resolves relative imports to absolute paths
- Handles various import styles: ES6, CommonJS, TypeScript
- Persists graph to disk for fast startup
- Updates incrementally on file changes
- Provides topological sorting for build order

### RAG Query Engine

**Purpose**: Orchestrate retrieval and AI-powered response generation for user queries.

**Interface**:

```typescript
interface RAGQueryEngine {
  // Process a user query (Context-Aware Chat)
  processQuery(query: string, context: QueryContext): Promise<QueryResult>;

  // Search for relevant files by feature description (Smart File Navigator)
  searchByFeature(description: string): Promise<FileSearchResult[]>;

  // Generate architecture overview (Get Started)
  generateArchitectureOverview(): Promise<string>;
}

interface QueryContext {
  activeFile?: string;
  selectedText?: string;
  scope: "file" | "workspace";
}

interface QueryResult {
  answer: string;
  sources: CodeChunk[];
  processingTime: number;
}

interface FileSearchResult {
  filePath: string;
  relevanceScore: number;
  contextSnippet: string;
  matchedChunks: CodeChunk[];
}
```

**Query Processing Pipeline**:

1. Generate query embedding using Bedrock Titan
2. Retrieve relevant chunks from Vector Store via semantic search
3. Rank chunks by cosine similarity
4. Construct prompt with top 10 chunks as context
5. Generate response using Claude Sonnet 4.5
6. Return formatted result with clickable file references

**Behavior**:

- Context-Aware Chat: Retrieves relevant code patterns and generates answers
- Smart File Navigator: Groups search results by file, shows relevance indicators
- Architecture Overview: Analyzes dependency graph and generates README-style summary
- Includes file paths and line numbers in all responses
- Tracks query history

### Worker Thread

**Purpose**: Execute indexing operations without blocking the main thread.

**Interface**:

```typescript
interface WorkerThread {
  // Start indexing a batch of files
  indexFiles(files: FileInfo[]): Promise<void>;

  // Cancel ongoing indexing
  cancel(): void;

  // Get indexing progress
  getProgress(): IndexingProgress;

  // Register progress callback
  onProgress(callback: (progress: IndexingProgress) => void): void;
}

interface IndexingProgress {
  totalFiles: number;
  processedFiles: number;
  currentFile: string;
  status: "idle" | "indexing" | "cancelled" | "error";
  error?: string;
}
```

**Behavior**:

- Runs in separate Node.js worker thread
- Communicates via message passing
- Processes files in priority order (active file first)
- Batches Bedrock API calls for efficiency
- Reports progress every 5 files
- Handles errors gracefully without crashing main thread

### Orchestrator

**Purpose**: Coordinate all components and manage extension lifecycle.

**Interface**:

```typescript
interface Orchestrator {
  // Initialize extension
  activate(context: ExtensionContext): Promise<void>;

  // Shutdown extension
  deactivate(): Promise<void>;

  // Handle file change events
  onFileChange(event: FileChangeEvent): Promise<void>;

  // Handle query requests
  onQuery(query: string, context: QueryContext): Promise<QueryResult>;

  // Get extension status
  getStatus(): ExtensionStatus;
}

interface ExtensionStatus {
  indexingStatus: IndexingProgress;
  vectorStoreStats: VectorStoreStats;
  cacheStats: CacheStats;
  cloudStatus: "online" | "offline" | "degraded";
}

interface CacheStats {
  hitRate: number;
  totalEntries: number;
  sizeBytes: number;
}
```

**Lifecycle**:

1. **Activation**: Load configuration, initialize components, start indexing active file
2. **Runtime**: Handle queries, file changes, and user interactions
3. **Deactivation**: Cancel ongoing operations, persist state, cleanup resources

## Data Models

### Code Chunk

Represents a semantic segment of code with embeddings and metadata.

```typescript
interface CodeChunk {
  id: string; // UUID
  filePath: string; // Absolute path
  content: string; // Actual code text
  startLine: number; // 1-indexed
  endLine: number; // 1-indexed
  embedding?: number[]; // 1024-dimensional vector (Titan V2)
  symbols: string[]; // Function/class names
  language: string; // TypeScript, Python, etc.
  hash: string; // SHA-256 of content
  metadata: ChunkMetadata;
}
```

### File Index Entry

Tracks indexing status for each file.

```typescript
interface FileIndexEntry {
  filePath: string;
  hash: string;
  lastIndexed: Date;
  chunkCount: number;
  status: "pending" | "indexed" | "error";
  error?: string;
}
```

### Dependency Edge

Represents an import relationship between files.

```typescript
interface DependencyEdge {
  from: string; // Importing file
  to: string; // Imported file
  importType: "es6" | "commonjs" | "dynamic";
  line: number; // Line number of import
}
```

### Query History Entry

Stores past queries for autocomplete and analytics.

```typescript
interface QueryHistoryEntry {
  query: string;
  timestamp: Date;
  context: QueryContext;
  resultCount: number;
  processingTime: number;
}
```

## Key Correctness Guarantees

The following correctness guarantees are critical for MVP functionality:

### Indexing Correctness

- **Complete Scanning**: All non-excluded files in workspace are discovered and indexed
- **Smart Exclusions**: node_modules, .git, and sensitive files (.env, .pem, etc.) are automatically skipped
- **Incremental Updates**: Only modified files are re-indexed, not the entire workspace
- **Active File Priority**: Currently open file is indexed first for immediate usability

### Code Chunking Correctness

- **Boundary Preservation**: Functions and classes are never split across chunk boundaries
- **Fallback Safety**: Invalid syntax triggers line-based chunking instead of failure
- **Metadata Completeness**: Every chunk includes file path, line numbers, symbols, and content hash

### Dependency Graph Correctness

- **Import Accuracy**: All import/require statements are extracted and resolved to absolute paths
- **Graph Consistency**: Dependency graph accurately reflects all file relationships
- **Impact Analysis Accuracy**: Dependent file queries return all files that import the target

### Search and RAG Correctness

- **Ranking Quality**: Search results ordered by semantic similarity (highest first)
- **Context Minimization**: Only top-K relevant chunks sent to Claude for generation
- **Local-First Privacy**: Embeddings stored locally; searches work offline

### Caching Correctness

- **Cache Hits**: Unchanged files (same SHA-256 hash) skip re-embedding
- **Cache Invalidation**: Modified files detected via hash change and re-embedded
- **Cost Efficiency**: ~90% reduction in Bedrock API calls via caching

### Configuration Correctness

- **Credential Validation**: AWS credentials validated via test API call before storage
- **Secure Storage**: Credentials stored using VS Code SecretStorage API

### Error Handling Correctness

- **Network Failure Detection**: Offline mode detected; local search continues working
- **Graceful Degradation**: Clear error messages when AI generation unavailable offline
- **Automatic Recovery**: Cloud features resume automatically when connectivity restored

## Error Handling

### Error Categories

**Network Errors**:

- AWS Bedrock API unavailable
- Timeout on API requests
- Rate limiting from AWS
- Invalid credentials

**File System Errors**:

- File not found during indexing
- Permission denied reading files
- Disk full when writing embeddings
- Corrupted vector store index

**Parsing Errors**:

- Invalid syntax in code files
- Unsupported file encoding
- Binary files mistaken for text
- Extremely large files exceeding memory

**User Input Errors**:

- Invalid query syntax
- Empty search queries
- Malformed configuration settings
- Invalid regex patterns for exclusions

### Error Handling Strategies

**Graceful Degradation**:

- When AWS Bedrock is unavailable, continue providing search from local Vector_Store
- When a file cannot be parsed, fall back to line-based chunking
- When embeddings fail, log error and continue with remaining files
- Display clear status indicators for degraded functionality

**Retry Logic**:

- Exponential backoff for transient network errors (max 3 retries)
- Retry with smaller batch sizes if batch embedding fails
- Automatic reconnection attempts when network is restored

**User Feedback**:

- Display toast notifications for critical errors
- Show detailed error messages in Webview for query failures
- Provide actionable suggestions (e.g., "Check AWS credentials")
- Log all errors to extension output channel for debugging

**Data Integrity**:

- Use atomic operations for Vector_Store updates
- Maintain transaction log for cache operations
- Validate data before persisting to disk
- Provide recovery mechanism for corrupted indexes

### Error Recovery

**Corrupted Vector Store**:

1. Detect corruption on startup (checksum validation)
2. Attempt to rebuild index from cached embeddings
3. If cache is also corrupted, trigger full re-indexing
4. Notify user of recovery process

**Failed Indexing**:

1. Mark failed files in index with error status
2. Retry failed files on next activation
3. Provide manual "Retry Failed Files" command
4. Skip persistently failing files after 3 attempts

**Invalid Configuration**:

1. Validate settings on change
2. Reject invalid values with clear error message
3. Fall back to default values if validation fails
4. Preserve last known good configuration

## Testing Strategy (MVP)

### Test Approach

For MVP, focus on **unit tests** and **integration tests** that validate the 4 core features:

**Core Feature Tests**:

1. **Architecture Overview**: Generate overview, parse components, display in Webview
2. **Smart File Navigator**: Semantic search, result ranking, file opening
3. **Impact Analysis**: Dependency graph queries, visualization, warning notifications
4. **Context-Aware Chat**: Query processing, RAG retrieval, Claude response generation

**Infrastructure Tests**:

- File Walker: Exclusion patterns, .gitignore parsing
- AST Chunker: Parsing, chunking, fallback on invalid syntax
- Vector Store: Add/search embeddings, persistence
- Cache Manager: Hash computation, cache hits/misses
- Dependency Graph: Import extraction, graph construction

**Integration Tests**:

- End-to-end indexing: file scan → chunk → embed → store
- Query flow: user input → search → retrieve → generate → display
- File change: modify → detect → re-index → update graph

### Performance Targets

- Search latency: <300ms
- AI response generation: <5s
- Indexing: Complete on file open <1s

### Mocking

- Mock AWS Bedrock for unit tests (return fake embeddings)
- Use real Bedrock for integration tests (with test AWS account)
- Mock VS Code API for all tests
