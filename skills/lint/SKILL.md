---
name: lint
description: Periodic wiki health check across 3 dimensions covering immune (consistency), connective (graph density), and generative (research direction) functions. Reports findings as a structured table grouped by dimension; never auto-fixes — user reviews and instructs which to fix. Use when user says "lint", "lint wiki", "health check", or for monthly wiki passes.
---

# Lint

Wiki-wide health check. One pass produces one structured report grouped by check type.

Pattern: **cheap scans → LLM analyses → one grouped report → user-directed fixes**.

User-facing output (report contents, finding text) is in **English** by default. To switch to another language, tell your LLM at the start of the session, or edit this file directly to change the language directive.

## Preconditions

- Working directory is `research/`.
- Wiki schema in `schemas/research-wiki.md` (mounted as `research/CLAUDE.md` in the default layout). The Tag conventions table defines the singular type tags lint enforces.
- Lint reads `wiki/` and `index.md`. Never touches `raw/`.

## Three dimensions

Per `schemas/research-wiki.md`, lint serves three wiki health functions. Every finding belongs to exactly one:

- **Immune**: defends internal consistency. Checks: stale-claims, alpha-aging, type-drift, contradiction-symmetry.
- **Connective**: densifies the graph. Checks: missing-entities, missing-cross-refs, orphan-pages, concept-overlap.
- **Generative**: suggests directions. Checks: direction-suggestions.

v1 implements 9 checks across the three dimensions.

## Steps

### 1. Cheap scans (deterministic)

Run via Bash. No LLM judgment needed.

**Type tag drift**:
For each `wiki/<dir>/*.md`, extract the first frontmatter tag. Flag mismatches against the directory's expected singular type per the schema's tag conventions.

```bash
# Example pattern (adapt to your scan script)
for f in wiki/concepts/*.md; do
  first_tag=$(awk '/^tags:/{getline; print}' "$f" | grep -oE '^\[?[a-z-]+' | head -1)
  # compare against expected: concept
done
```

**Alpha aging**:
For each entity page with `alpha` in tags, parse `last-updated`. Flag if `(today - last-updated) > 28` days.

**Orphan pages**:
For each `wiki/<type>/<page>.md` (excluding `wiki/sources/*` — sources are entry points by design), count incoming `[[wiki/<this-path>]]` and `[[<basename>]]` references. Flag pages with 0 inbound links.

### 2. Cross-reference scan (grep + LLM judgment)

**Missing cross-references**:
For each entity page, grep its body for plain-text mentions of other entity slugs (case-insensitive, word-bounded). Flag mentions NOT wrapped in `[[...]]`.

LLM filter: discard cases where the plain-text mention is intentional (a quote, a code block string, the page's own self-reference). Surface only genuine missing-link cases.

### 3. LLM analyses (read pages strategically)

Use `index.md` to know what to load. Don't load the whole wiki blindly.

**Stale claims**:
1. List sources by date from `wiki/sources/*` frontmatter.
2. For each entity page, check whether claims trace to older sources where a newer source has updated info.
3. Output: (page, stale claim, newer source, suggested update).

**Contradiction symmetry**:

Structural check, not semantic — catches the failure mode where ingest noted a contradiction on one page but didn't propagate to its referenced counterpart.

1. Grep all `wiki/**/*.md` for contradiction markers — typical forms (per the ingest template):
   - `New (contradicts above):`
   - `Old:` paired with `New:` on adjacent lines
   - any line containing `contradict` inside `## Key numbers` / `## Core idea`
2. For each finding, extract any `[[wiki/<path>]]` or `[[<entity>]]` references inside the contradiction block.
3. For each referenced target page, check whether it contains a mirroring contradiction marker citing back to the origin (or to the same source pair). Use `grep -F "[[wiki/<origin-slug>]]" <target>` plus a textual check for `contradict`.
4. **Asymmetric finding**: target lacks mirror → flag.
5. **Post-merge re-scan**: after any concept-merge fix executed in this lint pass (see step 5), re-scan the merged page for duplicate metric/claim subjects with conflicting values — flag as merge-induced self-contradiction.
6. Output: (origin page, target page, contradiction context, suggested mirror text — or for post-merge: which lines conflict).

Out of scope (would be v3): semantic-level "page A says P, page B says ¬P with no marker on either side". This requires actual claim-level NLP, not symmetry of existing markers.

**Missing entities**:
1. Read all entity pages.
2. Identify proper-noun / concept-name mentions appearing in ≥ 2 distinct pages without a corresponding entity page.
3. Output: (term, mentioning pages, suggested type/path).

**Concept overlap / candidate merges** (the headline check):
1. Read all pages of one type (start with `wiki/concepts/` — usually highest overlap rate).
2. Cluster by semantic similarity (one-line definition + Core idea + tag overlap + shared sources).
3. For each high-overlap pair (judged > 60% conceptual overlap), classify:
   - **Synonym** → propose merge; canonical name = the more-cited one
   - **Subset/superset** → propose hierarchy; fold subset into superset's `## Subtopics` section
   - **Adjacent but distinct** → propose adding `## Boundary with X` section to both
   - **Same name, different concepts** → propose disambiguation page
4. Output: (pages, overlap nature, recommendation, alternative options).

Always present at least 2 options when overlap nature is ambiguous — don't decide for the user.

**Direction suggestions** (generative, max 5):
1. Read `index.md` + `wiki/open-questions/*` + the 5 most-linked entity pages.
2. Identify:
   - **Underdeveloped**: high backlink count + thin body (demand exceeds supply)
   - **Missing companions**: many methods cited but no benchmark / comparison page
   - **Under-followed sources**: a person/lab cited often but only 1 source ingested
   - **Adjacent gaps**: implied by existing concept clusters but not yet present
3. Output: ≤ 5 suggestions, each with (suggestion, evidence in wiki, concrete next action — e.g., "follow X on twitter", "ingest paper Y", "create benchmark page Z").

### 4. Build the report

Render this exact structure:

```
# Lint Report (YYYY-MM-DD)

## Overview
- N total findings
- Immune: A | Connective: B | Generative: C
- Severity: high X / medium Y / low Z

## Immune checks

### Stale claims (n)
| # | Page | Old claim | Newer source | Suggestion | Severity |
|---|---|---|---|---|---|

### Alpha aging (n)
| # | Page | last-updated | Days since | Suggestion (promote / fold / split) | Severity |

### Type tag drift (n)
| # | Page | Current first tag | Expected | Severity |

### Contradiction asymmetry (n)
| # | Origin page | Target page | Contradiction context | Suggested mirror | Severity |

## Connective checks

### Missing entities (n)
| # | Candidate entity | Mentioned in | Mention count | Suggested path | Severity |

### Missing cross-references (n)
| # | Page | Should link to | Context | Severity |

### Orphan pages (n)
| # | Page | Notes | Severity |

### Concept overlap candidates (n)
| # | Pages | Overlap nature | Suggestion | Alternatives | Severity |
| 1 | [[harness-engineering]], [[agent-scaffold]] | adjacent but distinct | add boundary section | merge / split into hierarchy | medium |

## Generative directions

### Direction suggestions (≤5)
1. **<suggestion>** — Evidence: <wiki citation> — Action: <concrete next>
2. ...

---

## How to act

Per item: `fix #N` / `skip #N`
Bulk: `fix all type-drift` / `fix all high severity`
Discuss: just say what you think, e.g. `#7 I lean toward Option C`
```

**Severity calibration:**
- **high**: data integrity at risk (overlap that's actually a duplicate; stale claim contradicting current understanding; type drift breaking schema query)
- **medium**: noticeable but non-blocking (missing cross-ref; alpha aging without strong signal; weak overlap)
- **low**: stylistic / informational (orphan that may be intentional; soft direction suggestion)

### 5. Present and wait

Output the full report. Do **not** start fixes.

User responses:
- `fix #N` → execute as a normal Edit operation
- `skip #N` → mark deferred for this session, don't re-surface
- `fix all <category>` → batch within that category
- Discussion / override (`#7 I lean toward Option C`) → adapt and re-render the affected section

For multi-page fixes (especially concept merges), present a sub-plan first and wait for explicit `go`:

```
Merge plan for #N:
- Canonical: keep [[wiki/concepts/X]]
- Source: fold [[wiki/concepts/Y]] content into X
- Redirect: replace all [[Y]] references → [[X]] (M references in K pages)
- Delete: wiki/concepts/Y.md
```

Don't merge pages without showing this plan.

### 6. Append to log.md

After the lint pass (regardless of how many fixes happened):

```
## [YYYY-MM-DD] lint | <one-line summary>
- 9 checks / N findings: immune A / connective B / generative C
- Severity: high X / medium Y / low Z
- Actions: fix N / skip M / discussed K / pending L
- Modified files: <list if fixes happened>
```

If no fixes happened, drop the last line — empty action is also a valid lint outcome (the report itself is the value).

## Style constraints

- Findings need a unique `#` within their section so the user can reference precisely.
- Direction suggestions are bounded: max 5 per run; each must cite evidence in the wiki.
- Concept-overlap recommendations always present multiple options when applicable.
- Severity must be calibrated — don't make everything high or everything low. Distribution should reflect real risk.

## Edge cases

- **Sparse wiki** (< 10 entity pages of a type): skip concept-overlap for that type — clustering needs critical mass.
- **Recent ingest** (any wiki page with `last-updated` = today): defer overlap check on those specific pages — they need settling time before being judged for merge.
- **Large wiki** (> 200 pages): split concept-overlap by type across multiple lint runs (one type per run); warn the user in Overview.
- **No findings in a category**: still list the section header with "0 items" — explicit absence is informative.
- **Concurrent ingest in progress** (most recent log entry has no completion record AND was added < 1h ago): warn user, suggest waiting until ingest completes before lint.

## Do not

- Do not fix anything without explicit user instruction.
- Do not modify `raw/`.
- Do not surface findings the user explicitly skipped this session.
- Do not auto-merge concept-overlap candidates — even "obvious" synonyms need approval (canonical name choice + redirect strategy).
- Do not exceed 5 direction suggestions — quality over quantity; user must be able to act on each.
- Do not run lint while another lint or ingest is in progress.
- Do not dispatch subagents — v1 runs entirely in the main agent. Subagent parallelism may come in v2 if scaling requires it.
