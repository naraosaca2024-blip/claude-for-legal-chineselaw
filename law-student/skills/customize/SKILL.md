---
name: customize
description: >
  Guided customization of your law-student study profile — change one thing
  without re-running the whole cold-start interview. Adjust current classes,
  learning style, outline preferences, bar prep subjects, seed materials,
  or study session cadence. Use when the user says "change my [thing]",
  "add a class", "update my profile", "new semester", or "customize".
argument-hint: "[section name, or describe what you want to change]"
---

<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->


# /customize

## 何时运行

用户输入了 `/law-student:customize`。他们想要更改学习档案中的某些内容——课程、学习风格偏好、bar prep 科目——而无需重新运行整个冷启动访谈，也无需手动编辑 YAML。

## 做什么

1. **阅读配置。** 阅读
   `~/.laude/plugins/config/claude-for-legal/law-student/CLAUDE.md`。
   如果插件配置不存在或仍包含 `[PLACEHOLDER]`
   值，说：

   > 你尚未运行设置。首先运行 `/law-student:cold-start-interview`——customize 用于调整你已有的档案。

2. **显示可自定义地图。** 列出档案中的内容，分组，并附带当前值的一行摘要：

   - **学生档案** —— 姓名、学校、年级（1L/2L/3L/LLM）、bar 司法管辖区、注册的诊所或期刊
   - **当前课程** —— 课程名称、教授、教学大纲路径、考试格式（闭卷/开卷、论文/MBE/混合）、cold-call 风格
   - **学习风格** —— Socratic 与摘要、你想要多少反击、插件是重写你的工作还是仅结构性批评
   - **大纲偏好** —— 大纲格式（IRAC/CREAC/案例摘要风格）、规则详细程度、是否包括政策讨论、保存的大纲模板
   - **Bar prep** —— 哪个考试（UBE/州）、轮换科目、薄弱科目标记、MBE 与论文节奏
   - **种子材料** —— 案例书路径、先前大纲、评分论文、旧考试、MBE 套题、教学大纲、论文
   - **学习工作流** —— 会话长度、flashcard Leitner 分桶时间表、考试预测节奏、cold-call prep 时机
   - **集成** —— 文档存储 / flashcard 应用（如有）状态、备用方案

3. **询问他们想要更改什么。**

   > 你想调整什么？选择一个部分，或用自己的话描述更改。

4. **进行更改。** 显示当前值，询问新值，解释下游更改内容，确认，然后将其写入配置。

   示例：
   - *添加新课程：* "`/outline-builder` 将为此课程搭建新大纲。`/flashcards` 将添加新的主题存储桶。当你为此课程调用 `/cold-call-prep` 时，它会询问座位和主题。"
   - *学习风格 Socratic → summary-first：* "`/socratic-drill` 不会要求你先回答——它会展示规则和示例，然后测试你的应用。"
   - *添加 bar 科目：* "`/bar-prep-questions` 将在轮换中包括此科目，如果你将其标记为薄弱，则权重更高。"

5. **关闭。**

   > 完成。你的下一个输出将反映更改。还有其他吗？你可以随时运行 `/law-student:customize`。

## 护栏

- **永不删除部分。** 如果用户想要"删除"课程，提供将其标记为 `[Archived — retain seed materials]` 并解释 flashcard 和大纲行为的变化。
- **标记内部不一致。** 如果更改会使档案不一致（例如，"summary-first"学习风格 + "maximum pushback"Socratic 设置），标记紧张关系。
- **标记护栏降级。** `/legal-writing` 和 `/irac-practice` 上的"不重写你的写作"规则是承重的——skill 的价值是结构性反馈，而不是代写。如果用户要求关闭此功能，确认他们理解插件不会为他们撰写作品。
- **一次更改一个。** 不要重新询问整个访谈。
