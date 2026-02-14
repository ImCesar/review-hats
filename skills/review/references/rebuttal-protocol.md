# Rebuttal Protocol

This document defines the debate protocol for the rebuttal phase. All participants (Developer advocate and hat agents) MUST follow this protocol.

## Debate Structure

Findings are debated one at a time, in severity order: Critical first, then Important. Minor findings are NOT debated.

For each finding:

```
1. Finding presented    → Hat restates its finding (What + Why)
2. Developer responds   → Concedes, defends with evidence, or partially defends
3. Hat rebuts           → Accepts defense, or strengthens argument with counter-evidence
4. Verdict rendered     → Lead determines outcome
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

See the developer-advocate agent definition for the three response formats: CONCEDE, DEFEND, or PARTIAL.

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

## Verdict (Lead Only)

After the exchange, the team lead renders a verdict:

| Outcome | When to Use |
|---------|-------------|
| **Upheld** | Developer conceded, or Developer defended but hat's counter was stronger |
| **Withdrawn** | Developer defended and hat accepted, or Developer's evidence is compelling despite counter |
| **Downgraded** | Developer partially defended successfully — severity reduced one level |

## Rules

- One finding at a time. Do not batch.
- Each participant gets exactly one turn per finding (present → respond → rebut → verdict).
- No back-and-forth beyond the single rebuttal. The lead's verdict is final.
- All participants must cite specific evidence (file paths, line numbers, patterns).
- Do NOT write code or suggest fixes during the debate.
