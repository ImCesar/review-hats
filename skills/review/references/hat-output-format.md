# Hat Output Format

All hat agents MUST produce output in this exact format. Do not deviate.

```markdown
## [Color] Hat: [Domain] Review

### Findings

#### Critical
- **[Title]** - `file.ts:42`
  - What: [description of the issue]
  - Why: [impact, risk, or consequence if not addressed]

#### Important
- **[Title]** - `file.ts:15`
  - What: [description of the issue]
  - Why: [impact, risk, or consequence if not addressed]

#### Minor
- **[Title]** - `file.ts:88`
  - What: [description of the issue]

### Strengths
<!-- Only include this section if verbose mode was requested -->
- [What's done well, with file:line references where applicable]

### Verdict: [PASS | WARN | FAIL]
[1-2 sentence summary from this hat's perspective]
```

## Severity Guidelines

- **Critical**: Will cause bugs, security vulnerabilities, data loss, or outages. Must fix before merge.
- **Important**: Significant code quality, design, or correctness concerns. Should fix before merge.
- **Minor**: Style, naming, small improvements. Nice to fix but not blocking.

## Verdict Guidelines

- **PASS**: No Critical or Important findings.
- **WARN**: No Critical findings, but has Important findings.
- **FAIL**: Has one or more Critical findings.

## Rules

- If a severity section has no findings, omit it entirely (don't print empty sections).
- Always include file paths with line numbers in findings.
- Be specific and actionable. "Could be improved" is not a finding.
- Do NOT suggest fixes or write code. Report findings only.
- Reference the specific diff lines where you found the issue.

## Rebuttal Phase

When the review triggers a rebuttal (WARN or FAIL verdict), hat agents that produced Critical or Important findings may be recalled as teammates in an agent team to debate their findings with the Developer Advocate.

During rebuttal, respond to each defense with one of:

```
**Rebuttal: ACCEPT**
The defense is valid. [1 sentence explaining why.]
```

```
**Rebuttal: COUNTER**
The defense is insufficient. [Explain why the finding still stands.]
- Counter-evidence: [specific evidence that the concern remains]
```

Only COUNTER if you have specific evidence the defense is insufficient. See `references/rebuttal-protocol.md` for the full debate protocol.
