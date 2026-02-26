# Rebuttal Phase Orchestration

When the Phase 1 verdict is WARN or FAIL, the Blue Hat follows this document to run the rebuttal phase using Task tool subagents in three passes.

## Collect Debatable Findings

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

## Pass 1: Developer Advocate Evaluation

Spawn the Developer Advocate as a **single Task tool subagent**. It evaluates all debatable findings by exploring the codebase and forming positions.

~~~
Task tool (review-hats:developer-advocate):
  description: "Developer Advocate rebuttal evaluation"
  model: sonnet
  prompt: |
    You are the Developer Advocate evaluating findings from a code review.

    ## Changed Files
    <one file path per line>

    ## Diff
    <complete git diff output>

    ## Debatable Findings
    <all findings from the "Collect Debatable Findings" step above, in severity order>

    ## Instructions
    - Follow your agent definition exactly
    - For EACH finding: gather evidence from the codebase, then form your position
    - Process findings in order (Critical first, then Important)
    - Return your positions in the summary format defined in your agent definition
~~~

When the DA subagent returns, you have positions for all findings.

## Pass 2: Hat Agent Responses

For each hat that produced debatable findings, spawn a **Task tool subagent** to respond to the DA's positions. Dispatch all hats in a **single message** for parallel execution.

~~~
Task tool (review-hats:[agent-name]):
  description: "[Color] Hat rebuttal response"
  model: sonnet
  prompt: |
    You are the [Color] Hat responding to the Developer Advocate's challenge of your Phase 1 findings.

    ## Your Phase 1 Findings
    <this hat's Critical and Important findings>

    ## Developer Advocate Positions
    <DA's positions for this hat's findings only>

    ## Diff
    <complete git diff output>

    ## Instructions
    - Read `references/rebuttal-protocol.md` for the response format
    - For each finding, review the DA's position and cited evidence
    - Use Read, Grep, and Glob tools to verify claims made in defenses
    - Respond with ACCEPT or COUNTER for each finding
    - Only COUNTER if you have specific evidence the defense is insufficient
    - Return your responses in this exact format:

    ## Rebuttal Responses

    ### [ID]: [Title]
    - Response: ACCEPT | COUNTER
    - Reason: [1-2 sentences]
    - Counter-evidence: [if COUNTER — file:line, specific evidence]

    [repeat for each finding]
~~~

When all hat subagents return, you have responses for all findings.

## Pass 3: Render Verdicts

The Blue Hat (you) renders verdicts by applying the verdict table from `references/rebuttal-protocol.md` to each finding:

| DA Position | Hat Response | Verdict |
|-------------|-------------|---------|
| CONCEDE (no severity suggestion) | N/A | **Upheld** |
| CONCEDE (with severity suggestion) | ACCEPT | **Downgraded** |
| DEFEND | ACCEPT | **Withdrawn** |
| DEFEND | COUNTER (with evidence) | **Upheld** |
| PARTIAL | ACCEPT | **Downgraded** |
| PARTIAL | COUNTER (with evidence) | **Upheld** |

**Disambiguation:** When DA says PARTIAL and hat says ACCEPT, the verdict is always **Downgraded**. ACCEPT on a PARTIAL confirms severity reduction, not absence of concern.

**Downgrade targets:** Use the DA's suggested severity. If none suggested, downgrade one level: Critical -> Important, Important -> Minor.

## Produce the Revised Report

Using the rendered verdicts, produce the revised report following the format in `references/revised-report-format.md`.

Recalculate each hat's verdict based on remaining upheld findings:
- FAIL if any Upheld Critical findings
- WARN if any Upheld Important findings (no Upheld Criticals)
- PASS if all Critical/Important were Withdrawn or Downgraded below Important
