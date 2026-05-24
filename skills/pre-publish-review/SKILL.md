---
name: pre-publish-review
description: "16-agent pre-publish release gate"
---

# pre-publish-review

**Description:** Nuclear-grade 16-agent pre-publish release gate. Runs /get-unpublished-changes to detect all changes since last npm release, spawns up to 10 ultrabrain agents for deep per-change analysis, invokes /review-work (5 agents) for holistic review, and 1 oracle for overall release synthesis. Use before EVERY npm publish. Triggers: 'pre-publish review', 'review before publish', 'release review', 'ready to publish?'.

---

# Pre-Publish Review — 16-Agent Release Gate

Three-layer review before publishing to npm:

| Layer | Agents | Type | What They Check |
|-------|--------|------|-----------------|
| Per-Change Deep Dive | up to 10 | ultrabrain | Each logical change group individually |
| Holistic Review | 5 | review-work | Goal compliance, QA, code quality, security, context |
| Release Synthesis | 1 | oracle | Overall readiness, version bump, breaking changes, risk |

---

## Phase 0: Detect Unpublished Changes

Run `/get-unpublished-changes` FIRST.

```bash
PUBLISHED=$(npm view oh-my-opencode version 2>/dev/null || echo "not published")
LOCAL=$(node -p "require('./package.json').version")
COMMITS=$(git log "v${PUBLISHED}"..HEAD --oneline)
DIFF_STAT=$(git diff "v${PUBLISHED}"..HEAD --stat)
CHANGED_FILES=$(git diff --name-only "v${PUBLISHED}"..HEAD)
```

---

## Phase 1: Parse Changes into Groups

1. Start from `/get-unpublished-changes` analysis (already grouped by feat/fix/refactor/docs)
2. Split by module/area — same module changes belong together
3. Target up to 10 groups
4. For each group: name, commits, files, diff

---

## Phase 2: Spawn All Agents

Launch ALL agents in a single turn (`run_in_background=true`).

### Layer 1: Ultrabrain Per-Change Analysis (up to 10)

Each ultrabrain reviews ONE change group. Checklist:
1. Intent clarity — what is this trying to do?
2. Correctness — trace logic for 3+ scenarios
3. Breaking changes — public API, config, CLI, hooks?
4. Pattern adherence — follows codebase conventions?
5. Edge cases — empty arrays, undefined, concurrency?
6. Error handling — no swallowed errors?
7. Type safety — no `as any`, `@ts-ignore`?
8. Test coverage — meaningful tests?
9. Side effects — who depends on what changed?
10. Release risk — SAFE / CAUTION / RISKY

Output:
```
<verdict>PASS or FAIL</verdict>
<risk>SAFE / CAUTION / RISKY</risk>
<has_breaking_changes>YES or NO</has_breaking_changes>
<blocking_issues>MUST fix before publish</blocking_issues>
```

### Layer 2: Holistic Review via /review-work (5 agents)

Spawn review-work with full changeset. Internally launches 5 parallel agents.

### Layer 3: Oracle Release Synthesis (1 agent)

Full picture review — all commits, full diff. Checklist:
1. Release coherence — coherent story or grab-bag?
2. Version bump — PATCH/MINOR/MAJOR with justification
3. Breaking changes audit — exhaustive list
4. Migration requirements — what steps do users need?
5. Dependency changes — new deps, supply chain risk?
6. Changelog draft — feat/fix/refactor/breaking/docs
7. Deployment risk — SAFE/CAUTION/RISKY/BLOCK
8. Post-publish monitoring recommendations

---

## Phase 3: Collect Results

Track all agents:

| # | Agent | Type | Status | Verdict |
|---|-------|------|--------|---------|
| 1-10 | Ultrabrain per group | ultrabrain | pending | — |
| 11 | Review-Work Coordinator | unspecified-high | pending | — |
| 12 | Release Synthesis Oracle | oracle | pending | — |

---

## Phase 4: Final Verdict

**BLOCK** if: Oracle verdict is BLOCK, any ultrabrain CRITICAL, review-work failed any MAIN agent
**RISKY** if: Oracle is RISKY, multiple ultrabrains CAUTION/FAIL
**CAUTION** if: Oracle is CAUTION, minor issues flagged
**SAFE** if: Oracle is SAFE, all ultrabrains passed, review-work passed

### Final Report Format
```markdown
# Pre-Publish Review
## Release: v{PUBLISHED} -> v{LOCAL}
**Commits:** {N} | **Files:** {M} | **Agents:** {12}

## Overall Verdict: SAFE / CAUTION / RISKY / BLOCK
## Recommended Version Bump: PATCH / MINOR / MAJOR

## Per-Change Analysis
| # | Group | Verdict | Risk | Breaking? | Blocking |
|---|-------|---------|------|-----------|----------|

## Breaking Changes
[Exhaustive list with migration steps]

## Changelog Draft
[Ready-to-use changelog]
```

