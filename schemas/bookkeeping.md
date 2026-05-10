# Bookkeeping Files: index.md and log.md

Each wiki maintains two bookkeeping files at its root:

- `index.md` — live catalog of every entity page
- `log.md` — append-only ledger of every operation

Both are **gitignored in this showcase repo** because they reflect the original author's vault content. **When you fork this repo, you bootstrap them yourself** — see "Bootstrapping" below.

## index.md — Live catalog

**Purpose**: a quick index of every entity page, regenerated/updated after each ingest. Lets the LLM avoid blindly listing the whole `wiki/` directory when it needs to discover existing pages.

**Format**:

```markdown
# <Wiki name> Index

Last updated: YYYY-MM-DD

## Concepts (N)
- [[wiki/concepts/foo]] — one-line description
- [[wiki/concepts/bar]] — one-line description

## Methods (N)
- ...

## Models (N)
- ...

## Sources (N)
- [[wiki/sources/baz]] — title, author, date

(... one section per `wiki/<subdir>/`)
```

`index.md` is updated as part of every `ingest` pass (see `skills/ingest/SKILL.md`, step 8).

## log.md — Time-ordered ledger

**Purpose**: append-only record of every operation (triage move, ingest, sweep, lint pass). Lets you audit "what was done when, and why" and lets the LLM avoid re-ingesting the same source.

**Format**: each entry is one H2 with a date, followed by details:

```markdown
## [YYYY-MM-DD] triage | filename → destination

## [YYYY-MM-DD] ingest | source-title → N pages
- New: wiki/concepts/foo, wiki/people/bar
- Updated: wiki/methods/baz

## [YYYY-MM-DD] lint | summary
- 9 checks / N findings: immune A / connective B / generative C
- Severity: high X / medium Y / low Z
- Actions: fix N / skip M
- Modified: list of files

## [YYYY-MM-DD] sweep | filename
- Reason: <one-line reason>
```

Newest entries go at the bottom.

## Bootstrapping (after fork)

After cloning this repo, each subdomain's `index.md` and `log.md` don't exist. To create them:

**Option A — Tell your LLM**:

> Bootstrap `research/index.md` and `research/log.md` per the format in `schemas/bookkeeping.md`. Both should be empty (just the headers / no entries).

**Option B — Manual** (one-time):

```bash
cd research/

cat > index.md << 'EOF'
# Research Wiki Index

Last updated: $(date +%Y-%m-%d)

## Concepts (0)
## Methods (0)
## Models (0)
## Benchmarks (0)
## Labs (0)
## People (0)
## Products (0)
## Open questions (0)
## My papers (0)
## Sources (0)
EOF

touch log.md

# Same for interview-prep/
```

After bootstrap, the first `triage` or `ingest` pass will start populating them.
