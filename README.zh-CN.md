# llm-wiki-vault

把 [Obsidian](https://obsidian.md) vault 结构化为 LLM 管理的知识库的一种实现，灵感来自 [Karpathy 的 LLM-wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)。

> **语言**：[English](README.md) · [中文](README.zh-CN.md)

## 这是什么

一套布局 + 几个 [Anthropic Skill](https://docs.claude.com/en/docs/claude-code/skills.md)，用来管理一个 LLM 维护的知识 vault：

- **你**：把源（web clipping、论文、talk）扔进 `Clippings/`（inbox），然后提问、做判断、定方向。
- **LLM**：读源、把源合成进每个子领域 `wiki/` 下密集互链的 entity 页、维护 bookkeeping（`index.md`、`log.md`）。

skill 实现四个操作：**triage**（把 inbox 路由进对应子领域）、**ingest**（一源触动多页，写之前先 plan + 确认）、**lint**（周期巡检 wiki 健康度）、**triage-classifier**（Haiku 并行分类的 subagent）。

skill 用 `SKILL.md` 格式（Anthropic Skill 规范），但**内容是 framework-agnostic 的**：任何能 dispatch subagent + 编辑文件的 LLM agent 都能 adapt。详见 [INTEGRATION.md](INTEGRATION.md)。

## 仓库布局

```
llm-wiki-vault/
├── README.md / README.zh-CN.md
├── INTEGRATION.md           # Claude Code / Cursor / 其他 agent 集成指南
│
├── skills/                  # ← skill 的 canonical 位置（framework-agnostic）
│   ├── triage/SKILL.md
│   ├── ingest/SKILL.md
│   └── lint/SKILL.md
│
├── agents/                  # ← subagent 的 canonical 位置
│   └── triage-classifier.md
│
├── schemas/                 # ← schema 文档（通过 symlink 挂为 CLAUDE.md）
│   ├── vault-routing.md     # = 根 CLAUDE.md
│   ├── research-wiki.md     # = research/CLAUDE.md
│   ├── interview-prep-wiki.md
│   └── bookkeeping.md       # index.md 和 log.md 的格式说明
│
├── CLAUDE.md → schemas/vault-routing.md
│
├── .claude/                 # Claude Code 的接线（symlink 指回 skills/、agents/）
│   ├── skills/triage → ../../skills/triage
│   └── agents/triage-classifier.md → ../../agents/triage-classifier.md
│
├── research/                # 一个示例子领域
│   ├── CLAUDE.md → ../schemas/research-wiki.md
│   ├── .claude/skills/lint → ../../../skills/lint
│   ├── .claude/skills/ingest → ../../../skills/ingest
│   ├── raw/                 # 不可变源（gitignored —— 你的私人笔记）
│   ├── wiki/                # 合成层（gitignored）
│   ├── index.md             # fork 后自己 bootstrap —— 见 schemas/bookkeeping.md
│   └── log.md
│
├── interview-prep/          # 另一个示例子领域
│   └── （同上）
│
└── life/                    # 一个不带 wiki 的子领域（gitignored）
```

**为什么用 symlink？** `skills/` / `agents/` / `schemas/` 是 single source of truth，而 `.claude/` 和子领域的 `CLAUDE.md` 又留在 Claude Code（以及其他自动加载 `CLAUDE.md` 的工具）期望的位置。改一处，两边同步。

## 层次化的 skill 放置

skill 分两层放置：

| Skill | 位置 | 原因 |
|---|---|---|
| `triage` | vault 根（`skills/triage/`） | 跨域 —— 把 Clippings 路由到正确的子领域 |
| `ingest`、`lint` | 子领域内（`research/.claude/skills/`） | 域特定 —— 一次只操作一个 wiki，需要该 wiki 的 schema |

这映射了你实际的使用方式：在 vault 根启动 LLM agent 做 triage，然后 `cd research/` 开新 session 做 ingest/lint。每个 session 只看到与其 scope 相关的 skill。

**如果你只管理一个 wiki**（比如只有 research，没有单独的 interview-prep），把所有 skill 都放根目录：把 `ingest` 和 `lint` 移进根的 `skills/`，跳过子领域结构。让你的 LLM 帮你做这个调整即可。

## 快速开始

### 用 Claude Code

```bash
git clone <this-repo> llm-wiki-vault
cd llm-wiki-vault

# Bootstrap bookkeeping 文件（gitignored，你自己生成）
# 让 Claude 干：
claude
> Bootstrap research/index.md and research/log.md per schemas/bookkeeping.md.
> Same for interview-prep/.
```

完事 —— `.claude/` 的 symlink 已经接好了，下次你说 "triage" Claude 就能找到 skill。

开始用：把一个文件（论文 PDF、保存的网页）丢进 `Clippings/`，然后说 "triage"。triage 把它路由到 `research/raw/<sub>/` 后，`cd research/` 说 "ingest <filename>" 把它消化进 wiki。

### 用其他 agent 框架

skill 文件就是 plain markdown —— 见 [INTEGRATION.md](INTEGRATION.md) 了解怎么在 Cursor、Aider、或者你自己写的 LangGraph / Anthropic SDK agent 里接入。

## 切换语言

所有 skill / agent / schema 文件默认输出英文（表格、总结、JSON `reason` 字段等）。要切成中文 / 日文 / 其他语言，**直接告诉你的 LLM 在 session 开头切换，或者修改对应 `SKILL.md` 顶部的 language 指令**——这就是这套工作流默认依赖的"改你自己的 skill"循环。

## 灵感来源

- Andrej Karpathy —— [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)（原始 gist）
- Anthropic Skills —— [docs](https://docs.claude.com/en/docs/claude-code/skills.md)
- Obsidian —— graph view + `[[wikilink]]` 语法
