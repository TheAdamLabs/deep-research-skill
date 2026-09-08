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

**Answer format:** what is the most natural structure for the answer? (narrative, ranked list, recommendation+evidence, timeline, comparison)

**Deliverables:** decide here what files to produce, based on query complexity:

| Complexity | Typical deliverables |
|---|---|
| Simple factual / quick lookup | `dashboard.html` only (minimal: summary + findings sections) |
| Moderate | `evidence.md` (working file) + `dashboard.html` |
| Complex / multi-dimensional | `evidence.md` (working file) + `dashboard.html` with full charts |

`evidence.md` is an intermediate pipeline file (subagents write to it; you read it to populate the dashboard). It is not a user deliverable. `dashboard.html` is the single user-facing output for every run.

`dashboard.html` always earns its cost: it is the only output file users read. For simple factual queries, generate a minimal dashboard (summary + findings only, skip charts and confidence table).

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

Adjust the plan if needed before proceeding.

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

Hold in context -- do not write to a file yet.

Synthesize what you found into a clear, direct answer. Write it as you naturally would: prose narrative, structured lists, a recommendation with evidence, whatever form best communicates the findings. Do not force a format.

Every factual claim must end with an inline source link `[[domain.com]](https://full-url)`. Note where sources conflict. Note what you could not find.

For the confidence donut: count `CONFIDENCE: high/medium/low/conflicted` lines in `evidence.md` -- you will need these totals for `data-high/med/low/conf` on `#confidence-donut` in Phase 8.

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

Write `[output-dir]/dashboard.html` from scratch in a single `Write` call.

**Stack: DaisyUI 5 + Tailwind CSS 4 via CDN** (file requires internet to open).
Full DaisyUI component reference: `.agents/skills/daisyui/` (read any component file if needed).

### Page skeleton

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
  <meta charset="UTF-8" /><meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>[Query title]: Deep Research</title>
  <link href="https://cdn.jsdelivr.net/npm/daisyui@5" rel="stylesheet" type="text/css" />
  <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
  <style type="text/tailwindcss">
    .donut{width:120px;height:120px;border-radius:50%;background:#e2e8f0;position:relative;flex-shrink:0;}
    .donut::after{content:'';position:absolute;inset:22px;background:var(--color-base-100);border-radius:50%;}
  </style>
</head>
<body class="bg-base-200 min-h-screen">
  <div class="drawer lg:drawer-open">
    <input id="nav" type="checkbox" class="drawer-toggle" />
    <div class="drawer-content">
      <div class="navbar bg-neutral text-neutral-content lg:hidden sticky top-0 z-50 shadow">
        <label for="nav" class="btn btn-ghost btn-sm drawer-button">☰</label>
        <span class="font-bold">Deep Research</span>
      </div>
      <main class="p-6 lg:p-10 max-w-4xl mx-auto">
        <h1 class="text-3xl font-bold tracking-tight mb-1">[Query title]</h1>
        <p class="text-base-content/60 text-sm mb-10">[Date] · [N] sources · [query type]</p>
        <!-- sections: #summary #charts #findings #confidence #sources #gaps -->
      </main>
    </div>
    <div class="drawer-side z-40">
      <label for="nav" class="drawer-overlay"></label>
      <nav class="min-h-full w-56 bg-neutral text-neutral-content flex flex-col p-5 gap-1">
        <div class="text-lg font-bold mb-5">Deep<span class="text-primary">Research</span></div>
        <a href="#summary"    class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Summary</a>
        <a href="#charts"     class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Charts</a>
        <a href="#findings"   class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Findings</a>
        <a href="#confidence" class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Evidence</a>
        <a href="#sources"    class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Sources</a>
        <a href="#gaps"       class="btn btn-ghost btn-sm justify-start text-neutral-content/80">Gaps</a>
        <div class="mt-auto text-xs opacity-40 border-t border-neutral-content/20 pt-4 leading-relaxed">
          [Date]<br>[N] sources · [N] searches<br>Score: [X]/12
        </div>
      </nav>
    </div>
  </div>
  <script>
    const d=document.getElementById('confidence-donut');
    if(d){const h=+(d.dataset.high)||0,m=+(d.dataset.med)||0,l=+(d.dataset.low)||0,c=+(d.dataset.conf)||0;
    const t=h+m+l+c||1;let a=0;
    const seg=(deg,col)=>{const s=`${col} ${a.toFixed(1)}deg ${(a+deg).toFixed(1)}deg`;a+=deg;return s;};
    const td=n=>(n/t)*360;
    d.style.background=`conic-gradient(${[seg(td(h),'#16a34a'),seg(td(m),'#d97706'),seg(td(l),'#dc2626'),seg(td(c),'#7c3aed')].join(',')})`;}
  </script>
</body>
</html>
```

### Section guide

**`#summary`** -- put the answer first, scores second.
1. Headline: `<div class="border-l-4 border-primary bg-base-100 rounded-r-xl px-5 py-4 font-medium leading-relaxed mb-6">` (1-2 sentence direct answer)
2. Key insights: 4-6 `<li>` rows in `bg-base-100 rounded-xl border border-base-200 px-4 py-3`, each ending with a `<a class="link link-primary text-xs">` source link
3. Scores: `<div class="stats stats-horizontal shadow bg-base-100 w-full">` with `stat` children (completeness / accuracy / relevance / artifact fit)
4. Quality: `<div role="alert" class="alert alert-success">Research quality: X/12</div>`
5. Donut: `<div id="confidence-donut" data-high="N" data-med="N" data-low="N" data-conf="N" class="donut">` in a flex row with a text legend
6. Takeaways: 3-5 nuance bullets in the same row style as insights

**`#charts`** -- skip for simple queries. Good options:
- `<progress class="progress progress-primary w-full" value="N" max="100">` for comparison bars
- Inline `<svg>` in `<div class="overflow-x-auto">` for timelines or concept maps

**`#findings`** -- free-form synthesis, best DaisyUI component for the data:
- `card card-body` for grouped findings
- `alert alert-success/warning/error` for a clear verdict or recommendation
- `timeline timeline-vertical` for chronological events
- `table table-zebra` for compact structured data (prefer cards for anything wider than 3 columns)
- `collapse` / `accordion` for expandable detail sections

Every factual claim ends with: `<a href="URL" target="_blank" rel="noopener" class="link link-primary text-xs">[domain]</a>`

**`#confidence`** -- `<table class="table table-zebra table-sm w-full">`: columns are confidence (`<span class="badge badge-success/warning/error/secondary badge-sm">`), claim, source link. Curate to 20-40 key claims.

**`#sources`** -- one `<li>` per unique URL. Include confidence badge and freshness date.

**`#gaps`** -- `<div role="alert" class="alert alert-warning mb-3">` per gap; `alert-info` per follow-up question.

Source link rule: every URL must be a real `<a href="..." target="_blank" rel="noopener" class="link link-primary">`. Never plain text.

```bash
open [output-dir]/dashboard.html   # macOS
```

---

## Output Rules

Final chat message: short, always.

```
Research complete. Score: [X]/12 · [N] searches · [N] sources
Charts: [list chart types included, e.g. "donut, comparison bars": or "none" for simple queries]

📁 [output directory path]
  dashboard.html: [one-phrase description of the artifact]

Key finding: [one sentence]
Gaps: [one sentence about what's still unknown]
Follow-up: [1-2 questions worth pursuing]
```

- Never paste research content into chat
- `dashboard.html` is always produced; `evidence.md` is a working file, do not mention it in the output summary
- If the answer is genuinely unresolvable, say so directly in key finding
- **Never use em dashes (:) in any output file or chat message.** Replace with a colon, comma, semicolon, period, or parentheses as appropriate.

---

## Defaults

- **Minimum searches**: 25 web searches + 5 full-page fetches per run
- **Sub-questions**: 6-10
- **Patch rounds**: max 2
- **Stop signal**: same sources reappearing, or all evidence at `high`/`medium`
- **Search tool**: web search for discovery, full-page fetch for in-depth reading
