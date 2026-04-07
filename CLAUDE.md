# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in natural language, and Claude AI generates React code that renders in real-time. All files exist in a virtual file system (never written to disk), allowing instant preview updates.

## Development Commands

```bash
# Initial setup (install dependencies + generate Prisma client + run migrations)
npm run setup

# Start dev server with Turbopack
npm run dev

# Run all tests with Vitest
npm test

# Run single test file
npm test path/to/file.test.ts

# Database operations
npx prisma generate          # Regenerate Prisma client after schema changes
npx prisma migrate dev       # Create and apply new migration
npm run db:reset             # Reset database (WARNING: deletes all data)

# Linting
npm run lint
```

**Important**: All npm scripts use `NODE_OPTIONS='--require ./node-compat.cjs'` to enable Node.js compatibility in Next.js (required for Prisma on Windows).

## Architecture

### Virtual File System (VFS)

The core innovation is a fully in-memory file system that never writes to disk:

- **Implementation**: `src/lib/file-system.ts` - Tree structure with FileNode objects
- **React Context**: `src/lib/contexts/file-system-context.tsx` - Provides CRUD operations to UI
- **Serialization**: Stored as JSON in SQLite `Project.data` field for persistence
- **Import Alias**: All user files use `@/` import prefix (e.g., `import Button from '@/components/Button'`)

The VFS supports directories, file creation/update/delete/rename, and can be serialized to/from JSON for database storage.

### AI Code Generation Flow

1. **User Message** → `src/app/api/chat/route.ts` API endpoint
2. **System Prompt** from `src/lib/prompts/generation.tsx` injected with cache control
3. **Vercel AI SDK** streams response with tool calling enabled
4. **Two Tools Available**:
   - `str_replace_editor` (create/str_replace/insert) - defined in `src/lib/tools/str-replace.ts`
   - `file_manager` (rename/delete) - defined in `src/lib/tools/file-manager.ts`
5. **Tool Execution** operates directly on VirtualFileSystem instance
6. **React Context** processes tool calls and triggers UI refresh via `handleToolCall()`
7. **onFinish Hook** serializes final VFS state and messages to SQLite for logged-in users

### Live Preview System

Preview runs entirely in an iframe with blob URLs (no bundler):

- **Transformation**: `src/lib/transform/jsx-transformer.ts` uses Babel Standalone to transpile JSX/TSX
- **Import Map**: Creates ES module import map with:
  - External deps from `esm.sh` (React, etc.)
  - Local files as blob URLs (generated from transformed code)
  - `@/` alias resolution
  - CSS file collection and injection
- **Entry Point**: Always looks for `/App.jsx` first (convention), falls back to other .jsx/.tsx files
- **Error Handling**: Syntax errors displayed in styled error UI, runtime errors caught by ErrorBoundary
- **Preview HTML**: Generated in `createPreviewHTML()` with Tailwind CDN and inline styles

Key constraint: User code MUST export a default component from `/App.jsx` for preview to work.

### Database Schema

Prisma with SQLite (`prisma/dev.db`):

- **User**: email/password authentication (bcrypt hashed)
- **Project**: Belongs to User (optional - anonymous users store projectId in localStorage)
  - `messages`: JSON array of chat history
  - `data`: JSON serialized VirtualFileSystem tree
  - Anonymous projects: `userId` is null, tracked client-side via `src/lib/anon-work-tracker.ts`

**Prisma Client Location**: Generated to `src/generated/prisma` (not default `node_modules`)

### Authentication

JWT-based auth using `jose` library:

- **Implementation**: `src/lib/auth.ts` with `createSession()`, `getSession()`, `deleteSession()`
- **Session Storage**: HttpOnly cookie named "session"
- **Middleware**: `src/middleware.ts` protects routes (currently minimal)
- **UI Components**: `src/components/auth/` for SignIn/SignUp dialogs
- **Hook**: `src/hooks/use-auth.ts` provides client-side auth state

### Key Conventions

- **Entry File**: AI always creates `/App.jsx` as the root component
- **Import Paths**: User code uses `@/` prefix (e.g., `import Foo from '@/components/Foo'`)
- **No HTML Files**: Preview system doesn't use HTML files - only JSX/TSX components
- **Tailwind Styling**: All components styled with Tailwind classes (CDN loaded in preview iframe)
- **No Disk I/O**: Virtual file system means no actual files created during development

### Mock Provider Mode

If `ANTHROPIC_API_KEY` is not set in `.env`:

- Falls back to mock provider (see `src/lib/provider.ts`)
- Returns static code instead of AI-generated
- Uses fewer max steps (4 vs 40) to prevent repetition
- Useful for development/testing without API costs

## Testing

- **Framework**: Vitest with React Testing Library
- **Config**: `vitest.config.mts` with jsdom environment
- **Test Files**: Located alongside source in `__tests__/` directories
- **Coverage**: Tests for contexts, file system, transformers, and UI components

## Tech Stack

- **Next.js 15**: App Router with Server Actions
- **React 19**: Latest with automatic runtime
- **TypeScript**: Strict mode enabled
- **Tailwind CSS v4**: Using new PostCSS plugin architecture
- **Prisma**: SQLite for development (easily swapped for production DB)
- **Vercel AI SDK**: Tool calling and streaming
- **Babel Standalone**: Client-side JSX transformation
- **Monaco Editor**: Code editing (from `@monaco-editor/react`)
