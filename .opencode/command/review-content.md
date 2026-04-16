---
description: Reviews and improves Spanish (.es.md) syllabus content — one PR per module with atomic commits
agent: build
---

# /review-content — Module review with atomic commits in a single PR

Technical reviewer for a cybersecurity syllabus. Improves Spanish `.es.md` content by diagnosing issues, then applying all approved fixes as atomic commits on a single branch — **one PR per module**.

Argument: `$ARGUMENTS` (module directory or file path).

---

## 1. Setup (detect dynamically, never hardcode)

```bash
REPO_ROOT=$(git rev-parse --show-toplevel)
GH_USER=$(gh api user --jq .login)

for remote in $(git remote); do
  url=$(git remote get-url "$remote")
  owner=$(echo "$url" | sed -E 's#^(https://|git@)github\.com[:/]([^/]+)/.*#\2#')
  if [ "$owner" = "$GH_USER" ]; then
    FORK_REMOTE="$remote"; FORK_OWNER="$owner"
  else
    UPSTREAM_REMOTE="$remote"; UPSTREAM_OWNER="$owner"
    UPSTREAM_REPO=$(echo "$url" | sed -E 's#.*[:/]([^/]+)/([^/]+?)(\.git)?$#\2#')
  fi
done

INITIAL_BRANCH=$(git rev-parse --abbrev-ref HEAD)
```

### Constraints

- **Only `.es.md` files are edited.** Reject `.md` (English) paths.
- Module `README.md` files are an exception — they serve as the Spanish lesson index and may be edited for cross-file blocks.
- Module scope = directory name without numeric prefix: `02-linux` → `linux`, `04-seguridad-redes-1` → `redes-1`, `11-ISO-27001` → `iso-27001`.

---

## 2. Preflight

1. `gh auth status` — abort if not logged in.
2. Both `FORK_REMOTE` and `UPSTREAM_REMOTE` must be detected. Abort with setup instructions if missing.
3. `git status --porcelain` — if dirty, ask: stash / abort / ignore.
4. `git fetch "$UPSTREAM_REMOTE" main --quiet`.
5. Sync fork main (best-effort, non-blocking):
   ```bash
   gh repo sync "${FORK_OWNER}/${UPSTREAM_REPO}" \
     --source "${UPSTREAM_OWNER}/${UPSTREAM_REPO}" --branch main 2>/dev/null || true
   ```

---

## 3. Target resolution

`$ARGUMENTS` may be:

- **Empty** → list modules (`ls -d [0-9][0-9]-*`), ask user to pick.
- **Module name** (`02-linux`) → all `*.es.md` files in that module.
- **File path** → resolve against `REPO_ROOT`, must end in `.es.md`.

Validate existence. If not found, suggest similar names.

---

## 4. Diagnosis (read-only, no edits)

Read all target files. Produce a structured diagnosis along **four axes**, grouped into **atomic blocks**. Each block = one future commit.

### Axis A — Technical content

- Factual errors (dates, RFCs, CVEs, protocols, versions).
- Outdated statements (deprecated tools, retired protocols).
- Dangerous oversimplifications (security, crypto, permissions).
- Imprecise terminology (kernel vs OS, encryption vs hash).

### Axis B — Structure & pedagogy

- Concept ordering (simple → complex).
- Duplicated sections or repeated paragraphs.
- Heading hierarchy violations (`#` → `###` skipping `##`).
- Terms used before being defined.
- Poor didactic examples.
- Missing summary, next-steps, or references.

### Axis C — Markdown quality

- Malformed tables (misaligned pipes, missing separator).
- Code blocks without language fences.
- Inconsistent callouts.
- Images without `alt` text.
- Broken or bare-URL links.
- Formatting inconsistencies.

### Axis D — Cross-file coherence (module-level only)

When reviewing an entire module:

- **Duplicated content** across files (tables, command lists, explanations).
- **Lesson ordering** vs README dependency graph.
- **Scope overlap** — accidental vs intentional.
- **Orphaned references** ("como vimos anteriormente" pointing to nonexistent content).

### Output format

```
## Diagnosis: <file.es.md>

### Block N — <short title>
- **Axis**: content | structure | visualization | cross-file
- **Severity**: critical | high | medium | low
- **Problem**: <description>
- **Proposal**: <what to change>
- **Why**: <reason>
```

Cross-file blocks go in a separate section with `Files involved` field.

**After diagnosis**: ask which blocks to apply — `all`, `1,3,5`, `only critical`, or `none`.

---

## 5. Execution (autonomous, single branch, single PR)

### 5.1 Create module branch

```bash
git fetch "$UPSTREAM_REMOTE" main --quiet
BRANCH="docs/<scope>/review"
git checkout -b "$BRANCH" "${UPSTREAM_REMOTE}/main"
```

If branch exists, append `-v2`, `-v3`.

### 5.2 Apply all approved blocks

Process blocks in severity order (critical → low). For each block:

1. Apply targeted edit (never rewrite entire file).
2. Verify by re-reading the modified region.
3. Commit atomically:
   ```bash
   git add "<path/to/file.es.md>"
   git commit -m "<type>(<scope>): <summary>"
   ```

**No per-block confirmation.** The user already approved the block list in §4. Apply them autonomously.

**Only pause if** a change is destructive (deleting significant content) or genuinely ambiguous.

### 5.3 Push and open PR

```bash
git push -u "$FORK_REMOTE" "$BRANCH"

gh pr create \
  --repo "${UPSTREAM_OWNER}/${UPSTREAM_REPO}" \
  --base main \
  --head "${FORK_OWNER}:${BRANCH}" \
  --draft \
  --title "docs(<scope>): review <module-name> content" \
  --body "$(cat <<'EOF'
## What

Comprehensive review of `<module>/` Spanish content.

## Changes

- <one bullet per commit: type + description>

## Why

<Overall motivation: accuracy, pedagogy, formatting consistency.>

## Files

- `<file1.es.md>`
- `<file2.es.md>`
EOF
)"
```

### 5.4 Return to initial branch

```bash
git checkout "$INITIAL_BRANCH"
```

Pop stash if one was created in preflight.

---

## 6. Commit rules

- Conventional Commits: `type(scope): summary`.
- Types: `docs`, `fix`, `refactor`, `style`. Content is almost always `docs`.
- Scope: module without prefix (`linux`, `redes`, `pentesting`, `iso-27001`).
- Summary: lowercase, imperative, < 72 chars, English.
- **No body.** Summary line only.
- **Forbidden**: `Co-authored-by`, `Generated-by`, AI/tool mentions.

---

## 7. Hardblocks

- Never `git push --force`.
- Never push to upstream `main`.
- Never `git add -A` or `git add .` — only the specific file.
- Never mix changes from two blocks in the same commit.
- Never touch English `.md` content files.
- Never mention AI, tools, or generation process in commits, PRs, or branches.
- Never add attribution trailers.

### Failure handling

- Push rejected → report, do not force.
- PR creation fails → show error, leave branch pushed, ask user.
- Edit fails verification → `git checkout -- <file>`, report.
- Merge conflict → abort, report, do not auto-resolve.

---

## 8. Session summary

```
## Summary

**Module**: <name>
**Blocks identified**: N
**Blocks applied**: M
**Blocks skipped**: K

### PR
- [#<num>] <title> → <url>

### Pending (next session)
- Block X: <description>
```

---

## 9. Tone

- Interaction language: follow user's `AGENTS.md` preference (default: Spanish).
- Commits, branches, PR content: always English.
- If a proposed change is wrong, say so before applying.
- If a change is purely cosmetic with no real benefit, flag it and skip unless user insists.
