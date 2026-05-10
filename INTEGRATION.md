# Integration Guide

The skills in this repo are written as `SKILL.md` files (Anthropic Skill format), but their **content** — the steps, templates, judgment criteria — is plain markdown that any LLM agent can adapt. This guide shows how to wire them into different agent frameworks.

## Claude Code (default)

Already wired. After `git clone`, the symlinks under `.claude/` make every skill discoverable to Claude Code:

- Vault root: `triage` (and the `triage-classifier` subagent)
- `cd research/`: `ingest`, `lint`
- `cd interview-prep/`: bring your own (see "Bring your own subdomain skills" below)

Bootstrap step (one-time): see [schemas/bookkeeping.md](schemas/bookkeeping.md) — `index.md` and `log.md` are gitignored, so you create them after cloning.

## Generic LLM agents (LangGraph, AISDK, Anthropic SDK directly)

Each `SKILL.md` is a self-contained workflow description. Treat the file as a **system prompt** (or a `<workflow>` block within one):

```python
# Example with the Anthropic SDK
import anthropic

client = anthropic.Anthropic()

with open("skills/triage/SKILL.md") as f:
    triage_workflow = f.read()

resp = client.messages.create(
    model="claude-opus-4-7",
    system=f"You are a triage agent. Follow this workflow:\n\n{triage_workflow}",
    messages=[{"role": "user", "content": "triage Clippings/"}],
    tools=[...]  # provide Read, Bash, your own subagent dispatch tool
)
```

The `agents/triage-classifier.md` file is similarly a system prompt for a small worker model — use Haiku for cost, give it `Read` + `Bash` only.

For ingest and lint, the workflow assumes the LLM can call file-edit tools (Read, Write, Edit, Bash). The skill steps explicitly call out tool-result discipline ("absence of error ≠ success"), which is good practice for any agent runtime.

## Cursor

1. Copy `skills/triage/SKILL.md` to `.cursor/rules/triage.md` (or include via `@-mention`).
2. Cursor doesn't have a native "skill dispatch" mechanism, so for triage you'd inline the classifier logic instead of dispatching subagents — modify the skill to do classification serially in a single agent loop.
3. The schemas (`schemas/research-wiki.md` etc.) work as-is when included in the chat context — they describe the page format the agent should produce.

## Aider, Continue.dev, Cline, etc.

Same pattern as the generic case — the `SKILL.md` files are workflow prompts, the schema files are format specs. Each tool has its own way of pinning context (`.aider.conf.yml`, `.continuerc.json`, `.clinerules`); point it at the schema file relevant to your current working directory.

## Bring your own subdomain skills

The example layout has `research/` (full-fat) and `interview-prep/` (lighter). The schema for `interview-prep/` describes the operations (ingest, lint, sweep) but doesn't ship dedicated `SKILL.md` files for them — adapt the research-wiki ones, or write your own:

```
interview-prep/.claude/skills/
├── ingest/SKILL.md   # adapted from skills/ingest/SKILL.md
└── lint/SKILL.md     # adapted from skills/lint/SKILL.md
```

When you write them, follow the same structure: clear preconditions, step-numbered workflow, explicit verify step, English output by default.

## Why framework-agnostic?

The Anthropic Skill format is just frontmatter + markdown body. The frontmatter (`name`, `description`, optional `model` / `tools`) is what Claude Code reads to discover and route. The body is plain prose that any LLM can follow.

Putting these files at the top-level `skills/` (instead of burying them under `.claude/skills/`) signals: this is content, not Claude Code config. Claude Code happens to use them via the `.claude/` symlinks; other agents can use them by reading the markdown directly.
