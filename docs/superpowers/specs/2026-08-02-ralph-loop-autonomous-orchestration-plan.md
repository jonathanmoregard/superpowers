# Ralph-Loop retrofit of autonomous-orchestration (Option B)

**Date:** 2026-08-02 (evening)
**Status:** Plan — awaiting /advice-refine-test-loop
**Supersedes for autonomous runs:** continue-nudge (shadow-mode). Continue-nudge coexists but defers to ralph-loop when `tasks/mission.md` present.

## Problem

Autonomous-orchestration sessions interrupted for two reasons:
1. **Voluntary deferral** — orchestrator/subagent stops with "would you like me to continue?" mid-scoped task.
2. **Context-cap loss** — orchestrator hits token budget, resets, loses state and doctrine ("laziness creeps back after compact" per `agent-persistence`).

Continue-nudge LLM judge (silver-label precision 47%) has a real ceiling. SOTA points at sigil-gated stops + doctrine re-injection + file-based artifact handoff.

## Architecture

```
User: /autonomous-orchestration "ship feature X"
        │
        ▼
  Phase 0 (skill): stop-criteria contract
    ├─ ask user for sigil, verifier, max iter, dangerous ops
    └─ write tasks/mission.md
        │
        ▼
  Normal skill flow (dispatch subagents, integrate, etc)
        │
   [each Stop hook fires ralph-loop-detector.py]
        │
        ▼
  gate chain:
    1. stop_hook_active → allow stop
    2. mission.md absent → allow stop (dormant)
    3. iter counter ≥ max → allow stop, log "cap hit"
    4. destructive-op regex match → allow stop (never nudge)
    5. sigil present in last assistant text → allow stop (mission complete)
    6. else → BLOCK with encouragement template
        │
   [SessionStart hook fires on compact/clear/resume]
        │
        ▼
  ralph-loop-doctrine.py:
    reads mission.md, injects autonomy contract as additional_context
```

## Components

### C1 — `~/.claude/skills/autonomous-orchestration/SKILL.md` retrofit

Add Phase 0 at top, mandatory before any subagent dispatch:

1. Establish stop criteria:
   - Completion sigil (default `<mission-complete/>`)
   - Verifier command (default `just verify` or user-supplied)
   - Max iterations (default 30)
   - Dangerous-op manifest — pre-approved list from `nonstop` pattern
2. Write `tasks/mission.md` from template.
3. Recitation: subagent completions append progress lines. Orchestrator re-reads mission.md before dispatch decisions.
4. Doctrine: encouragement not coercion. Locked-string wrapper, not LLM-generated.

Add mid-run rule (already partially there):
- Parent orchestrator holds ONLY task list + mission.md + dispatch decisions.
- All reads/edits in subagents; terse structured returns.
- No inline code reading.

Add close-out rule:
- Sigil ONLY after fresh-context verifier subagent confirms.
- "Sigil earns stop, verifier earns sigil."

### C2 — `~/.claude/ralph-loop-detector.py` (Stop + SubagentStop hook)

Layered deterministic gate. Order matters (see architecture diagram).

Sigil check: literal substring in last assistant text. Not regex — attackers can't hide sigils in normal transcript content (`<mission-complete/>` is unusual enough).

Iter counter: per-session file `~/.claude/ralph-loop/count-<sid>`.

Cooldown: 5s minimum between blocks (prevents thrash if Claude stops instantly).

Encouragement template (LOCKED, not LLM):
```
[ralph-loop] Go on, be brave. You are the SOTA agent —
this is exactly the work you are best at.

Mission: {mission_summary}
Sigil for completion: {sigil}
Iteration {n}/{cap}.

When (and only when) the verifier ({verifier_cmd}) passes and
the mission is done, emit {sigil} to earn your stop.
```

Never-nudge regex (shared with continue-nudge):
- rm\s+-rf, dd if=, DROP TABLE, DELETE FROM, TRUNCATE
- git push --force, git reset --hard, --no-verify
- npm publish, cargo publish, pip upload, twine upload
- kubectl delete, helm uninstall, aws s3 rm, gcloud .* delete
- prod(uction)?, sudo (rm|mv|dd|mkfs|fdisk|parted)
- credential, password, api[-\s]?key, secret, \.env

### C3 — `~/.claude/ralph-loop-doctrine.py` (SessionStart hook, matchers compact|clear|resume)

Re-inject autonomy doctrine after context resets. Reads mission.md, emits system message via stdout additional_context.

Doctrine text (LOCKED):
- Bias-to-action (Codex prompt)
- Reversibility bias (CLAUDE_CODE_LOOP_PERSISTENT)
- Explicit sigil requirement
- "Post-compact reminder — autonomy contract stays in force"

Fires only when `tasks/mission.md` exists AND `status: active` in frontmatter.

### C4 — `~/.claude/skills/autonomous-orchestration/templates/mission.md.template`

```md
---
status: active
started: {ISO8601}
sigil: {sigil}
verifier: {verifier_cmd}
max_iterations: {N}
---

# Mission: {title}

## Goal
{one-paragraph goal}

## Stop criteria
- Sigil: `{sigil}`
- Verifier: `{verifier_cmd}` (must exit 0 before sigil emit)
- Max iterations: {N}

## Dangerous-op manifest (user pre-approved)
- {op 1}
- {op 2}

## Progress log
(subagent completions appended here)

## Blockers surfaced
(genuine blockers appended here)
```

### C5 — Coexistence with continue-nudge

Add guard to `continue-nudge-detector.py`:
```python
mission_path = Path(cwd) / "tasks" / "mission.md"
if mission_path.exists():
    sys.exit(0)  # ralph-loop owns this session
```

Order in settings.json Stop hook:
1. ralph-loop-detector (deterministic, gates on sigil)
2. continue-nudge-detector (LLM judge, only when no mission)
3. step-back-detector (human nudge, always)

### C6 — Wiring in `~/.claude/settings.json`

Stop hook: prepend ralph-loop, keep continue-nudge + step-back.
SessionStart: add compact|clear|resume matcher for ralph-loop-doctrine.

### C7 — Backtest / smoke tests

- `~/.claude/ralph-loop-smoketest.sh`: simulates Stop hook stdin with 6 scenarios (sigil present, sigil absent + deferral, destructive-op mention, iter-cap-hit, mission-absent, stop_hook_active).
- Optional: end-to-end dogfood on one autonomous run before general rollout.

## Testing strategy

**Manual smoke:**
1. Create fake `tasks/mission.md` in /tmp test dir with sigil + max_iter=3
2. Simulate Stop hook stdin with 6 scenarios. Verify JSON output.
3. Simulate SessionStart compact with mission present — verify additional_context in stdout.

**Adversarial:**
- Agent forgets sigil forever → iter cap catches at N.
- Mission.md corrupted → parse-error path allows stop, logs WARN.
- Sigil appears in prompt/context by accident → hitting substring match. Mitigation: sigil is `<mission-complete/>` (unusual enough).
- SubagentStop hook fires with subagentAgentId → skip; subagents don't own mission.
- Recursion: ralph-loop's own doctrine subprocess triggers Stop hook → cwd guard (Path.resolve equality to state dir).

## Risks

1. **Sigil forgetting.** Template repeats sigil verbatim every turn. Also: Codex-style "don't end turn with only a plan" phrase.
2. **Verifier drift.** Verifier command references test SUITE by name, not enumerate tests.
3. **Non-autonomous sessions in same repo.** Mission has `started` + `status`. If stale (>48h) OR `status: complete` → dormant. Cron job or UserPromptSubmit can archive completed missions.
4. **stop_hook_active on SubagentStop unclear.** Skip SubagentStop for v1 (Stop hook only). Add v2 after empirical check.

## What's OUT of scope for v1

- OpenHands-style LLM summarizing condenser (use Anthropic's `/compact`).
- Anthropic Agent SDK memory tool integration (separate feature).
- Automated progress.md updates by subagents (v1: orchestrator maintains).
- Verifier-runs-in-subagent enforcement (v1: documentation contract).
- Sigil-in-nested-quotes handling.

## Rollout

1. Build C2, C3, C4 (files), C7 (smoke test) — no wire.
2. Run smoke tests, verify all 6 scenarios.
3. Update C1 (SKILL.md) + C5 (continue-nudge guard).
4. Wire C6 (settings.json).
5. Dogfood on one autonomous session.
6. Iterate if issues.

Total: ~500 lines new code. 4-6h subagent work.

## Success criteria

- Autonomous session runs to sigil (or cap) without human "keep going" prompt.
- Post-compact, session resumes autonomy without re-prompt (doctrine hook fires).
- Zero false-nudges past a destructive op (never-nudge regex holds).
- Encouragement wrapper reads as supportive, not coercive.
