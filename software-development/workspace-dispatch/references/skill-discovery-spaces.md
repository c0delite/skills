# Skill Discovery: fullstack-dev, next-best-practices, supabase-postgres-best-practices

These skills appear in `hermes skills list` as "local" but `skill_view(name)` returns "not found".

**Likely cause:** Directory names have spaces, causing the skill resolver to fail on name lookup even though the skill is technically installed.

- `Full-Stack Development Practices` → expected name: `fullstack-dev`
- `Next.js Best Practices` → expected name: `next-best-practices`
- `Supabase Postgres Best Practices` → expected name: `supabase-postgres-best-practices`

**Fix:** Rename the directories in `~/.hermes/skills/` to strip spaces, then retry.

```bash
cd ~/.hermes/skills
# Rename spaces to hyphens
for dir in *" "*; do
  target=$(echo "$dir" | tr ' ' '-')
  mv "$dir" "$target"
  echo "Renamed: $dir -> $target"
done
```

After renaming, `skill_view(name)` should work using the hyphenated name derived from the directory.

**Prevention:** When cloning skills repos, clone to safe (no-space) directory names first, then copy.
