# Revised Report Format (Post-Rebuttal)

When the rebuttal phase runs, the Blue Hat produces this revised report instead of the standard report.

```markdown
# Review Hats Report (Revised)

**Scope**: [description of what was reviewed]
**Hats**: [list of hats that ran, with color emoji circles]
**Rebuttal**: Developer Advocate challenged [N] findings — [M] upheld, [X] withdrawn, [Y] downgraded

---

## Critical Findings

<!-- Only findings with original or current severity of Critical -->

- 🔴 **[Hat]** [Title] - `file:line` — **Upheld**
  - What: [description]
  - Why: [impact]
  - Defense: [Developer's argument]
  - Rebuttal: [Hat's counter-argument]

- 🔴 **[Hat]** [Title] - `file:line` — **Withdrawn**
  - What: [description]
  - ~~Why: [original impact]~~
  - Defense: [Developer's successful defense]

## Downgraded Findings

<!-- Findings whose severity was reduced -->

- **[Hat]** [Title] - `file:line` — **Downgraded** (was: Critical → Important)
  - What: [description]
  - Original Why: [original impact assessment]
  - Defense: [Developer's partial defense]

## Important Findings

<!-- Findings with original or current severity of Important -->

- **[Hat]** [Title] - `file:line` — **Upheld**
  - What: [description]
  - Why: [impact]
  - Defense: [Developer's argument, if any]
  - Rebuttal: [Hat's counter-argument, if any]

## Minor Findings

<!-- Unchanged from Phase 1 — minors are not debated -->
- **[Hat]** [Title] - `file:line`
  - What: [description]

---

## Verdict

| Hat | Original | Revised |
|-----|----------|---------|
| 🔴 Red (Security) | FAIL | WARN |
| ⚪ White (Correctness) | WARN | PASS |
| ... | ... | ... |

**Overall: [PASS | WARN | FAIL]** (was: [original verdict])

[2-3 sentence synthesis: what changed during rebuttal and why. Highlight any withdrawn or downgraded findings.]
```

## Revised Verdict Logic

After the rebuttal, recalculate each hat's verdict based on remaining findings:

- **FAIL** if any **Upheld** Critical findings remain
- **WARN** if any **Upheld** Important findings remain (and no Upheld Criticals)
- **PASS** if all Critical/Important findings were Withdrawn or Downgraded below Important

Downgraded findings count at their NEW severity level for verdict calculation.

## Rules

- Always show the original verdict alongside the revised verdict
- List Withdrawn findings with strikethrough on the Why line for transparency
- Group Downgraded findings in their own section
- Minor findings are copied verbatim from Phase 1 (no debate)
- If a severity section has no findings, omit it entirely
- Include the rebuttal summary line in the header (N challenged, M upheld, etc.)
