# Monorepo Scaffold Patterns

## Standard Agency Platform Stack

This session established a reusable stack for agency/client SaaS platforms:

| Layer | Choice | Rationale |
|-------|--------|-----------|
| Frontend | Next.js 14 App Router, TypeScript, Tailwind, shadcn/ui | App Router for dashboard, SEO pages need separate SSR |
| Backend | Node.js + Fastify + TypeScript | Faster than Express, built-in validation, modern |
| ORM | Prisma | Type-safe, works well with Supabase |
| Auth | Clerk | Drop-in auth, handles sessions, orgs — avoids rolling auth |
| Payments | Stripe | Standard for agency billing |
| Database | Supabase (PostgreSQL) | Postgres + auth + storage + realtime in one managed service |
| Container | Docker + docker-compose | Development orchestration |
| CI/CD | GitHub Actions | Tight git integration, free for open source |

## Directory Structure Convention

```
project/
├── frontend/          # Next.js — complete standalone app
│   ├── src/app/      # App Router pages
│   ├── src/components/
│   ├── src/lib/
│   ├── e2e/          # Playwright tests
│   └── src/test/     # Vitest unit tests
├── api/              # Fastify — complete standalone API
│   ├── src/routes/
│   ├── src/middleware/
│   ├── src/lib/
│   └── prisma/       # Schema + migrations
├── docker-compose.yml
├── Dockerfile.frontend
├── Dockerfile.api
├── SPEC.md           # Single source of truth
└── README.md
```

## Docker Multi-Stage Build Pattern

### Frontend (Next.js)
```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package*.json ./
RUN npm ci

FROM base AS development
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

FROM base AS production
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

### API (Node/Fastify)
```dockerfile
FROM node:20-alpine AS base
WORKDIR /app

FROM base AS deps
COPY package*.json ./
RUN npm ci

FROM base AS development
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 4000
CMD ["npm", "run", "dev"]

FROM base AS production
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build   # tsc
EXPOSE 4000
CMD ["node", "dist/index.js"]
```

### docker-compose hot-reload pattern
```yaml
services:
  frontend:
    build:
      context: ./frontend
      target: development
    volumes:
      - frontend-app:/app        # named volume for source
      - /app/node_modules       # bind mount to PRESERVE container node_modules
    ports:
      - "3000:3000"
```

The key insight: mount the source AS a named volume (`frontend-app:/app`) so the container sees source code for hot-reload, but separately mount `/app/node_modules` as a bind mount pointing to a host path that doesn't exist — this makes Docker ignore the host's (empty/missing) node_modules and use the container's installed node_modules instead.

## Supabase Schema Conventions

```prisma
model User {
  id        String   @id @default(cuid())
  clerkId   String   @unique   // Clerk ID, not local password
  email     String   @unique
  name      String?
  role      String   @default("member")  // admin, manager, member
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}

model Client {
  id        String    @id @default(cuid())
  name      String
  email     String?
  company   String?
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
  projects  Project[]
  invoices  Invoice[]
}

model Project {
  id          String   @id @default(cuid())
  name        String
  status      String   @default("active")
  hourlyRate  Float    @default(0)
  clientId    String
  client      Client   @relation(...)
  // ...
}
```

Use `cuid()` not `uuid()` for Prisma — Supabase handles UUIDs differently.

## Test Stack Convention

| Test Type | Tool | Location |
|-----------|------|----------|
| Backend unit | Jest + ts-jest | `api/src/routes/*.test.ts` |
| Backend API | Supertest (on top of Jest) | Same as above |
| Frontend unit | Vitest + jsdom + @testing-library/react | `frontend/src/test/` |
| E2E | Playwright | `frontend/e2e/` |

Jest config key setting for ESM TypeScript:
```js
{
  extensionsToTreatAsEsm: ['.ts'],
  transform: { '^.+\\.tsx?$': ['ts-jest', { useESM: true }] },
}
```

## API Route Pattern (Fastify)

```typescript
import { FastifyInstance } from 'fastify'
import { z } from 'zod'
import { prisma } from '../lib/prisma'

const createSchema = z.object({
  name: z.string().min(1),
  email: z.string().email().optional(),
})

export async function resourceRoutes(fastify: FastifyInstance) {
  fastify.get('/resources', async (req, res) => {
    return prisma.resource.findMany({ include: { _count: { select: { ... } } } })
  })

  fastify.post('/resources', async (req, res) => {
    const data = createSchema.parse(req.body)
    return prisma.resource.create({ data })
  })

  fastify.get('/resources/:id', async (req, res) => {
    const { id } = req.params as { id: string }
    const resource = await prisma.resource.findUnique({ where: { id } })
    if (!resource) return res.code(404).send({ error: 'Not found' })
    return resource
  })
}
```

Register with prefix: `fastify.register(resourceRoutes, { prefix: '/api/v1' })`

## CI/CD Pipeline Pattern (GitHub Actions)

Three-stage gate:
1. **Test** (parallel): backend Jest + frontend Vitest + Playwright E2E
2. **Build** (on main only, after tests pass): Docker images to GHCR
3. **Deploy**: add Railway/Fly.io/Render commands

E2E in CI requires starting the built app:
```yaml
- name: Start frontend
  run: npm start &
  env: { ... }
- name: Wait for server
  run: npx wait-on http://localhost:3000 --timeout 60000
- name: Run Playwright
  run: npx playwright test
```
