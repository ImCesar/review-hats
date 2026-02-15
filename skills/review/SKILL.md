---
name: review
description: >
  Run a multi-perspective code review using specialized hat agents.
  Each hat reviews from a distinct angle (security, architecture, correctness, etc.)
  and produces findings grouped by severity. Uses agent teams to keep hat agents alive
  for rebuttal debate when findings are challenged.
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

## Step 4: Create Review Team and Dispatch Hats

### Check Agent Teams Availability

Try to create an agent team named `review-hats`. If agent teams are unavailable (feature not enabled), fall back to the Task tool:

- Spawn hat agents using the Task tool in a **single message** (fire-and-forget)
- Note internally that rebuttal will be unavailable (Phase 2 requires agent teams)
- If the overall verdict is WARN or FAIL, append to the report: "Rebuttal phase skipped — enable agent teams with `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings for automated finding challenges."

### Agent Team Dispatch (preferred)

If agent teams are available, spawn all selected hat agents as teammates in a **single message** so they run in parallel.

Each agent's spawn prompt MUST include:

1. The full diff output (from Step 2)
2. The list of changed files
3. Whether verbose mode is on (for the Strengths section)
4. Instructions to read surrounding code files for context using Read/Grep/Glob tools
5. A reminder to follow the standardized output format from `references/hat-output-format.md`
6. Instructions to send findings to the lead via SendMessage and then wait

**Spawn prompt for each hat teammate:**

~~~
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
- After completing your review, send your full report to the team lead.
- Then wait — you may be recalled for rebuttal if your findings are challenged.
~~~

### Collect Results

Wait for all hat agents to send their findings via messages. As each hat goes idle after sending its report, its findings are available. Proceed to Step 5 once all hats have reported.

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
- Phase 1 fell back to Task tool (agent teams unavailable)

**If skipped**, present the Phase 1 report from Step 5 as the final output, clean up the team (if it exists), and stop.

### Collect Debatable Findings

From the Phase 1 report, extract all Critical and Important findings into this structure:

~~~
Hat: [color] [domain]
Severity: [Critical | Important]
Title: [finding title]
Location: [file:line]
What: [description]
Why: [impact]
~~~

Group by severity: Criticals first, then Important. This structure is the contract between Phase 1 (Step 5) and Phase 2 (Step 6) — the debate loop expects each finding in this format.

### Spawn Developer Advocate

Add the Developer Advocate to the existing review team as a new teammate:

~~~
You are the Developer Advocate in a rebuttal review. Your job is to defend the code author's design choices.

## Changed Files
<one file path per line>

## Diff
<complete git diff output from Step 2>

## Instructions
- Follow your agent definition exactly
- You will receive findings one at a time via messages from the team lead
- For each finding, respond with CONCEDE, DEFEND, or PARTIAL
- Explore the codebase thoroughly before responding to each finding
- Follow the rebuttal protocol in references/rebuttal-protocol.md
~~~

### Run the Debate

For each debatable finding, in severity order:

1. **Message the relevant hat teammate** asking it to present the finding
2. **Message the Developer Advocate** with the hat's finding presentation
3. **Message the hat teammate** with the Developer's response
4. **Render verdict** (Upheld, Withdrawn, or Downgraded) based on the exchange

Record each verdict and the key arguments from both sides. The hat agents retain their full Phase 1 reasoning context, so their rebuttals draw on the analysis they already performed.

### Produce the Revised Report

After all findings have been debated, produce the revised report following the format in `references/revised-report-format.md`.

Recalculate each hat's verdict based on remaining upheld findings:
- FAIL if any Upheld Critical findings
- WARN if any Upheld Important findings (no Upheld Criticals)
- PASS if all Critical/Important were Withdrawn or Downgraded below Important

### Clean Up the Team

After the rebuttal phase completes (whether report generation succeeds or fails):
1. Send shutdown requests to all teammates
2. Clean up the team resources (TeamDelete)

## Important Notes

- Dispatch ALL selected hats in a single message for parallel execution (Phase 1)
- Hat agents stay alive between Phase 1 and Phase 2 — do NOT shut them down before rebuttal
- If a hat agent fails or returns malformed output, note it in the report but continue with other hats
- Keep synthesis concise — the hat reports have the details, the synthesis highlights what matters most
- If agent teams are unavailable, fall back to Task tool for Phase 1 and skip rebuttal gracefully
- Always clean up the team after completing the review (whether Phase 1 only or after rebuttal)
