# Research Wiki Schema

A research knowledge base in the spirit of [Karpathy's LLM-wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). The example here is calibrated for LLM research, but the structure works for any field.

This file is mounted as `research/CLAUDE.md` in the default layout (via symlink), so any LLM agent that auto-loads `CLAUDE.md` from a working directory of `research/` will see it.

## Role split

- **User**: curate sources, ask questions, set direction, make judgments.
- **LLM**: read, write, maintain the wiki, do bookkeeping.

**Constraint**: the LLM does not write to `raw/`; the user does not write to `wiki/`.

## Directory layout

```
research/
├── CLAUDE.md          # symlink to schemas/research-wiki.md (this file)
├── index.md           # catalog (updated after each ingest) — see schemas/bookkeeping.md
├── log.md             # timeline (append-only)            — see schemas/bookkeeping.md
├── raw/               # immutable sources
│   ├── papers/        # arxiv PDFs / paper screenshots
│   ├── blogs/         # long-form technical blogs
│   ├── books/         # multi-chapter books / tutorials. Prefer MD source (single file or one file per chapter); fall back to PDF.
│   ├── threads/       # long Twitter threads
│   ├── talks/         # conference talks / YouTube transcripts
│   ├── code-notes/    # your analysis of a specific repo (not the repo itself)
│   └── personal/      # meeting notes, conversations with collaborators, your own thoughts
└── wiki/              # LLM-maintained synthesis layer
    ├── methods/        # algorithms / techniques (flash-attention, MoE, RLHF, ...)
    ├── concepts/       # abstract concepts (in-context learning, scaling laws, ...)
    ├── models/         # specific models (GPT-4, Claude, Llama, ...)
    ├── benchmarks/     # eval sets (MMLU, HumanEval, SWE-bench, ...)
    ├── labs/           # research orgs (Anthropic, OpenAI, DeepMind, ...)
    ├── people/         # key researchers / high-density information sources
    ├── products/       # industry products (Claude Code, Cursor, ...)
    ├── open-questions/ # your list of open questions
    ├── my-papers/      # wiki pages about your own work (not paper drafts)
    └── sources/        # summary pages for long sources (one long source = one page)
```

## Page templates

### Wiki entity page (methods / concepts / models / ...)

```markdown
---
tags: [entity-type, topic1, topic2]
first-seen: YYYY-MM-DD
last-updated: YYYY-MM-DD
sources: ["[[wiki/sources/...]]", "[[raw/blogs/...]]"]
---

# Entity name

**One-line definition.**

## Core idea
2–3 sentences on the mechanism / core idea.

## Key numbers
Include if available; skip otherwise.

## Appears in
- [[link]] — one sentence on how it's relevant

## Open questions
- Questions you've thought of but not answered (cross-link with [[wiki/open-questions/...]])
```

**Constraint**: each entity page stays within one screen. If it grows beyond, split.

### Source summary page (long content only)

```markdown
---
type: paper | blog | book | talk
authors: [...]
date: YYYY-MM-DD
url: ...
raw-file: "[[raw/.../...]]"
---

# Title

## TL;DR
2–3 sentences.

## Key contributions
1. ...
2. ...

## Pages touched
- [[wiki/methods/...]] — how it touched this page
- [[wiki/concepts/...]] — how it touched this page

## My take
Discussion notes from ingest: judgments, doubts, connections.
```

**Short content (tweets, short blogs) gets no source page** — fold directly into relevant entity pages, leave the original in `raw/`.

## Tag conventions

Entity page `tags` field format: `[type, topic1, topic2, ..., lifecycle?]`.

### Type tag (first position, required, singular)

The first tag = entity type, matching the `wiki/` subdirectory it lives in:

| Subdirectory | Type tag |
|---|---|
| `wiki/concepts/` | `concept` |
| `wiki/methods/` | `method` |
| `wiki/models/` | `model` |
| `wiki/benchmarks/` | `benchmark` |
| `wiki/labs/` | `lab` |
| `wiki/people/` | `person` |
| `wiki/products/` | `product` |
| `wiki/open-questions/` | `open-question` |
| `wiki/my-papers/` | `my-paper` |

**Always singular** — avoids `concept`/`concepts`, `person`/`people` drift. If you find existing pages using plurals, fix them next time you touch them.

### Lifecycle tag (optional, last position)

- `alpha` — concept is named but **definition / boundary / typology is still evolving**; expect future rename / split / absorption into another page.
  - Examples: `[[wiki/concepts/throwaway-editor]]`, `[[wiki/concepts/skills]]`
  - **Not a permanent tag** — alpha pages should eventually be promoted (drop the tag, declare stable), folded (merged into another page), or split (broken into multiple pages). Lint watches for this.
- No lifecycle tag = stable by default.
- `deprecated` — superseded by a newer understanding; kept for historical reference but no longer updated. Pages linking to it should redirect.

### Topic tags (middle positions, freeform)

`agent` / `attention` / `training` / `alignment` / `knowledge-management` etc. **Reuse existing tags** — don't invent a one-off tag for a single page (an orphan tag = no tag).

## Operations

- **Ingest** — pull a new source into the wiki; one source touches multiple pages. See `skills/ingest/SKILL.md`.
- **Lint** — periodic wiki health check (consistency / graph connectivity / research direction). See `skills/lint/SKILL.md`.

Ad-hoc `raw/` cleanup (deleting sources that turned out to be low-value + feeding the lesson back to the triage classifier) is handled as an edge case of ingest — see the ingest skill's edge cases.

## Working style

- Pages are **terse**. Each page fits within one screen.
- Linking is **dense**. The Obsidian graph is the core value.
- Contradictions are **flagged honestly**, not papered over.
- Don't **pad**: if a source touches no existing wiki page, just leave a source summary — don't fabricate a new entity to justify it.
- Don't **change silently**: every operation's edits are recorded in `log.md` so the user can audit.
