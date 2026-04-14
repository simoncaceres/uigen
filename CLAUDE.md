# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (run all tests)
npm run db:reset     # Reset SQLite database (destructive)
```

Set `ANTHROPIC_API_KEY` in `.env.local` to use real Claude. Without it, the app falls back to a mock provider that returns hardcoded example components.

## Architecture

UIGen is a Next.js 15 (App Router) app that lets users describe React components in natural language and see them generated and previewed live.

### Core Data Flow

1. User sends a message in `ChatInterface`
2. `ChatContext` POSTs to `/api/chat/route.ts` with message history + serialized virtual file system
3. Backend streams a Claude response; Claude uses two tools to write files:
   - `str_replace_editor` — view/create/edit file content
   - `file_manager` — rename/delete files and folders
4. Frontend tool handler in `ChatContext` applies tool calls to the in-memory `VirtualFileSystem`
5. `PreviewFrame` compiles updated JSX via Babel standalone and renders a live preview
6. On stream completion, the backend persists messages + file system to SQLite (authenticated users only)

### Key Modules

| Path | Role |
|------|------|
| `src/app/api/chat/route.ts` | Core backend: injects system prompt (with prompt caching), streams Claude, saves to DB |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — all file state lives here, serializes to JSON |
| `src/lib/contexts/chat-context.tsx` | Manages conversation state, drives tool execution loop |
| `src/lib/contexts/file-system-context.tsx` | Wraps `VirtualFileSystem` in React state |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude for component generation |
| `src/lib/tools/str-replace.ts` | `str_replace_editor` tool implementation |
| `src/lib/tools/file-manager.ts` | `file_manager` tool implementation |
| `src/lib/provider.ts` | Selects real `claude-sonnet-*` model or `MockLanguageModel` |
| `src/lib/auth.ts` | JWT session (jose + bcrypt); tokens in httpOnly cookies |
| `src/actions/` | Next.js Server Actions for project CRUD |
| `src/app/main-content.tsx` | Root UI: resizable split-pane, chat left / preview+code right |
| `src/app/[projectId]/page.tsx` | Loads project from DB and initializes both contexts |

### Virtual File System

`VirtualFileSystem` is a pure in-memory data structure (no disk I/O). It serializes to/from JSON for DB persistence and is passed wholesale to the API on each request so Claude has full context of the current file state.

### Authentication & Projects

- JWT tokens in cookies; `getSession()` returns `userId` or null for anonymous sessions
- `Project` rows store `messages` and `data` (file system) as JSON strings
- Anonymous users get ephemeral projects; authenticated users get persisted projects

### Preview Rendering

`PreviewFrame` uses `@babel/standalone` to transpile JSX in-browser. Components must be self-contained or use only libraries explicitly shimmed in the preview environment.

### Mock Provider

`MockLanguageModel` (activated when no API key) returns deterministic hardcoded components (Counter, Form, Card) and caps tool steps at 4 to prevent loops. Useful for UI development without API costs.

## Database

SQLite via Prisma. Schema: `User` (email/password) → `Project` (name, messages JSON, data JSON). Generated client outputs to `src/generated/prisma/`.

After schema changes: `npx prisma migrate dev --name <description>`
