<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# Launch Radar — 托管 Agent 模板

## 概述

定期扫描产品团队的发布追踪器（Jira、Linear 或 Asana），筛查未来几周内可能需要法律审查的产品发布。按照产品法律顾问的风险校准对每次发布进行分诊，并生成每周雷达备忘录：近期发布动态、需要法律关注的事项、触发标记的内容。与产品法律 Claude Code 插件中的 [`launch-watcher`](../../product-legal/agents/launch-watcher.md) Agent 同源——本目录是 `POST /v1/agents` 的托管 Agent Cookbook。

这是一个 **Cookbook，不是成品**。它开箱后无法直接使用。您需要将 MCP 连接器指向您的追踪器、加载风险校准配置、设置执行周期，并配置备忘录的发送目标。详见下方适配说明。

## ⚠️ 部署前须知

- **雷达分诊是路由决定，不是法律审查。** "需要审查"表示产品法律顾问应该查看；"仅供参考"不意味着该发布没有问题；"跳过"不等于放行该发布。审查完整雷达报告，不要只看被标记的条目——您可能在未被标记的条目中漏掉需要关注的内容。
- **风险分类使用您插件配置中的校准数据。** 如果校准数据过时，分诊结果也会过时。新产品线、新监管机构、新地理范围和新第三方依赖，在进入校准配置之前，雷达无法据此路由。
- **触发关键词列表有一定主观性。** 如果您的产品范围与默认配置不匹配（例如，您的业务侧重生物特征识别、受 FedRAMP 约束或涉及关键词未覆盖的方式处理未成年人数据），请在首次运行前重新调整，否则备忘录会遗漏它本应捕捉的情况。
- **追踪器工单是不可信输入。** PM 可以在标题或描述中写任何内容，攻击者也可以提交工单。分诊基于内容进行路由，不为工单内容背书。

## 部署

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export LINEAR_MCP_URL=... ATLASSIAN_MCP_URL=... ASANA_MCP_URL=... GDRIVE_MCP_URL=...
../../scripts/deploy-managed-agent.sh launch-radar
```

只设置您实际使用的追踪器对应的 MCP URL。编排器和 `tracker-reader` 会跳过未配置的 MCP。

## 引导事件

参见 [`steering-examples.json`](./steering-examples.json)。典型节奏是每周扫描一次（4–6 周前瞻期），加上当 PM 向产品法律顾问发消息询问"这有问题吗？"时的按需单工单分诊。

## 安全与交接

追踪器工单是不可信输入。产品经理可以在标题、描述或评论中输入任意文字——攻击者也可以提交工单。三层隔离机制：

| 层级 | 接触不可信追踪器内容？ | 工具 | 连接器 |
|---|---|---|---|
| **`tracker-reader`** | **是** | 仅 `Read`、`Grep` | Linear、Jira（atlassian）、Asana（只读） |
| `risk-classifier` / 编排器 | 否 | `Read`、`Grep`、`Glob`、`Agent` | 仅编排器：Linear / Jira / Asana / Drive（只读） |
| **`memo-writer`**（写入端） | 否 | `Read`、`Write`、`Edit` | 无 |

`tracker-reader` 返回长度受限、经 schema 验证的发布 JSON 列表。`risk-classifier` 无 MCP、无网络；它基于已验证的列表和用户校准文件运行。`memo-writer` 是唯一拥有写入权限的 worker，生成 `./out/launch-radar-<date>.md`。编排器不持有写入权限，也不自行解析原始工单正文。

**交接：** 当某次发布需要完整法律审查备忘录而非雷达条目时，编排器发出 `handoff_request` 交由 `launch-review` skill 在新会话中处理，而不是内联起草备忘录。`scripts/orchestrate.py` 负责路由。

## 适配说明

在实际使用前需要修改的内容：

- **追踪器指向。** 将 [`agent.yaml`](./agent.yaml) 和 [`subagents/tracker-reader.yaml`](./subagents/tracker-reader.yaml) 中的 `mcp_servers` 修改为您追踪器的 MCP URL。如果您只使用 Jira/Linear/Asana 中的一个，删除其他两个。如果您的追踪器不在列表中，换用您实际使用的 MCP，并相应更新 `tracker-reader` 的系统提示。
- **风险校准。** `risk-classifier` 从 `../../product-legal/CLAUDE.md`（由 `/product-legal:cold-start-interview` 生成）读取用户校准数据。如果您尚未运行 cold-start，请先运行，或在首次扫描前手动编写一个包含"通常阻止 / 通常需要处理 / 通常仅供参考"表格的 CLAUDE.md。没有校准数据时，分类器仅依赖关键词触发，噪音较多。
- **扫描周期与前瞻期。** 默认每周扫描，前瞻 6 周。您的发布节奏可能需要每天或每两周一次；较短的准备周期需要更长的前瞻期。在您的调度器（cron、Temporal、Airflow、EventBridge）中配置周期，不要在 Agent 内部配置。前瞻期通过引导事件传入。
- **交付渠道。** 备忘录默认输出到 `./out/`。如需同时或替代地发送到 Slack，可以：(a) 在 Cookbook 中添加 Slack MCP 并更新 `memo-writer` 在写入后发送，或 (b) 由编排层拾取 `./out/launch-radar-<date>.md` 后转发。这种模式将交付逻辑保留在 Agent 之外，便于测试；请选择适合您运营方式的方案。
- **触发关键词。** `launch-watcher` 系统提示中的关键词列表有一定主观性（COPPA、HIPAA、AI 供应商名称等）。删除与您产品无关的类别，添加特定领域术语（FedRAMP、PCI、HITRUST、TCPA、生物特征识别等），并对照校准表重新调整严重性阈值。修改后重新部署。
- **特权抬头。** `memo-writer` 会添加插件配置中的工作成果抬头。部署前请与您的总法律顾问确认具体措辞——不同司法管辖区存在差异。

## 您能获得什么，不能获得什么

- **您能获得：** 一个可用的清单、一个安全分层的流水线、一份将每次发布与其追踪器 URL 关联起来的备忘录，以及一条通向完整 launch-review skill 的交接路径。
- **您不能获得：** 一个生产就绪的 Agent。请将其指向您的追踪器，加载校准数据，设置执行周期，运行评估，并让产品法律顾问在信任它之前，对照他们自己对相同工单的判断审查最初几份输出。
- **您尤其不能获得：** 产品法律顾问的替代品。本 Agent 负责分诊。律师负责审查、标记和决策。备忘录中的每个"需要审查"条目是一个线索，不是一个裁决。
