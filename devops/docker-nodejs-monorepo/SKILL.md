---
name: docker-nodejs-monorepo
description: "Docker build patterns for Node.js monorepos and workspaces: lockfile isolation, Prisma generation, multi-stage builds, hot-reload, and multi-service orchestration with docker compose."
version: 1.0.0
---

# Docker + Node.js Monorepo Build Patterns

## When to Use

- Multi-service Node.js app built with Docker (frontend + API in one repo)
- Workspace uses npm workspaces, pnpm workspaces, or independent package.json dirs
- Prisma (or similar codegen tool) must run inside Docker
- Hot-reload needed inside containers during development

## Core Problem: Lockfile Isolation

Docker copies `package*.json` from each service's directory and runs `npm install` there. If the lockfile is missing or stale, npm resolves to the latest semver-matching version — NOT the version pinned in your workspace lockfile.

**Example failure:** Workspace has Prisma v5.22.0 pinned. Docker `FROM deps AS prisma` stage runs `npm install` in `api/` context — no lockfile there — so npm installs Prisma v7.x. Prisma v7 has breaking changes (datasource `url` property removed), causing `prisma generate` to fail with:
```
Error: The datasource property `url` is no longer supported in schema files.
```

## Solution: Copy Lockfiles Into Docker Contexts

After any `package-lock.json` change, copy it into every service subdir that Docker will build:

```bash
# After npm install/update in workspace root:
cp package-lock.json api/package-lock.json
cp package-lock.json frontend/package-lock.json
```

Then rebuild:
```bash
docker compose up -d --build
```

## Dockerfile Patterns

### API Dockerfile (Node.js + Prisma)

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps

FROM deps AS prisma
COPY prisma/ ./prisma/
RUN npx prisma generate          # uses ./node_modules/.bin/prisma
COPY .prisma/ /app/node_modules/.prisma/

FROM deps AS builder
COPY --from=prisma /app/node_modules/.prisma/ /app/node_modules/.prisma/
COPY src/ ./src/
RUN npm run build

FROM runner AS prod
WORKDIR /app
COPY --from=builder /app/dist/ ./dist/
COPY --from=prisma /app/node_modules/.prisma/ /app/node_modules/.prisma/
COPY package*.json ./
RUN npm ci --production --legacy-peer-deps
CMD ["node", "dist/index.js"]
```

Key points:
- The `prisma` stage copies the lockfile-pinned Prisma binary (from `node_modules/.bin/prisma`)
- `npx prisma generate` in the `prisma` stage uses the local `./node_modules/.bin/prisma` (from deps stage), NOT a globally installed version
- Copy `.prisma/` into `node_modules` so the prod stage can run `npm ci --production` without regenerating

### Frontend Dockerfile (Next.js)

```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --legacy-peer-deps

FROM deps AS builder
COPY src/ ./src/
RUN npm run build

FROM runner AS prod
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/ ./.next/
COPY --from=builder /app/public/ ./public/
COPY package*.json ./
RUN npm ci --production --legacy-peer-deps
CMD ["npm", "start"]
```

## Hot Reload

For development with hot-reload, mount source as a **writable** volume (never `:ro`):

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.api
    volumes:
      - ./apps/api/src:/app/src        # NOT :ro — hot-reload requires write
      - ./packages/shared/src:/app/packages/shared/src
    ports:
      - "3002:3001"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=${DATABASE_URL}
    depends_on:
      - db

  frontend:
    build:
      context: .
      dockerfile: Dockerfile.frontend
    volumes:
      - ./apps/web/src:/app/src        # NOT :ro — Next.js turbopack needs write
      - ./packages/shared/src:/app/packages/shared/src
      - ./apps/web/public:/app/public
    ports:
      - "3003:3000"
    environment:
      - NODE_ENV=development
      - NEXT_PUBLIC_API_URL=http://localhost:3002
```

**Warning:** Volume mounts override the image's `src/` — any `node_modules` inside `src/` in the image are hidden. Ensure the image builds a working `node_modules` first.

## Docker Compose Commands

| Command | Purpose |
|---------|---------|
| `docker compose up -d` | Start all services |
| `docker compose up -d --build` | Rebuild and start |
| `docker compose exec api sh` | Shell into running container |
| `docker compose logs api -f` | Follow API logs |
| `docker compose restart api` | Restart API without rebuilding |
| `docker compose down` | Stop all services |

Note: Use `docker compose` (v2, hyphenated) not `docker-compose` (v1, separate binary).

## Prisma in Docker: The npx Problem

`npx prisma generate` inside Docker does NOT use your local Prisma version. It downloads the latest `@prisma/client` and `prisma` that match the schema's `engineType`. To force the lockfile version:

1. Ensure `package-lock.json` is present in the Docker build context (copy it as shown above)
2. Run `./node_modules/.bin/prisma generate` instead of `npx prisma generate`
3. Or pin Prisma to exact version: `"prisma": "5.22.0"` (no `^`)

## Port Conflicts

If local port 3000 is occupied, remap in docker-compose:

```yaml
ports:
  - "3002:3000"   # host:container
```

Set `NEXT_PUBLIC_API_URL=http://localhost:4000` in the frontend container's environment to match.

## Exit Criteria for a Working Docker Setup

After running `docker compose up -d --build`:
- `docker compose ps` shows all services as `Up`
- `curl http://localhost:3002` returns the frontend
- `curl http://localhost:4000/health` returns 200
- `docker compose exec api npx prisma --version` matches workspace version

## Common Failures

| Symptom | Cause | Fix |
|---------|-------|-----|
| `sh: docker: not found` in container | `command:` in docker-compose.yml overrides Dockerfile CMD — if the override runs `docker compose up`, you get infinite recursion | Remove `command:` / `command: npm run dev` lines from docker-compose.yml service definitions. The Dockerfile CMD already specifies the correct start command. |
| Hot-reload not working | Volumes mounted `:ro` (read-only) prevent the dev server from watching files | Mount volumes **without** `:ro` suffix. Use `./src:/app/src` not `./src:/app/src:ro` |
| Turbopack/RSC 500 errors persisting after file fix | Stale SSR chunks baked into Docker image layer cache | **Full purge required:** `docker rm -f <service>` + `docker rmi <image>` + `rm -rf apps/web/.next` + `docker compose up --build`. Partial rebuilds (touch, `--no-cache` on compose) do NOT clear Turbopack's internal chunk cache embedded in the image. |
| `npx prisma` installs latest | Missing lockfile in Docker context | Copy `package-lock.json` to service subdir |
| Port already in use | Host port conflict | Change host port in `ports:` mapping |
| Module not found | Volume mount hides image's node_modules | Rebuild image first, ensure `npm ci` ran in image |
| `ECONNREFUSED` db | Database not ready | Add `depends_on` + healthcheck, or retry logic |
| `git clone` fails "could not read Username" | Cloning to a directory that doesn't exist, or PAT used in URL without credential helper | Always `cd` to parent dir first, or clone to `/tmp` then move; for automated clones use `GITHUB_TOKEN=$(grep github_token ~/.hermes/.env | cut -d= -f2)` then `git clone https://${GITHUB_TOKEN}@github.com/org/repo.git /target/path` |
| Frontend 500 on API call | CORS not configured | Set `CORS_ORIGIN=http://localhost:3002` on API |

## Monorepo Prisma Operations

In a workspace monorepo (e.g., `frontend/` + `api/` subdirs), the Prisma CLI binary and engine binaries live at the **workspace root `node_modules/`**, not inside a service subdir's `node_modules/`. This affects how you invoke Prisma from both CLI and Docker.

**Correct invocation:**
```bash
# From workspace root (where node_modules/ lives):
./node_modules/.bin/prisma db push --schema=api/prisma/schema.prisma
./node_modules/.bin/prisma migrate dev --schema=api/prisma/schema.prisma
./node_modules/.bin/prisma generate --schema=api/prisma/schema.prisma
```

**Wrong — appears to work but uses wrong version:**
```bash
cd api && npx prisma ...   # Downloads latest, ignores workspace lockfile
```

**Dockerfile: use the local binary, not npx:**
```dockerfile
# In api/Dockerfile — copy workspace lockfile first so Prisma v5 stays pinned
COPY package-lock.json ./
RUN npm ci --legacy-peer-deps

# Then invoke with local binary, not npx
RUN ./node_modules/.bin/prisma generate --schema=prisma/schema.prisma
```

**Common error if you use `npx` in Docker:** `Error: The datasource property 'url' is no longer supported` — Prisma v7 downloaded instead of lockfile-pinned v5. See `references/prisma-v7-breaks-schema-url.md`.

## Prisma + Supabase Pooler Timeout (P1001)

`prisma db push` or `prisma migrate` through Supabase's pgBouncer pooler (port 6543) can hang indefinitely (600s+ timeout). This is a known pgBouncer + Prisma interaction issue.

**Workaround — generate SQL and apply via Supabase SQL Editor:**

```bash
cd ~/agency-platform/api
npx prisma migrate diff --from-empty --to-schema-datamodel prisma/schema.prisma --script > ../supabase-migration.sql
```

Then paste `supabase-migration.sql` contents into Supabase Dashboard → SQL Editor → New Query → Run.

**If pooler is unavoidable**, run in background with long timeout:
```bash
npx prisma db push 2>&1 &
sleep 900 && echo "check output"
```

**Supabase direct port (5432) may be unreachable via IPv6** — pooler (6543) works but times out. The Supabase REST API (`curl`) works fine even when Prisma hangs, confirming the issue is pgBouncer session establishment, not the network.

## `@clerk/fastify` + Fastify Version Mismatch

`@clerk/fastify@3.x` requires Fastify v5. Projects on Fastify v4 will see peer dependency warnings and potential runtime breakage.

**Fix:** pin to v2:
```bash
npm install @clerk/fastify@2 --save-exact
```

## References

- `references/nextjs-rsc-client-component-pitfalls.md` — Next.js 15 + React 19 `'use client'` boundary issues, shadcn/ui client component rules, Turbopack cache purge procedure
- `references/supabase-connection-issues.md` — Supabase connection failure triage (auth vs. network vs. pooler timeout), now includes IPv6 routing issues and pooler timeout workarounds
- `references/prisma-v7-breaks-schema-url.md` — exact error transcript, root cause, fix sequence, and known-good version pins for Prisma/Clerk/Fastify
- `references/clerk-nextjs-gotchas.md` — Clerk + Next.js App Router critical pitfalls: middleware.ts naming, matcher regex, `<Show>` vs deprecated `<SignedIn>`, `auth()` server-side, env var names

## Related Skills

- `pm2-workload-management` — for running Node.js in development without Docker
- `github-repo-management` — for setting up CI that builds and pushes Docker images
