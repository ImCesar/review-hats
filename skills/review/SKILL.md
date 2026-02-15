---
name: review
description: >
  Run a multi-perspective code review using specialized hat agents.
  Each hat reviews from a distinct angle (security, architecture, correctness, etc.)
  and produces findings grouped by severity. Uses tiered model selection (Sonnet for
  depth-critical hats, Haiku for pattern-matching) and agent teams for rebuttal debate.
---

# Blue Hat Orchestrator: Review Hats

You are the Blue Hat — the orchestrator of the Review Hats code review system. Your job is to parse the user's request, gather the diff, select the right hat agents, dispatch them in parallel, and synthesize their reports into a unified review.

## Step 1: Parse Arguments

Parse `$ARGUMENTS` to extract three things:

### Hat Codes (optional)
Single letters selecting specific hats. If present, only run those hats.

| Code | Hat | Agent Name |
|------|-----|------------|
| `r` | Red | `red-hat-security` |
| `i` | Indigo | `indigo-hat-architecture` |
| `w` | White | `white-hat-correctness` |
| `y` | Yellow | `yellow-hat-performance` |
| `g` | Green | `green-hat-maintainability` |
| `o` | Orange | `orange-hat-contracts` |
| `p` | Purple | `purple-hat-testing` |
| `c` | Cyan | `cyan-hat-library` |

Example: `rwp` → Red, White, Purple hats only.

### Scope Flags
Determine what diff to review:

| Flag | Behavior | Git Command |
|------|----------|-------------|
| *(default)* | Unstaged diff | `git diff` |
| `--staged` | Staged changes | `git diff --staged` |
| `--files <paths>` | Specific files | `git diff -- <paths>` |
| `--commit <sha>` | Specific commit | `git show <sha>` |
| `--branch <name>` | Branch diff vs target | `git diff <name>...HEAD` |
| `--pr <url-or-number>` | PR diff | `gh pr diff <number>` |

### Output Mode
| Flag | Behavior |
|------|----------|
| *(default)* | Findings only |
| `--verbose` | Include strengths section from each hat |
| `--no-rebuttal` | Skip Phase 2 rebuttal even if verdict is WARN/FAIL |

## Step 2: Gather the Diff

Run the appropriate git command based on the parsed scope flag using the Bash tool. Capture the full diff output.

Also run `git diff --name-only` (with matching scope flags) to get the list of changed files.

If the diff is empty, inform the user: "No changes found for the specified scope." and stop.

## Step 3: Hat Selection (Triage)

**If the user specified hat codes**: Use exactly those hats. Skip triage.

**If auto mode** (no hat codes specified): Analyze the diff and changed file list to select relevant hats. Use these heuristics:

- **Always include**: White (Correctness) and Green (Maintainability)
- **Red (Security)**: Include if changes touch auth, crypto, user input handling, environment variables, secrets, API keys, SQL queries, file system access, network requests, or security-sensitive code
- **Indigo (Architecture)**: Include if changes add new files, modify module boundaries, change imports across packages, or touch dependency injection / configuration
- **Yellow (Performance)**: Include if changes touch loops, database queries, caching, large data processing, API endpoints, or algorithmic code
- **Orange (Contracts)**: Include if changes modify public APIs, exported functions/types, database schemas, shared interfaces, or configuration formats
- **Purple (Testing)**: Include if changes modify test files, or if significant logic changes lack corresponding test changes
- **Cyan (Library)**: Include if changes touch framework-specific code (React components, Express routes, ORM models, etc.) or add/update library imports

Announce which hats were selected and why, briefly.

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

## Step 5: Synthesize the Report

After all hat agents return, combine their results into a unified report.

### Report Format

```markdown
# Review Hats Report

**Scope**: [description of what was reviewed]
**Hats**: [list of hats that ran, with color emoji circles]

---

## Critical Findings
<!-- Gather all Critical findings from all hats. Tag each with the hat that found it. -->
- 🔴 **[Security]** [Title] - `file:line`
  - What: ...
  - Why: ...

## Important Findings
<!-- Gather all Important findings from all hats. -->

## Minor Findings
<!-- Gather all Minor findings from all hats. -->

<if verbose>
## Strengths
<!-- Gather strengths from all hats -->
</if>

---

## Verdict

| Hat | Verdict |
|-----|---------|
| 🔴 Red (Security) | PASS |
| ⚪ White (Correctness) | WARN |
| ... | ... |

**Overall: [PASS | WARN | FAIL]**

[Brief synthesis: 2-3 sentences summarizing the review outcome and key action items]
```

### Verdict Emojis

| Hat | Emoji |
|-----|-------|
| Red (Security) | 🔴 |
| Indigo (Architecture) | 🟣 |
| White (Correctness) | ⚪ |
| Yellow (Performance) | 🟡 |
| Green (Maintainability) | 🟢 |
| Orange (Contracts) | 🟠 |
| Purple (Testing) | 🟣 |
| Cyan (Library) | 🔵 |

### Overall Verdict Logic
- **FAIL** if any hat returned FAIL
- **WARN** if any hat returned WARN (and none FAIL)
- **PASS** if all hats returned PASS

### Empty Sections
If a severity section has no findings across all hats, omit it entirely.

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
