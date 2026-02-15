# Review Hats

A Claude Code plugin that runs multi-perspective code reviews using specialized AI agents inspired by de Bono's Six Thinking Hats. Each "hat" reviews your code from a distinct angle — security, architecture, correctness, performance, maintainability, contract safety, testing, and library usage — then produces a unified report.

## How It Works

A Blue Hat orchestrator (the skill) analyzes your diff, selects the relevant hats, dispatches them **in parallel** using tiered model selection (Sonnet for depth-critical reviews, Haiku for pattern-matching), and synthesizes their findings into a single report grouped by severity. When findings are challenged, a Developer Advocate coordinates the rebuttal debate via direct messages with hat agents, then reports verdicts back to the orchestrator.

### The 8 Review Hats

| Hat | Code | Focus |
|-----|------|-------|
| 🔴 Red | `r` | Security — vulnerabilities, injection, auth flaws, secrets |
| 🟣 Indigo | `i` | Architecture — patterns, coupling, separation of concerns |
| ⚪ White | `w` | Correctness — logic bugs, edge cases, error handling |
| 🟡 Yellow | `y` | Performance — complexity, N+1 queries, memory, bottlenecks |
| 🟢 Green | `g` | Maintainability — readability, naming, conventions |
| 🟠 Orange | `o` | Contract Safety — breaking changes, API surface, schemas |
| 🟣 Purple | `p` | Testing — coverage gaps, test quality, missing cases |
| 🔵 Cyan | `c` | Library/Framework — idiomatic usage, anti-patterns, deprecated APIs |

## Installation

### From the Marketplace

```
/plugin marketplace add ImCesar/review-hats
/plugin install review-hats@review-hats-marketplace
```

### Local Development

```bash
claude --plugin-dir /path/to/review-hats
```

## Usage

```
/review-hats:review                     # Auto-select hats, review unstaged diff
/review-hats:review rwp                 # Only Red, White, Purple hats
/review-hats:review --staged            # Review staged changes
/review-hats:review --branch main       # Diff current branch vs main
/review-hats:review --commit abc123     # Review a specific commit
/review-hats:review --pr 42             # Review a pull request
/review-hats:review --files src/foo.ts  # Review specific files
/review-hats:review --verbose           # Include strengths, not just findings
/review-hats:review rw --staged         # Combine hat codes with flags
/review-hats:review --no-rebuttal     # Skip rebuttal phase on WARN/FAIL
```

### Hat Selection

**Auto mode** (no hat codes): The orchestrator reads the diff and picks the relevant hats. Correctness (W) and Maintainability (G) are always included.

**Manual mode**: Pass one-letter codes to select specific hats. `rwp` runs Red, White, and Purple.

### Scope Flags

| Flag | What It Reviews |
|------|-----------------|
| *(default)* | Unstaged changes (`git diff`) |
| `--staged` | Staged changes (`git diff --staged`) |
| `--files <paths>` | Specific files (`git diff -- <paths>`) |
| `--commit <sha>` | A specific commit (`git show <sha>`) |
| `--branch <name>` | Current branch vs target (`git diff <name>...HEAD`) |
| `--pr <number>` | A pull request (`gh pr diff <number>`) |

### Output Modes

| Flag | Behavior |
|------|----------|
| *(default)* | Findings only — actionable items grouped by severity |
| `--verbose` | Full report with strengths from each hat |
| `--no-rebuttal` | Skip rebuttal phase (see [Rebuttal Phase](#rebuttal-phase)) |

### Rebuttal Phase

When a review produces a **WARN** or **FAIL** verdict, review-hats automatically runs a rebuttal phase using Claude Code agent teams. A **Developer Advocate** explores the codebase to understand design intent, then debates each Critical and Important finding with the hat that produced it.

Each finding gets a verdict:
- **Upheld** — Finding stands after debate
- **Withdrawn** — Finding dismissed; Developer showed it was intentional
- **Downgraded** — Finding valid but severity reduced

The revised report shows both original and final verdicts.

To skip the rebuttal phase:
```
/review-hats:review --no-rebuttal
```

> **Note**: Rebuttal requires the experimental agent teams feature. Enable it by adding `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` to your settings. If not enabled, the Phase 1 report is presented as-is.

### Severity Levels

- **Critical** — Must fix before merge. Bugs, vulnerabilities, data loss risks.
- **Important** — Should fix before merge. Significant quality or design concerns.
- **Minor** — Nice to fix. Style, small improvements, low-risk issues.

### Verdicts

Each hat produces a verdict. The overall verdict is the worst across all hats:

- **PASS** — No Critical or Important findings
- **WARN** — Important findings but no Critical
- **FAIL** — One or more Critical findings

## Project Structure

```
review-hats/
├── .claude-plugin/
│   ├── plugin.json                    # Plugin manifest
│   └── marketplace.json               # Marketplace catalog
├── skills/
│   ├── review/
│   │   ├── SKILL.md                   # Blue Hat orchestrator
│   │   └── references/
│   │       ├── hat-output-format.md   # Shared output format
│   │       ├── rebuttal-protocol.md   # Debate protocol for rebuttal
│   │       └── revised-report-format.md # Post-rebuttal report format
├── agents/
│   ├── developer-advocate.md          # Debate coordinator for rebuttal
│   ├── red-hat-security.md
│   ├── indigo-hat-architecture.md
│   ├── white-hat-correctness.md
│   ├── yellow-hat-performance.md
│   ├── green-hat-maintainability.md
│   ├── orange-hat-contracts.md
│   ├── purple-hat-testing.md
│   └── cyan-hat-library.md
├── commands/
│   └── review.md                      # Thin command alias
└── README.md
```

## License

MIT
