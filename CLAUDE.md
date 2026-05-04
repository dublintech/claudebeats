# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude creates/modifies them via tool calls against a virtual in-memory file system. A live preview renders the output in an iframe using Babel for JSX transformation.

## Commands

```bash
npm run dev          # Start dev server (Next.js + Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (all tests)
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset the SQLite database
```

Run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

> **Do not run `npm audit fix`** — package versions are pinned for compatibility.

## Environment

- `ANTHROPIC_API_KEY` — optional; falls back to a mock model returning canned demo components if missing or set to placeholder
- `JWT_SECRET` — optional; defaults to a dev key; controls cookie security

## Architecture

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Server calls `streamText()` (Vercel AI SDK) with `claude-haiku-4-5`, up to 40 tool steps
3. Claude invokes tools (`str_replace_editor`, `file_manager`) to create/edit virtual files
4. Tool call results stream back to the client
5. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) applies mutations to the in-memory virtual FS
6. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) re-renders the iframe

If the user is authenticated and `projectId` is provided, the project (messages + serialized FS) is auto-saved to SQLite via Prisma on each `onFinish`.

### Virtual File System

`src/lib/file-system.ts` — `Map<string, FileNode>` keyed by absolute path. No files are written to disk. Serializes to/from a flat `Record<string, FileNode>` for DB persistence. The `VirtualFileSystem` class is instantiated on the server per request and reconstructed from the serialized snapshot sent in the request body.

`FileSystemContext` wraps the client-side instance and exposes `handleToolCall`, which is the bridge between incoming Vercel AI SDK `onToolCall` callbacks and FS mutations. It also manages `selectedFile` and a `refreshTrigger` counter that downstream consumers watch to know when to re-render.

### Preview Pipeline

`PreviewFrame` → `createImportMap()` → `transformJSX()` (Babel standalone in-browser) → blob URLs → ES module import map → iframe `srcdoc`.

Key behaviors:
- Entry point is auto-discovered: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → `/src/App.jsx`
- Third-party package imports (e.g. `import { motion } from 'framer-motion'`) are automatically resolved via `https://esm.sh/<package>`
- CSS files are stripped from JS and injected as `<style>` blocks
- Files with syntax errors show a styled error overlay inside the iframe; healthy files still render
- Missing local imports get stub placeholder modules so the rest of the component tree still loads

### AI Tools (sent to Claude)

Defined in `src/lib/tools/`:
- **`str_replace_editor`** — `view`, `create`, `str_replace`, `insert` operations on virtual files. `undo_edit` is accepted by schema but always returns an error — use `str_replace` to revert.
- **`file_manager`** — `rename` and `delete` files/directories

### Generation Prompt Constraints

`src/lib/prompts/generation.tsx` — the system prompt Claude receives. Key rules it enforces:
- Every project must have a `/App.jsx` root file with a default export
- Always start new projects by creating `/App.jsx` first
- Style with Tailwind, not hardcoded styles
- Import local files using the `@/` alias (e.g. `@/components/Button`, not `./components/Button`)
- No HTML files — `App.jsx` is the entry point

When modifying the generation prompt, understand these constraints exist because the preview pipeline expects them (e.g. `@/` alias is wired into the import map builder in `jsx-transformer.ts`).

### Key Contexts

Both are client-side React contexts; `FileSystemProvider` must wrap `ChatProvider` because `ChatProvider` calls `useFileSystem()` internally.

- **`FileSystemProvider`** (`src/lib/contexts/file-system-context.tsx`) — virtual FS state, processes incoming tool calls via `handleToolCall`
- **`ChatProvider`** (`src/lib/contexts/chat-context.tsx`) — messages, input state, streaming via Vercel AI SDK `useChat`; also tracks anonymous work to sessionStorage via `src/lib/anon-work-tracker.ts`

### Auth

JWT-based (`src/lib/auth.ts`) with HTTP-only cookies; 7-day expiry. Auth is optional — anonymous users can generate components without signing in. `src/middleware.ts` only guards `/api/projects` and `/api/filesystem` routes. Uses `server-only` to prevent auth code from leaking to the client bundle.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` are JSON strings (serialized chat history and virtual FS snapshot). Server actions in `src/actions/` handle all DB operations.

### Prompt Caching

The generation system prompt uses Anthropic ephemeral caching (`providerOptions.anthropic.cacheControl`) to reduce token costs on repeated requests (5-minute TTL).

### Mock Provider

`src/lib/provider.ts` detects a missing/placeholder API key and substitutes `MockLanguageModel`. It runs a 4-step scripted sequence (create `App.jsx`, create a component file, `str_replace` it, then summarize) and detects the component type (counter/form/card) from keywords in the user prompt.

## Conventions

- **Path alias**: `@/` maps to `src/` (configured in `tsconfig.json`)
- **UI components**: shadcn/ui (Radix UI + Tailwind) lives in `src/components/ui/`; add new shadcn components via `npx shadcn@latest add <component>`
- **Server actions** go in `src/actions/` with `"use server"` directive
- **Client components** requiring browser APIs use `"use client"` directive
- Tests use Vitest + Testing Library with jsdom; config in `vitest.config.mts`
