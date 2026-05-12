# Researching Prior Art: Pure-Function Survey Skill

**Date:** 2026-05-12
**Status:** Approved
**Scope:** `skills/researching-prior-art/SKILL.md` (new), `skills/brainstorming/SKILL.md` (modify)

## Problem

Brainstorming currently moves from clarifying questions straight to proposing 2-3 approaches. The approaches are drawn from Claude's training data and the current conversation — there is no step that surveys the live landscape for existing tools, libraries, plugins, or SOTA patterns that already solve the problem. As a result, designs reinvent wheels, miss readymade fork targets, and overlook common-problem patterns that the ecosystem has already converged on.

Adjacent skill collections (`affaan-m/everything-claude-code/search-first`, `dwmkerr/claude-toolkit/research`, `acking-you/myclaude-skills/research`, `trevors/dot-claude/github-prior-art`, `wanshuiyin/.../idea-creator`) have each codified a prior-art research step. Superpowers does not.

This fork adds one — as a tool-agnostic, pure-function skill that brainstorming can call and that other skills (e.g. `writing-plans`) can reuse later.

## Goals

- Survey existing tools / libraries / plugins / SOTA approaches before brainstorming commits to a design.
- Keep the skill tool-agnostic: it states **what** to research and **how to evaluate**, never **which tool** to use. Tool selection is Claude's call at invocation time.
- Pure function: skill returns findings, does not chain to a next skill. Caller owns continuation.
- Cheap to skip for trivial work (single-file edits, behavior tweaks, already-researched-this-session).
- Reusable from brainstorming, writing-plans, or manual invocation.

## Non-Goals

- Mandatory invocation on every conversation. Skip gate exists for trivial work.
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
    SKILL.md              # modified — insert step 3.5
```

### New Skill: `researching-prior-art/SKILL.md`

#### Frontmatter

```yaml
---
name: researching-prior-art
description: Survey existing tools, libraries, plugins, and SOTA approaches before committing to a design. Use when starting a new feature, choosing a library, or asked "how do I build X" / "what's the best way to" / "which library should I use" — produces an Adopt/Extend/Compose/Build matrix with citations. Pure function: returns findings, does not chain.
---
```

The description doubles as auto-trigger criteria (frontmatter phrases) and as caller-side documentation.

#### Body Sections

1. **Overview** — one paragraph stating the skill is a pure function: input = topic/problem, output = matrix + recommendation, no continuation.

2. **Skip gate** — return immediately with `skipped: trivial` when any of these hold:
   - Single-file edit, formatting change, rename, or one-line fix.
   - Change inside a subsystem already researched earlier this session.
   - Caller explicitly marks the request trivial.

3. **Available channels** — at start, enumerate what's reachable for research in this session (search tools, fetch tools, MCP research agents, local indexes). Report unavailable channels honestly in the output. Never fabricate when blocked — escalate to the caller with `status: blocked` and a one-line reason.

4. **Local-before-external** — before touching external sources, check in order:
   1. `rg` the current repo for prior implementations or comments.
   2. Project package manifest (`package.json`, `pyproject.toml`, `Cargo.toml`, `flake.nix`, etc.) — is something already a dependency?
   3. Installed MCP servers (`~/.claude/settings.json`, plugin marketplace entries).
   4. Installed skills (`~/.claude/skills/`, `~/.codex/skills/`, plugin skill dirs).
   5. Only after the above, external sources.

5. **External survey** — operating instructions, framed as how to operate whatever research method is in use:
   - **Hard cap:** 15 minutes wall clock. Stop at the cap, report what's found.
   - **Query strategy:** at least 5 distinct query formulations covering: exact-match search, related-problem search, anti-pattern search ("X considered harmful"), comparison search ("X vs Y"), license/maintenance search.
   - **Source scan:** review 10-15 candidate sources before recommending. Filter by quality signals: active maintenance, recent commits, license clarity, ecosystem fit. Discard stale/abandoned/unlicensed candidates.
   - **Datapoint floor:** at least 2-3 independent datapoints per recommendation. Below floor → mark recommendation as `low-confidence`.

6. **Output: Adopt / Extend / Compose / Build matrix** — primary deliverable. Markdown table, one row per credible candidate:

   | Option | Decision | Source | License | Pros | Cons | Evidence |
   |--------|----------|--------|---------|------|------|----------|

   Decision values:
   - **Adopt** — exact-match library/tool, use as-is.
   - **Extend** — close match, fork or wrap.
   - **Compose** — combine multiple existing pieces.
   - **Build** — nothing suitable exists, justify why.

   Footer: top recommendation in one sentence + open questions for the caller.

7. **Anti-patterns to avoid** (called out explicitly in the skill body):
   - Rabbit-holing past the 15-minute cap.
   - Fabricating citations when a channel is blocked — escalate instead.
   - Silently skipping unavailable channels — list them in the output.
   - Endorsing stale, low-star, or unlicensed repos without flagging the risk.
   - "Apply X to Y" recommendations without genuine novelty or evidence of fit.
   - Naming a specific research tool in the recommendation as if the caller must use it — stay tool-agnostic.

8. **Pure-function contract** — explicit closing section: "Return the report and stop. Do not invoke any other skill. Do not begin implementation. The caller decides what happens next."

#### Output Shape (Returned to Caller)

```markdown
## Prior-Art Survey: <topic>

**Status:** done | skipped: trivial | blocked: <reason>
**Channels used:** <list> | **Skipped:** <list with reason>
**Wall clock:** <minutes>
**Datapoints:** <count>

### Local findings
- ...

### External findings
- ...

### Decision Matrix
| Option | Decision | Source | License | Pros | Cons | Evidence |
|--------|----------|--------|---------|------|------|----------|
| ...    | Adopt    | ...    | MIT     | ...  | ...  | ...      |

### Recommendation
<one sentence>

### Open questions
- ...
```

### Brainstorming Integration

Modify `skills/brainstorming/SKILL.md`:

**Checklist (line 22-32):** insert new item 4, renumber subsequent items:

```markdown
4. **Survey prior art** — invoke `superpowers:researching-prior-art` for non-trivial features. Skip explicitly for micro-edits. Use the returned matrix to seed the 2-3 approaches in the next step.
```

Existing items 4–9 become 5–10.

**Process flow DOT graph (line 36-64):** insert node `"Survey prior art"` between `"Ask clarifying questions"` and `"Propose 2-3 approaches"`. Add edge `"Ask clarifying questions" -> "Survey prior art" -> "Propose 2-3 approaches"`. Remove the direct edge.

**The Process section (line 95-99 "Exploring approaches"):** prepend one sentence:

```markdown
- For non-trivial features, the prior-art survey from the previous step has already produced an Adopt/Extend/Compose/Build matrix. Use it as the starting set of candidate approaches before adding alternatives from your own knowledge.
```

**Key Principles list (line 153-165):** add one bullet:

```markdown
- **Survey prior art before approaches** - For non-trivial features, the `researching-prior-art` skill produces a candidate matrix. Use it to seed approaches rather than starting from scratch.
```

**No changes** to the terminal-state language ("The ONLY skill you invoke after brainstorming is writing-plans"). The new skill is called *within* brainstorming, not after it.

## Tradeoffs

- **Skill-as-pure-function vs inline section:** chosen pure function. Costs one extra file and a frontmatter description. Buys reusability (writing-plans, manual invocation), decoupling (swap implementation without touching brainstorming), and easier independent eval. Matches superpowers' existing decomposition (writing-plans, executing-plans, subagent-driven-development are all separate skills).
- **Auto-trigger description vs caller-only:** chosen auto-trigger. Description includes trevors-style trigger phrases so the skill fires on its own when the user asks "which library should I use" outside of brainstorming. Caller-only would have been simpler but loses standalone utility.
- **Tool-naming vs tool-agnostic:** chosen tool-agnostic per user direction. Costs: Claude must select a tool at invocation time, which adds one micro-decision. Buys: skill survives MCP/tool churn, works across harnesses (Claude Code, Codex, etc.), no hard dependency on `research-agent` MCP.
- **15-minute cap vs strict 5-10:** chosen 15 generous. Costs occasional extra latency. Buys ability to cover complex domains without truncating mid-survey.
- **Adopt/Extend/Compose/Build matrix vs freeform report:** chosen matrix. Forces a concrete per-option recommendation and a uniform shape downstream consumers (brainstorming step 4) can read mechanically.

## Out of Scope

- Caching prior-art findings across sessions.
- Sharing matrices across worktrees or with team members.
- Auto-invoking the skill from `writing-plans` (future work — possible once the skill stabilizes).
- Upstream PR.
- Evaluation harness for the skill (no before/after eval planned; this is fork-only).

## Success Criteria

1. New skill file exists at `skills/researching-prior-art/SKILL.md` with all body sections above.
2. Brainstorming SKILL.md checklist, DOT graph, exploring-approaches text, and key-principles list are all updated consistently.
3. Invoking `superpowers:researching-prior-art` standalone returns a matrix-shaped report.
4. Running brainstorming on a non-trivial feature shows the prior-art step firing before approaches are proposed.
5. Running brainstorming on a one-line fix shows the prior-art step skipping with `skipped: trivial`.
6. Skill body contains no reference to a specific research tool (no "research-agent", "WebSearch", "exa", etc.).
