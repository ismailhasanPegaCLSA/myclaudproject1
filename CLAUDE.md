# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — An AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates code using tool-use, updates a virtual file system, and renders the result in a live iframe preview.

## Commands

```bash
npm run setup          # Install deps + generate Prisma client + run migrations
npm run dev            # Start dev server with Turbopack
npm run dev:daemon     # Start dev server in background, logs to logs.txt
npm run build
npm run lint
npm run test           # Run all tests with vitest
npm run db:reset       # Reset database (force)
```

To run a single test file:
```bash
npx vitest run src/path/to/file.test.ts
```

Tests live in `src/lib/__tests__/` and `src/lib/contexts/__tests__/`.

## Environment

Requires an `ANTHROPIC_API_KEY` environment variable. Without it, the app falls back to `MockLanguageModel` (`src/lib/provider.ts`) which returns canned responses — useful for UI development without API costs. The mock provider uses `maxSteps: 4`; the real provider uses `maxSteps: 40`.

## Architecture

### Three-Panel UI (`src/app/main-content.tsx`)
The main layout: **Chat** (left) → **Code Editor** (center) → **Live Preview** (right). State flows from the AI stream → virtual file system → preview iframe.

### AI Integration (`src/app/api/chat/route.ts`)
- Uses Vercel AI SDK `streamText` with `@ai-sdk/anthropic`
- Claude is given two tools: `str_replace_editor` (`src/lib/tools/str-replace.ts`) for in-place edits, and `file_manager` (`src/lib/tools/file-manager.ts`) for create/delete
- After streaming completes, the project is persisted to the database
- Model is configured in `src/lib/provider.ts` (currently `claude-haiku-4-5`)
- `/api/chat` is **not** auth-protected; `/api/projects` and `/api/filesystem` are

### Virtual File System (`src/lib/file-system.ts`)
An in-memory file tree that tracks all generated files. Managed via `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`). The entire state is serializable to JSON for database persistence. Key conventions:
- Every project must have a root `/App.jsx` as the iframe entrypoint
- Non-library imports within the virtual FS use the `@/` alias (e.g., `@/components/Button`)

### Live Preview (`src/components/preview/PreviewFrame.tsx`)
Renders the virtual file system in an iframe. `src/lib/transform/jsx-transformer.ts` uses Babel standalone to compile JSX at runtime and builds an import map for the sandbox environment.

### Authentication (`src/lib/auth.ts`)
JWT-based sessions (7-day expiry) using `jose`. Passwords hashed with `bcrypt`. Session cookie is HttpOnly. Server actions in `src/actions/` handle sign-up, sign-in, logout, and project CRUD.

### Database (`prisma/schema.prisma`)
SQLite via Prisma. Two models:
- `User`: email + hashed password
- `Project`: name, optional userId (anonymous projects allowed), `messages` (JSON), `data` (JSON file system snapshot)

### Prompts (`src/lib/prompts/`)
System prompts for the AI are defined here. Changes here directly affect how Claude generates components. The generation prompt enforces `/App.jsx` as entrypoint, Tailwind CSS for styling, and the `@/` import alias convention.
