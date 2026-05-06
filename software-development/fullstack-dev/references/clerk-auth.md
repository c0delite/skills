# Clerk Auth Integration — agency-webapp

## Setup

```bash
npm install @clerk/nextjs  # in apps/web
```

## Files

| File | Purpose |
|------|---------|
| `apps/web/src/middleware.ts` | Exports `clerkMiddleware()` — MUST be named `middleware.ts`, NOT `proxy.ts` |
| `apps/web/src/app/layout.tsx` | Wraps app with `<ClerkProvider>`, adds auth header |
| `apps/web/src/app/(auth)/sign-in/[[...sign-in]]/page.tsx` | `<SignIn fallbackRedirectUrl="/dashboard" />` |
| `apps/web/src/app/(auth)/sign-up/[[...sign-up]]/page.tsx` | `<SignUp fallbackRedirectUrl="/dashboard" />` |
| `apps/web/src/app/dashboard/page.tsx` | Protected route using `auth()` from `@clerk/nextjs/server` |

## Environment Variables

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/dashboard
```

## Critical Gotchas

### middleware.ts, NOT proxy.ts
Next.js App Router specifically looks for `middleware.ts` at the project root. Clerk errors at runtime if it can't find `clerkMiddleware()`:
```
auth() was called but Clerk can't detect usage of clerkMiddleware()
```
Fix: rename `proxy.ts` → `middleware.ts`.

### OAuth SSO — `fallbackRedirectUrl` prop overrides env vars
For social login (Gmail OAuth), `fallbackRedirectUrl` on `<SignIn>`/`<SignUp>` components takes precedence over `NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL` env var. Set BOTH to the same value:
```tsx
<SignIn fallbackRedirectUrl="/dashboard" />
// AND in .env:
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/dashboard
```
If only the env var is set, OAuth flows will still redirect to `/`.

### `<Show>` not `<SignedIn>`/`<SignedOut>`
Clerk v7+ uses `<Show when="signed-in">` and `<Show when="signed-out">`. The old `<SignedIn>`/`<SignedOut>` components are deprecated.

### Protected Routes
```tsx
import { auth } from '@clerk/nextjs/server';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const { userId } = await auth();
  if (!userId) redirect('/sign-in');
  // ...
}
```

## After-Setup Verification

1. `/` → 200 with auth header (Sign in/Sign up buttons visible)
2. `/sign-in` → 200 with Clerk `<SignIn>` component
3. `/sign-up` → 200 with Clerk `<SignUp>` component  
4. `/dashboard` unauthenticated → 307 redirect to `/sign-in`
5. Gmail SSO login → lands on `/dashboard`
