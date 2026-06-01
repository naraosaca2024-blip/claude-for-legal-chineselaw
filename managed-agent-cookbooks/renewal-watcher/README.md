<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# Renewal Watcher — 托管 Agent 模板

## 概述

扫描合同存储库中即将到来的续约和取消截止日期，与团队的行动手册进行交叉比对，标记存在即将到期截止日期、偏离行动手册和升级触发条件的合同，并生成告警报告。与商业法律 Claude Code Agent [`renewal-watcher`](../../commercial-legal/agents/renewal-watcher.md) 以及 [`renewal-tracker`](../../commercial-legal/skills/renewal-tracker) skill 同源——本目录是 `POST /v1/agents` 的托管 Agent Cookbook。

这是一个 **Cookbook，不是成品**。它默认以 Ironclad 作为合同生命周期管理系统（CLM），因为与其配对的插件也作此假设；使用 Agiloft、Ironclad 替代品、iManage 或 Google Drive 存放签署版 PDF 的团队，应相应替换 MCP 端点。

## ⚠️ 部署前须知

- **从合同元数据中提取的取消截止日期和续约条款可能有误。** CLM 元数据会与已执行文件产生偏差——修正案签署后未重新导入、实际生效日期与签署日期不同、自动续约机制有时标记有误。在依据计算出的截止日期作出终止或续约决定之前，执照律师必须对照签署协议及所有修正案进行核实。
- **升级路由遵循已配置的矩阵；它不作升级判断。** 被标记的行动手册偏差在具体情境下仍可能是可接受的；未被标记的条款也可能仍需关注。该矩阵是路由器，不是审查者。
- **平静的周并不意味着清洁的周。** 未被呈现的合同可能在 CLM 中缺失、标记有误，或已超过通知窗口期但元数据未反映这一情况。全部清洁的页脚意味着 Agent 运行了，不意味着不需要做任何事。

## 部署

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export IRONCLAD_MCP_URL=...
export GDRIVE_MCP_URL=...
# 可选——如果签署协议存放于此，请在清单中启用
export IMANAGE_MCP_URL=...
export DOCUSIGN_MCP_URL=...
../../scripts/deploy-managed-agent.sh renewal-watcher
```

## 引导事件

参见 [`steering-examples.json`](./steering-examples.json)。默认的周一早晨扫描使用第一个示例。其他两个示例分别覆盖按交易对手范围的临时运行和签署后偏差检查。

## 安全与交接

合同文本、交易对手消息和 CLM 评论均为**不可信输入**。三层隔离机制：

| 层级 | 接触不可信文档？ | 工具 | 连接器 |
|---|---|---|---|
| **`repo-reader`** | **是** | 仅 `Read`、`Grep` | ironclad、gdrive（只读）；imanage 默认关闭 |
| `deadline-calculator` / 编排器 | 否 | `Read`、`Grep`、`Glob`、`Agent` | 无 |
| **`alert-writer`**（写入端） | 否 | `Read`、`Write`、`Edit` | 无 |

`repo-reader` 返回长度受限、经 schema 验证的 JSON。`deadline-calculator` 是对该 JSON 和磁盘上的行动手册配置进行纯计算——无 MCP、无网络。`alert-writer` 生成 `./out/renewal-alerts-<YYYY-MM-DD>.md` 并发出 `handoff_request` 用于 Slack 投递。

**交接：** 编排器将 `alert-writer` 发出的 `handoff_request` 路由到 Slack 发送 worker，使用部署团队团队风格配置中指定的频道。Agent 本身不直接发送 Slack 消息。

**关联 Agent：** 当需要签署后偏差检查时，`handoff_request` 也可路由到 [`deal-debrief`](../../commercial-legal/agents/deal-debrief.md)；当续约时的偏差积累成规律时，可路由到 [`playbook-monitor`](../../commercial-legal/agents/playbook-monitor.md)。命名 Agent 之间绝不直接互相调用——路由是编排器的职责。

**不提供保证：** 本 Agent 推荐行动；律师决定是否取消、重新谈判或让续约自动执行。

## 适配说明

在将输出接入您的工作流之前：

- **指向您的 CLM。** `IRONCLAD_MCP_URL` 是默认值。如果签署协议存放于 iManage，请在 `agent.yaml` 和 `subagents/repo-reader.yaml` 中将 `imanage` 的 `default_config: { enabled: true }` 并设置 `IMANAGE_MCP_URL`。如果存放于 Google Drive 文件夹，依赖 `gdrive` 和 repo-reader 的备用搜索路径。如果存放于没有公共 MCP 的 CLM（Agiloft、Conga），请接入自定义连接器并更新 MCP server 块。
- **设置 Slack 频道。** alert-writer 发出的 `handoff_request` 会指定一个 Slack 频道。编排器从您的行动手册配置的**团队风格 → 续约告警**字段中读取该频道。请在首次定时运行前设置，否则交接会失效。
- **调整前瞻窗口。** deadline-calculator 的默认层级为逾期 / 30 / 60 / 90 / 180 天。如果您的续约周期较短（一年以下的 SaaS 订单）或较长（具有 12 个月通知窗口的多年企业级 MSA），请在 deadline-calculator 提示词和 `alert-writer.yaml` 的相应章节中调整层级阈值。
- **调整升级矩阵。** deadline-calculator 读取您行动手册中的升级矩阵，以决定是否设置 `escalation_needed: true` 以及路由对象。启用定时运行前请确认矩阵反映您当前的审批权限（谁签批允许自动续约失效，谁签批超过一定金额阈值的重新谈判）。[`escalation-flagger`](../../commercial-legal/skills/escalation-flagger) skill 已加载到 `alert-writer` 中用于格式化。
- **确认工作成果抬头。** `agent.yaml` 中的无头追加指令会让 Agent 在告警报告前添加您行动手册配置中的工作成果抬头。启用前请与您的总法律顾问核实抬头措辞。
- **执行周期。** 默认每周一次。高量团队应每天运行；小团队可以每月运行。执行周期在您自己的工作流引擎中设置——Cookbook 不自行调度。
