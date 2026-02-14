---
name: review
description: >
  Run a multi-perspective code review using specialized hat agents.
  Each hat reviews from a distinct angle (security, architecture, correctness, etc.)
  and produces findings grouped by severity. Dispatches hats in parallel and synthesizes a unified report.
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

## Step 4: Dispatch Hat Agents in Parallel

Spawn all selected hat agents using the Task tool in a **single message** so they run in parallel.

Each agent's prompt MUST include:

1. The full diff output (from Step 2)
2. The list of changed files
3. Whether verbose mode is on (for the Strengths section)
4. Instructions to read surrounding code files for context using Read/Grep/Glob tools
5. A reminder to follow the standardized output format from `references/hat-output-format.md`

**Template for each agent dispatch:**

```
Review the following code changes from the perspective described in your agent definition.

## Changed Files
<list of changed files>

## Diff
<full diff output>

## Instructions
- Analyze the diff thoroughly from your specialized perspective
- Use Read, Grep, and Glob tools to examine surrounding code for context
- Follow the output format exactly as specified in references/hat-output-format.md
- Severity: Critical = must fix before merge, Important = should fix, Minor = nice to fix
- Verdict: FAIL if any Critical, WARN if any Important, PASS otherwise
<if verbose>- Include a "Strengths" section noting what's done well</if>
<if not verbose>- Do NOT include a Strengths section</if>
- Do NOT suggest fixes or write code. Report findings only.
```

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

## Step 6: Rebuttal Phase (Agent Teams)

**Skip this step if:**
- The overall verdict is PASS (nothing to debate)
- The user passed `--no-rebuttal`
- No Critical or Important findings exist

**If skipped**, present the Phase 1 report from Step 5 as the final output and stop.

### 6a: Check Agent Teams Availability

Before attempting to create an agent team, verify the feature is available. If agent teams fail to initialize, fall back gracefully:

- Present the Phase 1 report as-is
- Append a note: "Rebuttal phase skipped — enable agent teams with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings for automated finding challenges."

### 6b: Collect Debatable Findings

From the Phase 1 report, extract all Critical and Important findings. For each finding, note:
- Which hat produced it
- The severity level
- The full finding text (What + Why)

Group by severity: Criticals first, then Important.

### 6c: Create the Agent Team

Tell Claude to create an agent team for the rebuttal. The team structure:

- **Lead (you, Blue Hat)**: Judge — renders verdicts, does NOT argue
- **Developer Advocate**: One teammate using the `developer-advocate` agent
- **Hat teammates**: One teammate per hat that produced Critical or Important findings

Spawn prompt for the Developer Advocate:
```
You are the Developer Advocate in a rebuttal review. Your job is to defend the code author's design choices.

## Changed Files
<list of changed files>

## Diff
<full diff output>

## Instructions
- Follow your agent definition exactly
- You will receive findings one at a time
- For each finding, respond with CONCEDE, DEFEND, or PARTIAL
- Explore the codebase thoroughly before responding to each finding
- Follow the rebuttal protocol in references/rebuttal-protocol.md
```

Spawn prompt for each hat teammate:
```
You are the [Color] Hat in a rebuttal review. You produced findings during Phase 1 that are now being challenged.

## Your Phase 1 Findings
<this hat's findings from Phase 1>

## Instructions
- The Developer Advocate will respond to each of your findings
- For each response, reply with ACCEPT or COUNTER per the rebuttal protocol
- Follow the rebuttal protocol in references/rebuttal-protocol.md
- Only counter if you have specific evidence the defense is insufficient
```

### 6d: Run the Debate

For each debatable finding, in severity order:

1. **Message the relevant hat teammate** asking it to present the finding
2. **Message the Developer Advocate** with the hat's finding presentation
3. **Message the hat teammate** with the Developer's response
4. **Render verdict** (Upheld, Withdrawn, or Downgraded) based on the exchange

Record each verdict and the key arguments from both sides.

### 6e: Produce the Revised Report

After all findings have been debated, produce the revised report following the format in `references/revised-report-format.md`.

Recalculate each hat's verdict based on remaining upheld findings:
- FAIL if any Upheld Critical findings
- WARN if any Upheld Important findings (no Upheld Criticals)
- PASS if all Critical/Important were Withdrawn or Downgraded

### 6f: Clean Up the Team

After the revised report is produced:
1. Ask all teammates to shut down
2. Clean up the team resources

## Important Notes

- You are running inline (not forked) so you CAN use the Task tool to spawn agents
- Dispatch ALL selected hats in a single message for parallel execution (Phase 1)
- If a hat agent fails or returns malformed output, note it in the report but continue with other hats
- Keep synthesis concise — the hat reports have the details, the synthesis highlights what matters most
- Phase 2 (rebuttal) uses agent teams, NOT the Task tool — these are full Claude Code sessions
- If agent teams are unavailable or fail, fall back to the Phase 1 report gracefully
- Always clean up the agent team after the rebuttal phase completes
