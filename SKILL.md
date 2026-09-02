---
name: deep-research
description: Conducts deep, multi-step research on any topic by decomposing queries into a parallel execution plan, accumulating structured evidence with confidence levels, and saving output as files (dashboard.html, report.md, evidence.md) in a timestamped directory. Use when the user asks to research, investigate, analyze, compare, or deep-dive a topic - especially business ideas, markets, competitors, technologies, or complex questions that require synthesizing multiple sources.
---

# Deep Research

Produces **saved files**, not chat text. Every run creates a timestamped directory with a browsable dashboard, structured report, and raw evidence.

---

## Setup (before anything else)

1. Determine the output directory:
   - If inside a project: `[project root]/research/YYYY-MM-DD-HH-MM-[query-slug]/`
   - Otherwise: `~/research/YYYY-MM-DD-HH-MM-[query-slug]/`
   - `query-slug` = first 4-5 words of query, lowercased, hyphenated
2. Create the directory with `mkdir -p`
3. State the full output path before proceeding

Deliverables are decided in Phase 1 based on query complexity and type. The only constant: keep chat output to a short summary block — never paste full research into chat.

---

## Phase 1: Query Analysis

Classify the query. Write this to the top of `report.md` immediately.

**Type:**
- `comparative` — X vs Y vs Z
- `exploratory` — state of X, how X works
- `temporal` — history/evolution of X
- `decision` — should we do X
- `factual` — specific facts, numbers
- `synthesis` — best approaches to X

**Answer shape:** what does a correct answer look like as a data structure? (table / timeline / ranked list / brief+evidence / narrative)

**Failure mode:** what would make this answer wrong or useless? (outdated sources / biased sources / missing key dimension)

**Artifact template:** which Phase 6 template applies?

**Deliverables:** decide here what files to produce, based on query complexity:

| Complexity | Typical deliverables |
|---|---|
| Simple factual / quick lookup | `report.md` only — a short structured answer |
| Moderate (1 clear artifact) | `report.md` + `evidence.md` |
| Complex / multi-dimensional | `report.md` + `evidence.md` + `dashboard.html` |

`dashboard.html` only earns its cost when the artifact has multiple sections a user will want to navigate (comparison tables, timelines with many entries, multi-section synthesis). Don't generate it for a focused factual question.

State the chosen deliverables here. This is the plan — do not add files not listed.

---

## Phase 2: Research Plan (DAG)

Design as a dependency graph. Write to `report.md`.

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

Each independent sub-question from the DAG gets its own subagent running in parallel. Do not run searches yourself in this phase — spawn subagents and let them do the work.

### 3a: Launch independent sub-questions

In a **single message**, launch one `generalPurpose` subagent per independent sub-question. Each subagent:
- Gets its own isolated context window
- Runs its own searches (minimum 5 WebSearch + 2 WebFetch per subagent)
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
1. Run at least 5 WebSearch calls and 2 WebFetch calls to research your sub-question thoroughly.
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

After independent subagents complete, read their `evidence-[ID].md` files, then launch subagents for dependent sub-questions — passing relevant findings from dependencies as context in the prompt.

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

**Minimum search budget across all subagents combined: 25 WebSearch + 10 WebFetch.** If the total falls short (check by counting searches in evidence files), run additional searches yourself to fill the gap before moving to Phase 4.

---

## Phase 4: Evidence Store

Append to `evidence.md` after each search batch:

```
## [Sub-question ID]: [Sub-question text]

CLAIM: [one-sentence atomic fact]
SOURCE: [URL]
FRESHNESS: [date or "unknown"]
CONFIDENCE: high | medium | low | conflicted
NOTE: [only if conflicted - what each source says]

CLAIM: ...
```

**Confidence rules:**
- `high` — 2+ independent primary sources agree
- `medium` — 1 credible primary source, or multiple secondary sources
- `low` — single secondary source, or inference
- `conflicted` — sources disagree; document both positions

---

## Phase 5: Gap Analysis

After merging all evidence, run gap check against `evidence.md`:

- Sub-questions with only `low`/`conflicted` evidence → spawn a targeted subagent with a tighter, more specific prompt
- New sub-questions that emerged during research → add to plan and spawn a subagent
- Load-bearing claims with weak sourcing → spawn a subagent specifically tasked with finding a primary source or disconfirming evidence
- Same sources appearing repeatedly → stop, that search space is saturated

For each gap requiring more research, spawn one subagent per gap (in parallel if multiple gaps). Merge results back into `evidence.md`. One gap-fill round maximum.

---

## Phase 6: Output Artifact

Write the full artifact to `report.md`. **Do not write prose summaries** — use the template for the query type from Phase 1.

### Comparative → Table

```markdown
# [Query Title]
*Research date: [date] | Sources: N | Confidence: overall rating*

| Dimension | Option A | Option B | Option C |
|-----------|----------|----------|----------|
| ...       | ...†     | ...      | ...      |

† low-confidence source

## Conflicts & gaps
[What sources disagreed on, what couldn't be found]
```

### Temporal → Timeline

```markdown
# [Query Title]

| Year | Event | Source | Confidence |
|------|-------|--------|------------|
| 2020 | ...   | [url]  | high       |

## Key turning points
[2-3 sentences on inflection points]
```

### Decision → Brief + Evidence

```markdown
# [Query Title]

## Recommendation
[One direct sentence: yes / no / depends on X]

## Why
- [Reason 1] — [source]
- [Reason 2] — [source]
- [Reason 3] — [source]

## Risks / unknowns
- [What could invalidate this] — [confidence: low]

## Full evidence
[Organized by sub-question]
```

### Exploratory / Synthesis → Claim Map

```markdown
# [Query Title]

## High-confidence findings
- [Claim] — [source]

## Contested / unclear
- [Claim]: [Source A says X] / [Source B says Y]

## Couldn't find
- [What was searched for but not found]

## Summary
[Narrative only where structure doesn't fit — keep under 200 words]
```

### Always append

```markdown
---
**Sources consulted:** N  
**Primary sources:** X  
**Most recent data:** [date]  
**Gaps:** [What this didn't cover and why]  
**Follow-up questions:** 
- [Question 1]
- [Question 2]
- [Question 3]
```

---

## Phase 7: Output Evaluation + Patch Loop

Score the `report.md` against the original query:

```
COMPLETENESS   [0-3]  covers all meaningful dimensions?
ACCURACY       [0-3]  load-bearing claims have high-confidence sources?
RELEVANCE      [0-3]  answers the actual question, not an easier adjacent one?
ARTIFACT FIT   [0-3]  right format for the query type?
TOTAL: /12
```

- **10-12**: proceed to dashboard
- **7-9**: patch — spawn one subagent targeting the lowest-scoring dimension, re-synthesize only that section, re-score once
- **0-6**: restart Phase 1 — framing was wrong, patching won't fix it

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

Generate `dashboard.html` — a self-contained single file (inline CSS, no external dependencies).

Structure:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>[Query]</title>
  <style>
    /* clean, readable styles — dark sidebar, white content area */
    body { font-family: system-ui; margin: 0; display: flex; }
    nav { width: 220px; background: #1a1a2e; color: #eee; padding: 24px 16px; min-height: 100vh; }
    nav a { display: block; color: #aaa; text-decoration: none; padding: 6px 0; font-size: 13px; }
    nav a:hover { color: white; }
    main { flex: 1; padding: 40px; max-width: 900px; }
    .score-card { display: flex; gap: 16px; margin: 24px 0; }
    .score { background: #f5f5f5; border-radius: 8px; padding: 16px; text-align: center; flex: 1; }
    .score .num { font-size: 32px; font-weight: bold; }
    .high { color: #16a34a; } .medium { color: #d97706; } .low { color: #dc2626; }
    table { width: 100%; border-collapse: collapse; margin: 16px 0; }
    th { background: #f5f5f5; padding: 10px; text-align: left; }
    td { padding: 10px; border-bottom: 1px solid #eee; vertical-align: top; }
    .tag { display: inline-block; padding: 2px 8px; border-radius: 4px; font-size: 12px; }
    .tag-high { background: #dcfce7; color: #15803d; }
    .tag-med  { background: #fef9c3; color: #854d0e; }
    .tag-low  { background: #fee2e2; color: #b91c1c; }
    .tag-conf { background: #ede9fe; color: #6d28d9; }
  </style>
</head>
<body>
  <nav>
    <div style="font-weight:bold;margin-bottom:24px;color:white">Deep Research</div>
    <a href="#summary">Summary</a>
    <a href="#artifact">Findings</a>
    <a href="#confidence">Confidence</a>
    <a href="#sources">Sources</a>
    <a href="#gaps">Gaps</a>
  </nav>
  <main>
    <h1>[Query]</h1>
    <p style="color:#666">[Date] · [N] sources · [type] query</p>

    <section id="summary">
      <h2>Summary</h2>
      <!-- score card -->
      <div class="score-card">
        <div class="score"><div class="num">[N]/3</div>Completeness</div>
        <div class="score"><div class="num">[N]/3</div>Accuracy</div>
        <div class="score"><div class="num">[N]/3</div>Relevance</div>
        <div class="score"><div class="num">[N]/3</div>Artifact fit</div>
      </div>
      <!-- key takeaways as bullets -->
    </section>

    <section id="artifact">
      <h2>Findings</h2>
      <!-- paste the main artifact here: table, timeline, or claim map as HTML -->
    </section>

    <section id="confidence">
      <h2>Evidence confidence</h2>
      <!-- table of claims grouped by confidence level with .tag-high/.tag-med/.tag-low badges -->
    </section>

    <section id="sources">
      <h2>Sources</h2>
      <!-- ordered list of sources: URL, domain, date, confidence tag -->
    </section>

    <section id="gaps">
      <h2>Gaps & follow-up</h2>
      <!-- what couldn't be found + 3 follow-up questions -->
    </section>
  </main>
</body>
</html>
```

Fill in all sections from `report.md` and `evidence.md`. Write to `dashboard.html`. Then open it:

```bash
open dashboard.html   # macOS
```

---

## Output Rules

Final chat message: short, always. List only the files that were actually produced.

```
Research complete. Score: [X]/12 · [N] searches · [N] sources

📁 [output directory path]
  [file1] — [one-phrase description]
  [file2] — [one-phrase description]

Key finding: [one sentence]
Gaps: [one sentence — what's still unknown]
Follow-up: [1-2 questions worth pursuing]
```

- Never paste report content into chat
- If a deliverable wasn't produced (e.g. no dashboard for a simple query), don't mention it
- If the answer is genuinely unresolvable, say so directly in key finding

---

## Defaults

- **Minimum searches**: 25 WebSearch + 5 WebFetch per run
- **Sub-questions**: 6-10
- **Patch rounds**: max 2
- **Stop signal**: same sources reappearing, or all evidence at `high`/`medium`
- **Search tool**: WebSearch for discovery, WebFetch for full-page reading
