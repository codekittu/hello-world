# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # Install deps, generate Prisma client, run migrations
npm run dev            # Start dev server (Next.js 15 + Turbopack) at localhost:3000
npm run build          # Production build
npm run lint           # ESLint
npm run test           # Run all tests with Vitest
npx vitest run src/path/to/file.test.ts  # Run a single test file
npm run db:reset       # Reset and re-migrate the SQLite database
```

## Environment Variables

- `ANTHROPIC_API_KEY` — Optional. Without it, a mock provider returns static placeholder code.
- `JWT_SECRET` — Used for signing session tokens. Defaults work for dev but set a real value in production.

## Architecture

### AI Generation Pipeline

**API route**: `src/app/api/chat/route.ts`

1. Client sends chat messages + serialized virtual file system state
2. Server selects provider (`src/lib/provider.ts`): real Claude (Haiku 4.5) if API key set, else `MockLanguageModel`
3. `streamText()` streams responses with tool calls using the system prompt in `src/lib/prompts/generation.tsx`
4. AI uses two tools: `str_replace_editor` (create/view/edit files) and `file_manager` (rename/delete)
5. `onFinish` hook serializes the file system and persists to DB for authenticated users

### Virtual File System

`src/lib/file-system.ts` — Core in-memory file tree. Key operations: `createFile`, `updateFile`, `deleteFile`, `rename`, `replaceInFile`, `insertInFile`, `serialize`/`deserializeFromNodes`.

`src/lib/contexts/file-system-context.tsx` — React context wrapping the VFS. Handles AI tool call execution and maps them to VFS operations. Use `useFileSystem()` hook.

### Component Preview

`src/components/preview/PreviewFrame.tsx` — Renders a sandboxed iframe. Flow:
1. `src/lib/transform/jsx-transformer.ts` uses `@babel/standalone` to transform JSX → JS and extract imports
2. External packages map to `https://esm.sh/{package}`; local files become blob URLs
3. HTML is generated with inline Tailwind CDN, an ErrorBoundary, and an import map
4. Entry point: `/App.jsx` → `/App.tsx` → `/index.jsx` → `/index.tsx` → first `.jsx`/`.tsx`

### Authentication

JWT-based sessions via HttpOnly cookies (`src/lib/auth.ts`). 7-day expiration, HS256 signed.
Server actions in `src/actions/index.ts`: `signUp`, `signIn`, `signOut`, `getUser`.
Middleware (`src/middleware.ts`) protects `/api/projects` and `/api/filesystem`.

### Database

Prisma + SQLite (`prisma/schema.prisma`). Two models:
- `User`: email/password (bcrypt, salt 10)
- `Project`: belongs to User; `messages` and `data` fields store JSON strings (chat history and serialized VFS)

Anonymous users get no persistence — file system is lost on refresh. Authenticated users auto-save on each AI response.

### UI Layout

`src/app/main-content.tsx` — Root layout: resizable panels (35% chat | 65% preview/code).
- Chat: `src/components/chat/` — uses Vercel `useChat` hook from `@ai-sdk/react`
- Preview: `src/components/preview/PreviewFrame.tsx`
- Code editor: `src/components/editor/CodeEditor.tsx` (Monaco) + `src/components/file-tree/`

### Testing

Vitest + jsdom + React Testing Library. Tests live in `src/**/__tests__/` directories alongside source files.
