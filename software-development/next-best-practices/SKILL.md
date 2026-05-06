---
name: next-best-practices
description: Next.js best practices for agency-webapp — file conventions, RSC boundaries, data patterns, async APIs, metadata, error handling, route handlers, Clerk auth integration, Turbopack quirks, hot-reload.
category: software-development
tags: [nextjs, react, rsc, turbopack, clerk, typescript]
---

# Next.js Best Practices

Project: **agency-webapp** `~/app/ndr-agency`, Next.js 15 with Turbopack.

## RSC Boundary Rules

- Any component with event handlers (`onClick`, `onChange`, etc.) MUST have `'use client'` at the top of the file.
- Plain `next/link` `<Link>` is RSC-safe — prefer it over UI library buttons with click handlers.
- Server Components can import Client Components; Client Components cannot import Server Components.
- `'use client'` directive must survive Docker rebuilds — Turbopack caches SSR chunks inside images.

## File Conventions

| Path | Rule |
|------|------|
| `app/layout.tsx` | Root layout — `<html><body>` wrapped by `<ClerkProvider>` |
| `app/page.tsx` | Server Component by default — use `<Link>` not buttons with `onClick` |
| `middleware.ts` | MUST exist at app root — exports `clerkMiddleware()` |
| `app/(auth)/sign-in/[[...sign-in]]/page.tsx` | Auth page |
| `app/(auth)/sign-up/[[...sign-up]]/page.tsx` | Auth page |
| `components/ui/*.tsx` | `'use client'` if they have any event handlers |
| `lib/utils.ts` | `cn()` utility for class merging (shadcn) |

## Clerk Auth

See `fullstack-dev/references/clerk-auth.md` for the full integration guide.

Key rules:
- `clerkMiddleware()` must be exported from `middleware.ts` (not `proxy.ts`)
- `<ClerkProvider>` inside `<body>` in `app/layout.tsx`
- OAuth SSO: set `fallbackRedirectUrl` prop AND `NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL` env var to same URL
- Use `<Show when="signed-in">` not `<SignedIn>` (deprecated in v7+)

## Turbopack Development

- Type errors from `node_modules/@clerk/` are pre-existing TS config issues — Turbopack skips type checking at build time, these don't block dev.
- Hot-reload: volumes must be writable (not `:ro`) in docker-compose.
- To clear SSR cache: `docker rm -f ndr-web && docker rmi ndr-agency-web && docker compose up -d --build web`
- `docker compose up -d --build web` builds fresh image, NOT just `docker compose up -d` (which only starts existing images).

## Protected Routes

```tsx
import { auth } from '@clerk/nextjs/server';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const { userId } = await auth();
  if (!userId) redirect('/sign-in');
  // ...
}
```

## Metadata & SEO

```tsx
import type { Metadata } from 'next';
export const metadata: Metadata = { title: 'Dashboard', description: '...' };
```

## API Routes (Route Handlers)

```tsx
// app/api/example/route.ts
import { NextResponse } from 'next/server';
export async function GET() {
  return NextResponse.json({ ok: true });
}
```
