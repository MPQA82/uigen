# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install + prisma generate + migrate
npm run dev            # Development server (Turbopack)
npm run dev:daemon     # Dev server in background, logs to logs.txt
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Vitest (all tests)
npm run db:reset       # Reset SQLite database
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

## Environment Variables

- `ANTHROPIC_API_KEY` — optional; omit to use the mock provider (returns static code, full agent loop still runs)
- `JWT_SECRET` — optional; defaults to `development-secret-key`

## Architecture

**UIGen** is an AI-powered React component generator with live preview. Claude generates components into a virtual (in-memory) file system; an iframe renders them live.

### Request flow

1. User sends a message in `ChatInterface`
2. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) uses Vercel AI SDK's `useChat` to POST to `/api/chat`
3. `/api/chat/route.ts` streams text + tool calls back using `streamText` with Anthropic Claude
4. `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) executes tool calls client-side
5. Virtual file system updates → `PreviewFrame` re-renders the iframe
6. On generation finish, authenticated users' messages + file system are persisted to SQLite via server actions

### Virtual file system

`src/lib/file-system.ts` — in-memory file tree, serializable to JSON. No disk writes. All generated code lives here until persisted to the `Project.data` column.

### AI tool integration

Two tools available to Claude during generation:
- `str_replace_editor` — view, create, str_replace, insert into files
- `file_manager` — rename, delete files/folders

Tool calls execute on the client; results are sent back for agentic looping (`maxSteps`: 4 mock / 40 real).

Prompt caching is enabled (ephemeral) on the system message (`src/lib/prompts/generation.tsx`).

### Authentication

JWT tokens in HTTP-only cookies (7-day expiry). `src/middleware.ts` protects API routes. Works without auth — anonymous sessions track work in localStorage (`src/lib/anon-work-tracker.ts`) with no DB persistence.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` (email + bcrypt password) and `Project` (name, messages as JSON string, file system as JSON string).

### Key files

| File | Purpose |
|---|---|
| `src/app/main-content.tsx` | Layout shell: ResizablePanels for chat / preview / code editor |
| `src/app/api/chat/route.ts` | Streaming AI endpoint |
| `src/lib/file-system.ts` | Virtual file system core |
| `src/lib/contexts/chat-context.tsx` | Chat state + AI SDK hook |
| `src/lib/contexts/file-system-context.tsx` | File system state + tool execution |
| `src/lib/provider.ts` | Selects Claude or mock model |
| `src/lib/prompts/generation.tsx` | System prompt for generation |
| `src/components/preview/PreviewFrame.tsx` | iframe renderer |
| `src/actions/index.ts` | Auth server actions |

## Conventions

- Import alias `@/` maps to `src/` — always use it for non-library imports
- `"use client"` on interactive components, `"use server"` on server actions
- Tailwind CSS only — no CSS modules or inline styles
- Server actions return `{ success: boolean; error?: string }`
- Generated components must have `App.jsx` as entry point, use `@/` imports, Tailwind only
