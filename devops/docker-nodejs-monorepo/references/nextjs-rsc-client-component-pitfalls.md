# Next.js 15 + React 19 RSC / Client Component Pitfalls

## The `'use client'` Directive Is Not Transitive in Next.js 15

In Next.js 15 with Turbopack, `'use client'` does NOT bubble up through component composition. Every component that uses browser APIs, event handlers, or passes them as props needs its own `'use client'` directive.

**Failure mode:** `Event handlers cannot be passed to Client Component props` — SSR throws 500 even though the page compiles and hot-reload shows no errors.

**Root cause:** If a Server Component renders a Client Component that renders another component with an `onClick`, the innermost component is still treated as a Server Component during SSR. The `onClick` handler cannot be serialized across the RSC boundary.

## shadcn/ui Components Are Not Automatically Client Components

Every shadcn/ui component you `npx shadcn@latest add` must be annotated with `'use client'` at the top of the file if it:
- Uses `onClick` or any event handler
- Uses `useState`, `useEffect`, or other React hooks
- Renders a native `<button>`, `<input>`, or similar with browser APIs

**The `Button` component specifically:** If you copy the shadcn `button.tsx` into your project without adding `'use client'` at the very top, any page (even a Server Component) that renders `<Button onClick={...}>` will throw a 500 during SSR.

**Correct:**
```tsx
'use client';

import * as React from 'react';
import { Slot } from '@radix-ui/react-slot';
// ...
```

## Practical Rule

| Component type | Needs `'use client'`? |
|---|---|
| Page (app/page.tsx) with no interactivity | No |
| Page with `<Link>` or plain `<a>` | No |
| Page with `onClick` or hooks | Yes |
| Component used inside a Server Component page, with event handlers | Yes (mark the component, not the page) |
| Shared UI component (Button, Input, Dialog) | Always yes if it uses browser APIs |

## When the Error Persists After Adding `'use client'`

**Symptom:** You added `'use client'` but the 500 still happens on reload.

**Cause:** Turbopack caches SSR chunks inside the Docker image layer. Adding `'use client'` to a file that was already built into the image does NOT clear the cached SSR chunk.

**Fix — full purge, not partial rebuild:**

```bash
# Remove containers and images
docker rm -f ndr-web ndr-api
docker rmi ndr-agency-web ndr-agency-api

# Clear Turbopack cache
rm -rf apps/web/.next

# Rebuild from zero
docker compose up --build -d
```

Partial rebuilds (`touch`, `--no-cache`, `docker compose up --build` alone) do NOT clear Turbopack's internal chunk cache embedded in the image. You must `docker rmi` to force a full rebuild of the image layers.

## Related

- Docker + Next.js hot-reload: volumes must NOT be `:ro` — see `docker-nodejs-monorepo` SKILL.md
- Next.js standalone output for Docker: `output: 'standalone'` in `next.config.js`
