# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Lint
npm lint

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Reset database
npm run db:reset
```

The app requires `ANTHROPIC_API_KEY` in `.env`. Without it, a `MockLanguageModel` is used that returns static component stubs — useful for development without API costs.

## Architecture

UIGen is a Next.js 15 App Router application where users describe React components in a chat interface and see a live preview rendered in an iframe.

### Request Flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from serialized state sent by the client, then calls `streamText` (Vercel AI SDK) with two tools: `str_replace_editor` and `file_manager`
3. The AI streams tool calls back; the client's `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) applies each tool call to its own in-memory `VirtualFileSystem` instance
4. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) watches `refreshTrigger` and re-renders by writing the files into an `<iframe srcdoc>` using Babel standalone (in-browser JSX transpilation via `src/lib/transform/jsx-transformer.ts`)
5. On finish, if the user is authenticated and a `projectId` exists, the server saves both the full message history and the serialized file system to the `Project` row in SQLite via Prisma

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode` objects (no disk I/O). It has two serialization formats:
- `serialize()` → `Record<string, FileNode>` (sent over the wire with each chat request)
- `deserialize()` / `deserializeFromNodes()` → rebuilds the tree on the server per request

The client holds an authoritative copy in `FileSystemContext`; the server reconstructs a fresh copy from the payload on every request.

### AI Tools

- **`str_replace_editor`** (`src/lib/tools/str-replace.ts`): `create`, `str_replace`, `insert`, `view` commands — the primary way the AI writes/edits files
- **`file_manager`** (`src/lib/tools/file-manager.ts`): `rename`, `delete`, `list` commands

### Preview Pipeline

`src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` to transpile JSX/TSX in the browser. It:
- Resolves `@/` import aliases to blob URLs of other virtual files
- Strips `.css` imports
- Creates an import map so each file is a proper ES module blob URL
- Generates a full HTML document written into `iframe.srcdoc`

The AI is instructed (via `src/lib/prompts/generation.tsx`) to always create `/App.jsx` as the entry point and use `@/` for cross-file imports within the virtual FS.

### Auth

Cookie-based JWT auth (`src/lib/auth.ts`) using `jose`. Passwords are hashed with `bcrypt`. Sessions last 7 days. The middleware (`src/middleware.ts`) is minimal — auth checks are done in server actions.

Anonymous users can use the app without signing in. Their work is tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts` and can be migrated to a project on sign-up.

### Data Model

Two Prisma models (SQLite):
- `User`: email + bcrypt password
- `Project`: belongs to an optional `User`; stores `messages` (JSON string of chat history) and `data` (JSON string of serialized `VirtualFileSystem`)

### Key Paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | AI streaming endpoint |
| `src/lib/file-system.ts` | VirtualFileSystem implementation |
| `src/lib/contexts/file-system-context.tsx` | Client-side FS state + tool call handler |
| `src/lib/transform/jsx-transformer.ts` | In-browser Babel JSX → ES module pipeline |
| `src/lib/provider.ts` | Returns real Claude model or `MockLanguageModel` |
| `src/lib/prompts/generation.tsx` | System prompt for the AI |
| `src/components/preview/PreviewFrame.tsx` | Live preview iframe |
| `src/components/chat/ChatInterface.tsx` | Chat UI, wires AI SDK to FS context |
| `prisma/schema.prisma` | DB schema |

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live alongside the code they test in `__tests__/` subdirectories.
