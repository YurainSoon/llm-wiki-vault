---
name: triage
description: Classify files in Clippings/ and route them to research/ or interview-prep/ subdirectories. Use at vault root when the user says "triage", "process clippings", or has accumulated unsorted clippings to process.
---

# Triage

Parallel classification of files in `Clippings/`, routing each to `research/raw/<sub>/` or `interview-prep/raw/<sub>/`.

Pattern: **dispatch subagents → propose batch → user confirms → main agent executes**.

User-facing output (table contents, summary report) is in **English** by default. To switch to another language, tell your LLM at the start of the session, or edit this file directly to change the language directive.

## Steps

### 1. List items

```bash
find Clippings -mindepth 1 -maxdepth 1 \( -type f -o -type d \)
```

Items can be either single files (most common) OR directories. Two common directory shapes:
- a multi-chapter **book** repo (honkit / mdbook source tree) → routes to `research/raw/books/`
- a cloned **source-code repo** kept verbatim (has `package.json` / `src/` / build config, not just prose `.md`) → routes to `research/raw/repos/` — distinct from `code-notes/`, which holds *notes about* a repo, not the repo itself

Treat each top-level item — file or directory — as **one** classification unit. Do not descend into directories to dispatch per-file classifiers.

If empty → report `Clippings/ is empty, nothing to triage` and stop.

### 2. Dispatch subagents in parallel

For each file, in **a single message**, issue multiple `Agent` tool calls with `subagent_type: "triage-classifier"` (loaded from `agents/triage-classifier.md`, or `.claude/agents/triage-classifier.md` if you're using Claude Code with the symlink layout). Each call passes the file's absolute path as input.

Each subagent returns a JSON object:

```json
{
  "file": "<absolute path>",
  "action": "keep" | "delete",
  "destination": "<absolute path to target subdir>" | null,
  "reason": "<one English sentence>",
  "confidence": "high" | "medium" | "low"
}
```

**Parallelism is critical.** Do not call subagents serially — issue all `Agent` tool uses in one assistant turn.

### 3. Collect

Gather all returned JSON proposals. Split into two groups:

- **Batch proposals** — `confidence` of `high` or `medium`
- **Needs human judgment** — `confidence` of `low`, OR any subagent that flagged ambiguity in its reason

### 4. Present

Output a **single table** to the user:

```
## Triage Proposal

| File | Action | Destination | Reason |
|---|---|---|---|
| foo.md | keep | research/raw/blogs/ | LLM agent architecture analysis |
| bar.md | delete | -- | Shallow content, duplicates existing material |
| baz.md | keep | interview-prep/raw/system-design/ | "Design X" problem genre |

## ⚠ Needs your judgment

| File | Why uncertain | Candidates |
|---|---|---|
| qux.md | Looks like both research and interview-prep | research/raw/blogs/ or interview-prep/raw/companies/ |

Reply `go` to confirm, or edit any row first.
```

### 5. Wait for explicit confirmation

Do NOT execute any `mv` or `rm` before the user confirms.

Accepted confirmation signals: `go`, `ok`, `confirm`, `execute`. Ambiguous replies (`looks fine`, `sure`) → ask again for an explicit signal.

If the user edits any row → apply the edit, re-render the table, wait for confirmation again.

### 6. Execute in batch

After confirmation:
- `mv` all keepers in parallel (multiple `mv` in one Bash call, or multiple parallel Bash calls)
- `rm` all deletes in parallel

### 6.5 Verify execution

After the batch, confirm filesystem matches intent. **Tool absence of error ≠ success** — always check.

For each kept file:
```bash
test -e "<destination>"   # target should exist
test ! -e "<source>"      # source should be gone
```

For each deleted file:
```bash
test ! -e "<source>"      # source should be gone
```

Batch into one Bash call by chaining checks (echo any failures). On any failure:

- **Auto-retry the failed `mv`/`rm` ONCE.** Re-run the original command, then re-check.
- If retry succeeds → continue to step 7, but record which ops needed retry (signal something fragile in the source/destination — e.g., permission, symlink, name collision).
- If retry also fails → **STOP**. Do not proceed to logs. Report which file(s), the original command, and the error to the user. Ask whether to debug / abort / proceed-with-only-the-successful-ones.

Never proceed to step 7 without all verify checks passing (or explicit user override).

### 7. Update logs

For each kept file, append one line to the **receiving subdomain's** `log.md` (NOT the root):

```
## [YYYY-MM-DD] triage | <filename> → <destination relative to vault root>
```

Use today's date in `YYYY-MM-DD` format.

### 8. Report

Concise summary:
- Moved N, deleted M
- Whether `Clippings/` is now empty (this is a healthy state — say so)
- Suggest next step: user can `cd research/` to run ingest

## Edge cases

- **Cloned source-code repo**: route the whole directory to `research/raw/repos/` (verbatim code). Do not split it into per-file notes — that's the ingest skill's job, and the synthesized notes land in `research/raw/code-notes/` / the wiki, not here.
- **Neither research nor interview-prep fits**: subagent returns low confidence. Surface in the ⚠ section. User decides (could go to a different subdomain like `life/`, stay in Clippings for later, or be deleted).
- **Binary / unreadable files**: subagent returns `confidence: low` with `reason: "unreadable"`. Surface in ⚠ section.
- **Filename mismatches content** (e.g., a PDF named `.md`): subagent flags in reason, sets confidence low.
- **Obvious media attachments** (image/audio/video standalone in Clippings): probably an attachment that got separated from a parent doc — flag for user.

## Do not

- Move or delete any file before user confirmation.
- Mix low-confidence items into the batch proposals — they belong in the ⚠ section.
- Read Clippings files yourself — let subagents do it. Keep main context clean.
- Write any wiki content — triage only routes; wiki updates are for the ingest skill.
