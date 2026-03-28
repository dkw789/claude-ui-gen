# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language; Claude generates code via tool calls, which is rendered in a sandboxed iframe with no files written to disk.

## Commands

```bash
npm run setup        # First-time setup: install deps + generate Prisma client + run migrations
npm run dev          # Dev server on http://localhost:3000 (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run db:reset     # Wipe and recreate the SQLite database
```

Run a single test file:
```bash
npx vitest run src/lib/transform/__tests__/jsx-transformer.test.ts
```

## Environment

Requires a `.env` file with `ANTHROPIC_API_KEY`. Without it, the app falls back to a mock provider that returns static placeholder components.

## Architecture

### Request flow

1. User types in **ChatInterface** → Vercel AI SDK `useChat` hook POSTs to `/api/chat`
2. `/api/chat/route.ts` calls Claude with two tools: `str_replace_editor` and `file_manager`
3. Tool call results update the **VirtualFileSystem** (in-memory, no disk I/O)
4. **FileSystemContext** propagates changes → **PreviewFrame** re-renders
5. **jsx-transformer.ts** Babel-transforms JSX, rewrites local imports to esm.sh CDN URLs, and injects an import map into a sandboxed iframe

### Key files

| Purpose | Path |
|---|---|
| AI chat endpoint | `src/app/api/chat/route.ts` |
| Main layout (resizable panels) | `src/app/main-content.tsx` |
| Virtual file system class | `src/lib/file-system.ts` |
| Chat state (useChat wrapper) | `src/lib/contexts/chat-context.tsx` |
| FS state + update logic | `src/lib/contexts/file-system-context.tsx` |
| Babel transform + import map | `src/lib/transform/jsx-transformer.ts` |
| Claude system prompt | `src/lib/prompts/generation.tsx` |
| str_replace tool | `src/lib/tools/str-replace.ts` |
| file_manager tool | `src/lib/tools/file-manager.ts` |
| JWT auth helpers | `src/lib/auth.ts` |
| Prisma schema | `prisma/schema.prisma` |

### State management

- **ChatContext**: wraps Vercel AI SDK `useChat`, exposes messages and submit handler
- **FileSystemContext**: holds the `VirtualFileSystem` instance; tool call results are applied here by the chat context after each streaming response
- No Redux or Zustand — React Context only

### Database

SQLite via Prisma. Schema has two models: `User` (email + bcrypt password) and `Project` (name, userId, messages JSON, virtualFS JSON). Anonymous users get no persistence; registered users have projects auto-saved after each turn.

Reference `prisma/schema.prisma` for the authoritative data model whenever working on anything database-related.

### Auth

JWT sessions (7-day, HttpOnly cookie). `src/middleware.ts` protects routes. Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD.

### Code execution

`jsx-transformer.ts` uses `@babel/standalone` to transpile JSX/TSX in the browser. CSS imports are stripped (browser incompatible). The output is injected into an iframe via `createPreviewHTML()` with a generated import map pointing to esm.sh.

### Prompt caching

`/api/chat/route.ts` marks the system prompt with `cache_control: { type: "ephemeral" }` to reduce Anthropic API costs on long conversations.
