# Requirements Document: CodeSailor VS Code Extension

## Introduction

CodeSailor is a VS Code extension that solves the "Lost at Sea" effect for developers navigating unfamiliar codebases. Using RAG (Retrieval-Augmented Generation) with AWS Bedrock, it creates an instant "map" of any project, allowing developers to understand system architecture, trace data flows, and predict impact of changes—without relying on outdated documentation or senior engineers.

### The Problem

- **Lost at Sea Effect**: New developers spend 70% of their time reading and tracing code
- **Slow Onboarding**: 3-6 months before new hires become fully productive
- **Documentation Drift**: Static documentation is rarely up-to-date with live code

### The Solution

CodeSailor is a **Knowledge Transfer** tool (not a code generator). It indexes your entire repository and lets you query system structure directly—e.g., "Trace the payment flow" returns specific file paths, dependency graphs, and entry points without manual traversal.

### Unique Selling Proposition

1. **Predictive Impact Analysis**: Warns you which services might break if you edit a file
2. **Task-to-File Navigation**: Describe a task, get exact files you need to touch

## Glossary

- **CodeSailor**: The VS Code extension system
- **Indexer**: Component responsible for scanning and processing code files
- **Vector_Store**: Local storage system for code embeddings (FAISS with vector extension)
- **RAG_Engine**: Retrieval-Augmented Generation query processing system
- **AST_Chunker**: Abstract Syntax Tree-based code chunking component
- **Bedrock_Client**: AWS Bedrock API integration component
- **Cache_Manager**: Component managing hash-based caching of embeddings
- **Dependency_Graph**: Import/require relationship mapping between files
- **Webview**: VS Code sidebar panel for user interface

## Functional Requirements

### FR-1: Smart Project Indexing

**User Story:** As a developer, I want the extension to automatically index my project files, so that I can query my codebase without manual setup.

**Type:** Functional

#### Acceptance Criteria

1.1. Extension activation triggers recursive workspace scan

1.2. Automatically skip excluded directories (node_modules, .git, .gitignore patterns)

1.3. Prioritize currently open file for immediate indexing

1.4. Background indexing continues after active file is indexed

1.5. File modifications trigger incremental re-indexing (single file only)

### FR-2: AST-Based Code Chunking

**User Story:** As a developer, I want code to be chunked intelligently at function and class boundaries, so that semantic search returns meaningful context.

**Type:** Functional

#### Acceptance Criteria

2.1. Parse code files into Abstract Syntax Tree (AST) for semantic understanding

2.2. Preserve complete function/class definitions within chunk boundaries (no splitting)

2.3. Include 500-token overlap between adjacent chunks for context continuity

2.4. Fallback to line-based chunking when AST parsing fails

2.5. Attach complete metadata to each chunk: - File path and line number ranges - Symbol names (functions, classes, variables) - Content hash for cache validation

### FR-3: Context-Aware Chat

**User Story:** As a developer, I want to ask questions based on existing code patterns (e.g., "How do I add a new API endpoint in this framework?"), so that I can follow project conventions without asking senior engineers.

**Type:** Functional

#### Acceptance Criteria

3.1. Retrieve semantically similar code chunks from Vector Store on query submission

3.2. Send top-K relevant chunks to Bedrock Claude for context-aware generation

3.3. Display AI-generated answers with clickable file references in Webview

3.4. Use Claude Sonnet 4.5 for all chat responses

3.5. Complete end-to-end query processing within 5 seconds

### FR-4: Smart File Navigator

**User Story:** As a developer, I want to describe a feature (e.g., "Where is the payment logic?") and get a list of relevant files, so that I can navigate large projects without manual searching.

**Type:** Functional

#### Acceptance Criteria

4.1. Perform semantic search across all indexed chunks in Vector Store

4.2. Return search results within 300ms for responsive navigation

4.3. Rank results by similarity score and group by file for clarity

4.4. Display results with: - File paths - Relevance indicators (similarity scores) - Context snippets (surrounding code)

4.5. Clicking a result opens the file and highlights the relevant code section

### FR-5: Dependency Graph Construction

**User Story:** As a developer, I want the extension to track import and require relationships, so that I can understand how files depend on each other.

**Type:** Functional

#### Acceptance Criteria

5.1. Extract all import/require statements during file indexing

5.2. Resolve relative imports to absolute file paths

5.3. Construct complete dependency graph after indexing all files

5.4. Persist dependency graph to local storage for fast startup

5.5. Update only affected graph edges when files are modified (incremental updates)

### FR-6: Impact Analysis (USP Feature)

**User Story:** As a developer, I want to be warned which services might break when I edit a file, so that I can avoid introducing bugs in dependent code.

**Type:** Functional

#### Acceptance Criteria

6.1. Query dependency graph for dependents when user opens a file

6.2. Display Impact Analysis Graph with visual connections in Webview

6.3. Show warning notification on file save: - Format: "⚠️ This change affects N other files" - Includes count of dependent files

6.4. Clicking warning highlights affected files in dependency graph

6.5. Clicking dependent file in graph opens it in editor

### FR-7: Architecture Overview (Get Started Feature)

**User Story:** As a new developer joining a project, I want to generate a README-style architecture overview with one click, so that I can understand the project structure in minutes instead of days.

**Type:** Functional

#### Acceptance Criteria

7.1. Analyze dependency graph to identify major components/modules

7.2. Retrieve representative code samples and entry points per component

7.3. Generate high-level architecture summary with AI: - Project purpose and tech stack - Component responsibilities - Data flow and interactions

7.4. Display summary in README-style format with sections per component

7.5. Make component names clickable to show associated files

### FR-8: Webview User Interface

**User Story:** As a developer, I want a sidebar panel for interacting with CodeSailor, so that I can access all features without leaving my editor.

**Type:** Functional

#### Acceptance Criteria

8.1. Register sidebar Webview panel on activation with 3 tabs: - Chat (context-aware Q&A) - Navigator (file search) - Impact Analysis (dependency visualization)

8.2. Display "Get Started" button on first open to generate architecture overview

8.3. Process chat queries and display responses with clickable file links

8.4. Render code snippets with syntax highlighting matching source language

8.5. Show real-time indexing progress in status bar

### FR-9: Configuration Management

**User Story:** As a developer, I want to configure AWS credentials, so that CodeSailor can access Bedrock for embedding and generation.

**Type:** Functional

#### Acceptance Criteria

9.1. Prompt for AWS credentials on first activation: - Access Key ID - Secret Access Key - AWS Region

9.2. Validate credentials via test API call to Bedrock before storing

9.3. Display clear error messages with troubleshooting steps for invalid credentials

9.4. Store validated credentials securely using VS Code SecretStorage API

9.5. Provide "Reconfigure AWS Credentials" command for credential updates

## Non-Functional Requirements

### NFR-1: Performance - Non-Blocking Indexing

**User Story:** As a developer, I want indexing to happen in the background, so that my editor remains responsive while working.

**Type:** Non-Functional (Performance)

#### Acceptance Criteria

NFR-1.1. Spawn Worker Thread for all indexing operations (non-blocking)

NFR-1.2. Allow normal editor operations to continue during indexing

NFR-1.3. Send progress updates from Worker Thread to main thread per file completion

NFR-1.4. Display indexing progress in status bar in real-time

NFR-1.5. Show completion notification when indexing finishes

### NFR-2: Performance - Fast Search

**User Story:** As a developer, I want search results to appear instantly, so that my workflow is not interrupted.

**Type:** Non-Functional (Performance)

#### Acceptance Criteria

NFR-2.1. Use FAISS vector extensions for fast similarity search

NFR-2.2. Return top 10 search results within 200ms

NFR-2.3. Display results immediately upon search completion

NFR-2.4. Maintain sub-200ms performance with 100,000+ indexed chunks

NFR-2.5. Log performance metrics when search latency degrades

### NFR-3: Security - Local-First Privacy

**User Story:** As a developer working on proprietary code, I want embeddings stored locally, so that my code never leaves my machine except for ephemeral queries.

**Type:** Non-Functional (Security/Privacy)

#### Acceptance Criteria

NFR-3.1. Send code chunks to AWS Bedrock Titan Embeddings V2 for embedding generation

NFR-3.2. Store all embeddings in local FAISS index (no cloud storage)

NFR-3.3. Perform searches using local index only (no network calls)

NFR-3.4. Send only minimal context (top-K chunks) to Bedrock for AI generation

NFR-3.5. Provide option to delete all local embeddings on extension uninstall

### NFR-4: Security - Sensitive Data Filtering

**User Story:** As a developer, I want the extension to automatically exclude sensitive files, so that API keys and credentials are never sent to the cloud.

**Type:** Non-Functional (Security)

#### Acceptance Criteria

NFR-4.1. Automatically skip sensitive file patterns: - Environment files: .env, .env._ - Certificates: .pem, .key, .cert - Secrets: secrets._, credentials.\*

NFR-4.2. Apply regex filters to detect sensitive content in files: - API tokens and keys - Passwords and credentials - Private keys

NFR-4.3. Exclude files with detected sensitive content and log warning

NFR-4.4. Inform user when querying excluded files ("Excluded for security")

NFR-4.5. Support custom exclusion patterns via user configuration

### NFR-5: Efficiency - Hash-Based Caching

**User Story:** As a developer, I want unchanged files to skip re-embedding, so that indexing is fast and cost-efficient.

**Type:** Non-Functional (Efficiency/Cost)

#### Acceptance Criteria

NFR-5.1. Compute SHA-256 hash of file content during indexing

NFR-5.2. Check for cached embeddings matching the computed hash

NFR-5.3. Reuse cached embeddings when hash matches (skip Bedrock API call)

NFR-5.4. Detect file modifications via hash changes and trigger re-embedding

NFR-5.5. Implement LRU eviction when cache exceeds 1GB storage limit

### NFR-6: Efficiency - Model Routing

**User Story:** As a developer, I want the extension to use cost-effective models, so that I can minimize AWS costs while maintaining quality.

**Type:** Non-Functional (Efficiency/Cost)

#### Acceptance Criteria

NFR-6.1. Use AWS Bedrock Titan Embeddings V2 for all embedding generation

NFR-6.2. Use Claude Sonnet 4.5 for all AI-generated chat responses

NFR-6.3. Include model-specific parameters in all Bedrock API requests

NFR-6.4. Log errors and notify user when models are unavailable

NFR-6.5. Allow user override of default model selection via configuration

### NFR-7: Reliability - Graceful Degradation

**User Story:** As a developer, I want the extension to work offline for cached queries, so that temporary network issues don't block my work.

**Type:** Non-Functional (Reliability)

#### Acceptance Criteria

NFR-7.1. Detect network failures when Bedrock API is unreachable

NFR-7.2. Continue providing local search results from Vector Store when offline

NFR-7.3. Display clear error message for AI generation requests while offline: - Explain cloud features are unavailable - Indicate local search still works

NFR-7.4. Automatically resume cloud features when connectivity is restored

NFR-7.5. Display degraded mode indicator in Webview status area
