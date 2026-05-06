# Clerk + Next.js App Router Critical Gotchas

## 1. Middleware File MUST Be Named `middleware.ts`

Next.js 15 App Router **requires** the middleware to be at `./middleware.ts` (or `./src/middleware.ts`). Naming it anything else — `proxy.ts`, `auth.ts`, `clerk.ts` — will silently fail.

```
apps/web/src/proxy.ts          ← WRONG — Clerk can't find it
apps/web/src/middleware.ts     ← CORRECT — Next.js convention
```

**Error when misnamed:**
```
Clerk: auth() was called but Clerk can't detect usage of clerkMiddleware().
Please ensure the following:
- Your middleware file exists at ./src/middleware.(ts|js)
```

**Fix:** Rename the file and rebuild the Docker image (Turbopack SSR chunks are cached).

## 2. Matcher Regex Must Cover All Target Routes

The `config.matcher` in `middleware.ts` controls which routes the Clerk middleware intercepts. If a route isn't matched, `auth()` will fail on that route.

```typescript
export const config = {
  matcher: [
    '/((?!_next|[^?]*\\.(?:html?|css|js(?!on)|jpe?g|webp|png|gif|svg|ttf|woff2?|ico|csv|docx?|xlsx?|zip|webmanifest)).*)',
    '/(api|trpc)(.*)',
  ],
};
```

The first pattern excludes static assets and Next internals. The second pattern covers API routes. Make sure your protected pages (e.g., `/dashboard`) are not excluded.

## 3. Use `<Show>` — NOT `<SignedIn>` / `<SignedOut>`

```tsx
// WRONG — these are deprecated
<SignedIn><UserButton /></SignedIn>
<SignedOut><SignInButton /><SignUpButton /></SignedOut>

// CORRECT
<Show when="signed-out">
  <SignInButton />
  <SignUpButton />
</Show>
<Show when="signed-in">
  <UserButton />
</Show>
```

## 4. Server-Side Auth: Use `auth()` from `@clerk/nextjs/server`

```tsx
import { auth } from '@clerk/nextjs/server';

export default async function ProtectedPage() {
  const { userId } = await auth();
  if (!userId) redirect('/sign-in');
  // ...
}
```

**NOT** `getAuth()` (deprecated) or `currentUser()` from `@clerk/nextjs`.

## 5. ClerkProvider Must Be Inside `<body>` in `layout.tsx`

```tsx
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <ClerkProvider>
          {children}
        </ClerkProvider>
      </body>
    </html>
  );
}
```

## 6. Required Environment Variables

```env
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up
NEXT_PUBLIC_CLERK_AFTER_SIGN_IN_URL=/
NEXT_PUBLIC_CLERK_AFTER_SIGN_UP_URL=/
```

`NEXT_PUBLIC_CLERK_SIGN_IN_URL` and `NEXT_PUBLIC_CLERK_SIGN_UP_URL` are required for the Clerk components to work properly.

## 7. Auth Routes: Use Clerk's Component Props or Catch-All

For sign-in/sign-up, use Clerk's built-in components:

```tsx
// app/(auth)/sign-in/[[...sign-in]]/page.tsx
import { SignIn } from '@clerk/nextjs';
export default Page = () => <SignIn />;

// app/(auth)/sign-up/[[...sign-up]]/page.tsx
import { SignUp } from '@clerk/nextjs';
export default Page = () => <SignUp />;
```

The catch-all `[...sign-in]` / `[...sign-up]` route groups allow Clerk to handle all sub-routes (callback, OAuth redirect, etc.).

## 8. Docker Rebuild Required After Middleware Changes

Turbopack SSR chunks are baked into Docker image layers. After changing `middleware.ts` or installing `@clerk/nextjs`:

```bash
docker rm -f ndr-web
docker rmi ndr-agency-web
rm -rf apps/web/.next
docker compose up -d --build web
```

## 9. `@clerk/fastify` + Fastify Version Mismatch

`@clerk/fastify@3.x` requires Fastify v5. If the API server uses Fastify v4, pin to v2:
```bash
npm install @clerk/fastify@2 --save-exact
```

## Checklist Before Claiming Clerk Is Working

- [ ] `middleware.ts` exists (not `proxy.ts` or any other name)
- [ ] `clerkMiddleware()` is called in `middleware.ts`
- [ ] `ClerkProvider` wraps `{children}` in `app/layout.tsx`
- [ ] `<Show when="signed-in/out">` used (not `<SignedIn>/<SignedOut>`)
- [ ] `auth()` used server-side, not deprecated `getAuth()`
- [ ] Env vars: `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `NEXT_PUBLIC_CLERK_SIGN_IN_URL`, `NEXT_PUBLIC_CLERK_SIGN_UP_URL`
- [ ] Protected routes redirect unauthenticated users (`307` from curl means it's working)
- [ ] Docker image rebuilt after installing Clerk
