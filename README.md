# MCP AI Server for Visual Studio

**Other tools read your files. MCP AI Server understands your code.**

[![Install from Marketplace](https://img.shields.io/badge/Install-VS%20Marketplace-purple.png)](https://marketplace.visualstudio.com/items?itemName=LadislavSopko.mcpserverforvs)
[![VS 2022](https://img.shields.io/badge/VS-2022%20%7C%202026-purple.png)](https://visualstudio.microsoft.com/)
[![Free](https://img.shields.io/badge/Price-Free-green.png)](https://marketplace.visualstudio.com/items?itemName=LadislavSopko.mcpserverforvs)
[![Tools](https://img.shields.io/badge/MCP%20Tools-20-blue.png)](https://modelcontextprotocol.io/)
[![Website](https://img.shields.io/badge/Website-0ics.com-blue.png)](https://www.0ics.com/static/vs-mcp.html)

The first and only Visual Studio extension that gives AI assistants access to the C# compiler (Roslyn) through the Model Context Protocol. 20 tools. 12 powered by Roslyn. Semantic understanding, not text matching.

## Install

**[Install from Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=LadislavSopko.mcpserverforvs)**

Or search for **"MCP AI Server"** in Visual Studio Extensions Manager.

### See it in Action

[![Watch Demo Video](https://www.0ics.com/static/youtube-min.jpeg)](https://youtu.be/fYV4oqY9n5g)

*Click to watch on YouTube*

---

## Why MCP AI Server?

AI coding tools (Claude Code, Codex CLI, Gemini CLI, OpenCode, Cursor, Copilot, Windsurf...) operate at the filesystem level — they read text, run grep, execute builds. They don't understand your code the way Visual Studio does.

MCP AI Server bridges this gap by exposing IntelliSense-level intelligence as MCP tools. Your AI assistant gets the same semantic understanding that powers F12 (Go to Definition), Shift+F12 (Find All References), and safe refactoring.

### What becomes possible:

| You ask | Without MCP AI Server | With MCP AI Server |
|---------|---------------|-------------|
| "Find WhisperFactory" | grep returns 47 matches | `Class, Whisper.net, line 16` — one exact answer |
| "Rename ProcessDocument" | sed breaks ProcessDocumentAsync | Roslyn renames 23 call sites safely |
| "What implements IDocumentService?" | Impossible via grep | Full inheritance tree with interfaces |
| "What calls AuthenticateUser?" | Text matches, can't tell direction | Precise call graph: callers + callees |

---

## 20 Tools

### Semantic Navigation (Roslyn-powered)
| Tool | Description |
|------|-------------|
| `FindSymbols` | Find classes, methods, properties by name — semantic, not text |
| `FindSymbolDefinition` | Go to definition (F12 equivalent) |
| `FindSymbolUsages` | Find all references, compiler-verified (Shift+F12 equivalent) |
| `GetSymbolAtLocation` | Identify the symbol at a specific line and column |
| `GetDocumentOutline` | Semantic structure: classes, methods, properties, fields |

### Code Understanding (Roslyn-powered)
| Tool | Description |
|------|-------------|
| `GetInheritance` | Full type hierarchy: base types, derived types, interfaces |
| `GetMethodCallers` | Which methods call this method (call graph UP) |
| `GetMethodCalls` | Which methods this method calls (call graph DOWN) |

### Refactoring (Roslyn-powered)
| Tool | Description |
|------|-------------|
| `RenameSymbol` | Safe rename across the entire solution — compiler-verified |
| `FormatDocument` | Visual Studio's native code formatter |

### Project & Build
| Tool | Description |
|------|-------------|
| `ExecuteCommand` | Build or clean solution/project with structured diagnostics |
| `ExecuteAsyncTest` | Run tests asynchronously with real-time status |
| `GetSolutionTree` | Solution and project structure |
| `GetProjectReferences` | Project dependency graph |
| `TranslatePath` | Convert paths between Windows and WSL formats |

### Editor Integration
| Tool | Description |
|------|-------------|
| `GetActiveFile` | Current file and cursor position |
| `GetSelection` / `CheckSelection` | Read active text selection |
| `GetLoggingStatus` / `SetLogLevel` | Extension diagnostics |

---

## Compatible Clients

Works with any MCP-compatible AI tool:

**CLI Agents:**
- **Claude Code** — Anthropic's terminal AI coding agent
- **Codex CLI** — OpenAI's terminal coding agent
- **Gemini CLI** — Google's open-source terminal agent
- **OpenCode** — Open-source AI coding agent (45k+ GitHub stars)
- **Goose** — Block's open-source AI agent
- **Aider** — AI pair programming in terminal

**Desktop & IDE:**
- **Claude Desktop** — Anthropic's desktop app
- **Cursor** — AI-first code editor
- **Windsurf** — Codeium's AI IDE
- **VS Code + Copilot** — GitHub Copilot with MCP
- **Cline** — VS Code extension
- **Continue** — Open-source AI assistant

**Any MCP client** — open protocol, universal compatibility

---

## Quick Start

1. **[Install](https://marketplace.visualstudio.com/items?itemName=LadislavSopko.mcpserverforvs)** from Visual Studio Marketplace
2. **Open** your .NET solution in Visual Studio
3. **Configure port** in MCP Server Settings (default: 3010)
4. **Add to your MCP client:**

```json
"vs-mcp": {
  "type": "http",
  "url": "http://localhost:3010/sdk/"
}
```

5. **Start** asking your AI semantic questions about your code

---

## MCP Client Configuration

| AI Tool | Type | Config File |
|---------|------|-------------|
| **[Claude Code](https://claude.ai/code)** | CLI | `CLAUDE.md` |
| **[Claude Desktop](https://claude.ai/download)** | App | `claude_desktop_config.json` |
| **[Cursor](https://cursor.sh)** | IDE | `.cursor/rules/*.mdc` or `.cursorrules` |
| **[Windsurf](https://codeium.com/windsurf)** | IDE | `.windsurfrules` |
| **[Cline](https://cline.bot)** | VS Code Extension | `.clinerules/` directory |
| **[VS Code + Copilot](https://code.visualstudio.com/)** | IDE | `.github/copilot-instructions.md` |
| **[Continue](https://continue.dev)** | IDE Extension | `.continue/config.json` |
| **[Gemini CLI](https://github.com/google-gemini/gemini-cli)** | CLI | `GEMINI.md` |
| **[OpenAI Codex CLI](https://github.com/openai/codex)** | CLI | `AGENTS.md` |
| **[Goose](https://github.com/block/goose)** | CLI | `.goose/config.yaml` |

Any tool supporting [Model Context Protocol](https://modelcontextprotocol.io/) will work.

---

## Configure AI Preferences (Recommended)

Add these instructions to your project's AI config file (see table above) to ensure your AI **automatically** prefers MCP tools:

```markdown
## MCP Tools - ALWAYS PREFER

When `mcp__vs-mcp__*` tools are available, ALWAYS use them instead of Grep/Glob/LS:

| Instead of | Use |
|------------|-----|
| `Grep` for symbols | `FindSymbols`, `FindSymbolUsages` |
| `LS` to explore projects | `GetSolutionTree` |
| Reading files to find code | `FindSymbolDefinition` then `Read` |
| Searching for method calls | `GetMethodCallers`, `GetMethodCalls` |

**Why?** MCP tools use Roslyn semantic analysis - 10x faster, 90% fewer tokens.
```

---

## Multi-Solution: Understand Your Dependencies

Need the AI to understand a library you depend on? Clone the source from GitHub, open it in a second Visual Studio — each instance runs its own MCP server on a **configurable port**.

```
Your project                          → port 3010
Library source (cloned from GitHub)   → port 3011
Framework source                      → port 3012
```

Your AI connects to all of them. It can trace calls, find usage patterns, and understand inheritance **across your code and library code**.

**MCP client configuration — three options depending on your client:**

Clients with native HTTP support (Claude Desktop, Claude Code):
```json
{
  "mcpServers": {
    "vs-mcp": {
      "type": "http",
      "url": "http://localhost:3010/sdk/"
    },
    "vs-mcp-whisper": {
      "type": "http",
      "url": "http://localhost:3011/sdk/"
    }
  }
}
```

Clients without HTTP support (via mcp-remote proxy):
```json
{
  "mcpServers": {
    "vs-mcp": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:3010/sdk/"]
    },
    "vs-mcp-whisper": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "http://localhost:3011/sdk/"]
    }
  }
}
```

Codex CLI (TOML):
```toml
[mcp_servers.vs-mcp]
type = "stdio"
command = "npx"
args = ["mcp-remote", "http://localhost:3010/sdk/"]

[mcp_servers.vs-mcp-whisper]
type = "stdio"
command = "npx"
args = ["mcp-remote", "http://localhost:3011/sdk/"]
```

---

## Settings

**Tools** → **MCP Server Settings**

| Setting | Default | Description |
|---------|---------|-------------|
| **Port** | 3001 | Server port (configurable per VS instance) |
| **Path Format** | WSL | Output as `/mnt/c/...` or `C:\...` |
| **Tools** | All enabled | Enable/disable individual tool groups |

Changes apply on next server start.

---

## vs. Other Approaches

| Approach | Symbol Search | Inheritance | Call Graph | Safe Rename |
|----------|:---:|:---:|:---:|:---:|
| **MCP AI Server (Roslyn)** | Semantic | Full tree | Callers + Callees | Compiler-verified |
| AI Agent (grep/fs) | Text match | No | No | Text replace |
| Other MCP servers | No | No | No | No |

---

## Requirements

- **Visual Studio 2022** (17.13+) or **Visual Studio 2026**
- **Windows** (amd64 or arm64)
- Any MCP-compatible AI tool

---

## Issues & Feature Requests

Found a bug or have an idea? [Open an issue](https://github.com/LadislavSopko/mcp-ai-server-visual-studio/issues/new/choose)!

---

## About

**[0ics srl](https://0ics.it)** — Italian software company specializing in AI-powered development tools.

Built by **Ladislav Sopko** — 30 years of software development, from assembler to enterprise .NET.

---

*MCP AI Server: Because your AI deserves the same intelligence as your IDE.*
