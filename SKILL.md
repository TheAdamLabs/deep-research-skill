---
name: deep-research
description: Conducts deep, multi-step research on any topic by decomposing queries into a parallel execution plan, accumulating structured evidence with confidence levels, and saving output as a browsable dashboard.html in a timestamped directory. Use when the user asks to research, investigate, analyze, compare, or deep-dive a topic - especially business ideas, markets, competitors, technologies, or complex questions that require synthesizing multiple sources.
---

# Deep Research

Produces a **single `dashboard.html`** per run, saved to a timestamped directory. Chat output is always a short summary block pointing to it.

---

## Setup (before anything else)

1. Determine the output directory:
   - If inside a project: `[project root]/research/YYYY-MM-DD-HH-MM-[query-slug]/`
   - Otherwise: `~/research/YYYY-MM-DD-HH-MM-[query-slug]/`
   - `query-slug` = first 4-5 words of query, lowercased, hyphenated
2. Create the directory with `mkdir -p`
3. State the full output path before proceeding

Deliverables are decided in Phase 1 based on query complexity and type. The only constant: keep chat output to a short summary block. Never paste full research into chat.

---

## Phase 1: Query Analysis

Classify the query. Write this analysis in chat (briefly) before proceeding.

**Type:**
- `comparative`: X vs Y vs Z
- `exploratory`: state of X, how X works
- `temporal`: history/evolution of X
- `decision`: should we do X
- `factual`: specific facts, numbers
- `synthesis`: best approaches to X

**Answer shape:** what does a correct answer look like as a data structure? (table / timeline / ranked list / brief+evidence / narrative)

**Failure mode:** what would make this answer wrong or useless? (outdated sources / biased sources / missing key dimension)

**Artifact template:** which Phase 6 template applies?

**Deliverables:** decide here what files to produce, based on query complexity:

| Complexity | Typical deliverables |
|---|---|
| Simple factual / quick lookup | `dashboard.html` only (minimal: summary + findings sections) |
| Moderate | `evidence.md` (working file) + `dashboard.html` |
| Complex / multi-dimensional | `evidence.md` (working file) + `dashboard.html` with full charts |

`evidence.md` is an intermediate pipeline file (subagents write to it; you read it to populate the dashboard). It is not a user deliverable. `dashboard.html` is the single user-facing output for every run.

`dashboard.html` always earns its cost -- it is the only output file users read. For simple factual queries, generate a minimal dashboard (summary + findings only, skip charts and confidence table).

State the chosen deliverables here. This is the plan. Do not add files not listed.

---

## Phase 2: Research Plan (DAG)

Design as a dependency graph. Write to chat (briefly) before launching subagents.

```
INDEPENDENT (run in parallel):
  [A] Sub-question A
  [B] Sub-question B
  [C] Sub-question C

DEPENDS ON [A]:
  [D] Follow-up requiring A

DEPENDS ON [B] + [C]:
  [E] Synthesis question
```

Target **6-10 sub-questions**. Each must cover ground no other sub-question covers.

---

## Phase 2b: Plan Critique (Pre-Mortem)

Before executing, run a skeptic pass. Add any blind spots as new sub-questions.

- **Anchoring**: Am I only planning to confirm my initial framing? What's the opposite framing?
- **Missing voices**: Who disagrees with the mainstream view? Is a contrarian search planned?
- **Source homogeneity**: Will all planned queries return the same 5 SEO articles?
- **Scope drift**: Am I answering a slightly different (easier) question than asked?
- **Load-bearing assumption**: What does the entire plan rest on that hasn't been verified?

State findings. If framing is wrong, re-run Phase 1 and 2 before proceeding.

---

## Phase 3: Subagent Launch

Each independent sub-question from the DAG gets its own subagent running in parallel. Do not run searches yourself in this phase. Spawn subagents and let them do the work.

### 3a: Launch independent sub-questions

In a **single message**, launch one subagent per independent sub-question (use whatever subagent primitive your harness provides: Task tool in Cursor, `claude -p` via bash in Claude Code). Each subagent:
- Gets its own isolated context window
- Runs its own searches (minimum 5 web searches + 2 full-page fetches per subagent)
- Writes results to `evidence-[ID].md` in the shared output directory
- Returns a one-paragraph summary of what it found

**Subagent prompt template** (fill in the bracketed parts):

```
You are a research agent. Your job is to answer one specific sub-question as part of a larger research task.

ORIGINAL QUERY: [full original user query]
YOUR SUB-QUESTION: [sub-question text]
OUTPUT FILE: [full path to evidence-[ID].md]
OUTPUT DIRECTORY: [full output directory path]

Instructions:
1. Run at least 5 web searches and fetch 2 full pages to research your sub-question thoroughly.
   Vary your query angles:
   - Broad: "[topic] 2026"
   - Specific: "[topic] [metric] site:[authority-domain]"
   - Contrarian: "[topic] problems failure criticism"
   - Comparative: "[topic] vs [alternative]"
2. Extract atomic claims (not summaries). Each claim is one sentence.
3. Write ALL findings to [full path to evidence-[ID].md] using this format:

## [Sub-question text]

CLAIM: [one-sentence fact]
SOURCE: [URL]
FRESHNESS: [date or "unknown"]
CONFIDENCE: high | medium | low | conflicted
NOTE: [only if conflicted]

CLAIM: ...

4. After writing the file, return a short paragraph summarizing:
   - The 2-3 most important findings
   - The overall confidence level
   - Any significant gaps or conflicts you found
```

### 3b: Launch dependent sub-questions

After independent subagents complete, read their `evidence-[ID].md` files, then launch subagents for dependent sub-questions, passing relevant findings from dependencies as context in the prompt.

Use the same subagent prompt template, adding:
```
CONTEXT FROM PRIOR RESEARCH:
[paste relevant claims from dependency evidence files]
```

### 3c: Merge evidence

After all subagents complete, merge all `evidence-[ID].md` files into `evidence.md`:

```bash
cat [output-dir]/evidence-*.md > [output-dir]/evidence.md
```

**Minimum search budget across all subagents combined: 25 web searches + 10 full-page fetches.** If the total falls short (check by counting searches in evidence files), run additional searches yourself to fill the gap before moving to Phase 4.

---

## Phase 4: Evidence Store

Append to `evidence.md` after each search batch:

```
## [Sub-question ID]: [Sub-question text]

CLAIM: [one-sentence atomic fact]
SOURCE: [full URL; never a domain name or title alone]
FRESHNESS: [date or "unknown"]
CONFIDENCE: high | medium | low | conflicted
NOTE: [only if conflicted - what each source says]

CLAIM: ...
```

**SOURCE field rules - no exceptions:**
- Always the full URL (`https://...`). Never a bare domain, publication name, or "source unavailable".
- If a claim comes from a paywalled or unresolvable page, record the URL anyway and mark FRESHNESS "paywalled".
- One SOURCE per CLAIM. If two sources back the same claim, record the claim twice with different SOURCE lines.

**Confidence rules:**
- `high`: 2+ independent primary sources agree
- `medium`: 1 credible primary source, or multiple secondary sources
- `low`: single secondary source, or inference
- `conflicted`: sources disagree; document both positions

---

## Phase 5: Gap Analysis

After merging all evidence, run gap check against `evidence.md`:

- Sub-questions with only `low`/`conflicted` evidence: spawn a targeted subagent with a tighter, more specific prompt
- New sub-questions that emerged during research: add to plan and spawn a subagent
- Load-bearing claims with weak sourcing: spawn a subagent specifically tasked with finding a primary source or disconfirming evidence
- Same sources appearing repeatedly: stop, that search space is saturated

For each gap requiring more research, spawn one subagent per gap (in parallel if multiple gaps). Merge results back into `evidence.md`. One gap-fill round maximum.

---

## Phase 6: Synthesis

Synthesize your findings into a structured artifact. **Do not write to a file yet** -- the synthesis will be written directly into `dashboard.html` `#findings` in Phase 8. Hold it in context.

**Do not write prose summaries. Use the template for the query type from Phase 1.**

**Citation rule for every template below:** every factual claim must carry an inline source link using the format `[[domain.com]](https://full-url)`. No claim without a source. In tables, add a source column or superscript footnotes rather than omitting citations.

### Comparative -> Table

```markdown
# [Query Title]
*Research date: [date] | Sources: N | Confidence: overall rating*

| Dimension | Option A | Option B | Option C |
|-----------|----------|----------|----------|
| [dim]     | [value] [[src]](url) | [value] [[src]](url) | [value] [[src]](url) |

*† = low-confidence source*

## Conflicts & gaps
[What sources disagreed on, what couldn't be found (each conflict cites both URLs)]
```

**Visual supplements (comparative):** `#charts` -- horizontal comparison bars (`.hbar-group`, one per dimension; `data-val="0-100"`) + heat-map matrix (`.heatmap`, for 3+ options / 4+ dims). `#findings` -- mermaid `flowchart LR` decision tree as `<pre class="mermaid">`. See `dashboard-template.html` `#charts` comments for HTML examples.

### Temporal -> Timeline

```markdown
# [Query Title]

| Year | Event | Source | Confidence |
|------|-------|--------|------------|
| 2020 | [event] | [[domain.com]](https://url) | high |

## Key turning points
[2-3 sentences on inflection points, each ending with [[domain.com]](url)]
```

**Visual supplements (temporal):** `#charts` -- SVG timeline (`.timeline-svg`, viewBox 900x160); x = 40 + ((year-min)/(max-min)) * 820; circle r: high=8, med=6, low=4; conflicted adds `stroke-dasharray="4 2"`; each circle is an `<a>` to its source. Alternate label y: above=58, below=108. `#findings` -- mermaid `timeline` diagram as `<pre class="mermaid">`. See template for SVG skeleton.

### Decision -> Brief + Evidence

```markdown
# [Query Title]

## Recommendation
[One direct sentence: yes / no / depends on X] [[primary-source.com]](url)

## Why
- [Reason 1] [[source.com]](url)
- [Reason 2] [[source.com]](url)
- [Reason 3] [[source.com]](url)

## Risks / unknowns
- [What could invalidate this] [[source.com]](url) *(confidence: low)*

## Full evidence
[Organized by sub-question; every claim has [[domain]](url)]
```

**Visual supplements (decision):** `#charts` -- vertical bar chart (`.bar-chart`); score each option 0-10 from evidence weight; `data-val` = score * 10; color by tier (>=7 `var(--high)`, 4-6 `var(--med)`, <4 `var(--low)`). Tallest bar = recommended option. No extra diagram in `#findings` needed.

### Exploratory / Synthesis -> Claim Map

```markdown
# [Query Title]

## High-confidence findings
- [Claim] [[source.com]](url)

## Contested / unclear
- [Claim]: [[Source A]](urlA) says X / [[Source B]](urlB) says Y

## Couldn't find
- [What was searched for but not found]

## Summary
[Narrative only where structure doesn't fit; keep under 200 words, inline citations throughout]
```

**Visual supplements (exploratory/synthesis):** `#charts` -- concept cluster SVG (`.concept-svg`, viewBox 700x400); center node at (350,200) r=40; 5-8 satellites at r=140 evenly-spaced angles; spoke lines from center; satellite border color = confidence tier. `#findings` -- mermaid `mindmap` as `<pre class="mermaid">`.

**All query types:** `#summary` confidence donut -- count `CONFIDENCE: high/medium/low/conflicted` lines in `evidence.md`, set `data-high/med/low/conf` on `#confidence-donut`. The JS renders the `conic-gradient` automatically.

---

## Phase 7: Output Evaluation + Patch Loop

Score your Phase 6 synthesis (held in context) against the original query:

```
COMPLETENESS   [0-3]  covers all meaningful dimensions?
ACCURACY       [0-3]  load-bearing claims have high-confidence sources?
RELEVANCE      [0-3]  answers the actual question, not an easier adjacent one?
ARTIFACT FIT   [0-3]  right format for the query type?
TOTAL: /12
```

- **10-12**: proceed to Phase 8 (dashboard)
- **7-9**: patch: spawn one subagent targeting the lowest-scoring dimension, update the relevant section of your synthesis in context, re-score once
- **0-6**: restart Phase 1 (framing was wrong, patching won't fix it)

Max 2 patch rounds. After 2, deliver with explicit notes on what remains weak.

Patch subagent prompt adds:
```
PATCH CONTEXT: This is a targeted patch for a deep research run.
WEAK DIMENSION: [completeness / accuracy / relevance / artifact fit]
GAP: [specific description of what's missing]
EXISTING FINDINGS: [paste relevant section of evidence.md]
YOUR TASK: Find sources that specifically address this gap. Return findings in evidence format.
```

---

## Phase 8: Dashboard

**Do not generate the HTML structure from scratch.** Read the template file first, then write one complete filled-in `dashboard.html`.

### Steps

1. **Read the template:**
```bash
cat [skill-directory]/dashboard-template.html
```
The skill directory is the directory containing this SKILL.md file (e.g. `/Users/you/.cursor/skills/deep-research/`).

2. **Write the complete dashboard** to `[output-dir]/dashboard.html` using the template as your structural base. Populate every placeholder in a single `Write` call -- do not use StrReplace on placeholders. You have full freedom to adapt the structure for your query type.

**What to populate in each section:**

| Section | Content source |
|---|---|
| `<title>` and `<h1>` | Query title |
| `.nav-meta`, `.meta` | Date, source count, score |
| `#summary` score cards + donut | Phase 7 scores; count `CONFIDENCE: high/medium/low/conflicted` lines in `evidence.md` |
| `#charts` | Phase 6 visual supplements for your query type (see instructions above) |
| `#findings` | Your Phase 6 synthesis -- the full structured artifact (table / timeline / brief / claim map) |
| `#confidence` tbody | One `<tr>` per key claim from `evidence.md` (curate to most important ~20-40 for long runs) |
| `#sources` | One `<li>` per unique URL from `evidence.md`, ordered by confidence then freshness |
| `#gaps` | Gaps and follow-up questions from your synthesis |

**Source link rule:** every URL from `evidence.md` must appear as a real `<a href="..." target="_blank" rel="noopener">` link. Never render a URL as plain text.

3. **Open the dashboard:**
```bash
open [output-dir]/dashboard.html   # macOS
```

<!-- removed: template HTML lives in dashboard-template.html -->

---

## Output Rules

Final chat message: short, always.

```
Research complete. Score: [X]/12 · [N] searches · [N] sources
Charts: [list chart types included, e.g. "donut, comparison bars" -- or "none" for simple queries]

📁 [output directory path]
  dashboard.html: [one-phrase description of the artifact]

Key finding: [one sentence]
Gaps: [one sentence about what's still unknown]
Follow-up: [1-2 questions worth pursuing]
```

- Never paste research content into chat
- `dashboard.html` is always produced; `evidence.md` is a working file, do not mention it in the output summary
- If the answer is genuinely unresolvable, say so directly in key finding
- **Never use em dashes (—) in any output file or chat message.** Replace with a colon, comma, semicolon, period, or parentheses as appropriate.

---

## Defaults

- **Minimum searches**: 25 web searches + 5 full-page fetches per run
- **Sub-questions**: 6-10
- **Patch rounds**: max 2
- **Stop signal**: same sources reappearing, or all evidence at `high`/`medium`
- **Search tool**: web search for discovery, full-page fetch for in-depth reading
