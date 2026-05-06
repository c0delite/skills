# Supabase Connection Failure Triage

When `prisma db push` or `prisma migrate` fails against a Supabase PostgreSQL instance, the error falls into three buckets:

## Error Code P1000 — Authentication Failed

```
Error: P1000: Authentication failed against database server at `aws-1-ap-southeast-1.pooler.supabase.com`,
the provided database credentials for `postgres` are not valid.
```

**Cause:** The password in `DATABASE_URL` is wrong. Supabase rotates passwords; the connection string in the dashboard is the source of truth.

**Fix:**
1. Go to Supabase Dashboard → Project Settings → Database
2. Copy the full connection URI (not just the password)
3. Update `api/.env` with the correct `DATABASE_URL`

## Error Code P1001 / Connection Timeout — Network / Pooler

```
Error: P1001: The database server at `aws-1-ap-southeast-1.pooler.supabase.com:6543` was not reached
```

or timed out after 300s.

**Causes:**
- Supabase pooler is slow to respond from this network location
- Firewall blocks port 6543 (Supabase uses port 6543 for pooler, not 5432)
- Instance is paused or overloaded

**Fix:**
- Use `--accept-data-loss` flag (skips safety checks, speeds up)
- For long-running ops, run in background and wait: `prisma db push --accept-data-loss 2>&1 &`
- Check Supabase status page: https://status.supabase.com
- Try direct port 5432 instead of pooler port 6543 if available

## Error Code P2025 — Migration History Conflict

```
Error: P2025: An operation failed due to a pre-existing migration failure
```

**Cause:** A previous migration left the `_prisma_migrations` table in an inconsistent state.

**Fix:**
```bash
# Reset migration state ( destructive — only in dev):
prisma migrate resolve --applied 20231201000000_add_users  # mark as applied
# OR
prisma migrate resolve --rolled-back 20231201000000_add_users  # mark as rolled back
```

## Supabase Connection URL Format

Supabase uses a pooler at port 6543:
```
postgresql://postgres.<PROJECT_REF>:<PASSWORD>@aws-1-<REGION>.pooler.supabase.com:6543/postgres
```

The `PROJECT_REF` and `PASSWORD` are in the Supabase dashboard connection string. Common mistakes:
- Using `postgres` as the user instead of `postgres.<PROJECT_REF>`
- Using port 5432 instead of 6543 (direct connection vs. pooler)
- Copying the password from the wrong field

## Quick Diagnostic

```bash
# Test basic reachability (requires nc or timeout):
timeout 10 nc -zv aws-1-ap-southeast-1.pooler.supabase.com 6543

# Test with Prisma in dry-run mode (doesn't touch DB):
./node_modules/.bin/prisma db execute --stdin --schema=api/prisma/schema.prisma <<< 'SELECT 1' 2>&1 | head -5

# Force schema push with max verbosity:
DEBUG="*" ./node_modules/.bin/prisma db push --schema=api/prisma/schema.prisma 2>&1 | grep -E "(P1000|P1001|P2025|Error|connected)"
```
