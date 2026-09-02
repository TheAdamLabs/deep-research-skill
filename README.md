# deep-research-skill

A [Cursor Agent Skill](https://docs.cursor.com/agent/skills) for deep, multi-step AI research. Decomposes any research query into a parallel subagent execution plan, accumulates structured evidence with confidence ratings, and delivers the right output artifact — not a wall of text.

## What it does

- **Plans before searching** — classifies the query type and designs a dependency graph of sub-questions
- **Self-critiques the plan** — runs a pre-mortem to catch anchoring bias, missing voices, and scope drift before any searches run
- **Parallel subagents** — spawns one `generalPurpose` subagent per independent sub-question, each with its own clean context window and search budget
- **Structured evidence store** — extracts atomic claims with confidence levels (`high` / `medium` / `low` / `conflicted`), not raw text dumps
- **Right artifact for the query** — comparison tables, timelines, decision briefs, or claim maps depending on what the question calls for
- **Self-scores and patches** — evaluates output on completeness, accuracy, relevance, and artifact fit; runs targeted patch subagents if score is below threshold
- **Saves files, not chat** — writes `report.md`, `evidence.md`, and optionally a `dashboard.html` to a timestamped directory; chat output is a short summary block only

## Installation

### Personal (available in all projects)

```bash
mkdir -p ~/.cursor/skills/deep-research
curl -o ~/.cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

### Project-level (shared with collaborators)

```bash
mkdir -p .cursor/skills/deep-research
curl -o .cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

Then restart Cursor or start a new chat — the skill will appear in the catalog automatically.

## Usage

Reference the skill by name in any chat:

```
Use the deep-research skill to compare pricing models for B2B SaaS infrastructure tools
```

```
/deep-research: what is the current state of open source LLM inference frameworks?
```

```
deep-research: should I build on top of Stripe or a PSP aggregator for a marketplace product?
```

## Output

Every run writes files to a timestamped directory (e.g. `research/2026-09-02-14-30-compare-pricing-b2b/`):

| File | Description | When produced |
|---|---|---|
| `report.md` | Main structured artifact | Always |
| `evidence.md` | Raw evidence store with confidence ratings | Moderate+ queries |
| `dashboard.html` | Visual summary, open in browser | Complex multi-section queries |

Chat output is always a short block:

```
Research complete. Score: 10/12 · 31 searches · 18 sources

📁 research/2026-09-02-compare-pricing-b2b/
  report.md       — comparison table + decision brief
  evidence.md     — 47 claims across 6 sub-questions
  dashboard.html  — open in browser

Key finding: Stripe is the right default unless marketplace split payments or multi-currency are day-one requirements.
Gaps: No primary data on Stripe's enterprise pricing above $1M GMV.
Follow-up: How do Stripe Connect and Adyen Marketplaces compare on payout latency?
```

## Architecture

```
Main agent: query analysis → DAG plan → plan critique
    │
    ├── [parallel] Subagent A → evidence-A.md
    ├── [parallel] Subagent B → evidence-B.md
    └── [parallel] Subagent C → evidence-C.md
         │
         └── [depends on A] Subagent D → evidence-D.md
    │
Main agent: merge evidence → gap analysis
    │
    └── [parallel, if gaps] Gap subagent(s)
    │
Main agent: artifact → self-score (0–12)
    │
    └── [if score 7–9] Patch subagent → re-score
    │
Main agent: dashboard (if warranted) → summary to chat
```

## Requirements

- Cursor with agent skills support
- The agent needs `WebSearch` and `WebFetch` tool access (enabled by default in Cursor agent mode)
- For `dashboard.html`: any browser

## Design principles

- **Query type determines output format** — not the other way around
- **Evidence is structured, not summarized** — atomic claims with sources and confidence, so synthesis has data to work from
- **Subagents over long context** — parallel independent agents keep context windows focused; the orchestrator only sees structured evidence
- **Files over chat** — research output belongs in files you can revisit, share, and build on
- **Honest about gaps** — unresolved questions and low-confidence claims are surfaced, not buried

## License

MIT
