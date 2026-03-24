# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run migrations after schema changes
npx prisma migrate dev
```

## Environment

Copy `.env` and add `ANTHROPIC_API_KEY`. Without it, the app uses a `MockLanguageModel` that returns static component code — useful for development without API costs.

## Architecture

### Virtual File System

The core abstraction is `VirtualFileSystem` ([src/lib/file-system.ts](src/lib/file-system.ts)) — an in-memory tree of `FileNode` objects. No files are ever written to disk during generation. The FS serializes to/from plain JSON for persistence in the database and for sending over the wire to the API route.

### AI Tool Loop

The chat API route ([src/app/api/chat/route.ts](src/app/api/chat/route.ts)) uses Vercel AI SDK `streamText` with two tools:
- `str_replace_editor` — view/create/str_replace/insert operations on the virtual FS
- `file_manager` — rename/delete operations

The AI iterates (up to 40 steps with a real key, 4 with mock) calling these tools to build up files. On finish, the full message history and serialized FS are persisted to the `Project` row in SQLite — but only if the user is authenticated.

### Live Preview Pipeline

`PreviewFrame` ([src/components/preview/PreviewFrame.tsx](src/components/preview/PreviewFrame.tsx)) re-renders whenever `refreshTrigger` increments. It calls `createImportMap` then `createPreviewHTML` from [src/lib/transform/jsx-transformer.ts](src/lib/transform/jsx-transformer.ts):

1. Each `.jsx/.tsx` file is transpiled in-browser via `@babel/standalone`
2. Transpiled code is wrapped in a `Blob` and a `blob://` URL is created
3. An ES module import map maps file paths and `@/` aliases to these blob URLs
4. Third-party npm packages are resolved to `https://esm.sh/<package>`
5. Missing local imports get placeholder stub modules
6. The resulting HTML is injected into a sandboxed `<iframe>` via `srcdoc`
7. Tailwind CSS is loaded from CDN inside the iframe

The preview's entry point defaults to `/App.jsx` and falls back through `/App.tsx`, `/index.jsx`, `/index.tsx`, `/src/App.jsx`, `/src/App.tsx`.

### State Management (Contexts)

Two React contexts wrap the entire editor:

- `FileSystemContext` ([src/lib/contexts/file-system-context.tsx](src/lib/contexts/file-system-context.tsx)) — owns the `VirtualFileSystem` instance, exposes CRUD operations, and implements `handleToolCall` which processes incoming AI tool calls to mutate the FS and trigger preview refresh.
- `ChatContext` ([src/lib/contexts/chat-context.tsx](src/lib/contexts/chat-context.tsx)) — wraps Vercel AI SDK `useChat`, wires `onToolCall` to `FileSystemContext.handleToolCall`, and sends the serialized FS with every request body.

### Auth

JWT-based sessions stored in an `httpOnly` cookie (`auth-token`). Server-side helpers are in [src/lib/auth.ts](src/lib/auth.ts) (marked `server-only`). Passwords are hashed with bcrypt. The middleware ([src/middleware.ts](src/middleware.ts)) does not enforce auth on routes — auth is checked inside individual actions/routes. Anonymous users can generate freely; project saving requires authentication.

### Database

Prisma with SQLite (`prisma/dev.db`). The generated client outputs to `src/generated/prisma/`. Two models:
- `User` — email + bcrypt password
- `Project` — belongs to an optional `User`; `messages` and `data` fields store JSON-serialized chat history and virtual FS respectively.

### AI Model

Configured in [src/lib/provider.ts](src/lib/provider.ts). Uses `claude-haiku-4-5` via `@ai-sdk/anthropic`. Falls back to `MockLanguageModel` (same file) when `ANTHROPIC_API_KEY` is absent.

### Tests

Vitest with jsdom and React Testing Library. Test files live alongside their subjects in `__tests__` directories. The `@/` path alias works in tests via `vite-tsconfig-paths`.
