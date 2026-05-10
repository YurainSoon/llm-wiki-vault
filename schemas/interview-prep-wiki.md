# Interview Prep Wiki Schema

Holds **procedural interview prep content only** — a sibling to the research wiki, with a much narrower mandate.

**Boundary**: anything with independent technical or academic value goes to `research/`, even if it's "also useful for interviews". This subdomain only holds problems, interview experiences, company processes, and behavioral question material — content that exists **only for the purpose of interviews**.

This file is mounted as `interview-prep/CLAUDE.md` in the default layout (via symlink).

## Role split

- **User**: decides what to practice, which companies to research.
- **LLM**: maintains the patterns library, organizes interview experiences, tracks company intel.

## Directory layout

```
interview-prep/
├── CLAUDE.md          # symlink to schemas/interview-prep-wiki.md (this file)
├── index.md           # see schemas/bookkeeping.md
├── log.md             # see schemas/bookkeeping.md
├── raw/
│   ├── leetcode/         # problems + your solutions
│   ├── system-design/    # system design problems ("design X" genre)
│   ├── behavioral/       # behavioral question material / story raw material
│   ├── companies/        # source material on target companies (job pages, news, salary data)
│   └── experiences/      # interview experience write-ups
└── wiki/
    ├── algo-patterns/        # problem-type abstractions (two-pointer, monotonic stack, DP state design, ...)
    ├── sd-patterns/          # system design patterns (caching, consistency, rate limiting, sharding, ...)
    ├── behavioral-stories/   # STAR story library
    └── companies/            # one page per company: process, salary band, question types, people
```

## Three core operations

### Ingest

- **Problem** → extract problem-type pattern, update or create `wiki/algo-patterns/<pattern>.md`, link back to original.
- **System design problem** → same, update `wiki/sd-patterns/`.
- **Interview experience** → break down to corresponding patterns + update `wiki/companies/<company>.md`.
- **Behavioral raw material** → organize into `wiki/behavioral-stories/`, broken down by STAR.

(The interview-prep variant of ingest is a thinner version of the research-wiki ingest skill. You can adapt `skills/ingest/SKILL.md` or write your own dedicated skill at `interview-prep/.claude/skills/ingest/`.)

### Query

- "Which problem types am I underpracticing?" → count frequency and recent updates in `wiki/algo-patterns/`.
- "What is X company's interview like?" → read `wiki/companies/X.md`.
- "What approach should I use for this kind of problem?" → cross-reference `wiki/algo-patterns/`.
- "Do I have a STAR story covering conflict-type questions?" → read `wiki/behavioral-stories/`.

### Lint

- Which patterns have < 3 data points — under-practiced.
- Which company pages haven't been updated in > 3 months — likely stale.
- Does behavioral-stories cover the common types (leadership / conflict / failure / impact)?
- Which problem types appearing in interview experiences don't yet have pattern pages?

## Auxiliary operations

### Sweep (manual `raw/` cleanup)

Trigger: user manually reviews `raw/` and finds low-value problems / experiences / company material that triage let through.

1. User specifies files to delete, or LLM proposes (with reasoning).
2. Delete files + remove the corresponding `triage` entries from `log.md`.
3. Append to `log.md`: `## [YYYY-MM-DD] sweep | <description>` + delete list + reasons.
4. If the low-value pattern is generalizable → also update `agents/triage-classifier.md` to add a new demote signal.

## Working style

- Pattern pages must be **abstracted properly** — don't just copy "Two Sum" verbatim and call it a pattern.
- Company pages must be **dated** — salary, process, interviewer reputation all change over time.
- When unsure about boundary, **ask**: ambiguous cases probably belong in `research/`.
