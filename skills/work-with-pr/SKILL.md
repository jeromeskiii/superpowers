---
name: work-with-pr
description: "Full PR lifecycle with verification loop"
---

# work-with-pr

**Description:** Full PR lifecycle: git worktree → implement → atomic commits → PR creation → verification loop (CI + review-work + Cubic approval) → merge. Keeps iterating until ALL gates pass and PR is merged. Worktree auto-cleanup after merge. Triggers: 'create a PR', 'implement and PR', 'work-with-pr', 'PR workflow', 'implement end to end'.

---

# Work With PR — Full PR Lifecycle

```
Phase 0: Setup         → Branch + worktree in sibling directory
Phase 1: Implement     → Do the work, atomic commits
Phase 2: PR Creation   → Push, create PR targeting dev
Phase 3: Verify Loop   → Unbounded iteration until ALL gates pass:
  ├─ Gate A: CI         → gh pr checks (test, typecheck, build)
  ├─ Gate B: review-work → 5-agent parallel review
  └─ Gate C: Cubic      → cubic-dev-ai[bot] "No issues found"
Phase 4: Merge         → Squash merge, worktree cleanup
```

---

## Phase 0: Setup

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
BASE_BRANCH="dev"

# Create branch
BRANCH_NAME="feature/$(echo "$TASK_SUMMARY" | tr '[:upper:] ' '[:lower:]-')"
git fetch origin "$BASE_BRANCH"
git branch "$BRANCH_NAME" "origin/$BASE_BRANCH"

# Create worktree (sibling to repo)
WORKTREE_PATH="../${REPO_NAME}-wt/${BRANCH_NAME}"
mkdir -p "$(dirname "$WORKTREE_PATH")"
git worktree add "$WORKTREE_PATH" "$BRANCH_NAME"

cd "$WORKTREE_PATH"
[ -f "bun.lock" ] && bun install
```

---

## Phase 1: Implement

- **Scope discipline**: For bugs, stay minimal. Fix + test only.
- **Commit strategy**: Use git-master skill's atomic commits.
  - 3+ files → 2+ commits minimum
  - 5+ files → 3+ commits minimum
  - 10+ files → 5+ commits minimum
- Each commit pairs implementation with tests.

**Pre-push validation:**
```bash
bun run typecheck
bun test
bun run build
```

---

## Phase 2: PR Creation

```bash
git push -u origin "$BRANCH_NAME"

gh pr create \
  --base "$BASE_BRANCH" \
  --head "$BRANCH_NAME" \
  --title "$PR_TITLE" \
  --body "## Summary
[1-3 sentences]

## Changes
[Bullet list]

## Testing
- typecheck ✅
- test ✅
- build ✅"

PR_NUMBER=$(gh pr view --json number -q .number)
```

---

## Phase 3: Verification Loop

```
while true:
  1. Wait for CI          → Gate A
  2. If CI fails          → fix, commit, push, continue
  3. Run review-work      → Gate B
  4. If review fails      → fix blocking issues, commit, push, continue
  5. Check Cubic          → Gate C
  6. If Cubic has issues  → fix, commit, push, continue
  7. All three pass       → break
```

### Gate A: CI Checks
```bash
gh pr checks "$PR_NUMBER" --watch --fail-fast

# On failure:
RUN_ID=$(gh run list --branch "$BRANCH_NAME" --status failure --json databaseId --jq '.[0].databaseId')
gh run view "$RUN_ID" --log-failed
```

### Gate B: review-work
Launch 5 parallel sub-agents. All 5 must pass. On failure: fix blocking issues, commit, push, re-enter from Gate A.

### Gate C: Cubic Approval
```bash
# Approval signal: "**No issues found**" + confidence "**5/5**"
CUBIC_REVIEW=$(gh api "repos/${REPO}/pulls/${PR_NUMBER}/reviews" \
  --jq '[.[] | select(.user.login == "cubic-dev-ai[bot]")] | last | .body')

if echo "$CUBIC_REVIEW" | grep -q "No issues found"; then
  echo "Cubic: APPROVED"
else
  echo "Cubic: ISSUES FOUND"
fi
```

### Iteration discipline
1. Fix ONLY issues from failing gate
2. Commit atomically
3. Push
4. Re-enter from Gate A (code changed → full re-verification)

No "improvement" scope creep during fix iterations.

---

## Phase 4: Merge & Cleanup

```bash
# Squash merge
gh pr merge "$PR_NUMBER" --squash --delete-branch

# Sync .sisyphus state back
if [ -d "$WORKTREE_PATH/.sisyphus" ]; then
  mkdir -p "$ORIGINAL_DIR/.sisyphus"
  cp -r "$WORKTREE_PATH/.sisyphus/"* "$ORIGINAL_DIR/.sisyphus/" 2>/dev/null || true
fi

# Cleanup worktree
cd "$ORIGINAL_DIR"
git worktree remove "$WORKTREE_PATH"
git worktree prune
```

### Report
```
## PR Merged ✅
- **PR**: #{PR_NUMBER} — {PR_TITLE}
- **Branch**: {BRANCH_NAME} → {BASE_BRANCH}
- **Iterations**: {N} verification loops
- **Gates passed**: CI ✅ | review-work ✅ | Cubic ✅
- **Worktree**: cleaned up
```

