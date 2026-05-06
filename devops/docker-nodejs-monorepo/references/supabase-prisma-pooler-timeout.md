# Supabase + Prisma: pgBouncer Pooler Timeout Workaround

## Problem Statement

`prisma db push` hangs indefinitely (>600s) against Supabase pgBouncer pooler (port 6543) with timeout errors. Supabase REST API works fine. Direct PostgreSQL port 5432 may be unreachable (IPv6 routing).

## Session Transcript (2026-05-05)

```
Error: P1001: Can't reach database server at `db.xjrzdbyqihxwirolevax.supabase.co:5432`
Error: P1000: Authentication failed against database server at `aws-1-ap-southeast-1.pooler.supabase.com`
```

## Root Cause Chain

1. `.env` had wrong `DATABASE_URL` format — used direct connection format (`postgres:password@db.host:5432`) instead of pooler format (`postgres.PROJECT_REF:password@aws-REGION.pooler.supabase.com:6543`)
2. After fixing URL, pooler auth worked but pgBouncer session establishment was extremely slow (>600s)
3. Prisma hangs at "Creating database..." step — pgBouncer + Prisma 5.x have known slow-first-query issues
4. Supabase REST API (PostgREST) worked perfectly — confirmed database was reachable

## Diagnostic Checklist

```bash
# 1. Verify pooler port is reachable
timeout 10 nc -zv aws-1-ap-southeast-1.pooler.supabase.com 6543

# 2. Verify Supabase REST API works (confirms project is active)
curl -s "https://PROJECT_REF.supabase.co/rest/v1/" \
  -H "apikey: <ANON_KEY>"

# 3. Check if project is paused (free tier pauses after 7 days inactivity)
# → If REST returns 200, project is awake

# 4. Check IPv4 vs IPv6 for direct port
timeout 10 nc -zv db.PROJECT_REF.supabase.co 5432
# If "Network is unreachable" — IPv6 routing issue, use pooler

# 5. Check Prisma version compatibility
./node_modules/.bin/prisma --version
```

## Workaround: Apply SQL via Supabase Dashboard

```bash
# Generate migration SQL without touching the database
cd ~/agency-platform/api
npx prisma migrate diff --from-empty \
  --to-schema-datamodel prisma/schema.prisma \
  --script > ../supabase-migration.sql
```

Then in Supabase Dashboard → SQL Editor → paste contents of `supabase-migration.sql` → Run.

## Known Good Configuration (2026-05-05)

```json
{
  "node": "18.19.1",
  "prisma": "5.22.0",
  "@prisma/client": "5.22.0",
  "@clerk/fastify": "2.6.31",
  "fastify": "^4.28.0"
}
```

## Key Lesson

> When Prisma hangs against Supabase pooler but REST API works, skip Prisma entirely. Generate the SQL migration and paste it into the Supabase SQL Editor — it's faster and more reliable than debugging pgBouncer session issues.
