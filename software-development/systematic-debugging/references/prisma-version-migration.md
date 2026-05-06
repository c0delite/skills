# Prisma Version Compatibility Notes

## Prisma 7 Breaking Change: datasource url removed from schema

**Symptom:**
```
Error: The datasource property `url` is no longer supported in schema files.
Move connection URLs for Migrate to `prisma.config.ts`
```

**Cause:** Prisma 7 moved connection URL configuration out of `schema.prisma` into `prisma.config.ts`. The `datasource db { url = env("DATABASE_URL") }` block in schema.prisma is no longer valid.

**Fix options:**

### Option A — Pin to Prisma 5 (faster, stable)
```bash
cd api && npm install prisma@5 @prisma/client@5
# schema.prisma stays unchanged with datasource url
```

### Option B — Migrate to Prisma 7
1. Remove `url` from `api/prisma/schema.prisma`:
   ```prisma
   datasource db {
     provider = "postgresql"
     // url moved to prisma.config.ts
   }
   ```
2. Create `api/prisma.config.ts`:
   ```ts
   import path from 'node:path'
   import type { PrismaConfig } from 'prisma'
   
   export default {
     earlyAccess: true,
     schema: path.join(__dirname, 'prisma', 'schema.prisma'),
     migrate: {
       adapter: async () => {
         const { PrismaPg } = await import('@prisma/adapter-pg')
         return new PrismaPg({ connectionString: process.env.DATABASE_URL })
       }
     }
   } satisfies PrismaConfig
   ```
3. Update `api/src/lib/prisma.ts` to pass adapter to PrismaClient constructor

## Prisma generate inside Docker

**Problem:** `npx prisma generate` inside a Docker build requires a valid `DATABASE_URL` format but does NOT need an actual live connection. Prisma introspects the schema file, not the database, to generate the client.

**Fix:** Pass a valid-format connection string as a build arg, even if it points to a non-routable address:
```dockerfile
ARG DATABASE_URL="postgresql://placeholder:placeholder@localhost:5432/placeholder"
RUN npx prisma generate
```

Or copy the locally-generated `.prisma/client` into the Docker image:
```dockerfile
COPY api/node_modules/.prisma /app/node_modules/.prisma
```

## Docker Monorepo Root Cause (Lockfile Isolation)

**Problem:** Workspace has `prisma@5.22.0` pinned in `package-lock.json`. Docker builds `api/` context — no lockfile there — so `npm install` resolves `^5.10.2` to the latest, which is now Prisma v7. Prisma v7 rejects `url = env("DATABASE_URL")` in schema.prisma.

**Root cause chain:**
1. `api/package-lock.json` is missing from Docker context
2. `npm install` in Docker has no lockfile → semver range `^5.10.2` resolves to latest → Prisma v7
3. `npx prisma generate` uses Docker's Prisma v7, not workspace's v5.22.0

**Fix sequence (always run after workspace npm install):**
```bash
cp package-lock.json api/package-lock.json
cp package-lock.json frontend/package-lock.json
docker compose up -d --build
```

**Prevention:** Pin to exact version to prevent any upgrade path:
```bash
npm install prisma@5.22.0 @prisma/client@5.22.0 --save-exact
```

**Best practice for Docker monorepos:** Copy workspace lockfile into every service subdir before every build. This ensures Docker's `npm ci` uses the exact pinned versions from your workspace lockfile, not semver-resolved latest.

## Supabase connection URL format

```
postgresql://postgres.[REF]:[PASSWORD]@aws-0-[REGION].pooler.supabase.com:6543/postgres?pgbouncer=true
```

Get these from Supabase project settings → Connection Pooling.
