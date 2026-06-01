<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# Reg Monitor — 托管 Agent 模板

## 概述

按计划检查监管信息源，按部署团队的重要性阈值进行筛选，对始终重要的条目执行快速政策库差距检查，并生成摘要。与监管法律 Claude Code Agent [`reg-change-monitor`](../../regulatory-legal/agents/reg-change-monitor.md) 以及 [`reg-feed-watcher`](../../regulatory-legal/skills/reg-feed-watcher) / [`policy-diff`](../../regulatory-legal/skills/policy-diff) skill 同源——本目录是 `POST /v1/agents` 的托管 Agent Cookbook。

## ⚠️ 部署前须知

- **摘要条目是经筛选的线索，不是法律结论。** 重要性筛选器应用可配置的阈值，而非法律判断。Agent 分类为"信息性"的监管变化，对您的业务仍可能是实质性的；Agent 标记为"重要"的变化，最终可能并不适用于您。请审查每份摘要；由执照律师决定某条目是否需要行动、披露、政策修订或上报。
- **政策差距检查是初步审查，不是适用性的法律评估。** 差距界面使用启发式规则对照您的政策库比较新监管文本。"差距"是供律师评估的线索；"对齐"的结果不等于合规认证。
- **重要性阈值是您的校准，不是法律规定。** 如果您的 `## Materiality threshold` 部分过时或是针对不同风险状态调整的，分诊结果也会过时。启用定时运行前请重新确认。
- **监控名单是您声明的覆盖范围。** 不在监控名单上的监管机构仍可能发布对您有实质影响的内容。遗漏一个监管机构是配置缺陷，不是信息源缺陷。

## 部署

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export GDRIVE_MCP_URL=...
../../scripts/deploy-managed-agent.sh reg-monitor
```

## 引导事件

参见 [`steering-examples.json`](./steering-examples.json)。默认每周扫描使用第一个示例。其他两个示例分别覆盖对特定进展的定向深度检查，以及对已标记条目的差距分析。

## 安全与交接

监管信息源内容（联邦公报条目、机构 RSS 推送、付费信息源告警）是**不可信输入**。三层隔离机制：

| 层级 | 接触不可信文档？ | 工具 | 连接器 |
|---|---|---|---|
| **`feed-reader`** | **是** | 仅 `Read`、`Grep`、`WebFetch` | 无 |
| `materiality-filter` / 编排器 | 否 | `Read`、`Grep`、`Glob`、`Agent` | gdrive（仅编排器） |
| **`digest-writer`**（写入端） | 否 | `Read`、`Write`、`Edit` | 无 |

`feed-reader` 返回长度受限、经 schema 验证的 JSON。`materiality-filter` 是对该 JSON 和磁盘上的监管法律配置进行纯计算——无 MCP、无网络。`digest-writer` 生成 `./out/reg-digest-<YYYY-MM-DD>.md` 并发出 `handoff_request` 用于 Slack 投递。

**交接：** 编排器将 `digest-writer` 发出的 `handoff_request` 路由到 Slack 发送 worker，使用部署团队团队风格配置中指定的频道。Agent 本身不直接发送 Slack 消息。

**不提供保证：** 本 Agent 呈现变化并标记潜在政策差距；由律师决定某项监管变化是否需要采取行动以及由谁负责响应。

## 适配说明

在将输出接入您的工作流之前：

- **将 `feed-reader` 指向您的信息源。** 默认目标是联邦公报（免费公共 API，无需 MCP）。如果您的机构订阅了付费监管信息源（如 Bloomberg Law）或使用直接机构 RSS，请将端点添加到 feed-reader 的 web_fetch 允许列表，并调整编排器的扫描计划。如果您只有免费来源，仅使用联邦公报 API 也是可行的。
- **配置摘要交付频道。** digest-writer 发出的 `handoff_request` 会指定一个 Slack 频道。编排器从您的监管法律配置的**团队风格 → 监管摘要**字段中读取该频道。请在首次定时运行前设置，否则交接会失效。希望通过邮件接收摘要或保存到 Confluence 页面的团队，应在编排器允许列表中替换交接目标。
- **调整重要性阈值。** materiality-filter 读取您配置中的 `## Materiality threshold` 部分——始终重要 / 值得审查 / 仅供参考。启用定时运行前请确认各层级反映您当前的风险状态；阈值设置过低会导致摘要充斥大量条目，过高则会遗漏有截止日期的义务。
- **更新监控名单。** materiality-filter 还会读取 `## Regulators we watch` 表。随着您的业务范围变化，相应增减监管机构。
- **确认工作成果抬头。** `agent.yaml` 中的无头追加指令会让 Agent 在报告前添加您配置中的工作成果抬头。启用前请与您的总法律顾问核实抬头措辞。
- **执行周期。** 默认每周一次。活跃的监管环境（金融服务规则制定周期、跨境 AI 监管）可能需要每天执行。执行周期在您自己的工作流引擎中设置——Cookbook 不自行调度。
