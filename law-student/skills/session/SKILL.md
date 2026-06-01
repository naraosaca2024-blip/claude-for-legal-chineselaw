---
name: session
description: >
  Run a focused N-question study session on a subject — MBE, essay, or
  flashcards. Tracks performance and updates the study plan. Use when the
  user says "run me 10 questions on [subject]", "do a session on [subject]",
  "let's do 5 cards on [subject]", or wants to drill a fixed number of
  questions and have the plan adapt.
argument-hint: "<subject> <n> [--mbe | --essay | --flashcards]"
---

<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->


# /session

1. 解析 `$ARGUMENTS` —— 主题和 N。如果缺失，询问：
   > 什么主题，多少问题？（例如，`Evidence 10` 或 `Contracts 5 --essay`。）
2. 加载 `~/.claude/plugins/config/claude-for-legal/law-student/CLAUDE.md` → 司法管辖区、考试格式、薄弱学科。
3. 加载 `~/.claude/plugins/config/claude-for-legal/law-student/study-plan.yaml`（如存在）。阅读此主题的 `session_history` 以权倾向于学生薄弱的子主题。
4. 按方法标志路由：
   - `--mbe`（bar prep 科目的默认值）：加载 `bar-prep-questions` skill，运行 N 个 MBE 风格问题。应用司法管辖区处理（请参阅该 skill 的 `## Jurisdiction handling`）。将每个标记为 `[UBE/majority]` 或 `[state-specific]`。
   - `--essay`：加载 `bar-prep-questions`，运行 N 个论文提示。按论文模式 rubric 评分。
   - `--flashcards`：加载 `flashcards` skill，在 `--drill` 模式下运行 N 张卡片。
5. 每次运行 N 个问题。每次之后，解释对/错并在司法管辖区分歧时标记规则主体。
6. 会话结束时，写入会话结果：
   - 如果 `study-plan.yaml` 存在：根据 `study-plan` skill 的架构追加到 `session_history`。
   - 如果不存在：写入 `~/.claude/plugins/config/claude-for-legal/law-student/session-history.yaml`。
7. 报告：
   - 分数：X/N（百分比）
   - 未命中：附带子主题标签的列表
   - 本次会话的薄弱子主题
   - 与此主题上先前会话的模式（如历史记录有 2 次以上）
   - 计划现在推荐的下一步
