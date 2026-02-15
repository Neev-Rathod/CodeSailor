# CodeSailor 🧭

**Stop drowning in code. Start navigating with AI.**

CodeSailor is a VS Code extension that transforms how developers explore unfamiliar codebases. Using RAG (Retrieval-Augmented Generation) powered by AWS Bedrock, it creates an instant "map" of any project—eliminating the "Lost at Sea" effect that slows down onboarding.

---

## 🎯 The Problem

- **70% of developer time** spent reading and tracing code
- **3-6 months** before new hires become productive
- **Documentation is always outdated** and can't answer "How does this _actually_ work?"

## ✨ The Solution

CodeSailor indexes your **entire repository** (not just active files) and lets you query it like a knowledge base:

> **"Trace the payment flow"** → See exact file paths, dependency chains, and entry points  
> **"Where is authentication handled?"** → Get relevant files with context snippets  
> **"What breaks if I edit this?"** → Visualize impact across dependent services

---

## 🚀 Core Features

### 1️⃣ **Get Started & Architecture Overview**

Generate a README-style project explanation with one click. Understand the tech stack, major components, and data flow in **minutes instead of days**.

### 2️⃣ **Smart File Navigator**

Describe what you're looking for in plain language:

- _"Where is the payment logic?"_ → Opens `payment.service.ts`
- _"Find database connection setup"_ → Shows `db-config.ts` with context

### 3️⃣ **Impact Analysis Graph** (🔥 Unique Feature)

See which modules depend on the file you're editing. **Prevents breaking changes** by warning you before you save:

> ⚠️ _This change affects 7 other files_

### 4️⃣ **Context-Aware Chat**

Ask questions based on **your project's code patterns**:

- "How do I add a new API endpoint in this framework?"
- "What's the authentication flow for protected routes?"
- "Explain this service's role in the system"

---

## 🛠️ How It Works

```
┌─────────────────────────────────────────────────┐
│  1. Index Project (background, non-blocking)   │
│     • Scans workspace files                     │
│     • Chunks code at function/class boundaries  │
│     • Generates embeddings (AWS Bedrock Titan)  │
│     • Stores vectors locally (privacy-first)    │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  2. Query with Natural Language                 │
│     • Search via semantic similarity            │
│     • Retrieve relevant code chunks             │
│     • Generate answers (Claude Sonnet 4.5)      │
└─────────────────────────────────────────────────┘
                      ↓
┌─────────────────────────────────────────────────┐
│  3. Navigate & Understand Instantly             │
│     • Clickable file references                 │
│     • Dependency visualization                  │
│     • Impact warnings on edits                  │
└─────────────────────────────────────────────────┘
```

---

## 🔐 Privacy & Cost

- **Local-First**: Embeddings stored on your machine, not the cloud
- **Minimal Cloud Usage**: Only sends code chunks for queries (not entire files)
- **Smart Caching**: ~90% reduction in API calls via hash-based caching
- **Estimated Cost**: $0.10-$0.50 per 1000 files indexed (one-time), $0.01-$0.05 per query

---

## 🎯 USP (Unique Selling Proposition)

Unlike GitHub Copilot or ChatGPT (which focus on **code generation**), CodeSailor focuses on **knowledge transfer**:

| Tool           | Focus                | Context            | Use Case                |
| -------------- | -------------------- | ------------------ | ----------------------- |
| **Copilot**    | Code completion      | Current file only  | Writing new code        |
| **ChatGPT**    | General Q&A          | No project context | Generic questions       |
| **CodeSailor** | System understanding | Entire repository  | Onboarding & navigation |

**What makes us different:**

1. **Predictive Impact Analysis** - Warns you what breaks when you edit
2. **Task-to-File Mapping** - Describe task → Get exact files to modify

---

## 🚦 Getting Started

### Prerequisites

- VS Code 1.85+
- AWS Account with Bedrock access (Free tier available)
- AWS credentials (Access Key ID, Secret, Region)

### Installation

1. Install extension from VS Code Marketplace (coming soon)
2. Configure AWS credentials when prompted
3. Open any project and click **"Get Started"** in the CodeSailor sidebar

### First Use

```typescript
// 1. CodeSailor indexes your project automatically (background task)
// 2. Click "Get Started" to generate architecture overview
// 3. Try these queries in the Chat tab:
//    - "Explain how authentication works"
//    - "Where is error handling implemented?"
//    - "Show me all API endpoints"
```

---

## 🏗️ Technology Stack

- **Frontend**: VS Code Webview (React)
- **Backend**: TypeScript/Node.js
- **AI Models**:
  - AWS Bedrock Titan Embeddings V2 (code embeddings)
  - AWS Bedrock Claude Sonnet 4.5 (response generation)
- **Vector Store**: SQLite with vector extension (local storage)
- **Code Parsing**: Tree-sitter (AST-based chunking)

---

## 📊 Performance

- **Search**: <300ms across 100,000 code chunks
- **AI Responses**: <5s end-to-end
- **Indexing**: Active file indexed in <1s (background indexing continues)
- **Offline Mode**: Local search works without internet

---

## 🗺️ Roadmap

**MVP (Current Focus)**:

- ✅ Smart Project Indexing
- ✅ Context-Aware Chat
- ✅ Smart File Navigator
- ✅ Impact Analysis Graph
- ✅ Architecture Overview

**Future Enhancements**:

- Tutorial generation ("How do I add a new feature?")
- CodeLens inline decorators (reference counts)
- Multi-repository support
- Team knowledge sharing
- Custom model fine-tuning

---

## 🤝 Contributing

See [design.md](design.md) for architecture details and [requirements.md](requirements.md) for functional specifications.

---

## 💬 Support

- **Issues**: GitHub Issues (coming soon)
- **Docs**: See [design.md](design.md) and [requirements.md](requirements.md)

---

**Built for developers who are tired of being lost at sea in unfamiliar codebases.** 🌊⚓
