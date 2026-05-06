# Directory Copy Naming with Spaces

When copying skills repo directories to `~/.hermes/skills/`, directory names with spaces can cause issues.

**Symptom:** A skill shows in `hermes skills list` as "local" but `skill_view(name)` fails with "not found."

**Root cause:** `cp -r /tmp/skills-repo/* ~/.hermes/skills/` preserves spaces in directory names. The skill system may resolve names with spaces differently than expected.

**Workaround:** After copying, rename directories to strip/replace spaces:
```bash
cd ~/.hermes/skills
for dir in *" "*; do
  mv "$dir" "$(echo $dir | tr ' ' '-')"
done
```

**Better approach:** Clone with safe names, or use `rsync --exclude='* *'` before copying to a flat structure.

**Note:** Skills with spaces in directory names may still load fine in some contexts but fail in `skill_view()` lookups. If a skill is unfindable, check its actual directory name vs. its internal `name:` field in SKILL.md.
