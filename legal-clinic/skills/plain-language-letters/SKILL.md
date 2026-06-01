---
name: plain-language-letters
description: >
  参考：已弃用——常规通信使用 `/client-letter`，实质性更新使用 `/status client`。
  在 v2 重建期间拆分为两个更专注的 skills。保留为重定向用于迁移。
user-invocable: false
---

<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->


# [已弃用] 平实语言信函 → 参见 `/client-letter` 和 `/status client`

此 skill 在 v2 重建期间被拆分：

- **常规通信**（预约确认、文档请求、简短的"我们已提交"更新）→ `skills/client-letter/` — 使用 `/client-letter [type]`

- **实质性客户状态更新** → `skills/status/` 客户面向模式 — 使用 `/status client`

两者都应用来自 CLAUDE.md 的平实语言标准（阅读水平、无行话）。

完整工作流程参见相应的 SKILL.md 文件。
