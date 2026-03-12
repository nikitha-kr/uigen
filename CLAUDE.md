# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations
npm run dev          # Start dev server at http://localhost:3000 (Turbopack)
npm run build        # Production build
npm run lint         # ESLint
npm test             # Run all tests (Vitest)
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # Run a single test file
npm run db:reset     # Reset SQLite database (destructive)
```

All `next` commands require `NODE_OPTIONS='--require ./node-compat.cjs'` (already set in scripts).

## Environment

Copy `.env` and set:
- `ANTHROPIC_API_KEY` — if absent, a `MockLanguageModel` is used instead (returns static components)
- `JWT_SECRET` — defaults to `"development-secret-key"` in dev

## Architecture

### Request Flow
User message → `POST /api/chat` → Vercel AI SDK `streamText` → Claude (`claude-haiku-4-5`) with tools → streamed response updates virtual FS on client

### Virtual File System
The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory tree of `FileNode` objects. Generated React components live here, never on disk. The FS is serialized to JSON and stored in the `Project.data` column (SQLite) for authenticated users.

The client holds FS state via `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`), which processes AI tool calls (`str_replace_editor`, `file_manager`) to mutate the FS and trigger re-renders.

### AI Tools
Two tools are given to the model at inference time:
- `str_replace_editor` — create/edit files via string replacement or insert
- `file_manager` — rename/delete files

Both tools operate on a server-side `VirtualFileSystem` instance reconstructed from the client's serialized state per request.

### Auth
JWT-based, cookie-stored (`auth-token`), server-only (`src/lib/auth.ts`). Anonymous users can generate components but projects are only persisted for authenticated users. The `Project` model has an optional `userId`.

### Component Generation Rules (prompt)
- Every project must have `/App.jsx` as the entrypoint with a default export
- Use `@/` import alias for all non-library imports (e.g. `@/components/Button`)
- Style with Tailwind CSS only — no inline/hardcoded styles
- No HTML files

### Database
Prisma + SQLite (`prisma/dev.db`). Two models: `User` and `Project`. `Project.messages` and `Project.data` store JSON strings (chat history and FS snapshot). Reference `prisma/schema.prisma` for the full data structure.

## Code Style

Use comments sparingly. Only comment complex code.

### Preview
`PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) renders generated JSX live using `@babel/standalone` for in-browser JSX transformation.
