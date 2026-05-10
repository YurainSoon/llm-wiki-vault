# Vault Routing Schema

This is the schema for a multi-domain LLM-wiki vault. **The root only routes between subdomains; it holds no synthesized content of its own.** All synthesized content lives under each subdomain's `wiki/`.

This file is mounted as `CLAUDE.md` at the vault root in the default layout (via symlink), so any LLM agent that auto-loads `CLAUDE.md` will see it.

## Subdomains

The example layout in this repo has three subdomains:

- `research/` — knowledge base for your research field (in this example, LLM research)
- `interview-prep/` — job interview preparation
- `life/` — personal notes (no wiki, just freeform)

You can rename these or add others as long as each subdomain is self-contained. Each subdomain that uses the wiki pattern carries its own `CLAUDE.md` (schema) and its own `.claude/skills/` (e.g., `ingest`, `lint`) — see "Hierarchical skill placement" in the README.

## Triage workflow

Trigger: user says "triage", "process clippings", or "clean up the inbox".

For each item under `Clippings/`:

1. **Keep or delete**: is it worth keeping? Give one sentence of reasoning (based on content quality, relevance to user's current direction).
2. **Subdomain**: if keep, which subdomain does it belong to?

   The single rule (binary):
   > **"If I already had a job offer and stopped job-hunting, would I still keep this?"**
   - Yes → `research/`
   - No → `interview-prep/`

   **Anything with independent technical or academic value goes to `research/`** — even if it's "also useful for interviews". `interview-prep/` only holds procedural interview content (problems, interview experiences, company processes).

3. **Subdirectory**: pick the specific `<domain>/raw/<sub>/` to land in.

**Protocol**: batch propose → single confirmation → batch execute.

- List all files under `Clippings/`, then **dispatch `triage-classifier` subagents (Haiku) in parallel** — each reads one item and returns `{action, destination, reason, confidence}`.
- Aggregate proposals into one table for the user. **Low-confidence items go in a separate "⚠ Needs your judgment" section at the bottom** — never mix them into the batch.
- Wait for explicit user confirmation (`go` / `execute`) before any `mv` / `rm`.
- For each kept file, append one line to the **receiving subdomain's** `log.md`: `## [YYYY-MM-DD] triage | <filename> → <destination>`.

Detailed workflow: see `skills/triage/SKILL.md`.

## Don't

- Don't create wiki content at the root.
- Don't link across subdomains — they are disjoint domains by design.
- Don't treat `Clippings/` as permanent storage — it's an inbox. After triage it should be empty.
- Don't decide ambiguous cases for the user — ask.

## User profile

(Optional, but useful for the LLM to tailor its work.) Tell the LLM about your role and current focus. Example for the original author of this template:

> CS PhD focused on LLM research. Reading papers/blogs/Twitter, writing papers, prepping for the job market. The research wiki is the main workspace.

Adapt this to your own context.
