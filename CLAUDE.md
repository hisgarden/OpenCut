# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenCut is a free, open-source, privacy-first video editor for web, desktop, and mobile. Built with Next.js 16 + React 19 + TypeScript in a Turborepo monorepo.

## Commands

All commands use **Bun** (v1.2.18) as the package manager.

```bash
# Install dependencies
bun install

# Development (runs on http://localhost:3000)
bun dev:web

# Build
bun build:web

# Lint (Biome)
bun lint:web              # check
bun lint:web:fix          # auto-fix

# Format (Biome)
bun format:web

# Tests (Bun test runner)
bun test

# Database (from apps/web/)
bun run db:generate       # generate migrations
bun run db:migrate        # run migrations
bun run db:push:local     # push schema (dev)
```

## Local Dev Setup

```bash
cp apps/web/.env.example apps/web/.env.local
docker compose up -d db redis serverless-redis-http
bun install
bun dev:web
```

## Architecture

### Monorepo Structure

- `apps/web/` — Main Next.js application (App Router)
- `packages/env/` — Environment variable validation (Zod)
- `packages/ui/` — Shared icon components

### Core Editor System

Singleton `EditorCore` manages all editor state through specialized managers:

```
EditorCore (singleton)
├── playback: PlaybackManager    # Playback state, position, speed
├── timeline: TimelineManager    # Clips, tracks, timeline operations
├── scene: SceneManager          # Multiple scenes/editing contexts
├── project: ProjectManager      # Metadata, canvas presets, FPS
├── media: MediaManager          # Media library, uploads
├── renderer: RendererManager    # Preview canvas rendering
├── audio: AudioManager          # Audio mixing
├── save: SaveManager            # Auto-save
├── selection: SelectionManager  # Currently selected elements
└── command: CommandManager      # Undo/redo
```

**In React components**, always use the `useEditor()` hook (uses `useSyncExternalStore`, auto-re-renders on changes):
```typescript
const editor = useEditor();
editor.timeline.addTrack({ type: "media" });
```

**Outside React**, use `EditorCore.getInstance()` directly.

### State Management (Dual Pattern)

- **EditorCore** — Core business logic and editor state (singleton + subscriptions)
- **Zustand stores** (`src/stores/`) — UI-layer preferences persisted to localStorage (snapping, panels, keybindings)

### Actions & Commands

- **Actions** (`lib/actions/definitions.ts`) — User-triggered operations. Single source of truth. In components, use `invokeAction()` instead of calling `editor.*` directly.
- **Commands** (`lib/commands/`) — Undo/redo-able mutations. Each extends `Command` from `lib/commands/base-command` with `execute()` and `undo()`.
- **Handlers** (`hooks/use-editor-actions.ts`) — Connect actions to implementations via `useActionHandler()`.

### Directory Conventions (src/)

- `lib/` — Domain logic specific to OpenCut
- `utils/` — Generic reusable utilities (could be copy-pasted into any project)
- `services/` — External integrations and persistent concerns (renderer, storage, transcription)
- `core/` — EditorCore singleton and its managers
- `stores/` — Zustand UI state stores
- `hooks/` — Custom React hooks
- `types/` — TypeScript type definitions
- `constants/` — Constant values

### Path Alias

`@/*` maps to `src/*` (e.g., `import { useEditor } from "@/hooks/use-editor"`)

## Code Style

### Tooling

- **Biome** for linting and formatting (tab indentation, 80-char line width, double quotes)
- **Strict TypeScript** (`strictNullChecks: true`)

### Conventions

- **No abbreviations**: `event` not `e`, `element` not `el` (exception: widely understood short forms like `config`)
- **Comments**: Explain WHY, not WHAT. No changelog-style comments in code.
- **One file, one responsibility**: Extract shared logic into focused utility files. Split files >500 lines.
- **No TypeScript enums**: Use `as const` instead
- **No `any` type**: Maintain strict type safety
- **Accessibility**: Follow WCAG/ARIA best practices (see `.github/copilot-instructions.md` for full Ultracite rules)

## Areas Under Active Refactor (Avoid)

The preview system is being rewritten from DOM/HTML rendering to binary rendering. Avoid modifying:
- Preview panel (text fonts, stickers, effects)
- Export functionality
- Preview rendering optimizations
