# deep-research-skill

A [Cursor Agent Skill](https://docs.cursor.com/agent/skills) for deep, multi-step AI research. Decomposes any query into a parallel subagent execution plan, accumulates structured evidence with confidence ratings, and delivers the right artifact type rather than a wall of text.

## Install

**Personal** (available in all projects):

```bash
mkdir -p ~/.cursor/skills/deep-research
curl -o ~/.cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

**Project-level** (shared with collaborators via `.cursor/skills/`):

```bash
mkdir -p .cursor/skills/deep-research
curl -o .cursor/skills/deep-research/SKILL.md \
  https://raw.githubusercontent.com/TheAdamLabs/deep-research-skill/main/SKILL.md
```

Start a new chat to pick up the skill.

## Usage

```
Use the deep-research skill to compare pricing models for B2B SaaS infrastructure tools
```

The skill decides what files to produce based on query complexity. Results are saved to a timestamped directory under `research/`; chat output is always a short summary block pointing to the files.

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
Main agent: artifact -> self-score (0-12)
    |
    +-- [if score 7-9] Patch subagent -> re-score
    |
Main agent: dashboard (if warranted) -> summary to chat
```

## License

MIT
