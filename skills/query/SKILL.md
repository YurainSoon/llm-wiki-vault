---
name: query
description: Read the research wiki and answer the user's substantive question with a prose synthesis plus a bottom citation list. Use for definitional / comparative / causal / explanatory questions while working in research/ — "what is X", "how does Y compare to Z", "why W", "explain ABC". This is the default behavior for any non-operational question in research/. Skip for operational requests (ingest/lint/triage), chit-chat, or meta on the previous turn.
---

# Query

Single-question wiki retrieval and synthesis. One query = read index + drill candidates → synthesize prose answer → optionally fold back at session boundary.

Pattern: **index-first retrieval → adaptive drill-down → prose answer + bottom citations → fold-back at session boundary**.

User-facing output defaults to **the language the user writes in** — mirror the user's input language. To pin a fixed language instead, tell the LLM at the start of the session, or edit this line.

## Preconditions

- Working directory is `research/` (skill assumes the schema in `research/CLAUDE.md`).
- The user's message is a substantive question (see §1).

## Steps

### 1. Decide whether to fire

Fire when the user's message is a **substantive question** plausibly within the wiki's domain — definitional / comparative / causal / synthesis-type ("what is X", "how does Y differ from Z", "why did W happen", "explain ABC").

Do NOT fire when:
- **Operational** ("ingest this", "run lint", "create a page", "edit X")
- **Meta / chit-chat** ("what did you just say", "再说一遍", "thanks", "好的下一个")
- **Cross-subdomain** (interview-prep / life) — wiki is research-only
- **Trivial knowledge** wiki obviously won't have ("今天几号", "2+2 等于几")

If unsure, fire. Over-firing costs one `index.md` read; under-firing is the failure mode that made this skill exist.

### 2. Read `index.md`

Use the Read tool. `index.md` is the catalog of all wiki pages with one-line summaries — the designed entry point for any query (per Karpathy gist's "read index first, then drill in").

### 3. Pick candidate pages and decide depth

Match the question against `index.md` page names and one-liners. Adapt depth to question shape:

| Question type | Behavior |
|---|---|
| **Navigation / overview** ("what's in the wiki about X", "list pages on Y") | `index.md` alone may suffice — only drill if user wants depth |
| **Definitional / comparative / causal** (most cases) | Drill into 2–7 candidate pages |

Aim for breadth, not exhaustion — if 10+ pages look relevant, the question is too broad; ask user to narrow first.

### 4. Read candidates in parallel

Call the Read tool in parallel for all candidates in one assistant turn. Do not serialize.

### 5. Follow one hop on `[[link]]`

While reading candidates, note `[[link]]` pointers to **unread but clearly relevant** pages (judged from anchor + surrounding sentence). Read those in a second parallel batch. **One hop only** — do not chain further.

If after one hop the picture is still thin, fall back to:

```bash
grep -ril "<keyword>" wiki/
```

This is exceptional, not routine — well-summarized `index.md` should handle most cases.

### 6. Synthesize the answer

Write **prose** in natural language — read like a teacher explaining, not a footnoted document.

**Format rules:**

- **No inline `[[link]]` in the body.** All citations go to the bottom list.
- **No `[wiki]` / `[parametric]` tags.** Absence of citation = parametric or general knowledge; presence in citation list = informed by wiki. The user can tell from context.
- **Surface contradictions and non-consensus**: when wiki pages disagree with each other or with conventional wisdom, say so in the body using natural-language hedging ("本 vault 的整理认为 X，这和通常说法 Y 有出入" / "wiki 内部对这点的判断不一致"). The user explicitly wants to absorb these tensions, not have them smoothed away.
- **Wiki gaps**: if part of the question isn't covered, fill from parametric knowledge naturally — no special tag, the missing citation signals it.
- **Length matches the question**: short question → one paragraph; multi-part question → may run longer. Default to ≤ one screen.

After the prose, append the citation list:

```
---
**引用**
- [[wiki/<dir>/<slug>]]
- [[wiki/<dir>/<slug>]]
```

**Citation rules:**
- Only list pages that were **read** and **meaningfully informed** the answer.
- Order by centrality to the answer, not by directory.
- Do not pad with adjacent-but-unread pages.
- If no wiki page meaningfully shaped the answer (rare; means wiki is silent), say so in the body and **omit the citation block entirely** — an empty `**引用**` heading is worse than none.

### 7. Fold-back trigger (session boundary only)

After answering, **do not** immediately offer fold-back. Wait for one of:

| Signal | Action |
|---|---|
| User explicitly says "save this" / "fold back" / "回填" | Enter fold-back flow (§8) immediately |
| User starts a query on a **clearly different** topic | Before answering the new query, one-line offer fold-back on the previous one |
| User signals wrap-up ("明白了" / "thanks" / "ok 下一个" / "够了" / "明天再聊") | One-line offer fold-back |
| User keeps drilling on the same topic | Do NOT offer — discussion is still live |

**Only offer fold-back if the discussion produced new synthesis** — a comparison, a connection, a clarification not already in any read page. Pure retrieval / restatement → don't offer.

Offer format (one line, not a wall):

```
要不要把刚才关于 <topic> 的讨论回填进 wiki？候选：新建 wiki/<dir>/<slug> / 追加到 [[existing-page]] / 不回填
```

### 8. Fold-back execution (if user accepts)

Reuse `ingest` skill's plan → confirm → execute → verify pattern, adapted:

1. **Propose**: specific new slug + body draft, OR specific edits to listed existing pages. Be concrete (give the actual prose, not "I'll add a paragraph about X").
2. **Wait for `go`**. Ambiguous → ask again.
3. **Execute writes** in parallel where possible.
4. **Verify** per ingest §8.5 — for each change, grep that it landed.
5. **Update `index.md`** ONLY if a new page was created. Pure updates to existing pages don't change the catalog.
6. **Append to `log.md`**:
   ```
   ## [YYYY-MM-DD] query-foldback | <topic> → <K> pages
   - New: wiki/<dir>/<slug>
   - Updated: wiki/<dir>/<slug>
   ```
7. **Report** concisely: what was created / updated, total pages touched.

## Style

- **Prose, not lists**, in the body. Bullets are for skills and catalogs, not for explanations.
- **Teacher tone**: explain mechanisms and the *why*, not just facts.
- **Surface wiki's voice**: where the wiki makes a specific judgment ("本 vault 把这条标 alpha", "wiki 内部对 X 有矛盾"), let that voice through — don't launder it into generic statement.
- **One screen** per answer by default. Longer questions probably should be split.
- **Don't pretend coverage you don't have**: when wiki is silent on part of the question, say so in natural language.

## Edge cases

- **Empty wiki / no candidate pages from `index.md`**: state explicitly that wiki has no page on this, answer from parametric knowledge like a teacher would, then end the response with: "wiki 里目前没这个主题的页，要不要 ingest 一个源建一页？"
- **Question crosses subdomains**: only answer the research portion via wiki; for non-research portion say "这部分属于另一个 subdomain，wiki 这边不处理"。 Do not link across.
- **Wiki self-contradiction discovered**: surface both claims in the body — do not silently pick one. After answering, mention "这是 lint 该看的，要不要顺便记下" — but do not auto-trigger lint.
- **Page is stale** (`last-updated` is old + claims look superseded by newer sources also in wiki): mention in the body ("wiki 这页 last-updated 是 YYYY-MM-DD，下面的数字可能已经被更新的源推翻"). Do not silently use stale claims.
- **Operational vs substantive ambiguity** ("how does ingest work" — skill mechanics or concept?): pick the more likely reading; if truly ambiguous, ask one clarifying question before firing.
- **Question too broad** ("tell me everything about agents"): ask user to narrow before firing. Do not dump the index.
- **User explicitly asks for parametric / non-wiki answer** ("just tell me, don't look it up"): skip steps 2–5, answer directly, no citation block. Respect the override.

## Do not

- Do not embed `[[link]]` in the prose body. Citations go to the bottom list only.
- Do not tag sentences with `[wiki]` / `[parametric]`. Let context signal.
- Do not pad the citation list with pages you didn't actually read or that didn't shape the answer.
- Do not offer fold-back after every query — only at session boundaries (§7).
- Do not modify any file during query — query is read-only. Writes happen only in fold-back execution (§8).
- Do not dispatch subagents — synthesis must stay in one context (same nail as ingest).
- Do not invent wiki claims to fill gaps — if wiki is silent, say so.
- Do not silently smooth over contradictions in the wiki — the user wants to see them.
- Do not write to `raw/`. Ever.
