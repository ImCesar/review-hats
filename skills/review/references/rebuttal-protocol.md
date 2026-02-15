# Rebuttal Protocol

This document defines the debate protocol for the rebuttal phase. All participants (Developer Advocate and hat agents) MUST follow this protocol.

## Debate Structure

Findings are debated one at a time, in severity order: Critical first, then Important. Minor findings are NOT debated.

For each finding:

```
1. Developer requests   → Developer Advocate messages the hat agent to present the finding
2. Finding presented    → Hat restates its finding (What + Why)
3. Developer responds   → Concedes, defends with evidence, or partially defends
4. Hat rebuts           → Accepts defense, or strengthens argument with counter-evidence
5. Verdict rendered     → Developer Advocate determines outcome
```

## Finding Presentation (Hat)

When presenting a finding for debate, restate it concisely:

```
**Finding: [Title]** - `file:line`
- Severity: [Critical | Important]
- What: [description]
- Why: [impact]
```

## Developer Response

After a finding is presented, the Developer Advocate responds with one of three positions:

### Concede
Use when the finding is genuinely valid. Optionally suggest a lower severity.
```
**Position: CONCEDE**
This finding is valid. [1 sentence explaining why.]
- Suggested severity: [optional — e.g., Critical → Important]
```

### Defend
Use when specific evidence shows the code is intentional.
```
**Position: DEFEND**
This is intentional. [Explain the design choice with specific evidence.]
- Evidence: [cite file:line, commit message, pattern, or doc]
```

### Partially Defend
Use when the concern is real but severity is too high.
```
**Position: PARTIAL**
The concern is valid but the severity should be lower. [Explain why.]
- Evidence: [cite mitigating factors from the codebase]
- Suggested severity: [e.g., Critical → Important]
```

## Hat Rebuttal

After the Developer responds, the hat agent either:

### Accept the Defense
```
**Rebuttal: ACCEPT**
The defense is valid. [1 sentence explaining why.]
```

### Counter
```
**Rebuttal: COUNTER**
The defense is insufficient. [Explain why the finding still stands.]
- Counter-evidence: [specific evidence that the concern remains]
```

## Verdict (Developer Advocate)

After the exchange, the Developer Advocate renders a verdict:

| Outcome | When to Use |
|---------|-------------|
| **Upheld** | Developer CONCEDEd (without severity suggestion), OR hat COUNTERed with specific evidence and the finding still demonstrates real risk |
| **Withdrawn** | Hat ACCEPTed a DEFEND position, OR Developer cited specific evidence (code, docs, tests) that directly addresses the concern |
| **Downgraded** | Developer used PARTIAL (or CONCEDE with severity suggestion) and hat ACCEPTed the severity reduction — concern is real but impact is lower than claimed |

**Disambiguation:** When Developer says PARTIAL and hat says ACCEPT, the verdict is always **Downgraded** (never Withdrawn). ACCEPT on a PARTIAL confirms the severity reduction, not the absence of a concern.

**Downgrade targets:** When a finding is Downgraded, the new severity is determined by the Developer's suggestion (e.g., "Critical → Important"). If the Developer suggests a specific target, use it. If no target is suggested, downgrade by exactly one level: Critical → Important, Important → Minor.

### Verdict Guidelines

- **Evidence wins over assertion.** A position backed by file paths, line numbers, or documented patterns outweighs one that argues in the abstract.
- **Specificity wins over generality.** "This is handled at `auth.py:45`" beats "this is probably handled somewhere."
- **When in doubt, uphold.** A finding that survives debate is worth keeping. False negatives are costlier than false positives in code review.

## Rules

- One finding at a time. Do not batch.
- Each participant gets exactly one turn per finding (present → respond → rebut → verdict).
- No back-and-forth beyond the single rebuttal. The lead's verdict is final.
- All participants must cite specific evidence (file paths, line numbers, patterns).
- Do NOT write code or suggest fixes during the debate.
