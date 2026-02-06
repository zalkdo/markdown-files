# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development Commands

```bash
# Development server with Turbopack
npm run dev

# Alternative dev server without Turbopack
npm run dev2

# Production build
npm run build

# Production server
npm run start

# Lint checks
npm run lint
```

## Tech Stack

- **Framework**: Next.js 15.3.2 with App Router
- **UI Library**: React 19 with Mantine v8
- **Language**: TypeScript 5
- **Styling**: Tailwind CSS v4 + Mantine theme (dark/light mode)
- **State Management**: Zustand v5, React Query v5, custom RxJS reactive streams
- **HTTP Client**: Axios with interceptors

## Architecture Overview

### Reactive Streams Pattern
The app uses a custom RxJS-based pub/sub system for inter-component communication:
- `lib/reactive/reactivestreams.ts` - Core ReactiveStreamsProcessor extending RxJS Subject
- `contexts/ReactiveContext.ts` - React context providing the reactive instance
- `hooks/useReactive.ts` - Hook for publishing/subscribing to topics by group

Modals use this pattern - `useAlert()` and `useConfirm()` publish to reactive topics, and `AppAlertModal`/`AppConfirmModal` subscribe to receive triggers.

### Data Fetching
- React Query for caching and data fetching
- Axios interceptors configured in `contexts/QueryClientContext.tsx`
- API proxy: `/api/:path*` rewrites to `http://localhost:8080/:path*` (dev only)
- API clients live in `api/` directory (e.g., `api/clients.ts`)

### Layout Structure
- `app/layout.tsx` - Root layout wrapping with GlobalContext
- `components/AppLayout.tsx` - Main layout with Sidebar, modals, loading bar
- `components/Page*.tsx` - Modular page layout components (PageHeader, PageBody, PageLayout)

### Path Alias
Use `@/*` to import from root directory (configured in tsconfig.json).

## Key Directories

- `app/` - Next.js App Router pages
- `api/` - Axios API client instances
- `components/` - Reusable React components
- `contexts/` - React Context providers (GlobalContext, QueryClientContext, ReactiveContext)
- `hooks/` - Custom React hooks (useAlert, useConfirm, useReactive, useTenant)
- `lib/reactive/` - RxJS-based reactive streams implementation
- `stores/` - Zustand state stores

## Notes

- No testing framework is currently configured
- The app uses `next-themes` for theme switching with Mantine's color scheme manager
- Next.js output mode is `standalone` for containerization
- React Strict Mode is disabled
