# Token Efficiency Refactor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Switch Phase 1 from agent teams to Task tool subagents with tiered model selection, and make Phase 2 rebuttal spawn a minimal agent team only when needed.

**Architecture:** Phase 1 uses fire-and-forget Task tool subagents (Haiku for pattern-matching hats, Sonnet for White/Red). Phase 2 spawns a minimal agent team only on WARN/FAIL with just the DA and hats that produced debatable findings on Sonnet. Blue Hat context stays lean in both phases.

**Tech Stack:** Claude Code Task tool, agent teams (SendMessage), markdown skill definitions

---

### Task 1: Rewrite Step 4 — Task tool dispatch with model tiers

**Context:** Step 4 currently creates an agent team for Phase 1 and dispatches hats as teammates. We're switching to Task tool subagents (fire-and-forget) with tiered model selection. The agent team fallback logic is removed — Task tool IS the primary path now. Agent teams are only used in Step 6 for rebuttal.

**Files:**
- Modify: `skills/review/SKILL.md:77-126` (Step 4, replace entirely)

**Step 1: Replace Step 4**

Replace everything from `## Step 4:` (line 77) through line 126 (end of `### Collect Results`) with:

~~~markdown
## Step 4: Dispatch Hat Agents

Spawn all selected hat agents using the Task tool in a **single message** so they run in parallel.

### Model Selection

Use tiered model selection based on hat type:

| Hat | Model | Rationale |
|-----|-------|-----------|
| White (Correctness) | `sonnet` | Needs depth for logic bugs, edge cases |
| Red (Security) | `sonnet` | Needs depth for vulnerability analysis |
| Green (Maintainability) | `haiku` | Structured pattern-matching |
| Indigo (Architecture) | `haiku` | Module boundary analysis |
| Yellow (Performance) | `haiku` | Algorithmic complexity patterns |
| Orange (Contracts) | `haiku` | Breaking change detection |
| Purple (Testing) | `haiku` | Coverage gap patterns |
| Cyan (Library) | `haiku` | Framework usage patterns |

### Spawn Prompt

Each Task tool dispatch MUST include:

1. The full diff output (from Step 2)
2. The list of changed files
3. Whether verbose mode is on (for the Strengths section)
4. Instructions to read surrounding code files for context using Read/Grep/Glob tools
5. A reminder to follow the standardized output format from `references/hat-output-format.md`

**Task tool dispatch for each hat:**

~~~
Task tool (review-hats:[agent-name]):
  description: "[Color] Hat [domain] review"
  model: [sonnet or haiku per table above]
  prompt: |
    Review the following code changes from the perspective described in your agent definition.

    ## Changed Files
    <one file path per line>

    ## Diff
    <complete git diff output from Step 2>

    ## Instructions
    - Analyze the diff thoroughly from your specialized perspective
    - Use Read, Grep, and Glob tools to examine surrounding code for context
    - Follow the output format exactly as specified in references/hat-output-format.md
    - Severity: Critical = must fix before merge, Important = should fix, Minor = nice to fix
    - Verdict: FAIL if any Critical, WARN if any Important, PASS otherwise
    <if verbose>- Include a "Strengths" section noting what's done well</if>
    <if not verbose>- Do NOT include a Strengths section</if>
    - Do NOT suggest fixes or write code. Report findings only.
~~~

Dispatch ALL selected hats in a single message for parallel execution.
~~~

**Step 2: Verify the edit**

Read back `skills/review/SKILL.md` lines 77-130 and confirm:
- Step 4 heading is "Dispatch Hat Agents" (no mention of "Create Review Team")
- Model Selection table exists with White/Red on sonnet, rest on haiku
- Task tool dispatch template exists (not agent team spawn)
- No references to TeamCreate, SendMessage, or agent teams in Step 4
- No "Collect Results" subsection (Task tool returns results directly)

**Step 3: Commit**

```bash
git add skills/review/SKILL.md
git commit -m "refactor: switch Phase 1 from agent teams to Task tool subagents

Hats now run as fire-and-forget Task tool subagents with tiered
model selection: Sonnet for White/Red, Haiku for all others.
No persistent context windows during Phase 1."
```

IMPORTANT: Do NOT add Co-Authored-By lines to commits.

---

### Task 2: Rewrite Step 6 — Minimal agent team for rebuttal

**Context:** Step 6 currently assumes hats are already alive in an agent team from Phase 1. Since Phase 1 now uses Task tool (hats are dead), Step 6 needs to create a fresh agent team with only the DA + hats that produced debatable findings. The skip condition about "Task tool fallback" is removed since Task tool is now the primary Phase 1 path. The teammate name example should also be fixed to use correct full names (review finding from baseline).

**Files:**
- Modify: `skills/review/SKILL.md:195-272` (Step 6 + Important Notes, replace entirely)

**Step 1: Replace Step 6 and Important Notes**

Replace everything from `## Step 6: Rebuttal Phase` (line 195) through end of file (line 272) with:

~~~markdown
## Step 6: Rebuttal Phase

**Skip this step if:**
- The overall verdict is PASS (nothing to debate)
- The user passed `--no-rebuttal`
- No Critical or Important findings exist

**If skipped**, present the Phase 1 report from Step 5 as the final output and stop.

### Check Agent Teams Availability

Try to create an agent team named `rebuttal`. If agent teams are unavailable (feature not enabled):

- Present the Phase 1 report as the final output
- Append to the report: "Rebuttal phase skipped — enable agent teams with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings for automated finding challenges."
- Stop.

### Collect Debatable Findings

From the Phase 1 report, extract all Critical and Important findings into this structure:

~~~
[ID]: [Title]
Hat: [agent-name]
Severity: [Critical | Important]
Location: [file:line]
What: [description]
Why: [impact]
~~~

Number findings sequentially: C1, C2... for Critical, I1, I2... for Important. Include the hat's **agent name** (e.g., `white-hat-correctness`, `red-hat-security`) matching the names in the Step 1 table.

### Spawn Rebuttal Team

Create a minimal agent team with only the agents needed for debate:

1. **Developer Advocate** — debate coordinator (inherits user's model)
2. **Only the hat agents that produced Critical or Important findings** — on `sonnet` model for debate quality

Spawn all teammates in a **single message**:

**Developer Advocate spawn prompt:**
~~~
You are the Developer Advocate and debate coordinator for this rebuttal review.

## Changed Files
<one file path per line>

## Diff
<complete git diff output from Step 2>

## Debatable Findings
<all findings from the "Collect Debatable Findings" step above, in severity order>

## Instructions
- Follow your agent definition exactly
- For each finding, message the hat teammate by name (peer DM) to run the debate
- After all findings are debated, send your verdict summary back to the team lead
- Follow the rebuttal protocol in references/rebuttal-protocol.md
~~~

**Hat agent spawn prompt (for each hat with debatable findings):**
~~~
You are the [Color] Hat in a rebuttal review. You produced findings during Phase 1 that are now being challenged.

## Your Phase 1 Findings
<this hat's Critical and Important findings>

## Diff
<complete git diff output from Step 2>

## Instructions
- The Developer Advocate will message you to present each finding for debate
- Present findings using the format in references/rebuttal-protocol.md
- For each Developer response, reply with ACCEPT or COUNTER
- Only COUNTER if you have specific evidence the defense is insufficient
- Use Read, Grep, and Glob tools to verify claims made in defenses
~~~

### Wait for Verdict Summary

The DA will run the entire debate via peer DMs with hat agents. When finished, it sends back a single verdict summary message. Wait for this message.

If the DA goes idle without sending a summary, message it: "Send your verdict summary now."

### Produce the Revised Report

Using the DA's verdict summary, produce the revised report following the format in `references/revised-report-format.md`.

Recalculate each hat's verdict based on remaining upheld findings:
- FAIL if any Upheld Critical findings
- WARN if any Upheld Important findings (no Upheld Criticals)
- PASS if all Critical/Important were Withdrawn or Downgraded below Important

### Clean Up the Team

After the revised report is produced (whether successfully or not):
1. Send shutdown requests to all teammates
2. Clean up the team resources (TeamDelete)

## Important Notes

- Dispatch ALL selected hats in a single message for parallel execution (Phase 1, Task tool)
- Phase 1 hats are fire-and-forget — they return findings and die
- Phase 2 spawns a fresh agent team with only the DA and hats that had debatable findings
- The Developer Advocate coordinates the entire debate via peer DMs — the Blue Hat does NOT relay messages
- If a hat agent fails or returns malformed output, note it in the report but continue with other hats
- Keep synthesis concise — the hat reports have the details, the synthesis highlights what matters most
- Always clean up the rebuttal team after completing the review
~~~

**Step 2: Verify the edit**

Read back `skills/review/SKILL.md` from line 195 onward and confirm:
- Skip conditions do NOT mention "Phase 1 fell back to Task tool" (removed — Task tool is primary)
- "Check Agent Teams Availability" creates a team named `rebuttal` (not `review-hats`)
- Debatable findings use full agent names (e.g., `white-hat-correctness`, `red-hat-security`)
- "Spawn Rebuttal Team" creates DA + only relevant hats
- Hat spawn prompt gives them their Phase 1 findings and the diff
- Important Notes say "Phase 1 hats are fire-and-forget"
- No reference to hats "staying alive" between phases

**Step 3: Commit**

```bash
git add skills/review/SKILL.md
git commit -m "refactor: minimal agent team for rebuttal only

Phase 2 now creates a fresh agent team with only the DA and hats
that produced debatable findings, on Sonnet for debate quality.
No agent team spawned when verdict is PASS."
```

---

### Task 3: Update frontmatter and description

**Context:** The SKILL.md frontmatter description references agent teams for Phase 1, which is no longer accurate. Also fix the "the lead" leftover in rebuttal-protocol.md (review finding from baseline).

**Files:**
- Modify: `skills/review/SKILL.md:3-7` (frontmatter description)
- Modify: `skills/review/references/rebuttal-protocol.md:100` (fix "the lead")

**Step 1: Update frontmatter**

In `skills/review/SKILL.md`, replace lines 3-7:

```yaml
description: >
  Run a multi-perspective code review using specialized hat agents.
  Each hat reviews from a distinct angle (security, architecture, correctness, etc.)
  and produces findings grouped by severity. Uses agent teams to keep hat agents alive
  for rebuttal debate when findings are challenged.
```

with:

```yaml
description: >
  Run a multi-perspective code review using specialized hat agents.
  Each hat reviews from a distinct angle (security, architecture, correctness, etc.)
  and produces findings grouped by severity. Uses tiered model selection (Sonnet for
  depth-critical hats, Haiku for pattern-matching) and agent teams for rebuttal debate.
```

**Step 2: Fix "the lead" in rebuttal protocol**

In `skills/review/references/rebuttal-protocol.md`, find line 100:

```
- No back-and-forth beyond the single rebuttal. The lead's verdict is final.
```

Replace with:

```
- No back-and-forth beyond the single rebuttal. The Developer Advocate's verdict is final.
```

**Step 3: Verify both files**

- Read `skills/review/SKILL.md` lines 1-8 — frontmatter mentions "tiered model selection" and "Haiku"
- Read `skills/review/references/rebuttal-protocol.md` line 100 — says "Developer Advocate's verdict"

**Step 4: Commit**

```bash
git add skills/review/SKILL.md skills/review/references/rebuttal-protocol.md
git commit -m "fix: update skill description and fix leftover 'the lead' reference

Frontmatter now describes tiered model selection. Rebuttal protocol
rule correctly attributes verdict to Developer Advocate."
```

---

### Task 4: Update README

**Context:** The README "How It Works" paragraph mentions agent teams for dispatch. It should reflect the hybrid architecture. Keep it brief — one sentence change.

**Files:**
- Modify: `README.md:7` (How It Works paragraph)

**Step 1: Update How It Works**

In `README.md`, replace line 7:

```
A Blue Hat orchestrator (the skill) analyzes your diff, selects the relevant hats, dispatches them **in parallel** as an agent team, and synthesizes their findings into a single report grouped by severity. When findings are challenged, a Developer Advocate coordinates the rebuttal debate via direct messages with hat agents, then reports verdicts back to the orchestrator.
```

with:

```
A Blue Hat orchestrator (the skill) analyzes your diff, selects the relevant hats, dispatches them **in parallel** using tiered model selection (Sonnet for depth-critical reviews, Haiku for pattern-matching), and synthesizes their findings into a single report grouped by severity. When findings are challenged, a Developer Advocate coordinates the rebuttal debate via direct messages with hat agents, then reports verdicts back to the orchestrator.
```

**Step 2: Verify**

Read `README.md` line 7 — mentions "tiered model selection" and "Sonnet/Haiku".

**Step 3: Commit**

```bash
git add README.md
git commit -m "docs: update README for hybrid Task tool + agent teams architecture"
```

---

### Task 5: Test the refactored review

**Context:** Run the same review as our baseline (`--branch origin/main`) and compare token usage and finding quality.

**Step 1: Run the review**

```
/review-hats:review --branch origin/main
```

**Step 2: Compare results**

After the review completes, compare against the baseline (from `docs/plans/2026-02-15-token-efficiency-design.md`):

| Metric | Baseline | New |
|--------|----------|-----|
| Token usage | 17% of 5x plan | Target: <10% |
| Phase 1 findings | 9 Important, 6 Minor | Should be similar |
| Hats selected | White, Green, Indigo, Orange | Should be same |
| Overall verdict | WARN → PASS (after rebuttal) | Compare |

Report the comparison to the user.
