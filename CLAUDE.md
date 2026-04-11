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
npx vitest src/lib/__tests__/file-system.test.ts

# Lint
npm run lint

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Run new migrations after schema changes
npx prisma migrate dev
```

Set `ANTHROPIC_API_KEY` in `.env` to use the real Claude API. Without it, the app falls back to a `MockLanguageModel` in `src/lib/provider.ts`.

## Architecture

UIGen is a Next.js 15 App Router application that lets users describe React components in a chat, then generates and live-previews them using Claude.

### Core data flow

1. User sends a message in `ChatInterface` → POST to `/api/chat`
2. The route streams a response from Claude (or `MockLanguageModel`) using Vercel AI SDK's `streamText`
3. Claude calls two tools: `str_replace_editor` and `file_manager` to create/modify files in a `VirtualFileSystem`
4. The VFS is serialized and stored in React context (`FileSystemContext`)
5. `PreviewFrame` compiles files from the VFS using Babel standalone and renders them in an iframe

### Virtual File System (`src/lib/file-system.ts`)

All generated code lives in an in-memory `VirtualFileSystem` — nothing is written to disk. The VFS supports standard file operations plus text-editor-style commands (`viewFile`, `replaceInFile`, `insertInFile`). It serializes to/from plain `Record<string, FileNode>` for JSON storage in the Prisma `Project.data` field.

### AI tools (`src/lib/tools/`)

- `str_replace_editor` — view/create/str_replace/insert operations on the VFS, modeled after the Claude computer-use text editor tool
- `file_manager` — create, delete, rename, list operations on the VFS

### JSX compilation (`src/lib/transform/jsx-transformer.ts`)

Transforms JSX/TSX source from the VFS into executable JS using Babel standalone in the browser. Handles missing imports by substituting placeholder modules and strips CSS imports.

### Authentication (`src/lib/auth.ts`, `src/middleware.ts`)

JWT-based auth stored in an httpOnly cookie (`auth-token`). `getSession()` is server-only. `verifySession()` is used in middleware to protect `/api/projects` and `/api/filesystem`. Anonymous users can generate components; only authenticated users get project persistence.

### Persistence

- **Authenticated users**: Projects saved to SQLite via Prisma (`Project` model). `messages` and `data` (VFS snapshot) are stored as JSON strings.
- **Anonymous users**: Session work tracked in `sessionStorage` via `src/lib/anon-work-tracker.ts`. On sign-up/login, the anonymous work can be migrated to a new project.

### Key contexts (`src/lib/contexts/`)

- `FileSystemContext` — holds the VFS state and exposes file CRUD to components
- `ChatContext` — holds message history, streaming state, and the selected project

### Provider fallback (`src/lib/provider.ts`)

`getLanguageModel()` returns the real `anthropic("claude-haiku-4-5")` model when `ANTHROPIC_API_KEY` is set, otherwise returns `MockLanguageModel` which replays hardcoded component generation steps.
