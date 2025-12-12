# OpenCode Architecture Overview

**Date:** 2025-12-12
**Version:** 0.12.1
**Repository:** https://github.com/sst/opencode

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Core Components](#core-components)
3. [Package Structure](#package-structure)
4. [Data Flow](#data-flow)
5. [Technology Stack](#technology-stack)
6. [Key Design Patterns](#key-design-patterns)
7. [Extension Points](#extension-points)

---

## High-Level Architecture

OpenCode follows a **client-server architecture** with multiple frontend options:

```
┌─────────────────────────────────────────────────────────────┐
│                        FRONTENDS                             │
├──────────────┬──────────────┬───────────────┬───────────────┤
│   TUI (Go)   │  CLI (Bun)   │  Web (Solid)  │  SDK (TS/Go)  │
└──────┬───────┴──────┬───────┴───────┬───────┴───────┬───────┘
       │              │               │               │
       └──────────────┴───────┬───────┴───────────────┘
                              │
                    ┌─────────▼──────────┐
                    │   API SERVER       │
                    │   (Hono + Bun)     │
                    └─────────┬──────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
    ┌───────▼────────┐ ┌─────▼──────┐  ┌──────▼────────┐
    │   Session      │ │  Provider  │  │  Storage      │
    │   Manager      │ │  Registry  │  │  (Local)      │
    └───────┬────────┘ └─────┬──────┘  └───────────────┘
            │                │
    ┌───────▼────────────────▼──────────────────────┐
    │          TOOL EXECUTION LAYER                 │
    ├────┬────┬────┬────┬────┬────┬────┬────┬──────┤
    │Bash│Read│Edit│Grep│Glob│Web │MCP │Task│...   │
    └────┴────┴────┴────┴────┴────┴────┴────┴──────┘
            │
    ┌───────▼────────────────────────────────────┐
    │       PROJECT WORKSPACE                    │
    │   (Git-tracked, sandboxed environment)     │
    └────────────────────────────────────────────┘
```

---

## Core Components

### 1. Entry Point (`index.ts`)

**Purpose:** CLI argument parsing and command routing.

**Key Features:**
- Yargs-based command-line interface
- Global error handling (unhandled rejections, uncaught exceptions)
- Logging initialization
- Version and help display

**Available Commands:**
```typescript
- tui         // Launch terminal UI
- run         // Run a prompt directly
- generate    // Code generation
- serve       // Start API server
- auth        // Manage credentials
- agent       // Manage agents
- models      // List available models
- mcp         // MCP server management
- github      // GitHub integration
- stats       // Usage statistics
- export      // Export conversations
- attach      // Attach to running session
- debug       // Debug utilities
- upgrade     // Auto-update opencode
```

---

### 2. Session Management (`session/`)

**Purpose:** Orchestrates AI conversations and manages state.

**Files:**
- `index.ts` - Main session orchestration
- `message.ts` / `message-v2.ts` - Message handling
- `prompt.ts` - System prompts (50KB+ of carefully crafted instructions)
- `system.ts` - System integration
- `compaction.ts` - Message history compaction
- `revert.ts` - Change rollback

**Key Responsibilities:**
- AI model communication (via Vercel AI SDK)
- Tool execution coordination
- Conversation history management
- Context window management (compaction)
- Snapshot/rollback integration

**Architecture Pattern:**
```typescript
Session {
  - id: string
  - messages: Message[]
  - agent: Agent
  - provider: Provider

  async run(prompt: string) {
    while (!done) {
      response = await provider.generate(messages, tools)
      if (response.toolCalls) {
        results = await executeTools(response.toolCalls)
        messages.push(toolResults)
      } else {
        break
      }
    }
  }
}
```

---

### 3. Provider System (`provider/`)

**Purpose:** Abstracts LLM provider differences.

**Files:**
- `provider.ts` - Main provider interface and registry
- `models.ts` - Model definitions and mappings
- `transform.ts` - Request/response transformations

**Supported Providers:** 75+ including:
- Anthropic (Claude)
- OpenAI (GPT)
- Google (Gemini)
- Amazon Bedrock
- Azure OpenAI
- GitHub Copilot
- Groq
- DeepSeek
- Cerebras
- Local models (Ollama, LM Studio)
- Custom OpenAI-compatible APIs

**Provider Interface:**
```typescript
interface Provider {
  id: string
  name: string
  models: Map<string, Model>
  options: {
    apiKey?: string
    baseURL?: string
    headers?: Record<string, string>
  }
}
```

---

### 4. Tool System (`tool/`)

**Purpose:** Provides AI-accessible functions for code manipulation and information retrieval.

#### Available Tools:

**File Operations:**
- `read.ts` - Read file contents (with line ranges, image support)
- `write.ts` - Create new files
- `edit.ts` - Modify existing files (exact string replacement)
- `multiedit.ts` - Edit multiple files at once
- `patch.ts` - Apply unified diff patches

**Code Navigation:**
- `glob.ts` - Pattern-based file finding (`**/*.ts`)
- `grep.ts` - Content search (ripgrep integration)
- `ls.ts` - Directory listing
- `lsp-diagnostics.ts` - Language server diagnostics
- `lsp-hover.ts` - Symbol information via LSP

**Execution:**
- `bash.ts` - Sandboxed command execution
- `test.ts` - Test runner integration

**Information:**
- `webfetch.ts` - Fetch and process web pages
- `task.ts` - Launch sub-agents for complex tasks
- `todo.ts` - Task tracking/management

**Registry:**
- `registry.ts` - Tool registration and management
- `tool.ts` - Base tool interface

**Tool Interface:**
```typescript
interface Tool<TInput, TOutput> {
  name: string
  description: string
  parameters: JSONSchema
  execute(input: TInput, ctx: Context): Promise<TOutput>
}
```

#### Tool Description Files

Each tool has a `.txt` file (e.g., `bash.txt`, `edit.txt`) containing:
- Natural language description for the AI
- Usage guidelines
- Security constraints
- Examples

These are injected into the system prompt.

---

### 5. Configuration System (`config/`)

**Purpose:** Multi-source, hierarchical configuration management.

**Files:**
- `config.ts` - Configuration loader and validator
- `markdown.ts` - Markdown file reference parser (`@file/path` syntax)

**Configuration Sources (in order of precedence):**
1. `$OPENCODE_CONFIG` environment variable
2. `./opencode.json` or `./opencode.jsonc` (project-level)
3. `./.opencode/agent/*.md` and `./.opencode/command/*.md`
4. `~/.config/opencode/opencode.json` (global)
5. `~/.config/opencode/agent/*.md` and `~/.config/opencode/command/*.md`

**Key Features:**
- Variable substitution: `{env:VAR}`, `{file:path}`
- JSON and JSONC support
- Schema validation (Zod)
- Migration from legacy formats
- Agent and command definitions
- Provider customization

**Example Config:**
```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-4-20250514",
  "small_model": "anthropic/claude-3-5-haiku-20241022",
  "provider": {
    "anthropic": {
      "options": {
        "apiKey": "{env:ANTHROPIC_API_KEY}"
      }
    }
  },
  "agent": {
    "code-reviewer": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "You are a code reviewer...",
      "tools": {
        "write": false,
        "edit": false
      }
    }
  }
}
```

---

### 6. Snapshot System (`snapshot/`)

**Purpose:** Git-based change tracking for safe AI operations.

**Key Functions:**
- `track()` - Create snapshot hash (git commit hash)
- `patch(hash)` - List changed files since snapshot
- `diff(hash)` - Show textual diff
- `revert(patches)` - Undo changes
- `restore(hash)` - Full restore to snapshot

**How It Works:**
1. Before AI operations, take snapshot: `hash = await Snapshot.track()`
2. AI makes changes (creates/modifies/deletes files)
3. Track what changed: `patch = await Snapshot.patch(hash)`
4. If problems, rollback: `await Snapshot.revert([patch])`

**Implementation:**
- Uses Git under the hood
- No extra dependencies
- Handles edge cases (unicode, symlinks, large files)
- Isolated per project
- Tested extensively (30+ test cases)

---

### 7. Project Management (`project/`)

**Purpose:** Manages project context and workspace.

**Files:**
- `project.ts` - Project discovery and metadata
- `instance.ts` - Project instance and state management
- `state.ts` - Project state persistence
- `bootstrap.ts` - Project initialization

**Key Responsibilities:**
- Git repository detection
- Workspace boundaries
- `.gitignore` respecting
- Project-specific configuration
- Working directory management

---

### 8. Authentication (`auth/`)

**Purpose:** Credential storage and provider authentication.

**Storage Location:** `~/.local/share/opencode/auth.json`

**Supported Auth Flows:**
- API key (manual entry or generated)
- OAuth Device Flow (GitHub Copilot, Anthropic Pro)
- Browser-based auth
- Environment variables

**Security:**
- Credentials stored locally (not in git)
- Per-provider isolation
- Support for `{file:path}` for external secrets

---

### 9. MCP Integration (`mcp/`)

**Purpose:** Model Context Protocol server integration.

**Capabilities:**
- Load external MCP servers
- Expose tools to AI
- Resource management
- Server lifecycle management

**Example MCP Servers:**
- Filesystem
- Git operations
- Database queries
- API clients

---

### 10. Server (`server/`)

**Purpose:** HTTP API for remote access.

**Framework:** Hono (fast, lightweight, Bun-optimized)

**Endpoints:**
- Session management
- Streaming responses
- Tool execution
- Configuration
- Statistics

**Use Cases:**
- Remote TUI access
- Web interface backend
- SDK integration
- Multi-client support

---

### 11. CLI Commands (`cli/`)

**Purpose:** User-facing command implementations.

**Structure:**
```
cli/
├── cmd/
│   ├── run.ts         - Direct prompt execution
│   ├── tui.ts         - Launch terminal UI
│   ├── serve.ts       - API server
│   ├── auth.ts        - Credential management
│   ├── agent.ts       - Agent operations
│   ├── models.ts      - Model listing
│   ├── mcp.ts         - MCP server management
│   ├── github.ts      - GitHub integration
│   ├── export.ts      - Export conversations
│   ├── stats.ts       - Usage analytics
│   ├── attach.ts      - Attach to session
│   ├── debug/         - Debug utilities
│   └── ...
├── ui.ts              - UI utilities (logo, formatting)
├── error.ts           - Error formatting
└── bootstrap.ts       - CLI initialization
```

---

## Package Structure

### Monorepo Layout

```
opencode/
├── packages/
│   ├── opencode/          # Main CLI package ⭐
│   │   ├── src/           # TypeScript source
│   │   ├── test/          # Test suite
│   │   └── bin/opencode   # CLI entry point
│   │
│   ├── tui/               # Terminal UI (Go)
│   │   └── internal/      # Go implementation
│   │
│   ├── web/               # Web interface (Solid)
│   │   ├── src/           # SolidJS components
│   │   └── astro.config.ts
│   │
│   ├── app/               # Desktop app (future)
│   │
│   ├── sdk/               # Client SDKs
│   │   ├── js/            # JavaScript/TypeScript SDK
│   │   └── go/            # Go SDK (future)
│   │
│   ├── console/           # Admin console
│   │   ├── app/           # Console frontend
│   │   ├── core/          # Shared logic
│   │   ├── function/      # Serverless functions
│   │   └── resource/      # Infrastructure
│   │
│   ├── function/          # Shared functions
│   ├── plugin/            # Plugin system
│   └── identity/          # Auth/identity
│
├── infra/                 # SST infrastructure
├── specs/                 # API specifications
├── patches/               # NPM package patches
└── scripts/               # Build scripts
```

### Key Directories in `packages/opencode/src/`

```
src/
├── index.ts               # Entry point
├── cli/                   # CLI commands
├── session/               # AI session orchestration
├── tool/                  # AI tools (18+ tools)
├── provider/              # LLM provider integrations
├── config/                # Configuration management
├── snapshot/              # Git-based change tracking
├── project/               # Project management
├── auth/                  # Authentication
├── mcp/                   # MCP integration
├── server/                # HTTP API
├── agent/                 # Agent definitions
├── command/               # Command definitions
├── storage/               # Local storage
├── share/                 # Conversation sharing
├── plugin/                # Plugin system
├── permission/            # Permission system
├── format/                # Code formatters
├── file/                  # File utilities
├── lsp/                   # LSP client
├── ide/                   # IDE integration
├── bus/                   # Event bus
├── flag/                  # Feature flags
├── global/                # Global state
├── id/                    # ID generation
├── installation/          # Installation management
├── bun/                   # Bun-specific utilities
└── util/                  # Utilities
```

---

## Data Flow

### Typical Request Flow

```
1. User Input
   │
   ├─> CLI (index.ts)
   │   └─> Command (e.g., run.ts)
   │       └─> Session.create()
   │
2. Session Initialization
   │
   ├─> Load Config (config.ts)
   ├─> Initialize Provider (provider.ts)
   ├─> Load Tools (registry.ts)
   ├─> Create Snapshot (snapshot.ts)
   │
3. AI Conversation Loop
   │
   ├─> Format Messages (message.ts)
   ├─> Add System Prompt (prompt.ts)
   ├─> Call LLM (AI SDK)
   │   └─> Provider API
   │
   ├─> Receive Response
   │   │
   │   ├─> Text? → Display to user
   │   │
   │   └─> Tool Calls?
   │       └─> Execute Tools (tool/*)
   │           ├─> bash.ts
   │           ├─> edit.ts
   │           ├─> read.ts
   │           └─> ...
   │       └─> Add Results to Messages
   │       └─> Loop back to LLM
   │
4. Completion
   │
   ├─> Save Session (storage.ts)
   ├─> Show Summary
   └─> Exit
```

### Message Flow

```typescript
[
  {
    role: "system",
    content: "You are an AI coding assistant..."
  },
  {
    role: "user",
    content: "Fix the bug in auth.ts"
  },
  {
    role: "assistant",
    content: null,
    tool_calls: [
      {
        name: "read",
        input: { file_path: "/path/to/auth.ts" }
      }
    ]
  },
  {
    role: "tool",
    tool_call_id: "...",
    content: "// file contents..."
  },
  {
    role: "assistant",
    content: null,
    tool_calls: [
      {
        name: "edit",
        input: {
          file_path: "/path/to/auth.ts",
          old_string: "buggy code",
          new_string: "fixed code"
        }
      }
    ]
  },
  {
    role: "tool",
    tool_call_id: "...",
    content: "Success"
  },
  {
    role: "assistant",
    content: "I've fixed the bug by..."
  }
]
```

---

## Technology Stack

### Backend/CLI

**Runtime:**
- Bun (JavaScript/TypeScript runtime)
- Node.js compatibility layer

**Frameworks:**
- Hono (web server)
- Yargs (CLI)
- Vercel AI SDK (LLM integration)

**Validation:**
- Zod (schema validation)

**Database:**
- PlanetScale (MySQL) - for cloud features
- Local JSON - for CLI storage

**Storage:**
- Local filesystem (`~/.local/share/opencode/`, `~/.config/opencode/`)
- Git (for change tracking)

### Terminal UI

**Language:** Go 1.24.x

**Package:** `packages/tui/`

**Features:**
- Native TUI rendering
- Streaming support
- Performance optimized

### Web Interface

**Framework:** SolidJS

**Build Tools:**
- Vite
- Astro (static site generation)

**Styling:**
- TailwindCSS
- Custom components (@kobalte/core)

**Routing:**
- @solidjs/router

### Infrastructure

**IaC:** SST (Serverless Stack)

**Hosting:**
- Cloudflare (Workers, Pages)
- AWS (via SST)

**Services:**
- Stripe (payments)
- PlanetScale (database)

---

## Key Design Patterns

### 1. Provider Pattern

Abstracts LLM provider differences:

```typescript
interface Provider {
  generate(messages, tools) => Stream<Response>
}

// Usage
const response = await provider.generate(messages, tools)
```

### 2. Tool Registry Pattern

Dynamically registered, AI-accessible functions:

```typescript
const registry = new ToolRegistry()
registry.register(BashTool)
registry.register(EditTool)
// ...

const tools = registry.getAllForAgent(agent)
```

### 3. Snapshot Pattern

Git-based time-travel for safe AI operations:

```typescript
const before = await Snapshot.track()
// AI makes changes
const patch = await Snapshot.patch(before)
if (hasProblems) {
  await Snapshot.revert([patch])
}
```

### 4. Context Provider Pattern

Scoped context for project operations:

```typescript
await Instance.provide({
  directory: "/path/to/project",
  fn: async () => {
    // All operations use this project context
    const config = await Config.get()
  }
})
```

### 5. Event Bus Pattern

Decoupled communication between components:

```typescript
Bus.emit("tool.executed", { tool, result })
Bus.on("tool.executed", (data) => {
  // Handle event
})
```

### 6. Middleware Pattern

Request/response transformation:

```typescript
app.use(async (c, next) => {
  // Before request
  await next()
  // After request
})
```

---

## Extension Points

### 1. Custom Tools

Create new AI-accessible functions:

```typescript
// my-tool.ts
export const MyTool = {
  name: "my_tool",
  description: "Does something useful",
  parameters: z.object({
    input: z.string()
  }),
  async execute(input, ctx) {
    // Implementation
    return { result: "..." }
  }
}

// Register
registry.register(MyTool)
```

### 2. Custom Agents

Define specialized AI agents:

```json
{
  "agent": {
    "my-agent": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "You are a specialized agent for...",
      "temperature": 0.3,
      "tools": {
        "bash": false,
        "write": false
      }
    }
  }
}
```

Or via markdown:

```markdown
<!-- .opencode/agent/my-agent.md -->
---
model: anthropic/claude-sonnet-4-20250514
temperature: 0.3
tools:
  bash: false
---

You are a specialized agent for...
```

### 3. Custom Commands

Add shortcuts for common tasks:

```json
{
  "command": {
    "review": {
      "template": "Review the code changes and suggest improvements",
      "agent": "code-reviewer"
    }
  }
}
```

### 4. Custom Providers

Add custom LLM providers:

```json
{
  "provider": {
    "my-provider": {
      "npm": "@ai-sdk/openai-compatible",
      "options": {
        "baseURL": "https://api.my-provider.com/v1",
        "apiKey": "{env:MY_API_KEY}"
      },
      "models": {
        "my-model": {
          "name": "My Model"
        }
      }
    }
  }
}
```

### 5. MCP Servers

Integrate external tools via Model Context Protocol:

```json
{
  "mcp": {
    "my-server": {
      "command": "npx",
      "args": ["my-mcp-server"]
    }
  }
}
```

### 6. Plugins

(Future) Full plugin system for extending OpenCode.

---

## Code Style Guidelines

From `AGENTS.md`:

**DO:**
- ✓ Keep things in one function unless composable/reusable
- ✓ Use Bun APIs (Bun.file(), $\`command\`, etc.)
- ✓ Prefer single-word variable names
- ✓ Use `const` over `let`

**AVOID:**
- ✗ Unnecessary destructuring
- ✗ `else` statements unless necessary
- ✗ `try`/`catch` if avoidable
- ✗ `any` type
- ✗ `let` statements

---

## Performance Considerations

### 1. Bun Runtime
- Fast startup time
- Native TypeScript execution
- Built-in SQLite, test runner, bundler

### 2. Streaming
- Tool results streamed to user
- LLM responses streamed
- Large files handled in chunks

### 3. Context Window Management
- Message compaction (compaction.ts)
- Smart truncation
- Tool result summarization

### 4. Caching
- Configuration caching
- Provider instance reuse
- Tool registry singleton

---

## Security Considerations

### 1. Sandboxing
- Bash commands restricted to project directory
- No directory traversal outside project
- Path validation on all file operations

### 2. Credential Storage
- Local storage only (`~/.local/share/opencode/`)
- No credentials in git
- Support for external secret files

### 3. Snapshot Safety
- All changes tracked
- Easy rollback
- Git-based verification

### 4. Permission System
- Tool access control per agent
- Configurable permissions
- User approval prompts (optional)

---

## Development Workflow

### Local Development

```bash
# Install dependencies
bun install

# Run in development mode
cd packages/opencode
bun dev [arguments]

# Run tests
bun test

# Type checking
bun turbo typecheck

# Build
bun run build
```

### Testing

```bash
# Unit tests
cd packages/opencode
bun test

# Specific test file
bun test test/config/config.test.ts

# With coverage
bun test --coverage
```

### Deployment

```bash
# Deploy infrastructure
npx sst deploy --stage production

# Publish NPM package
npm publish
```

---

## Future Architecture Considerations

### Planned Features

1. **Desktop App** - Electron or Tauri wrapper
2. **Mobile Support** - React Native or PWA
3. **Plugin Marketplace** - Community extensions
4. **Team Collaboration** - Shared sessions
5. **Cloud Sync** - Cross-device history
6. **Advanced Agents** - Multi-agent orchestration

### Scalability

- Horizontal scaling via API server
- Session persistence in database
- Tool execution queue
- Rate limiting and quotas

---

## Conclusion

OpenCode's architecture is **modular, extensible, and security-focused**. Key strengths:

✅ **Clean separation of concerns**
- CLI, API, TUI, Web are independent
- Tools are self-contained
- Providers are pluggable

✅ **Safety first**
- Git-based change tracking
- Sandboxed execution
- Rollback capabilities

✅ **Developer experience**
- Bun-first for performance
- TypeScript for type safety
- Comprehensive configuration

✅ **Extensibility**
- Custom tools, agents, commands
- MCP integration
- Plugin system (planned)

The codebase is well-organized, follows modern patterns, and is designed for both stability and rapid iteration.

---

**Architecture Review Completed:** 2025-12-12
**Total Source Files:** ~100+ TypeScript files
**Total Tools:** 18+ AI-accessible functions
**Lines of Code:** ~15,000+ (excluding tests and dependencies)
**Test Coverage:** Strong (50+ test cases, focus on core functionality)
