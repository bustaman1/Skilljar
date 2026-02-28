# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # First-time setup: install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack on http://localhost:3000
npm run build          # Build production bundle
npm run lint           # Run ESLint
npm test               # Run Vitest test suite (watch mode)
npm run db:reset       # Reset SQLite database to clean state
npx prisma migrate dev # Run database migrations after schema changes
npx prisma generate    # Regenerate Prisma client after schema changes
```

Run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

**UIGen** is a Next.js 15 App Router application that uses Claude AI to generate React components with real-time preview.

### Virtual File System
The core abstraction is `VirtualFileSystem` (`src/lib/file-system.ts`) — an in-memory file structure that never writes to disk. All AI-generated files live here during a session. The system is JSON-serializable for database persistence. `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) exposes this to React components.

### AI Integration
- `/src/app/api/chat/route.ts` is the main AI endpoint using Vercel AI SDK with Anthropic Claude
- Claude is given two tools: `str_replace_editor` (view/create/edit files) and `file_manager` (manage files/directories), defined in `src/lib/tools/`
- The system prompt in `src/lib/prompts/generation.tsx` instructs Claude how to generate React components
- `src/lib/provider.ts` returns either the real Anthropic provider or a `MockLanguageModel` when `ANTHROPIC_API_KEY` is absent
- Prompt caching is enabled with ephemeral cache control on the system message

### Preview System
`PreviewFrame` (`src/components/preview/`) renders generated components in an iframe sandbox. `@babel/standalone` transpiles JSX in the browser, and a virtual module system maps all project files into importable modules using an import map to resolve React and other dependencies.

### Data Persistence
- SQLite via Prisma (`prisma/schema.prisma`) with two key models: `User` and `Project`
- `Project.messages` stores chat history as JSON; `Project.data` stores the serialized virtual file system
- Auth uses JWT (via `jose`) in HTTP-only cookies with bcrypt password hashing
- Server Actions in `src/actions/` handle auth and project CRUD
- Anonymous users' work is tracked via `src/lib/anon-work-tracker.ts` and promoted to a saved project after sign-up

### State Management
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) wraps Vercel AI SDK's `useChat` hook
- `FileSystemContext` manages virtual file system state
- Both contexts are composed in `src/app/main-content.tsx`

### Path Aliases
`@/*` maps to `./src/*` (configured in `tsconfig.json`).

## Environment

```
ANTHROPIC_API_KEY="sk-ant-..."   # Optional; falls back to mock provider if absent
```

## Tech Stack

- **Framework**: Next.js 15 (App Router), React 19, TypeScript
- **Styling**: Tailwind CSS v4, Shadcn/ui (Radix UI), lucide-react
- **Code Editor**: Monaco Editor (`@monaco-editor/react`)
- **AI**: Vercel AI SDK (`ai`, `@ai-sdk/anthropic`)
- **Database**: SQLite + Prisma ORM
- **JSX Compilation**: `@babel/standalone` (runs in browser)
- **Testing**: Vitest + React Testing Library (jsdom environment)
