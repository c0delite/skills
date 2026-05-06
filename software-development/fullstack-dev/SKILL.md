---
name: fullstack-dev
description: Full-stack backend architecture and frontend-backend integration guide for the agency-webapp project. Trigger when building full-stack apps with Express + Next.js + shadcn + TypeScript + Docker.
category: software-development
tags: [express, nextjs, typescript, docker, rest-api, auth, shadcn]
---

# Full-Stack Development Practices

Project context: **agency-webapp** at `~/app/ndr-agency`. Stack: Express API + Next.js 15 + shadcn + TypeScript + Docker monorepo (npm workspaces). Branch: `feat/*`. Deliverables: zero-bug, full integration, passing unit + e2e tests.

## Project Structure

```
apps/
  api/        — Express server (port 3002)
  web/        — Next.js 15 app (port 3003)
packages/
  shared/     — Shared types, utils
```

## Conventions

- Feature-first API structure: `apps/api/src/features/<name>/controller.ts | router.ts | schema.ts`
- Shared middleware: `apps/api/src/shared/middleware/`
- RSC-safe components: any component with event handlers needs `'use client'` at top of file
- Hot-reload: volumes mounted writable (not `:ro`) in docker-compose
- `npm -w apps/api run dev` for workspace-script invocation
- No `command:` overrides in docker-compose — rely on Dockerfile CMD

## This skill has a references directory

- `references/clerk-auth.md` — Clerk auth integration notes, gotchas, and redirect flow troubleshooting
