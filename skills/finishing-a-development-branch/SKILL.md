---
name: finishing-a-development-branch
description: Use when implementation is complete, all tests pass, and the branch needs integration, publication, preservation, or cleanup
---

# Finishing a Development Branch

## Overview

Guide completion of development work by following established delivery intent
and repository policy, asking only when a material choice remains.

**Core principle:** Verify tests → Determine the authorized outcome → Execute or
ask one focused question → Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with merge/PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

### Step 3: Determine the Delivery Outcome

Resolve the outcome from the strongest available signal, in this order:

1. The user's explicit request from any point in the conversation
2. Repository instructions or an established delivery workflow
3. The conventional safe outcome for the repository

When the outcome is established and authorized, proceed directly. Do not ask
the user to choose it again.

When policy requires a specific approval, ask only for that approval. Example:
`Open the PR?`

When multiple outcomes remain plausible and materially different, ask one
focused question with only the relevant choices and put the recommended choice
first. Do not show a generic menu.

**Never offer discard as a routine completion option.** Abandon work only when
the user has expressed intent to discard, abandon, supersede, or clean it up;
then use the confirmation in Step 4.

### Step 4: Execute Choice

#### Option 1: Merge Locally

```bash
# Switch to base branch
git checkout <base-branch>

# Pull latest
git pull

# Merge feature branch
git merge <feature-branch>

# Verify tests on merged result
<test command>

# If tests pass
git branch -d <feature-branch>
```

Then: Cleanup worktree (Step 5)

#### Option 2: Push and Create PR

```bash
# Push branch
git push -u origin <feature-branch>

# Create PR
gh pr create --title "<title>" --body "$(cat <<'EOF'
## Summary
<2-3 bullets of what changed>

## Test Plan
- [ ] <verification steps>
EOF
)"
```

Then: Keep the worktree. Review feedback may require more commits.

#### Option 3: Keep As-Is

Report: "Keeping branch <name>. Worktree preserved at <path>."

**Don't cleanup worktree.**

#### Option 4: Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation.

If confirmed:
```bash
git checkout <base-branch>
git branch -D <feature-branch>
```

Then: Cleanup worktree (Step 5)

### Step 5: Cleanup Worktree

**For Options 1 and 4:**

Check if in worktree:
```bash
git worktree list | grep $(git branch --show-current)
```

If yes:
```bash
git worktree remove <worktree-path>
```

**For Option 3:** Keep worktree.

## Quick Reference

| Option | Merge | Push | Keep Worktree | Cleanup Branch |
|--------|-------|------|---------------|----------------|
| 1. Merge locally | ✓ | - | - | ✓ |
| 2. Create PR | - | ✓ | ✓ | - |
| 3. Keep as-is | - | - | ✓ | - |
| 4. Discard | - | - | - | ✓ (force) |

## Common Mistakes

**Skipping test verification**
- **Problem:** Merge broken code, create failing PR
- **Fix:** Always verify tests before offering options

**Ignoring established delivery intent**
- **Problem:** Asking the user to choose an outcome they already requested or
  repository policy already determines
- **Fix:** Infer the authorized outcome before asking anything

**Generic completion menus**
- **Problem:** Irrelevant choices create friction and make destructive cleanup
  look routine
- **Fix:** Ask one focused question only when a material choice remains; never
  offer discard without abandonment intent

**Automatic worktree cleanup**
- **Problem:** Remove worktree when might need it (Option 2, 3)
- **Fix:** Only cleanup for Options 1 and 4

**No confirmation for discard**
- **Problem:** Accidentally delete work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing tests
- Merge without verifying tests on result
- Ask the user to repeat an established delivery choice
- Offer discard without evidence the user wants to abandon the work
- Delete work without confirmation
- Force-push without explicit request

**Always:**
- Verify tests before completing delivery
- Follow explicit user intent and repository delivery policy
- Ask only the focused question needed to resolve a genuine ambiguity or
  approval gate
- Get typed confirmation before discarding work
- Clean up worktree for Options 1 & 4 only

## Integration

**Called by:**
- **subagent-driven-development** (Step 7) - After all tasks complete
- **executing-plans** (Step 5) - After all batches complete

**Pairs with:**
- **using-git-worktrees** - Cleans up worktree created by that skill
