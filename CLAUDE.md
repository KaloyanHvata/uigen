# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest src/lib/__tests__/file-system.test.ts

# Reset database
npm run db:reset
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` (already included in npm scripts) due to Node.js compatibility shims.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat; Claude generates JSX files via tools, which are stored in a virtual file system and rendered in a sandboxed iframe.

### Data Flow

1. User sends message → `POST /api/chat` streams `streamText` response via Vercel AI SDK
2. Claude calls tools (`str_replace_editor`, `file_manager`) to create/edit files
3. Tool calls propagate to the client-side `FileSystemContext` (virtual in-memory FS)
4. `PreviewFrame` picks up FS changes, transforms JSX via Babel standalone, and renders in an iframe using esm.sh CDN import maps
5. For authenticated users, project state (messages + serialized FS) is persisted to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`): In-memory FS with no disk I/O. Serialized to JSON for DB persistence. All file operations happen here — the iframe loads content directly from this state.

**Provider** (`src/lib/provider.ts`): Returns `claude-haiku-4-5` if `ANTHROPIC_API_KEY` is set, otherwise falls back to `MockLanguageModel` which returns static component templates. This allows development without an API key.

**Contexts**:
- `ChatContext` (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, handles tool call results, tracks anonymous work in localStorage
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`): Manages virtual FS state; executes file tool calls and triggers preview refresh

**Tools** (server-side, `src/lib/tools/`):
- `str_replace_editor`: view/create/str_replace/insert operations on files
- `file_manager`: rename/delete operations

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Babel standalone transforms JSX/TSX to plain JS. Generates an HTML document with import maps pointing to esm.sh for React and other dependencies.

### Route Structure

- `src/app/page.tsx` — Root: redirects authenticated users to their latest project, or shows chat for anonymous users
- `src/app/[projectId]/page.tsx` — Project view
- `src/app/api/chat/route.ts` — Streaming chat endpoint; builds system prompt, tools, and handles DB persistence
- `src/actions/index.ts` — Server actions for auth (signUp, signIn, signOut, getUser)
- `src/middleware.ts` — Protects `/api/projects` and `/api/filesystem` routes with JWT validation

### Authentication

JWT sessions (7-day expiry) stored in httpOnly cookies via the `jose` library. Passwords hashed with bcrypt. Anonymous users can generate components but their work is only preserved in localStorage — sign-up migrates it to the DB.

### Database

SQLite (via Prisma). Two models: `User` and `Project`. Projects store `messages` and `data` (serialized virtual FS) as JSON strings.

### Testing

Tests live alongside source in `__tests__/` subdirectories. The suite covers: `VirtualFileSystem`, `FileSystemContext`, `ChatContext`, JSX transformer, and all chat/editor UI components. Uses Vitest + React Testing Library with jsdom.
