---
name: skills-maintenance
description: Skills ecosystem audit, rename, index, and sync — fix manifest mismatches, update deprecated names, regenerate INDEX.md, and push to GitHub hub (dazfx/hermes-skills).
version: 1.0
category: devops
---

# Skills Maintenance

## When to Use
- After renaming/moving skills on disk
- Manifest shows missing or mismatched entries
- Deprecated "openclaw-*" skills need cleanup
- INDEX.md is stale
- Skills were modified but GitHub hub isn't synced

## Architecture

```
~/.hermes/skills/              # Live skills (90 dirs, most via bundled plugin)
~/.hermes/skills/.bundled_manifest  # "name:hash" map — must match dir names
~/.hermes/hermes-skills/       # GitHub hub working copy (dazfx/hermes-skills)
~/.hermes/hermes-skills/skills/     # Custom skills in hub (subset of live)
~/.hermes/hermes-skills/INDEX.md    # Human-readable index for the repo
```

## Audit Checklist

### 1. Detect manifest mismatches
```bash
# Extract manifest keys and check they exist as dirs
grep -o '^[^:]*' ~/.hermes/skills/.bundled_manifest | while read key; do
  dir=$(find ~/.hermes/skills -maxdepth 3 -type d -name "$key" 2>/dev/null | head -1)
  [ -z "$dir" ] && echo "MISSING: $key"
done
```

### 2. Find skills on disk not in manifest
```bash
find ~/.hermes/skills -name 'SKILL.md' -exec dirname {} \; | xargs -I{} basename {} | sort > /tmp/on_disk.txt
grep -o '^[^:]*' ~/.hermes/skills/.bundled_manifest | sort > /tmp/in_manifest.txt
diff /tmp/on_disk.txt /tmp/in_manifest.txt
```

### 3. Detect "openclaw" legacy names
```bash
grep 'openclaw' ~/.hermes/skills/.bundled_manifest
find ~/.hermes/skills -path '*openclaw*' -name 'SKILL.md'
```

## Rename a Skill

```bash
OLD="openclaw-crash-test"
NEW="crash-test"
CAT="devops"

# 1. Copy dir with new name
cp -r ~/.hermes/skills/$CAT/$OLD ~/.hermes/skills/$CAT/$NEW

# 2. Update YAML frontmatter
sed -i "s/name: $OLD/name: $NEW/" ~/.hermes/skills/$CAT/$NEW/SKILL.md

# 3. Fix manifest entry
sed -i "s/^$OLD:/$NEW:/" ~/.hermes/skills/.bundled_manifest

# 4. Keep old dir for backward compat (or rm -rf it)

# 5. Sync to GitHub hub
rm -rf ~/.hermes/hermes-skills/skills/$NEW 2>/dev/null
cp -r ~/.hermes/skills/$CAT/$NEW ~/.hermes/hermes-skills/skills/$NEW
```

## Regenerate INDEX.md

INDEX.md lives at `~/.hermes/hermes-skills/INDEX.md`. It should be a markdown table with:
- Section per category
- Skill name + one-line purpose
- Deprecated skills table at the bottom
- Naming conventions section

Generate by running `skills_list` and grouping by category. Only list custom devops skills in detail; bundled plugin skills get compact tables.

## Sync to GitHub Hub

```bash
cd ~/.hermes/hermes-skills

# Copy modified custom skills
for skill in skills/*/; do
    name=$(basename "$skill")
    for cat in devops data-science; do
        if [ -d ~/.hermes/skills/$cat/$name ]; then
            rm -rf "skills/$name"
            cp -r ~/.hermes/skills/$cat/$name "skills/$name"
        fi
    done
done

git add -A
git commit -m "sync: <description>"
git push
```

## Pitfalls

- **Don't delete old dirs immediately** — manifest entries may still point to them; copy-then-deprecate is safer.
- **Manifest keys must match directory names exactly** — the loader uses exact string match.
- **Custom skills are in `devops/` and `data-science/` subdirs** on disk but flat `skills/` in the hub.
- **Bundled plugin skills** (autonomous-ai-agents, creative, etc.) are NOT in the GitHub hub — only listed in INDEX.md for reference.
- **state.db session files** won't break if skills move — the skill loader re-discovers each turn.
