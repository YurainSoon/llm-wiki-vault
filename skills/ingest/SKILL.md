---
name: ingest
description: Process one source from research/raw/ into the wiki — read it, discuss takeaways with the user, propose all wiki page creates/updates as a single plan, then execute on confirmation. Use when the user says "ingest <file>", "ingest this", or wants to integrate a specific raw source into the wiki.
---

# Ingest

Single-source wiki integration. One ingest = one source file → multiple wiki page writes.

Pattern: **read source → calibrate takeaways with user → build full plan → user confirms → batch execute**.

User-facing output (takeaway summary, plan table, report) defaults to **the language the user writes in** — mirror the user's input language. To pin a fixed language instead, tell your LLM at the start of the session, or edit this line.

## Preconditions

- Working directory is `research/` (skill assumes the schema in `schemas/research-wiki.md`, mounted as `research/CLAUDE.md` in the default layout).
- Input is one file under `research/raw/<sub>/<file>`.

## Steps

### 1. Validate input

- File must exist under `research/raw/`. Reject paths outside `raw/`.
- Check `log.md` for prior ingest of this source. **Log titles ≠ filenames** — entries use the source's human-readable title (often with author/year, no `.md`), so a literal `basename` grep is brittle and will miss most re-ingests. Do BOTH:

  ```bash
  # (a) List recent ingest lines and eyeball for a near-match:
  grep -E '^## \[.*\] ingest \|' log.md | tail -30

  # (b) Fuzzy match on 2–3 distinctive tokens from the filename
  #     (drop stopwords: the/a/our/of/and/to/for/in/on). Also peek the
  #     source's frontmatter `title:` if present and add tokens from it:
  grep -iE 'ingest \|.*(<token1>|<token2>)' log.md
  ```

  If any candidate appears → STOP and ask user: re-ingest as update, audit existing pages, or cancel. Do not proceed silently — even if (b) is empty, trust (a) over the grep when a recent line looks like the same source.

### 2. Read the source

Use `Read`. For long files (>50K tokens) read in two passes if needed: first pass scan structure, second pass focus on substantive sections.

For PDF books: Read tool caps at 20 pages per call; for a >20-page PDF you must pass `pages` (e.g., `pages: "1-20"`) and make multiple calls. First-pass scan TOC + intro + chapter heads to plan; second-pass deep-read the substantive chapters.

### 3. Calibrate takeaways with user (BRIEF)

Output **2–4 sentences**:
- One sentence: what the source is about
- 2–3 bullets: the key claims / contributions you noticed
- One question: "Anything you want to emphasize / ignore / focus on?"

Wait for user reply. User may say "go" / "just start" — that's fine, proceed.

**Do not** dump a long summary here. This is a calibration check, not an analysis report. The full analysis goes into the wiki pages.

### 4. Decide source-page

Word count thresholds (count substantive content, not boilerplate):

| Length | Source page? |
|---|---|
| > 2000 words / long talk / paper / book / long gist | **Yes** — create `wiki/sources/<slug>.md` |
| < 1000 words / short blog / thread | **No** — fold directly into entity pages |
| 1000 – 2000 words | **Propose** — state your recommendation in the plan; user decides |

Slug rule: lowercase, hyphenated, derived from title (drop noise words). E.g., `wiki/sources/llm-wiki-karpathy.md`.

### 5. Discover candidate existing wiki pages

Two-step discovery:

1. **Read `index.md` using the Read tool** (not bash `cat`). This both gives you the catalog AND satisfies the Read-before-Edit requirement when you update index.md in step 8.

2. **Bash to find candidate existing pages**:
   ```bash
   grep -ril "<key term>" wiki/
   ```

3. For each candidate existing page that may be touched in step 8, **Read it now with the Read tool** (not bash `cat`). The Edit/Write tools will refuse to modify any existing file that has not been Read in this session.

Build two lists:
- **Existing pages** that this source touches (need updates)
- **Missing pages** that should exist (need creation)

For an empty wiki (first ingest), the existing list is empty — that's expected.

### 6. Build the plan

Render this exact structure to the user:

```
## Ingest Plan: <source title>

### Source page
- [x] CREATE wiki/sources/<slug>.md  (long content, create source page)
  or [ ] SKIP  (short content, fold directly into entity pages)
  or [?] 1000–2000 words boundary — I recommend X, you decide

### New entity pages (N)
- wiki/concepts/<slug>.md — <one-line description of what this page covers>
- wiki/people/<slug>.md — <description>
- wiki/methods/<slug>.md — <description>
...

### Updated existing entity pages (M)
- wiki/concepts/<existing>.md — add reference to "Appears in"
- wiki/methods/<existing>.md — add reference + revise Core idea (new angle: ...)
- wiki/models/<existing>.md — flag contradiction: original page says X, this source says Y
...

### index.md
- Concepts: add N entries
- People: add M entries
- Sources: add 1 entry

### log.md
- Append: ## [YYYY-MM-DD] ingest | <source title> → touched K pages

Reply `go` to confirm, or edit any line first.
```

**Plan rules:**
- Be specific about *what* changes on each existing page (not just "update X.md")
- For new pages, give the slug AND a one-liner — user needs to validate naming
- Don't propose more than ~15 page touches per source. If a source legitimately touches more, surface that and ask whether to scope down.

### 7. Wait for explicit confirmation

Accepted: `go`, `ok`, `confirm`, `execute`. Ambiguous (`looks fine`, `sure`) → ask again.

If user edits any line (renames a slug, removes a page, splits one entity into two) → apply edits, re-render plan, wait again.

### 8. Execute writes

In order:
1. Create source page (if applicable) using the source-page template (see `schemas/research-wiki.md`)
2. Create new entity pages using the entity-page template
3. Update existing entity pages — add to `## Appears in`, modify `## Core idea` / `## Key numbers` if warranted, flag contradictions
4. Update `index.md` — add new entries, modify summaries if changed
5. Append to `log.md`:
   ```
   ## [YYYY-MM-DD] ingest | <source title> → <K> pages
   - New: wiki/concepts/foo, wiki/people/bar, ...
   - Updated: wiki/methods/baz, ...
   ```

Use today's date (YYYY-MM-DD format) — get from `date +%Y-%m-%d` if unsure.

Batch writes where possible (parallel `Write` / `Edit` tool calls in one assistant turn).

**Tool result discipline**: After each tool call, inspect the result. **Absence of an error message ≠ success.** If a tool returns an error, address it before proceeding. Common failures:
- `"File has not been read yet"` on existing files → Read the file with the Read tool, then retry.
- `"String to replace not found"` on Edit → Read the file fresh and recompute the exact `old_string`.

### 8.5 Verify writes (MUST pass before step 9)

For every change in the plan, verify it actually landed:

**New files** (source page + new entity pages):
- Use Read or `ls -la <path>` to confirm the file exists with non-trivial content.

**Updated existing entity pages**:
- `grep -F "[[wiki/sources/<source-slug>]]" wiki/<path>` — confirms the new "Appears in" link was added.

**index.md**:
- For each new entry, `grep -F "[[wiki/<type>/<slug>]]" index.md` — all must hit. If any miss → index.md update silently failed.

**log.md**:
- `tail -10 log.md` — confirm today's ingest entry is present.

**On any verify failure: auto-retry the failed write ONCE.**
- Re-Read the target file with the Read tool before retrying Edit/Write.
- After retry, re-run the failing verify check.
- If retry succeeds → continue to step 9, but record which writes needed retry.
- If retry also fails → **STOP**. Do not claim success. Report the failed write + the original tool error to the user. Ask whether to retry again / debug / abort.

**Never claim success in step 9 without all verify checks passing.**

### 8.6 Optional semantic verify

§8.5 verifies writes *landed*; this verifies they're *faithful* to the source. This is the ONE place ingest is allowed to dispatch a subagent — for independent verification, not synthesis.

**When to run** — default-ON if any of:
- touched ≥ 10 wiki pages
- source is book / long talk / paper > 4000 words
- ingest had ≥ 1 explicit contradiction flag
- user asked ("verify", "check", "audit")

Otherwise default-OFF. Always offer a one-line opt-in:
```
Run semantic verify? (touched N pages / source type X) [y/N]
```

User says no / silence treated as no → skip to §9.

**Mechanism** — dispatch ONE `general-purpose` subagent. Brief:

```
You are the semantic verifier for an ingest pass. You are independent
from the agent that wrote the wiki pages — do not assume what they
wrote is correct.

Inputs:
- Source: <absolute path under raw/>
- Wiki touches: <list of (page path, new|updated)>

Steps:
1. Read the source.
2. Read each touched wiki page in its current on-disk state.
3. Cross-check.

Report (under 300 words), four bullets:
- Drift — claims in wiki pages NOT supported by the source.
- Omission — substantive claims in source NOT reflected in any touched page.
- Internal contradiction — pages touched in this ingest making mutually
  inconsistent claims (lateral failure within one ingest).
- Naming — entity slugs that don't match the schema in schemas/research-wiki.md.

For each category: "none" if clean, otherwise list specific page + claim.
Do NOT propose fixes — only flag.
```

**Surface findings to user before §9 report.** User responses:
- `fix #N` → main agent applies edit, re-runs §8.5 grep verify on changed files
- `skip` → record in §9 report ("verifier flagged N issues, user chose skip")
- `discuss` → conversation continues, defer §9

Verifier output is informational, not blocking — user can ignore. But it MUST be surfaced; never silently drop.

### 9. Report

Concise summary:
- Source page: created / skipped
- Created N entity pages, updated M
- Total touched: K pages
- Index + log updated
- **If any verify required retry**: list which writes needed retry (signal for user to inspect underlying cause)
- (Optional) Suggest next ingest target if other unprocessed files in raw/

## Page conventions

Templates and frontmatter are defined in `schemas/research-wiki.md`. Follow them strictly:
- Entity pages: keep ≤ 1 page; tags + first-seen + last-updated + sources frontmatter
- Source pages: type/authors/date/url/raw-file frontmatter; TL;DR + Key contributions + Pages touched + My take sections
- Always use `[[wiki link]]` syntax (Obsidian-style), with paths relative to the vault root or just the entity name

## Style constraints

- Entity pages must be **dense and link-heavy**, not narrative essays.
- When updating an existing entity page, **append** to `## Appears in` rather than rewriting it.
- If the new source contradicts an existing claim, **add** the contradiction explicitly — don't smooth it over. Format:
  ```
  ## Key numbers
  - Old: X% on benchmark Y (per [[wiki/sources/foo]])
  - New (contradicts above): Z% on Y (per [[wiki/sources/bar]]) — difference is in measurement protocol
  ```
- Never modify files in `raw/` — it is immutable.
- Never write outside `wiki/`, `index.md`, or `log.md`.

## Edge cases

- **First ingest into empty wiki**: existing-pages list is empty; all changes are creates. Proceed normally.
- **Source already ingested** (per log.md): ask whether to re-ingest as update.
- **Source not worth ingesting** (you or user concludes value is too low after reading — typically at calibration step 3 or after seeing the plan in step 6): bail and clean up instead of ingesting.
  1. `rm` the raw file (after explicit user `go` confirmation)
  2. Edit `log.md` to remove the corresponding `triage` entry (keep history honest about what made it through)
  3. Append a new entry to `log.md`: `## [YYYY-MM-DD] sweep | <filename>` + one-line reason
  4. **If the low-value pattern is generalizable** (not unique to this one file): propose to user — "should I add a demote signal to `agents/triage-classifier.md` so similar files get filtered earlier?" If approved, edit the classifier prompt.

  This trigger absorbs what was previously documented as a standalone "sweep" operation. It can fire anywhere in the ingest flow (calibration, planning, even mid-execution if the source proves worthless).
- **Source mostly redundant** (everything it says is already in wiki): make this finding explicit in step 6 — propose a minimal source page (or no source page) + 0–1 page touches. Don't pad.
- **Source contradicts multiple existing pages**: surface all contradictions in the plan; user decides which to flag vs reconcile.
- **Source spans multiple entity types** (e.g., a survey paper covering many methods): cap at ~15 touched pages in one ingest. If more, suggest a follow-up `/ingest` pass focused on a different subset.
- **Book-length source** (`raw/books/`): item may be a single file (`.pdf` / `.md`) OR a directory of per-chapter `.md` files (honkit / mdbook source layout). For directory form: Read `README.md` + `SUMMARY.md` first to grasp structure, then read chapters in order. MD source preferred over PDF when both exist (better Claude readability + cleaner quotes for wiki).
  - Default to whole-book ingest if total fits in ~200K tokens (1M context handles it).
  - For longer books, ingest chapter-by-chapter sharing one source page: create the source-page skeleton on first ingest, then each subsequent ingest appends a `## Chapter N: <title>` section to it + does that chapter's wiki touches. The 15-page touch cap applies *per ingest call*, not per book.
- **Ambiguous entity naming** (should this be `wiki/concepts/X.md` or `wiki/methods/X.md`?): pick the better fit and flag the choice in the plan with `[?]`. Let user override.

## Do not

- Do not write any file before user confirmation.
- Do not update `raw/`.
- Do not invent content not supported by the source — if unsure, leave the section out of the entity page.
- Do not collapse contradictions for tidiness.
- Do not produce a long-form analysis at step 3 — keep takeaway calibration brief.
- Do not dispatch subagents in steps 1–8.5 — synthesis must stay in one coherent context. The only allowed subagent is the optional semantic verifier in §8.6 (independent verification, not synthesis).
