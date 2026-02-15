# Token Efficiency Refactor Design

## Problem

Running review-hats with agent teams costs ~17% of a 5x Max plan's token budget per review. Agent teams spawn persistent context windows for each teammate, and with 4 hat agents + DA + Blue Hat, that's 6 concurrent context windows. The official docs confirm agent teams use ~7x more tokens than standard sessions.

## Solution

Hybrid architecture: Task tool subagents for Phase 1 (cheap, fire-and-forget), agent teams only for Phase 2 rebuttal (minimal team, only when needed). Tiered model selection uses Haiku for pattern-matching hats and Sonnet for depth-critical hats.

## Architecture

### Phase 1: Task Tool Subagents

Use the **Task tool** to spawn hat agents as fire-and-forget subagents. Each hat runs independently, returns findings, and dies. No persistent context windows.

**Model tiers:**
| Hat | Model | Rationale |
|-----|-------|-----------|
| White (Correctness) | Sonnet | Needs depth for logic bugs, edge cases |
| Red (Security) | Sonnet | Needs depth for vulnerability analysis |
| Green (Maintainability) | Haiku | Structured pattern-matching, naming, conventions |
| Indigo (Architecture) | Haiku | Module boundary analysis, coupling patterns |
| Yellow (Performance) | Haiku | Algorithmic complexity, N+1 patterns |
| Orange (Contracts) | Haiku | Breaking change detection, API surface |
| Purple (Testing) | Haiku | Coverage gaps, test quality patterns |
| Cyan (Library) | Haiku | Framework usage patterns, deprecated APIs |

### Phase 2: Minimal Agent Team (Rebuttal)

Only triggers on WARN/FAIL verdict. Spawns a minimal agent team:

1. **Developer Advocate** on **inherit** (user's model) — debate coordinator
2. **Only hats that produced Critical/Important findings** on **Sonnet** — need debate capability

DA coordinates via peer DMs, reports verdicts back to Blue Hat (same as current architecture).

### Flow Diagram

```
Phase 1 (Task tool, parallel, fire-and-forget):

Blue Hat ──┬── Task(white-hat, sonnet) ──→ findings
           ├── Task(green-hat, haiku)  ──→ findings
           ├── Task(indigo-hat, haiku) ──→ findings
           └── Task(orange-hat, haiku) ──→ findings

Blue Hat synthesizes report → verdict

If PASS: done
If WARN/FAIL:

Phase 2 (Agent team, minimal):

Blue Hat ──→ TeamCreate("rebuttal")
           ├── Spawn DA (inherit)
           ├── Spawn white-hat (sonnet, only if had findings)
           └── Spawn green-hat (sonnet, only if had findings)

Blue Hat ──→ DA: all findings + hat names
DA ←→ hats (peer DMs)
DA ──→ Blue Hat: verdict summary

Blue Hat renders revised report, cleans up team
```

## Savings Levers

1. **No persistent hat context windows during Phase 1** — Task tool subagents die after returning results, freeing their context immediately
2. **Haiku for 6/8 hats** — ~10x cheaper per token than Opus
3. **Smaller rebuttal team** — only hats with debatable findings, not all hats
4. **No agent team overhead when PASS** — most reviews should PASS, meaning no agent team spawned at all

## Model Selection in Agent/Skill Definitions

Hat agent definitions will use `model: inherit` (unchanged). The model tier is controlled by the Blue Hat orchestrator in SKILL.md when it specifies the model parameter on Task tool or agent team spawn calls.

## Files to Change

1. **`skills/review/SKILL.md`** — Step 4: switch from agent team to Task tool for Phase 1. Step 6: spawn minimal agent team for rebuttal only.
2. **`README.md`** — Update "How It Works" to reflect hybrid architecture.

## Baseline (2026-02-15 review run)

### Token usage
- 17% of 5x Max plan budget
- 6 agents: 4 hats + DA + Blue Hat orchestrator
- All agents on Opus

### Phase 1 findings (for quality comparison)
- **White Hat (WARN)**: 4 Important (verdict logic contradiction, PARTIAL+COUNTER unhandled, Withdrawn ambiguity, DA omits Withdrawn condition), 1 Minor
- **Green Hat (WARN)**: 2 Important (ambiguous "the lead", inconsistent teammate names), 2 Minor
- **Indigo Hat (WARN)**: 2 Important (agents reference skill-internal paths, DA different role), 2 Minor
- **Orange Hat (WARN)**: 2 Important (teammate name contradiction, version gap), 1 Minor
- **Overall: WARN** (9 Important, 6 Minor)

### Phase 2 results
- 9 findings debated
- 0 upheld, 6 withdrawn, 3 downgraded
- **Revised: PASS**

## Success Criteria

After implementation, run the same review (`--branch origin/main`) and compare:
1. **Token usage**: Target <10% of 5x plan (down from 17%)
2. **Finding coverage**: Should catch similar Important findings (some variance expected)
3. **Rebuttal quality**: DA should still produce well-reasoned verdicts

Sources:
- [Claude Code cost management docs](https://code.claude.com/docs/en/costs)
- [Agent Teams vs Sub-Agents comparison](https://www.geeky-gadgets.com/agent-teams-token-usage/)
- [Qodo's use of Haiku 4.5 for code review](https://www.qodo.ai/blog/qodos-default-reviewers-why-we-picked-gpt-5-2-gemini-2-5-pro-and-claude-haiku-4-5/)
- [Claude Code agent teams docs](https://code.claude.com/docs/en/agent-teams)
