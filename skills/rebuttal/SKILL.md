---
name: rebuttal
description: >
  Run a rebuttal against existing review findings.
  Creates an agent team with a Developer Advocate and relevant hat agents to debate
  Critical and Important findings. Use after a review with --no-rebuttal, or to
  re-debate findings from a previous session.
---

# Blue Hat Orchestrator: Rebuttal

You are the Blue Hat running a standalone rebuttal phase. Your job is to take existing Phase 1 review findings, create an agent team, and run the debate protocol to challenge each Critical and Important finding.

## Step 1: Parse Arguments

Parse `$ARGUMENTS` to extract:

### Scope Flags (required)
The rebuttal needs the same diff the original review used, so the Developer Advocate can explore the code.

| Flag | Behavior | Git Command |
|------|----------|-------------|
| *(default)* | Unstaged diff | `git diff` |
| `--staged` | Staged changes | `git diff --staged` |
| `--files <paths>` | Specific files | `git diff -- <paths>` |
| `--commit <sha>` | Specific commit | `git show <sha>` |
| `--branch <name>` | Branch diff vs target | `git diff <name>...HEAD` |
| `--pr <url-or-number>` | PR diff | `gh pr diff <number>` |

## Step 2: Gather the Diff

Run the appropriate git command based on the parsed scope flag using the Bash tool. Capture the full diff output.

Also run `git diff --name-only` (with matching scope flags) to get the list of changed files.

If the diff is empty, inform the user: "No changes found for the specified scope." and stop.

## Step 3: Locate Phase 1 Findings

Look in the conversation history for the most recent Review Hats Report. Extract all Critical and Important findings into this structure:

~~~
Hat: [color] [domain]
Severity: [Critical | Important]
Title: [finding title]
Location: [file:line]
What: [description]
Why: [impact]
~~~

Group by severity: Criticals first, then Important.

If no Review Hats Report is found in the conversation, ask the user to provide the Phase 1 findings (paste the report or provide a file path).

If no Critical or Important findings exist, inform the user: "No findings to debate — all findings are Minor or the review was PASS." and stop.

## Step 4: Create the Rebuttal Team

### Check Agent Teams Availability

Try to create an agent team named `rebuttal`. If agent teams are unavailable:

- Inform the user: "Rebuttal requires agent teams. Enable them by adding `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` to your Claude Code settings, then try again."
- Stop.

### Spawn Teammates

Spawn these teammates in a **single message**:

**Developer Advocate:**
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

**One hat agent per hat that produced Critical or Important findings:**
~~~
You are the [Color] Hat in a rebuttal review. You produced findings during Phase 1 that are now being challenged.

## Your Phase 1 Findings
<this hat's findings extracted using the format from Step 3>

## Diff
<complete git diff output from Step 2>

## Instructions
- The Developer Advocate will respond to each of your findings
- For each response, reply with ACCEPT or COUNTER per the rebuttal protocol
- Follow the rebuttal protocol in references/rebuttal-protocol.md
- Only counter if you have specific evidence the defense is insufficient
- Use Read, Grep, and Glob tools to verify claims made in defenses
~~~

## Step 5: Run the Debate

For each debatable finding, in severity order:

1. **Message the relevant hat teammate** asking it to present the finding
2. **Message the Developer Advocate** with the hat's finding presentation
3. **Message the hat teammate** with the Developer's response
4. **Render verdict** (Upheld, Withdrawn, or Downgraded) based on the exchange

Record each verdict and the key arguments from both sides.

## Step 6: Produce the Revised Report

After all findings have been debated, produce the revised report following the format in `references/revised-report-format.md`.

Recalculate each hat's verdict based on remaining upheld findings:
- FAIL if any Upheld Critical findings
- WARN if any Upheld Important findings (no Upheld Criticals)
- PASS if all Critical/Important were Withdrawn or Downgraded below Important

## Step 7: Clean Up

After the revised report is produced (whether successfully or not):
1. Send shutdown requests to all teammates
2. Clean up the team resources (TeamDelete)

## Important Notes

- This skill runs INDEPENDENTLY of the review skill — it creates its own agent team
- Hat agents spawned here are fresh (they did not participate in Phase 1) — provide them with their findings and the diff so they can rebuild context
- Always clean up the team after completion
- If a teammate fails or returns malformed output, note it in the report but continue with others
