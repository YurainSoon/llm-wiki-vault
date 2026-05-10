# llm-wiki-vault

An [Obsidian](https://obsidian.md) vault structured as an LLM-managed knowledge base, in the spirit of [Karpathy's LLM-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f).

> **Read in**: [English](README.md) · [中文](README.zh-CN.md)

## What this is

A working layout + a small set of [Anthropic Skills](https://docs.claude.com/en/docs/claude-code/skills.md) for managing a knowledge vault where:

- **You** curate sources (web clippings, papers, talks) into `Clippings/` (an inbox) and ask questions.
- **The LLM** reads sources, synthesizes them into dense interlinked entity pages under each subdomain's `wiki/`, and maintains bookkeeping (`index.md`, `log.md`).

The skills here implement four operations: **triage** (route inbox items into subdomains), **ingest** (one source → many wiki page touches, planned and confirmed before write), **lint** (periodic wiki health check), and a **triage-classifier** subagent (parallel Haiku-driven classification).

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
│   └── lint/SKILL.md
│
├── agents/                  # ← canonical home of subagents
│   └── triage-classifier.md
│
├── schemas/                 # ← schema docs (mounted as CLAUDE.md via symlink)
│   ├── vault-routing.md     # = root CLAUDE.md
│   ├── research-wiki.md     # = research/CLAUDE.md
│   ├── interview-prep-wiki.md
│   └── bookkeeping.md       # spec for index.md and log.md
│
├── CLAUDE.md → schemas/vault-routing.md
│
├── .claude/                 # Claude Code wiring (symlinks point back to skills/, agents/)
│   ├── skills/triage → ../../skills/triage
│   └── agents/triage-classifier.md → ../../agents/triage-classifier.md
│
├── research/                # an example subdomain
│   ├── CLAUDE.md → ../schemas/research-wiki.md
│   ├── .claude/skills/lint → ../../../skills/lint
│   ├── .claude/skills/ingest → ../../../skills/ingest
│   ├── raw/                 # immutable sources (gitignored — your private notes)
│   ├── wiki/                # synthesized pages (gitignored)
│   ├── index.md             # bootstrapped after fork — see schemas/bookkeeping.md
│   └── log.md
│
├── interview-prep/          # another example subdomain
│   └── (same pattern)
│
└── life/                    # subdomain without a wiki (gitignored)
```

**Why symlinks?** Single source of truth in `skills/` / `agents/` / `schemas/`, while `.claude/` and per-subdomain `CLAUDE.md` remain in the locations Claude Code (and tools that auto-load `CLAUDE.md`) expect. Edit once, both surfaces update.

## Hierarchical skill placement

Skills live at one of two levels:

| Skill | Lives at | Why |
|---|---|---|
| `triage` | vault root (`skills/triage/`) | cross-domain — routes Clippings into the right subdomain |
| `ingest`, `lint` | per-subdomain (`research/.claude/skills/`) | domain-specific — operates on one wiki at a time, needs that wiki's schema |

This mirrors how you actually use them: launch your LLM agent at the vault root for triage, then `cd research/` and start a fresh session for ingest/lint. Each session sees exactly the skills relevant to its scope.

**If you only manage one wiki** (e.g., just a research vault, no separate interview-prep), put everything at the root: move `ingest` and `lint` into the root-level `skills/` and skip the subdomain structure. Tell your LLM to do this.

## Quick start

### With Claude Code

```bash
git clone <this-repo> llm-wiki-vault
cd llm-wiki-vault

# Bootstrap the bookkeeping files (gitignored, so you create them yourself).
# Tell Claude:
claude
> Bootstrap research/index.md and research/log.md per schemas/bookkeeping.md.
> Same for interview-prep/.
```

That's it — `.claude/` symlinks are already wired, so the next time you say "triage" Claude finds the skill.

To start using it: drop a file (a paper PDF, a saved web page) into `Clippings/`, then say "triage". When triage routes the file into `research/raw/<sub>/`, `cd research/` and say "ingest <filename>" to fold it into the wiki.

### With other agent frameworks

The skill files are plain markdown — see [INTEGRATION.md](INTEGRATION.md) for how to wire them up in Cursor, Aider, or your own LangGraph / Anthropic SDK agent.

## Languages

All skill / agent / schema files default to English output (tables, summaries, JSON `reason` fields, etc.). To switch the output language to Chinese / Japanese / etc., **just tell your LLM at the start of a session, or edit the language directive at the top of the relevant `SKILL.md` file** — that's the standard "edit your own skill" loop these workflows are built around.

## Inspiration

- Andrej Karpathy — [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) (the original gist)
- Anthropic Skills — [docs](https://docs.claude.com/en/docs/claude-code/skills.md)
- Obsidian — graph view + `[[wikilink]]` syntax
