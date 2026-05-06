# Prisma v7 Breaking Change: `url` Property Removed

## The Error

When Docker builds with Prisma v7.x (released March 2025) instead of v5.x, `prisma generate` fails:

```
Error code: P1012
error: The datasource property `url` is no longer supported in schema files.
Move connection URLs for Migrate to `prisma.config.ts`
```

Full Prisma 7 migration guide: https://www.prisma.io/docs/orm/more/upgrade-guides/upgrading-versions/upgrading-to-prisma-7

## Root Cause

Schema.prisma (Prisma v5 style):
```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

Prisma v7 requires adapter pattern in `prisma.config.ts`:
```typescript
export default defineConfig({
  earlyAccess: true,
  schema: './prisma/schema.prisma',
  migrate: {
    adapter: async () => {
      const { Pool } = await import('pg')
      return new Pool({ connectionString: process.env.DATABASE_URL })
    }
  }
})
```

## Why It Happens in Docker

1. Workspace has `package-lock.json` with `prisma: 5.22.0`
2. `api/package-lock.json` is deleted or missing (not copied into Docker context)
3. Docker `npm install` in `api/` context has no lockfile → resolves latest semver → installs `prisma@^5.10.2` which is now v7.x
4. Container runs `npx prisma generate` → downloads latest → Prisma v7 → breaks

## Fix Sequence

```bash
# Always after workspace npm install:
cp package-lock.json api/package-lock.json
cp package-lock.json frontend/package-lock.json

# Rebuild
docker compose up -d --build
```

Alternatively, pin to exact version to prevent any upgrade:
```bash
npm install prisma@5.22.0 @prisma/client@5.22.0 --save-exact
```

## Verified Versions (as of May 2025)

| Package | Working | Broken |
|---------|---------|--------|
| `prisma` | 5.22.0 | 7.x |
| `@prisma/client` | 5.22.0 | 7.x |
| `@clerk/fastify` | 2.x | 3.x (requires Fastify v5) |
| `fastify` | 4.x | 5.x (peer dep conflict with @clerk/fastify v2) |

## Peer Dep Conflict Pattern

When `@clerk/fastify@2` (Fastify v4 compatible) conflicts with `fastify@5`:
- Use `--legacy-peer-deps` on all `npm ci`/`npm install` commands
- Do NOT upgrade `@clerk/fastify` to v3 unless also upgrading Fastify to v5
