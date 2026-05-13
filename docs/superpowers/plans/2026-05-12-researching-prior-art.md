# Researching Prior Art Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a tool-agnostic, pure-function `researching-prior-art` skill and wire it into `brainstorming` as a prior-art survey step.

**Architecture:** One new skill file (`skills/researching-prior-art/SKILL.md`) that produces an Adopt/Extend/Compose/Build matrix and returns to caller without chaining. One existing skill file (`skills/brainstorming/SKILL.md`) modified in four spots to invoke the new skill between clarifying questions and proposing approaches.

**Tech Stack:** Markdown skill files. No code, no tests beyond hand-run smoke checks.

**Spec:** `docs/superpowers/specs/2026-05-12-researching-prior-art-design.md`

---

## File Structure

- **Create:** `skills/researching-prior-art/SKILL.md` — pure-function survey skill (frontmatter + 8 body sections)
- **Modify:** `skills/brainstorming/SKILL.md` — four targeted edits (checklist, DOT graph, exploring-approaches subsection, key-principles list)

No code files. No supporting docs (smoke-check results captured in commit messages per spec success criterion 14).

---

## Task 1: Create `skills/researching-prior-art/SKILL.md`

**Files:**
- Create: `skills/researching-prior-art/SKILL.md`

- [ ] **Step 1: Verify target directory does not yet exist**

Run: `ls skills/researching-prior-art 2>&1`
Expected: `ls: cannot access 'skills/researching-prior-art': No such file or directory`

If directory exists with content, stop and ask the user how to proceed.

- [ ] **Step 2: Write the full skill file**

Write to `skills/researching-prior-art/SKILL.md`:

````markdown
---
name: researching-prior-art
description: Use when starting a new feature, choosing a library, or before committing to a design. Triggers on "How do I [implement/build/create/add] X", "What's the best way to [solve/approach/handle] X", "How should I structure/organize/design X", "Which library/tool/framework should I use for X". Also use when an upstream skill (brainstorming, writing-plans) requests prior-art survey.
---

# Researching Prior Art

## Overview

Pure function. Input: a topic or problem. Output: a survey report with an Adopt/Extend/Compose/Build decision matrix. Return the report to the caller and stop — do not chain to another skill, do not begin implementation, do not write code. The caller decides what happens next.

If invoked standalone on an unscoped question (no concrete topic noun, no clear problem statement), ask one scoping question first before enumerating channels. If invoked while a brainstorming session is active and clarifying questions are not yet complete, defer — return `status:skipped:premature` and let brainstorming call you at the prior-art step.

## Impact Gate (When to Skip)

Two layers.

### Always-skip override (mechanical, no judgment)

Return `status:skipped:no-impact` immediately when:
- Pure refactor inside a single file
- Rename / formatting / lint fix / comment edit
- Version bump or dependency upgrade with no API change
- One-line bug fix in already-chosen library code

### Judgment skip

Return `status:skipped:no-impact` when:
- The question involves no third-party tool, library, or pattern choice
- The answer would not change the design
- The caller explicitly marks the request trivial

Do **not** skip on bare ambiguity ("add an endpoint", "add a CLI flag"). Apply the impact gate to the specifics: framework already chosen + mechanical addition → skip; fresh integration or library swap → run.

## Available Channels

Claude has no runtime "list my tools" API; the tool list is in the system prompt and read once. At start, scan the in-context tool list and build the `channels` list:

1. Scan the tool list visible in your system prompt for tool names or descriptions containing `search`, `fetch`, `research`, `web` (any MCP, plugin, or platform tool).
2. Note built-in `WebSearch` / `WebFetch` if listed as available in this session.
3. Local indexes are always available: `rg` over the current repo, package manifests, `~/.claude/settings.json`, `~/.claude/skills/`, `~/.codex/skills/`, plugin skill directories.

Report the enumerated list verbatim in the output's `Channels used` field. List unavailable channels in `Channels skipped` with one-line reasons. Never fabricate findings when a channel is blocked — return `status:blocked:<reason>` and let the caller decide.

## Local-Before-External

Exhaust local channels (channel group 3) before touching external ones (channel groups 1, 2). If local sources answer the question, return without external search.

## External Survey (External Channels Only)

Local channels do not take "anti-pattern" or "license" queries — this section applies only to external sources.

### Enforceable caps (count-based; Claude has no wall clock between calls)

- Max **5 distinct external queries**
- Max **15 sources reviewed** in total
- **Stop early** after two consecutive queries return zero new candidates

For human reviewers of agent logs, ~15 minutes wall-clock is a reasonable rough budget — not operationally enforceable inside the skill.

### Query strategy

The 5 queries should cover:
- Exact-match search
- Related-problem search
- Anti-pattern search ("X considered harmful")
- Comparison search ("X vs Y")
- License / maintenance search

### Source scan filters

Quality signals: active maintenance, recent commits, license clarity, ecosystem fit. Discard stale/abandoned/unlicensed candidates.

### Datapoint floor

At least **2** independent datapoints per recommendation. Below floor → mark recommendation as `low-confidence`.

## Output: Adopt / Extend / Compose / Build Matrix

Primary deliverable. The 4-tuple is borrowed from `affaan-m/everything-claude-code/skills/search-first/SKILL.md` (synthesized by that author; not a canonical analyst framework).

| Decision | Meaning | Components |
|----------|---------|------------|
| **Adopt** | Exact-match library/tool; use as-is | Single source URL |
| **Extend** | Close match; fork or wrap | Single source URL + wrap location |
| **Compose** | Combine multiple existing pieces | List of 2+ source URLs |
| **Build** | Nothing suitable exists | N/A; justify in Cons/Evidence; mark `low-confidence` if absence-of-evidence |

### Output shape

```markdown
## Prior-Art Survey: <topic>

**Status:** done | skipped:no-impact | skipped:premature | blocked:<reason> | no-prior-art-found
**Channels used:** <list> | **Channels skipped:** <list with reason>
**Queries run:** <count> / 5 max  |  **Sources reviewed:** <count> / 15 max
**Datapoints:** <count>

### Local findings
- ...  (or "none — local channels checked: <list>")

### External findings
- ...  (or "none — external channels checked: <list>")

### Decision Matrix
| Option | Decision | Components | Source | License | Pros | Cons | Evidence |
|--------|----------|------------|--------|---------|------|------|----------|
| ...    | Adopt    | ...        | ...    | MIT     | ...  | ...  | ...      |

### Recommendation
<one sentence; "low-confidence" tag if datapoints below floor or status = no-prior-art-found>

### Open questions
- ...
```

### Status values

- `done` — survey ran, matrix populated.
- `skipped:no-impact` — impact gate fired; no research performed.
- `skipped:premature` — invoked while a brainstorming session is mid-clarification; deferred.
- `blocked:<reason>` — channels unreachable; caller decides whether to retry or proceed without prior-art context.
- `no-prior-art-found` — channels worked, returned no usable candidates; matrix shows only `Build` row with low-confidence tag.

### Matrix omission

For non-`done` statuses (`skipped:*`, `blocked:*`, `no-prior-art-found` with no candidates), omit the Decision Matrix section entirely (replace with one-line "n/a — see status"). Local/External findings sections also omit if the corresponding channel set was not used.

## Anti-Patterns

- Rabbit-holing past the query/source caps.
- Fabricating citations when a channel is blocked — escalate instead.
- Silently skipping unavailable channels — list them in the output.
- Endorsing stale, low-star, or unlicensed repos without flagging the risk.
- "Apply X to Y" recommendations without genuine novelty or evidence of fit.
- Naming a specific research tool in the recommendation as if the caller must use it — stay tool-agnostic.

## Pure-Function Contract

<HARD-GATE>
Return the report and stop. Do NOT invoke writing-plans, brainstorming, executing-plans, subagent-driven-development, or any implementation skill. Do NOT begin coding. Do NOT create files outside the report. Do NOT call yourself recursively. The caller decides what happens next.

**Permitted:** Asking the caller a single scoping question before enumerating channels (per Overview) is permitted dialogue — it is not a "next-skill" invocation.
</HARD-GATE>
````

- [ ] **Step 3: Verify the file written matches the source-of-truth structure**

Run: `wc -l skills/researching-prior-art/SKILL.md && head -5 skills/researching-prior-art/SKILL.md`

Expected: ~125 lines; first 5 lines show the `---` frontmatter opener, `name: researching-prior-art`, the `description:` line, the closing `---`, and a blank line before `# Researching Prior Art`.

- [ ] **Step 4: Verify description is under 1024 chars**

Run: `awk '/^description:/,/^[a-zA-Z-]+:|^---$/' skills/researching-prior-art/SKILL.md | head -1 | wc -c`

Expected: a number under 1024 (target ~510).

- [ ] **Step 5: Verify no specific research tool is named in the body**

Run: `grep -niE 'research-agent|websearch|webfetch|\bexa\b|tavily|context7' skills/researching-prior-art/SKILL.md || echo "clean"`

Expected output: `clean`.

(If matches appear inside an "anti-patterns" example showing what NOT to do, that's acceptable — re-read context and decide. The body must not instruct using any specific tool.)

- [ ] **Step 6: Commit**

```bash
git add skills/researching-prior-art/SKILL.md
git commit -m "feat(skills): add researching-prior-art pure-function skill

Tool-agnostic prior-art survey. Produces Adopt/Extend/Compose/Build matrix
with citations and returns to caller without chaining. Triggered by
brainstorming step 4 (next task) or by the trigger phrases in the
frontmatter description.

Spec: docs/superpowers/specs/2026-05-12-researching-prior-art-design.md"
```

---

## Task 2: Brainstorming Checklist — Insert New Item, Renumber

**Files:**
- Modify: `skills/brainstorming/SKILL.md` (lines 22-32, the numbered checklist)

- [ ] **Step 1: Confirm current checklist state**

Run: `sed -n '22,32p' skills/brainstorming/SKILL.md`

Expected: 10 lines showing items 1-9 numbered exactly as quoted below:

```
You MUST create a task for each of these items and complete them in order:

1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

If the file differs from this expected content, stop and ask the user. The renumbering below assumes this exact starting point.

- [ ] **Step 2: Apply the Edit — replace the entire checklist block**

Edit tool, replacing this old_string:

```
1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Propose 2-3 approaches** — with trade-offs and your recommendation
5. **Present design** — in sections scaled to their complexity, get user approval after each section
6. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
7. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
8. **User reviews written spec** — ask user to review the spec file before proceeding
9. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

With this new_string:

```
1. **Explore project context** — check files, docs, recent commits
2. **Offer visual companion** (if topic will involve visual questions) — this is its own message, not combined with a clarifying question. See the Visual Companion section below.
3. **Ask clarifying questions** — one at a time, understand purpose/constraints/success criteria
4. **Survey prior art** — invoke `superpowers:researching-prior-art` for non-trivial features (impact gate decides). Use the returned matrix to seed the 2-3 approaches in step 5. If a prior-art matrix has already been produced this session for the same scope, reuse it instead of re-invoking — no second survey.
5. **Propose 2-3 approaches** — with trade-offs and your recommendation
6. **Present design** — in sections scaled to their complexity, get user approval after each section
7. **Write design doc** — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit
8. **Spec self-review** — quick inline check for placeholders, contradictions, ambiguity, scope (see below)
9. **User reviews written spec** — ask user to review the spec file before proceeding
10. **Transition to implementation** — invoke writing-plans skill to create implementation plan
```

- [ ] **Step 3: Verify renumbering**

Run: `sed -n '22,33p' skills/brainstorming/SKILL.md`

Expected: items 1-10, with new item 4 ("Survey prior art") inserted. The list now has 10 numbered entries instead of 9.

- [ ] **Step 4: Verify no other line in the file still says "step 4" referring to the old "Propose approaches" position**

Run: `grep -n 'step 4\|item 4' skills/brainstorming/SKILL.md || echo "clean"`

Expected: `clean`, or hits that are clearly inside the new item 4 itself (the reference "in step 5" is the only intentional one — that points to the new "Propose approaches" position, so it's correct).

- [ ] **Step 5: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): insert prior-art survey as step 4

New step 4 invokes superpowers:researching-prior-art for non-trivial
features. Existing items 4-9 renumber to 5-10. Reuse clause prevents
double-surveying when prior-art has already fired this session.

Spec: docs/superpowers/specs/2026-05-12-researching-prior-art-design.md"
```

---

## Task 3: Brainstorming DOT Graph — Add Node, Replace Edge

**Files:**
- Modify: `skills/brainstorming/SKILL.md` (the DOT graph block, currently lines 36-64)

- [ ] **Step 1: Confirm DOT graph current state**

Run: `sed -n '36,64p' skills/brainstorming/SKILL.md`

Expected: the `dot` code block containing nodes and edges. The relevant lines are the node declaration `"Ask clarifying questions" [shape=box];` and the edge `"Ask clarifying questions" -> "Propose 2-3 approaches";`.

- [ ] **Step 2: Add new node declaration**

Edit tool, replacing this old_string:

```
    "Ask clarifying questions" [shape=box];
    "Propose 2-3 approaches" [shape=box];
```

With this new_string:

```
    "Ask clarifying questions" [shape=box];
    "Survey prior art" [shape=box];
    "Propose 2-3 approaches" [shape=box];
```

- [ ] **Step 3: Replace the edge**

Edit tool, replacing this old_string:

```
    "Ask clarifying questions" -> "Propose 2-3 approaches";
```

With this new_string:

```
    "Ask clarifying questions" -> "Survey prior art";
    "Survey prior art" -> "Propose 2-3 approaches";
```

- [ ] **Step 4: Verify the DOT graph is well-formed**

Run: `sed -n '36,66p' skills/brainstorming/SKILL.md | grep -c 'Survey prior art'`

Expected: `3` (one node declaration, two edges referring to the new node).

- [ ] **Step 5: Verify no orphan edge remains**

Run: `grep -F '"Ask clarifying questions" -> "Propose 2-3 approaches"' skills/brainstorming/SKILL.md || echo "removed"`

Expected: `removed`.

- [ ] **Step 6: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): update DOT graph for prior-art step

Add 'Survey prior art' node. Replace the direct edge from clarifying
questions to approaches with two edges routing through the new node.

Spec: docs/superpowers/specs/2026-05-12-researching-prior-art-design.md"
```

---

## Task 4: Brainstorming Body — Exploring-Approaches Bullet + Key-Principles Bullet

**Files:**
- Modify: `skills/brainstorming/SKILL.md` (Exploring approaches subsection currently lines 95-99; Key Principles list currently lines 153-165)

- [ ] **Step 1: Confirm Exploring approaches current state**

Run: `sed -n '95,99p' skills/brainstorming/SKILL.md`

Expected:

```
**Exploring approaches:**

- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why
```

- [ ] **Step 2: Prepend the prior-art bullet**

Edit tool, replacing this old_string:

```
**Exploring approaches:**

- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why
```

With this new_string:

```
**Exploring approaches:**

- For non-trivial features, the prior-art survey from the previous step has already produced an Adopt/Extend/Compose/Build matrix. Use it as the starting set of candidate approaches before adding alternatives from your own knowledge.
- Propose 2-3 different approaches with trade-offs
- Present options conversationally with your recommendation and reasoning
- Lead with your recommended option and explain why
```

- [ ] **Step 3: Confirm Key Principles current state**

Run: `sed -n '153,165p' skills/brainstorming/SKILL.md`

Expected: the bulleted Key Principles list. Locate the bullet starting with `- **Explore alternatives**`.

- [ ] **Step 4: Insert the prior-art bullet after "Explore alternatives"**

Edit tool, replacing this old_string:

```
- **Explore alternatives** - Always propose 2-3 approaches before settling.
- **Incremental validation** - Present design, get approval before moving on.
```

With this new_string:

```
- **Explore alternatives** - Always propose 2-3 approaches before settling.
- **Survey prior art before approaches** - For non-trivial features, the `researching-prior-art` skill produces a candidate matrix. Use it to seed approaches rather than starting from scratch.
- **Incremental validation** - Present design, get approval before moving on.
```

- [ ] **Step 5: Verify both edits landed**

Run: `grep -c "prior art" skills/brainstorming/SKILL.md`

Expected: at least `4` (checklist item 4, two edges in DOT graph that reference "Survey prior art", exploring-approaches bullet, key-principles bullet — actually 5+; if fewer than 4, an edit was lost).

- [ ] **Step 6: Commit**

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat(brainstorming): wire prior-art matrix into approaches + principles

Exploring-approaches bullet prepended: matrix seeds the candidate set
before adding alternatives from training data.
Key-principles bullet added under 'Explore alternatives'.

Spec: docs/superpowers/specs/2026-05-12-researching-prior-art-design.md"
```

---

## Task 5: Smoke Checks (RED / GREEN / Adversarial)

These are hand-run checks. The agent executing this plan runs the brainstorming flow with the user on the fixture below and records the result. No code changes; commit captures the outcome.

**Files:**
- No files modified
- Outcome recorded in commit message at end of task

- [ ] **Step 1: Note RED-baseline expectation**

Fixture: *"Add a rate limiter to a Node Express API."*

Without the new skill installed, brainstorming would:
- Move from clarifying questions directly to proposing approaches
- Propose approaches drawn from training data (likely mentions express-rate-limit, possibly bottleneck, possibly hand-rolled middleware)
- Produce no Adopt/Extend/Compose/Build decision matrix
- Produce no citations

Record this expectation in the smoke-check commit message — no need to actually run the RED case since the skill is brand-new (RED state is empirically: how brainstorming behaved before today's commits).

- [ ] **Step 2: Run GREEN smoke check**

Open a new conversation in this repo. Tell the agent:

> "Brainstorm a small feature: I want to add a rate limiter to a Node Express API."

Expected behavior:
- Brainstorming fires.
- After clarifying questions, brainstorming invokes `superpowers:researching-prior-art`.
- The prior-art skill enumerates channels, surveys, and returns a Decision Matrix with at least these candidates: `express-rate-limit` (Adopt), `rate-limiter-flexible` (Adopt or Extend), `bottleneck` (Compose or Build).
- Each row has a Source URL, License, Pros, Cons, Evidence.
- Top recommendation is one sentence with a citation.
- Brainstorming then proposes 2-3 approaches using the matrix as the seed.

Record in a scratch note: did each expected behavior happen?

- [ ] **Step 3: Run adversarial smoke check**

Open a new conversation. Tell the agent:

> "I want to fix a one-line typo in our existing rate-limiter config — change `windowMs: 60000` to `windowMs: 90000`. Brainstorm this."

Expected behavior:
- Brainstorming fires (or refuses on triviality grounds).
- If brainstorming reaches step 4, the prior-art skill is invoked.
- Prior-art's impact gate fires the always-skip override (one-line bug fix in already-chosen library code → `status:skipped:no-impact`).
- No external queries are run.
- Brainstorming proceeds without a matrix.

Record outcome in scratch note.

- [ ] **Step 4: Commit smoke-check results**

```bash
git commit --allow-empty -m "test: smoke-check researching-prior-art (RED/GREEN/adversarial)

Fixture: 'Add a rate limiter to a Node Express API.'

RED (baseline, pre-skill state — historical, not re-run):
- Brainstorming proposed approaches from training data
- No Adopt/Extend/Compose/Build matrix
- No citations

GREEN (with skill installed):
- <fill in: did prior-art fire after clarifying questions? did matrix appear?>
- <fill in: which candidates appeared? which decision labels?>
- <fill in: were citations present?>

Adversarial (one-line bug fix in already-chosen library code):
- <fill in: did the always-skip override fire? status:skipped:no-impact?>
- <fill in: were any external queries run? (expected: zero)>

Spec: docs/superpowers/specs/2026-05-12-researching-prior-art-design.md"
```

Replace the `<fill in: ...>` placeholders with actual observed behavior before committing. If any expected behavior failed, do NOT commit — return to the spec, identify the gap, and propose a fix.

---

## Self-Review (run before declaring complete)

After Tasks 1-5 are done, check:

**Spec coverage:**

| Spec success criterion | Implementing task |
|------------------------|-------------------|
| 1. SKILL.md with HARD-GATE | Task 1 step 2 |
| 2. Description ≤1024 chars, when-to-use only | Task 1 steps 2, 4 |
| 3. Checklist renumber 4→5..9→10 | Task 2 step 2 |
| 4. DOT graph node + edge replacement | Task 3 steps 2, 3 |
| 5. Exploring-approaches bullet | Task 4 step 2 |
| 6. Key-principles bullet | Task 4 step 4 |
| 7. Reuse clause in checklist item 4 | Task 2 step 2 (clause is inside the new item) |
| 8. Standalone scoped invocation returns matrix | Task 5 GREEN |
| 9. Standalone unscoped → scoping question | Skill Overview section; verify in any future standalone use |
| 10. Mid-clarification → `skipped:premature` | Skill Overview section; verify in any future use |
| 11. Rate-limiter fixture GREEN | Task 5 step 2 |
| 12. One-line fix → `skipped:no-impact` | Task 5 step 3 |
| 13. No specific research tool named | Task 1 step 5 |
| 14. RED/GREEN/adversarial documented | Task 5 step 4 |

If a criterion has no implementing task, add one.

**Placeholder scan:**

Run: `grep -nE 'TBD|TODO|fill in|implement later|add appropriate' docs/superpowers/plans/2026-05-12-researching-prior-art.md`

Expected matches: only the `<fill in: ...>` markers inside Task 5's commit message template (those are intentional — they get replaced with actual observations).

**Type consistency:**

The skill's status string format is `status:detail` (single colon, no parens) throughout: `status:skipped:no-impact`, `status:skipped:premature`, `status:blocked:<reason>`, `status:done`, `status:no-prior-art-found`. The plan and the skill body match. ✓

---

## Execution Mode Decision

**Use `superpowers:executing-plans`** (not subagent-driven-development) because:
- Tasks are tiny markdown edits — subagent spawn overhead exceeds the work
- Task 5 needs human verification at each smoke check
- Single skill file + four targeted edits — no parallelism benefit
