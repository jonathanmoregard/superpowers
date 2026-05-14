# Researching Prior Art: Pure-Function Survey Skill

**Date:** 2026-05-12
**Status:** Approved (post round-2 advisor)
**Scope:** `skills/researching-prior-art/SKILL.md` (new), `skills/brainstorming/SKILL.md` (modify)

## Problem

Brainstorming currently moves from clarifying questions straight to proposing 2-3 approaches. The approaches are drawn from Claude's training data and the current conversation — there is no step that surveys the live landscape for existing tools, libraries, plugins, or SOTA patterns that already solve the problem. As a result, designs reinvent wheels, miss readymade fork targets, and overlook common-problem patterns that the ecosystem has already converged on.

Adjacent skill collections (`affaan-m/everything-claude-code/search-first`, `dwmkerr/claude-toolkit/research`, `acking-you/myclaude-skills/research`, `TrevorS/dot-claude/github-prior-art`, `wanshuiyin/.../idea-creator`) have each codified a prior-art research step. Superpowers does not.

This fork adds one — as a tool-agnostic, pure-function skill that brainstorming can call and that other skills (e.g. `writing-plans`) can reuse later.

## Goals

- Survey existing tools / libraries / plugins / SOTA approaches before brainstorming commits to a design.
- Keep the skill tool-agnostic: it states **what** to research and **how to evaluate**, never **which tool** to use. Tool selection is Claude's call at invocation time.
- Pure function: skill returns findings, does not chain to a next skill. Caller owns continuation.
- Cheap to skip when the answer would not change the design (impact gate).
- Reusable from brainstorming, writing-plans, or manual invocation.
- Conform to `superpowers:writing-skills` conventions (description = when-to-use only; HARD-GATE wraps non-negotiable contracts; iron law on testing).

## Non-Goals

- Mandatory invocation on every conversation. Impact gate exists for low-leverage cases.
- Specifying a particular research backend. The skill must work whether Claude has WebSearch, an MCP research agent, fetch tooling, or a local-only environment.
- Replacing brainstorming's design phase. Research informs approaches; it does not produce the design.
- Upstream PR to `obra/superpowers`. This is a fork-only customization. (Upstream's CLAUDE.md rejects fork-specific changes, and a behavior-shaping skill addition would require eval evidence beyond the scope of this work.)

## Design

### File Layout

```
skills/
  researching-prior-art/
    SKILL.md              # new — pure-function skill
  brainstorming/
    SKILL.md              # modified — insert one checklist item + DOT node + body lines
```

### New Skill: `researching-prior-art/SKILL.md`

#### Frontmatter

Per `superpowers:writing-skills` (lines 99-172): `description` must describe **when to use**, never summarize workflow. Trigger phrases ported verbatim from `TrevorS/dot-claude/github-prior-art`:

```yaml
---
name: researching-prior-art
description: Use when starting a new feature, choosing a library, or before committing to a design. Triggers on "How do I [implement/build/create/add] X", "What's the best way to [solve/approach/handle] X", "How should I structure/organize/design X", "Which library/tool/framework should I use for X". Also use when an upstream skill (brainstorming, writing-plans) requests prior-art survey.
---
```

(~510 chars, under writing-skills' 1024-char cap; trimmed two redundant trigger families — "What are people using for X" and "How might we [implement/architect] X" — that fully overlap the kept ones.)

No workflow summary. No mention of matrix shape. (That belongs in the body.)

#### Body Sections

1. **Overview** — one paragraph: pure function, returns findings, does not chain. Caller owns continuation.

2. **Impact gate (when to skip)** — borrowed from brainstorming's existing impact-gate pattern (line 85-89). Two layers:

   **Always-skip override (mechanical, no judgment required) — return `status: skipped:no-impact`:**
   - Pure refactor inside a single file
   - Rename / formatting / lint fix / comment edit
   - Version bump or dependency upgrade with no API change
   - One-line bug fix in already-chosen library code

   **Judgment skip — return `status: skipped:no-impact` when:**
   - The question involves no third-party tool, library, or pattern choice.
   - The answer would not change the design.
   - The caller explicitly marks the request trivial.

   Do **not** skip on bare ambiguity ("add an endpoint", "add a CLI flag") — apply the impact gate to the specifics: if the framework/library is already chosen and the addition is mechanical, skip; if it's a fresh integration or a library swap, run.

3. **Available channels — concrete enumeration procedure.** Claude has no runtime "list my tools" API; the tool list is in the system prompt and read once. At start, scan the in-context tool list and build the `channels` list:
   1. Scan the tool list visible in your system prompt for tool names or descriptions containing `search`, `fetch`, `research`, `web` (any MCP, plugin, or platform tool).
   2. Note built-in `WebSearch` / `WebFetch` if listed as available in this session.
   3. Local indexes always available: `rg` over the current repo, package manifests, `~/.claude/settings.json`, `~/.claude/skills/`, `~/.codex/skills/`, plugin skill directories.

   Report the enumerated list verbatim in the output's `Channels used` field. List unavailable channels in `Channels skipped` with one-line reason. Never fabricate findings when a channel is blocked — return `status: blocked` with the reason and let the caller decide.

4. **Local-before-external** — exhaust local channels (3.iii) before touching external ones (3.i, 3.ii). If local sources answer the question, return without external search.

5. **External survey — operating instructions** (this section is **external-channel-only**; local channels in section 4 don't take "anti-pattern" or "license" queries). Framed as "how to operate whatever research method is in use", not "use tool X":
   - **Enforceable caps (Claude has no wall clock between tool calls — use counts instead):**
     - Max **5 distinct external queries**.
     - Max **15 sources reviewed** in total.
     - **Stop early** after two consecutive queries return zero new candidates.
   - **Guidance for human reviewers:** ~15 minutes wall-clock if the agent operator wants a rough budget for log review. Not operationally enforceable.
   - **Query strategy:** the 5 queries should cover: exact-match, related-problem, anti-pattern ("X considered harmful"), comparison ("X vs Y"), license/maintenance.
   - **Source scan:** filter by quality signals: active maintenance, recent commits, license clarity, ecosystem fit. Discard stale/abandoned/unlicensed candidates.
   - **Datapoint floor:** at least **2** independent datapoints per recommendation. Below floor → mark recommendation as `low-confidence`.

6. **Output: Adopt / Extend / Compose / Build matrix** — primary deliverable. The 4-tuple is borrowed from `affaan-m/everything-claude-code/skills/search-first/SKILL.md` (synthesized by that author; not a canonical analyst framework). Markdown table:

   | Option | Decision | Components | Source | License | Pros | Cons | Evidence |
   |--------|----------|------------|--------|---------|------|------|----------|

   Decision values:
   - **Adopt** — exact-match library/tool, use as-is. `Components` = single source URL.
   - **Extend** — close match, fork or wrap. `Components` = single source URL + wrap location.
   - **Compose** — combine multiple existing pieces. `Components` = list of 2+ source URLs (this column exists for Compose).
   - **Build** — nothing suitable exists. `Components` = N/A. Justify why in the `Cons`/`Evidence` columns; mark `low-confidence` if absence-of-evidence (rather than evidence-of-absence) is the basis.

   Footer: top recommendation in one sentence + open questions for the caller.

7. **Anti-patterns to avoid** (called out explicitly in skill body):
   - Rabbit-holing past the 15-minute cap.
   - Fabricating citations when a channel is blocked — escalate instead.
   - Silently skipping unavailable channels — list them in the output.
   - Endorsing stale, low-star, or unlicensed repos without flagging the risk.
   - "Apply X to Y" recommendations without genuine novelty or evidence of fit.
   - Naming a specific research tool in the recommendation as if the caller must use it — stay tool-agnostic.

8. **Pure-function contract — HARD-GATE.** Wrap in `<HARD-GATE>` block (mirroring brainstorming line 12-14):

   > `<HARD-GATE>`
   > Return the report and stop. Do NOT invoke writing-plans, brainstorming, executing-plans, subagent-driven-development, or any implementation skill. Do NOT begin coding. Do NOT create files outside the report. Do NOT call yourself recursively. The caller decides what happens next.
   >
   > **Permitted:** Asking the caller a single scoping question before enumerating channels (per Overview section 1) is permitted dialogue — it is not a "next-skill" invocation.
   > `</HARD-GATE>`

#### Output Shape (Returned to Caller)

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
<one sentence; "low-confidence" tag if datapoints below floor or if status = no-prior-art-found>

### Open questions
- ...
```

Status values explicitly defined (single canonical format: `status:detail`, colon-separated, no parens):
- `done` — survey ran, matrix populated.
- `skipped:no-impact` — impact gate fired; no research performed.
- `skipped:premature` — invoked while a brainstorming session is mid-clarification; deferred (see Overview section 1).
- `blocked:<reason>` — channels unreachable; caller must decide whether to retry or proceed without prior-art context.
- `no-prior-art-found` — channels worked, returned no usable candidates; matrix shows only `Build` row with low-confidence tag.

**Matrix omission for non-`done` statuses:** when status is `skipped:*`, `blocked:*`, or `no-prior-art-found` with no candidates at all, omit the Decision Matrix section entirely (replace with one-line "n/a — see status"). Local/External findings sections also omit if the corresponding channel set was not used.

### Brainstorming Integration

Modify `skills/brainstorming/SKILL.md`:

**Checklist (current lines 22-32, 9 items numbered 1-9).** Insert as new item 4. Existing items 4-9 become 5-10 (resulting list has 10 items):

```markdown
4. **Survey prior art** — invoke `superpowers:researching-prior-art` for non-trivial features (impact gate decides). Use the returned matrix to seed the 2-3 approaches in step 5.
```

The conditional Visual Companion item (item 2) remains conditional; renumbering still produces a valid sequence whether or not item 2 fires.

**Process flow DOT graph (current lines 36-64).** Literal diff — implementer applies these exact edits:

1. Add node declaration after line 43 (`"Ask clarifying questions" [shape=box];`):
   ```
   "Survey prior art" [shape=box];
   ```
2. Replace the existing edge `"Ask clarifying questions" -> "Propose 2-3 approaches";` with two edges:
   ```
   "Ask clarifying questions" -> "Survey prior art";
   "Survey prior art" -> "Propose 2-3 approaches";
   ```

**The Process section, "Exploring approaches" subsection (current lines 95-99).** Prepend one bullet:

```markdown
- For non-trivial features, the prior-art survey from the previous step has already produced an Adopt/Extend/Compose/Build matrix. Use it as the starting set of candidate approaches before adding alternatives from your own knowledge.
```

**Key Principles list (current lines 153-165).** Add one bullet:

```markdown
- **Survey prior art before approaches** - For non-trivial features, the `researching-prior-art` skill produces a candidate matrix. Use it to seed approaches rather than starting from scratch.
```

**Auto-trigger collision mitigation (symmetric — both sides gate).** Brainstorming's frontmatter ("MUST use before any creative work") and prior-art's trigger phrases overlap. Both skills must be aware:

- **Prior-art Overview clause:** *"If invoked standalone on an unscoped question (no concrete topic noun, no clear problem statement), ask one scoping question first before enumerating channels. If invoked while a brainstorming session is active and clarifying questions are not yet complete, defer — return `status:skipped:premature` and let brainstorming call you at the prior-art step."*
- **Brainstorming reuse clause (add to checklist item 4):** *"If a prior-art matrix has already been produced this session for the same scope, reuse it instead of re-invoking — no second survey."*

These two clauses prevent: (a) doubled survey cost when user runs prior-art standalone then asks to brainstorm, (b) wasted survey when prior-art fires first on an unscoped question.

**Item numbering convention.** Item numbers in the brainstorming checklist are ordinal positions, not stable identifiers. The Visual Companion item (item 2) is conditional, so any session may render with or without it. Downstream references in this spec and in the skill bodies use ordinal-relative phrasing ("the prior-art step", "after clarifying questions") rather than literal numbers like "item 4". Implementers maintaining either skill should preserve this convention.

**No changes** to brainstorming's terminal-state language ("The ONLY skill you invoke after brainstorming is writing-plans"). The new skill is called *within* brainstorming, not after it.

### Testing & Eval (Iron Law from writing-skills)

Per `skills/writing-skills/SKILL.md` line 374-393: no skill without a failing test first. Fork-only context lowers the bar (no PR eval evidence required) but does not eliminate it. Minimum:

**Fixture:** *"Add a rate limiter to a Node Express API."* Has known prior art (express-rate-limit, rate-limiter-flexible, bottleneck), forces an Adopt-vs-Compose decision, small enough to run twice in <30 min, and the matrix difference between RED and GREEN is obvious.

- **RED:** run brainstorming on the fixture WITHOUT the new skill installed; document the approaches Claude proposes (expected: drawn from training data, no live SOTA references, no Decision Matrix).
- **GREEN:** install the skill, re-run; document the approaches (expected: matrix populated with the three candidates above plus citations, top recommendation in one sentence).
- **Adversarial pressure case:** invoke brainstorming on a one-line bug fix in already-chosen library code; verify prior-art skill fires the always-skip override and returns `status:skipped:no-impact` rather than running queries.

Tests live as hand-run smoke checks; no automated harness required for fork-only.

## Tradeoffs

- **Skill-as-pure-function vs inline section:** chosen pure function. Costs one extra file and a frontmatter description. Buys reusability (writing-plans, manual invocation), decoupling (swap implementation without touching brainstorming), and easier independent eval. Precedent: `verification-before-completion` and `systematic-debugging` are pure-function-ish (return after their phases, do not dispatch a next skill). Counter-precedent: `writing-plans`, `executing-plans`, `subagent-driven-development` chain into one another — this skill must explicitly *not* follow that pattern, hence the HARD-GATE.
- **Auto-trigger description vs caller-only:** chosen auto-trigger. Description includes verbatim trigger phrases from `TrevorS/dot-claude/github-prior-art` so the skill fires on its own when the user asks "which library should I use" outside of brainstorming. Caller-only would have been simpler but loses standalone utility. Risk of collision with brainstorming addressed via Overview scoping clause.
- **Tool-naming vs tool-agnostic:** chosen tool-agnostic per user direction. Costs: Claude must select a tool at invocation time, which adds one micro-decision. Buys: skill survives MCP/tool churn, works across harnesses (Claude Code, Codex, etc.), no hard dependency on `research-agent` MCP.
- **15-minute cap vs strict 5-10:** chosen 15 generous. Costs occasional extra latency. Buys ability to cover complex domains without truncating mid-survey.
- **Adopt/Extend/Compose/Build matrix vs freeform report:** chosen matrix. Forces a concrete per-option recommendation and a uniform shape downstream consumers (brainstorming step 5) can read mechanically. `Components` column added to honor Compose's N-ary nature without breaking one-row-per-candidate.
- **Impact gate vs trivial-list gate:** chosen impact gate. Trivial-list (single-file, formatting, renames) under-fires on ambiguous middle-ground (add endpoint, add CLI flag). Impact gate ("would the answer change the design?") generalizes correctly and matches brainstorming's existing assumption-first discipline.

## Out of Scope

- Caching prior-art findings across sessions.
- Sharing matrices across worktrees or with team members.
- Auto-invoking the skill from `writing-plans` (future work — possible once the skill stabilizes).
- Upstream PR.
- Automated evaluation harness (manual smoke checks only for fork-only).

## Success Criteria

1. New skill file exists at `skills/researching-prior-art/SKILL.md` with all body sections above, including `<HARD-GATE>` wrap on the pure-function clause.
2. Frontmatter description contains only when-to-use trigger phrases, no workflow summary (verified against `writing-skills` lines 150-172). Description is under 1024 chars (writing-skills hard cap).
3. Brainstorming `SKILL.md` checklist renumbered: existing items 4-9 become 5-10; new item inserted at position 4.
4. Brainstorming `SKILL.md` DOT graph updated: one new node `"Survey prior art" [shape=box];`, one edge replacement (remove `"Ask clarifying questions" -> "Propose 2-3 approaches"`, add the two replacement edges).
5. Brainstorming `SKILL.md` "Exploring approaches" subsection prepends the prior-art-matrix bullet.
6. Brainstorming `SKILL.md` Key Principles list adds the prior-art bullet.
7. Brainstorming `SKILL.md` checklist item 4 includes the "reuse existing matrix this session" clause.
8. Invoking `superpowers:researching-prior-art` standalone on a scoped question returns a matrix-shaped report.
9. Invoking it standalone on an unscoped question produces one scoping question before any channel enumeration.
10. Invoking it while brainstorming is mid-clarification returns `status:skipped:premature`.
11. Running brainstorming on the fixture feature (rate limiter) shows the prior-art step firing before approaches are proposed; matrix populated with citations.
12. Running brainstorming on a one-line bug fix shows the prior-art step returning `status:skipped:no-impact` via the always-skip override.
13. Skill body contains no reference to a specific research tool (no "research-agent", "WebSearch", "exa", etc.).
14. RED/GREEN/adversarial smoke checks documented in the skill directory or commit message.
