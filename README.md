# deep-research-skill

A [Cursor Agent Skill](https://docs.cursor.com/agent/skills) for deep, multi-step AI research. Decomposes any query into a parallel subagent execution plan, accumulates structured evidence with confidence ratings, and delivers a browsable `dashboard.html` built with DaisyUI.

## Install

The skill is a single file. No other dependencies required.

**Cursor - personal** (all projects):

```bash
mkdir -p ~/.cursor/skills/deep-research
curl -o ~/.cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

**Cursor - project** (shared via `.cursor/skills/`):

```bash
mkdir -p .cursor/skills/deep-research
curl -o .cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

**Claude Code - personal** (all projects):

```bash
mkdir -p ~/.claude/skills/deep-research
curl -o ~/.claude/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

**Claude Code - project** (shared via `.claude/skills/`):

```bash
mkdir -p .claude/skills/deep-research
curl -o .claude/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

Start a new chat to pick up the skill. No restart needed in Claude Code (skills hot-reload).

> **Note:** The generated `dashboard.html` loads DaisyUI and Tailwind CSS from a CDN, so it requires internet access to display correctly.

> **Note:** The parallel subagent phase requires harness support for spawning subagents. In Cursor this uses the Task tool. In Claude Code it uses `claude -p` via bash. The skill body uses generic language; the agent resolves to whatever its harness provides.

## Usage

```
Use the deep-research skill to compare pricing models for B2B SaaS infrastructure tools
```

Every run produces a single `dashboard.html` in a timestamped directory under `research/`. Chat output is always a short summary block pointing to it. An intermediate `evidence.md` is written during the run as a pipeline artifact (not the primary deliverable).

## Architecture

```
Main agent: query analysis -> DAG plan -> plan critique
    |
    +-- [parallel] Subagent A -> evidence-A.md
    +-- [parallel] Subagent B -> evidence-B.md
    +-- [parallel] Subagent C -> evidence-C.md
         |
         +-- [depends on A] Subagent D -> evidence-D.md
    |
Main agent: merge evidence -> gap analysis
    |
    +-- [parallel, if gaps] Gap subagent(s)
    |
Main agent: synthesis (held in context) -> self-score (0-12)
    |
    +-- [if score 7-9] Patch subagent -> re-score
    |
Main agent: write dashboard.html (DaisyUI + Tailwind CDN) -> open -> summary to chat
```

## License

MIT
