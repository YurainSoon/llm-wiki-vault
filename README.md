# llm-wiki-vault

An [Obsidian](https://obsidian.md) vault structured as an LLM-managed knowledge base, in the spirit of [Karpathy's LLM-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

> **Read in**: [English](README.md) · [中文](README.zh-CN.md)

## What this is

A working layout + a small set of [Anthropic Skills](https://docs.claude.com/en/docs/claude-code/skills.md) for managing a knowledge vault where:

- **You** curate sources (web clippings, papers, talks) into `Clippings/` (an inbox) and ask questions.
- **The LLM** reads sources, synthesizes them into dense interlinked entity pages under each subdomain's `wiki/`, and maintains bookkeeping (`index.md`, `log.md`).

The skills here implement five operations: **triage** (route inbox items into subdomains), **ingest** (one source → many wiki page touches, planned and confirmed before write), **lint** (periodic wiki health check), **query** (read the wiki and answer a question with a cited prose synthesis), and a **triage-classifier** subagent (parallel Haiku-driven classification).

The skills are written as `SKILL.md` files (Anthropic Skill format), but their content is **framework-agnostic**: any LLM agent that can dispatch subagents and edit files can adapt them. See [INTEGRATION.md](INTEGRATION.md).

## Repository layout

```
llm-wiki-vault/
├── README.md / README.zh-CN.md
├── INTEGRATION.md           # Claude Code / Cursor / generic LLM agent integration
│
├── skills/                  # ← canonical home of all skills (framework-agnostic)
│   ├── triage/SKILL.md
│   ├── ingest/SKILL.md
│   ├── lint/SKILL.md
│   └── query/SKILL.md
│
├── agents/                  # ← canonical home of subagents
│   └── triage-classifier.md
│
├── schemas/                 # ← schema docs (mounted as CLAUDE.md via symlink)
│   ├── vault-routing.md     # = root CLAUDE.md
│   ├── research-wiki.md     # = research/CLAUDE.md (once you scaffold it)
│   ├── interview-prep-wiki.md
│   └── bookkeeping.md       # spec for index.md and log.md
│
├── CLAUDE.md → schemas/vault-routing.md
│
├── .claude/                 # Claude Code wiring (symlinks point back to skills/, agents/)
│   ├── skills/triage → ../../skills/triage
│   └── agents/triage-classifier.md → ../../agents/triage-classifier.md
│
└── Clippings/               # inbox (dir kept as scaffolding; contents gitignored)

# Created locally on your fork — NOT in the repo. The subdomain folders
# themselves are gitignored (not just their contents), so they never appear
# in git. You scaffold them once after cloning (see Quick start):
#
#   research/         an example subdomain
#     ├── CLAUDE.md → ../schemas/research-wiki.md
#     ├── .claude/skills/{ingest,lint,query} → ../../../skills/...
#     ├── raw/        immutable sources (your private notes)
#     ├── wiki/       synthesized pages
#     └── index.md, log.md   # bootstrapped per schemas/bookkeeping.md
#   interview-prep/   another subdomain (same pattern)
#   life/             subdomain without a wiki
```

**Why subdomains aren't in the repo.** This repo ships only the reusable framework — skills, subagents, schemas, and the root Claude Code wiring. The actual knowledge lives in the subdomain folders (`research/`, `interview-prep/`, `life/`), which are private and gitignored *as folders*: the directories themselves never show up in git, not just their contents. You create them on your fork.

**Why symlinks?** Single source of truth in `skills/` / `agents/` / `schemas/`, while `.claude/` and per-subdomain `CLAUDE.md` sit where Claude Code (and tools that auto-load `CLAUDE.md`) expect. The root-level symlinks (`.claude/skills/triage`, `.claude/agents/…`) ship with the repo; the per-subdomain symlinks live inside the gitignored subdomain folders, so you recreate them when you scaffold a subdomain. Edit once, every surface updates.

## Hierarchical skill placement

Skills live at one of two levels:

| Skill | Lives at | Why |
|---|---|---|
| `triage` | vault root (`skills/triage/`, wired via `.claude/skills/triage` — ships in repo) | cross-domain — routes Clippings into the right subdomain |
| `ingest`, `lint`, `query` | per-subdomain (`research/.claude/skills/`, symlinked on your fork) | domain-specific — operates on one wiki at a time, needs that wiki's schema |

The canonical skill content always lives in the root `skills/` folder (which *does* ship in the repo). "Per-subdomain" only means *where the symlink that mounts it lives* — and since the subdomain folders are gitignored, you create those symlinks yourself when scaffolding a subdomain.

This mirrors how you actually use them: launch your LLM agent at the vault root for triage, then `cd research/` and start a fresh session for ingest/lint/query. Each session sees exactly the skills relevant to its scope.

**If you only manage one wiki** (e.g., just a research vault, no separate interview-prep), put everything at the root: keep `ingest`, `lint`, and `query` in the root-level `skills/`, symlink them under the root `.claude/skills/`, and skip the subdomain structure. Tell your LLM to do this.

## Quick start

### With Claude Code

```bash
git clone <this-repo> llm-wiki-vault
cd llm-wiki-vault

# The subdomain folders don't ship in the repo — scaffold one. Tell Claude:
claude
> Scaffold the research/ subdomain per schemas/research-wiki.md and
> schemas/bookkeeping.md: create research/CLAUDE.md → ../schemas/research-wiki.md,
> the raw/ + wiki/ subdirectory structure, the
> research/.claude/skills/{ingest,lint,query} symlinks back to ../../../skills/,
> and bootstrap research/index.md + research/log.md.
> Repeat for interview-prep/ if you want a second subdomain.
```

The root `.claude/` symlinks ship with the repo, so **triage works immediately** at the vault root. `ingest` / `lint` / `query` become available once you've scaffolded a subdomain and `cd` into it.

To start using it: drop a file (a paper PDF, a saved web page) into `Clippings/`, then say "triage". When triage routes the file into `research/raw/<sub>/`, `cd research/` and say "ingest <filename>" to fold it into the wiki.

### With other agent frameworks

The skill files are plain markdown — see [INTEGRATION.md](INTEGRATION.md) for how to wire them up in Cursor, Aider, or your own LangGraph / Anthropic SDK agent.

## Languages

All skills mirror **the language you write in** — ask in Chinese, get tables / summaries / `reason` fields back in Chinese; ask in English, get English. To pin a fixed output language regardless of input, **tell your LLM at the start of a session, or edit the language directive at the top of the relevant `SKILL.md` file** — that's the standard "edit your own skill" loop these workflows are built around.

## Inspiration

- Andrej Karpathy — [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (the original gist)
- Anthropic Skills — [docs](https://docs.claude.com/en/docs/claude-code/skills.md)
- Obsidian — graph view + `[[wikilink]]` syntax
