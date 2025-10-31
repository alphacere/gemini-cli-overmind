# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Gemini CLI is an open-source AI agent that brings Google Gemini directly into the terminal. It's a monorepo (npm workspaces) containing multiple packages:

- **@google/gemini-cli** - Main CLI application with interactive UI built on React + Ink
- **@google/gemini-cli-core** - Core AI engine, tool system, and services
- **@google/gemini-cli-vscode-ide-companion** - VS Code extension for IDE integration
- **@google/gemini-cli-a2a-server** - Cloud deployment server
- **@google/gemini-cli-test-utils** - Shared testing infrastructure

## Essential Development Commands

### Building

```bash
# Build all packages
npm run build

# Run complete development validation (clean, format, lint, build, typecheck, test)
npm run preflight

# Bundle for production
npm run bundle

# Build VS Code extension
npm run build:vscode

# Build sandbox Docker image
npm run build:sandbox
```

### Linting and Formatting

```bash
# Check linting (ESLint) across all packages
npm run lint

# Auto-fix linting and formatting issues
npm run lint:fix

# Format code with Prettier
npm run format
```

### Testing

```bash
# Run all unit and integration tests
npm run test

# Run CI test suite (includes script tests)
npm run test:ci

# Run integration tests without sandbox
npm run test:integration:sandbox:none

# Run integration tests with Docker sandbox
npm run test:integration:sandbox:docker

# Run specific test file in a workspace
npm run test --workspace @google/gemini-cli -- path/to/test.spec.ts

# Run tests with coverage
npm run test:coverage --workspaces
```

### Type Checking

```bash
# Check TypeScript types across all packages
npm run typecheck
```

### Development Server

```bash
# Start dev mode (watches and rebuilds)
npm run start

# Start with debugging enabled
npm run debug
```

## Codebase Architecture

### High-Level Architecture

```
CLI Layer (packages/cli)
    ↓
Core Logic (packages/core)
    ├─ AI Engine (GeminiClient, GeminiChat, Turn)
    ├─ Tool System (Registry & Execution)
    ├─ Services (File, Git, Shell, etc.)
    └─ Config Management
    ↓
External APIs & Services
    ├─ Gemini API
    ├─ Google Cloud
    └─ MCP Servers
```

### Directory Structure

**packages/cli/** (78K LOC) - Terminal UI and user-facing commands
- `src/ui/` - React/Ink components (AppContainer, ChatUI, etc.)
- `src/commands/` - CLI command handlers
- `src/config/` - User configuration and settings
- `src/services/` - Application-level services
- `src/core/` - CLI initialization and startup logic

**packages/core/** (107K LOC) - Core AI engine and tool system
- `src/core/client.ts` - GeminiClient (conversation management, API communication)
- `src/core/geminiChat.ts` - GeminiChat (single chat session management)
- `src/core/turn.ts` - Turn (single conversation turn)
- `src/tools/` - 48+ tools (read-file, shell, grep, web-fetch, mcp-client, etc.)
- `src/services/` - Backend services (file discovery, git operations, chat recording, loop detection, compression)
- `src/config/` - Configuration system (39K LOC) with models, auth, policies
- `src/mcp/` - Model Context Protocol support

**packages/vscode-ide-companion/** - VS Code extension
- Provides IDE context to Gemini CLI
- Enables inline code assistance

**packages/a2a-server/** - Cloud deployment
- Express-based HTTP server
- Supports remote agent access

### Key Architectural Concepts

#### GeminiClient (core/core/client.ts)
- Manages conversation state and history
- Handles tool registration and execution
- Communicates with Gemini API
- Implements context compression and loop detection
- Integrates IDE context when available

#### Tool System (core/tools/)
The tool registry is the central mechanism for extending functionality:
- Each tool is a module with metadata and execution logic
- MCP protocol allows loading external tools
- Tools handle file ops, shell commands, web searches, and more
- All tool execution goes through permission checks and user confirmation

#### Config System (core/config/)
Extensive configuration system that handles:
- Model selection and parameters
- Authentication methods (OAuth, API keys, Vertex AI, etc.)
- User settings and preferences
- Tool policies and permissions
- Telemetry configuration

### Key Files and Their Purposes

| File | Purpose | Key Exports |
|------|---------|-------------|
| `packages/cli/src/gemini.tsx` | CLI entry point and argument parsing | `main()` |
| `packages/cli/src/ui/AppContainer.tsx` | Main React component for interactive mode | Interactive UI |
| `packages/core/src/core/client.ts` | Gemini API client and conversation management | `GeminiClient` |
| `packages/core/src/core/geminiChat.ts` | Single chat session | `GeminiChat` |
| `packages/core/src/tools/[name].ts` | Individual tool implementations | Tool modules |
| `packages/core/src/config/config.ts` | Configuration management | `loadCliConfig()`, etc. |
| `packages/core/src/services/` | Reusable business logic | Service classes |

## Development Workflow

### Working on Features

1. **Understand the context** - Most new features involve both CLI and core changes
2. **Write tests first** (recommended)
   - Unit tests: `src/__tests__/` or `.spec.ts` alongside code
   - Integration tests: `integration-tests/` for end-to-end testing
3. **Make changes** to appropriate package(s)
4. **Run preflight** to validate: `npm run preflight`
5. **Test specific changes**: Use `npm run test --workspace @package-name`

### Code Organization Rules

- **CLI layer** (packages/cli) - UI, commands, user-facing features
- **Core layer** (packages/core) - Business logic, AI engine, tools
- **Services** - Reusable logic accessed by both layers
- **Config** - System-wide configuration and policies
- **Tools** - Discrete, composable units of functionality

### Type Safety

- Use TypeScript strict mode throughout
- Proper typing for all tool implementations
- Config types are important - update when adding new features

## Common Tasks

### Adding a New Tool

1. Create file in `packages/core/src/tools/[tool-name].ts`
2. Implement tool interface with metadata and execute function
3. Register in tool registry (typically in config.ts)
4. Add tests in `__tests__/`
5. Update documentation in `docs/tools/`

### Modifying Config System

- Update `packages/core/src/config/config.ts`
- Add type definitions to config interfaces
- Update schema/validation if needed
- Test with different config scenarios

### Adding CLI Commands

1. Create command handler in `packages/cli/src/commands/`
2. Register in command registry
3. Add UI components as needed in `packages/cli/src/ui/`
4. Test with `npm run start` or test suite

## Testing Strategy

### Test Structure

- **Unit tests** - Test individual functions/classes in isolation
- **Integration tests** - Full workflows with real tools
- **Sandbox variants** - Test with/without Docker/Podman isolation

### Running Tests

```bash
# Single test file
npm run test -- path/to/test.spec.ts

# Watch mode
npm run test -- --watch

# Specific test pattern
npm run test -- --grep "pattern"

# Coverage report
npm run test -- --coverage
```

## Build and Release Process

### Version Management

```bash
# Release new version (updates version across monorepo)
npm run release:version
```

### Bundling

```bash
# Create production bundle
npm run bundle

# Output appears in bundle/ directory for npm distribution
```

### Sandbox Image

```bash
# Build Docker sandbox image
npm run build:sandbox

# Image URI in package.json config.sandboxImageUri
```

## Configuration and Customization

### Authentication

The system supports multiple auth methods:
- OAuth with Google (free tier, 60 req/min)
- Gemini API Key (direct model control)
- Vertex AI (enterprise features)
- Cloud Shell integration

### User Settings

Located in `~/.gemini/settings.json`:
- Model preferences
- Tool permissions
- MCP server configurations
- Custom context

### GEMINI.md Files

Projects can include GEMINI.md for persistent context to Gemini CLI.

## Monorepo Workspace Management

The project uses npm workspaces. Work within specific packages:

```bash
# Install dependencies (root level)
npm ci

# Run command in specific workspace
npm run [script] --workspace @google/gemini-cli

# Run command across all workspaces
npm run [script] --workspaces
```

## Performance Considerations

### Context Compression

Large conversations are automatically compressed to stay within API limits. Compression strategy is in `ChatCompressionService`.

### Loop Detection

Prevents infinite tool-calling loops. Configured in `LoopDetectionService`.

### Streaming

The system uses AsyncGenerator streams for real-time output, important for long-running operations.

## Security and Permissions

### Tool Permissions

- Tools can require user confirmation before execution
- Fine-grained permission system in config
- Sandboxing support (Docker/Podman) for shell commands

### Trusted Folders

Different permission levels based on directory context.

### Data Handling

- Telemetry system (OpenTelemetry) collects usage data
- Can be disabled in settings
- No sensitive data (API keys, tokens) in logs

## Debugging Tips

### Enable Debug Logging

```bash
npm run debug

# Or with environment variable
DEBUG=1 npm run start
```

### MCP Server Issues

Check MCP configuration in settings.json and verify server processes are running.

### Tool Execution Failures

- Check tool logs in debug mode
- Verify tool permissions in config
- Test tool in isolation if possible

### API/Authentication Issues

- Verify credentials are set correctly
- Check GEMINI_API_KEY or OAuth tokens
- Review Google Cloud project settings for Vertex AI

## Key Dependencies

- **@google/genai** (1.16.0) - Gemini API SDK
- **@modelcontextprotocol/sdk** (1.15.1) - MCP protocol
- **React** (19.1.0) + **Ink** (6.2.3) - Terminal UI
- **TypeScript** (5.3.3) - Type safety
- **OpenTelemetry** - Observability
- **simple-git** (3.28.0) - Git operations
- **tree-sitter** (0.25.10) - Code parsing

## Version and Node Requirements

- **Node.js**: >= 20.0.0
- **Package Version**: v0.13.0-nightly (daily releases with main branch)
- **Release Schedule**:
  - Nightly: Daily at UTC 0000
  - Preview: Weekly on Tuesdays at UTC 2359
  - Stable: Weekly on Tuesdays at UTC 2000

## IDE Integration

VS Code companion extension in `packages/vscode-ide-companion/`:
- Sends editor context to Gemini CLI
- Provides inline suggestions
- Bidirectional communication

## Getting Help

- Check `docs/` directory for detailed guides
- Run `gemini /help` in the CLI for command reference
- Review existing test files for usage examples
- See troubleshooting guide at `docs/troubleshooting.md`
