<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# 法律领域托管 agent 模板

本仓库中的每个 agent **提供两种使用方式**：一种是今天即可安装的 Claude Code plugin（见仓库根目录下的各垂直目录），另一种是**Claude 托管 agent** 模板，由你的平台团队部署在你自己的工作流引擎之后。**同一个 agent，同一套 skill——选你喜欢的接入面。** 以下每个目录都是一个部署清单，引用了对应 plugin 中的权威系统提示和 skill，因此只存在一个事实来源。

这些是 **cookbook，不是成品。** 它们是起点。请根据你的文档管理系统、合同仓库、Slack 工作区、通知路由、审阅节奏进行适配。不做适配无法直接使用，这是有意为之。

运行 `../scripts/deploy-managed-agent.sh <slug>` 即可上传 skill、创建叶子 worker，并通过 `POST /v1/agents` 提交解析后的配置。每个模板都附带 [`steering-examples.json`](./reg-monitor/steering-examples.json) 和一份说明安全层级与交接方式的 agent 专属 README。

| Agent | 垂直 plugin | 监听对象 | CMA 触发事件 | 叶子 worker |
|---|---|---|---|---|
| [`reg-monitor`](./reg-monitor/) | regulatory-legal | 监管信息源（Federal Register、机构 RSS、TR） | `Check feeds as-of <date>, materiality: <threshold>` | feed-reader · materiality-filter · **digest-writer** |
| [`renewal-watcher`](./renewal-watcher/) | commercial-legal | 合同仓库（Ironclad），关注续约及取消截止日期 | `Scan renewals <X>–<Y> days out, flag playbook deviations` | repo-reader · deadline-calculator · **alert-writer** |
| [`diligence-grid`](./diligence-grid/) | corporate-legal | 虚拟数据室（Box、Datasite、Intralinks、iManage），监控新上传及批量审阅 | `Review folder <path> against schema <schema-id>` | doc-reader · extractor · normalizer · **grid-writer** |
| [`launch-radar`](./launch-radar/) | product-legal | 产品路线图/发布追踪器（Jira、Linear、Asana），筛选需要法律审阅的发布项 | `Scan tracker for launches in next <N> weeks` | tracker-reader · risk-classifier · **memo-writer** |
| [`docket-watcher`](./docket-watcher/) | litigation-legal | 法院案卷（Trellis、CourtListener），监控新文件、截止日期及待办事项 | `Watch docket <case-id> in <court>, matter <matter-id>` | docket-reader · deadline-mapper · **tracker-writer** |

**加粗**的叶子 = 唯一拥有 `Write` 权限的 worker。

## 清单与 API 的对应关系

`agent.yaml` 文件使用 `POST /v1/agents` 的真实字段名，并包含几个由部署脚本解析的便捷写法：

| 清单写法 | 解析结果 |
|---|---|
| `system: {file: ../../<plugin>/agents/<agent>.md, append: "..."}` | `system: "<内联内容 + append>"` |
| `system: {text: "..."}` | `system: "<text>"` |
| `skills: [{from_plugin: ../../<plugin>}]` | 上传该目录下的所有 `skills/*` → `[{type: custom, skill_id: ...}, ...]` |
| `skills: [{path: ../../...}]` | `skills: [{type: custom, skill_id: <uploaded-id>}]` |
| `callable_agents: [{manifest: ./subagents/x.yaml}]` | `callable_agents: [{type: agent, id: <created-id>, version: latest}]` |

> **研究预览：** `callable_agents`（多 agent 委托）仅支持**一级委托**。编排器可以调用 worker；worker 不能再调用下级 subagent。

## 跨 agent 交接

命名 agent 之间不互相直接调用。当一个 agent 需要另一个（例如 `launch-radar` 发现某个发布需要完整审阅备忘录）时，它在输出中发出 `handoff_request`；[`../scripts/orchestrate.py`](../scripts/orchestrate.py)（或你自己的事件总线）将其作为新的触发事件路由到目标会话。参考脚本对目标进行了硬编码白名单验证，并对有效载荷进行 schema 校验。

## 安全模型

法律文件和法院文书是**不可信输入**。每个 cookbook 采用三层 worker 隔离：

1. **Readers**（读取层）接触不可信文件，仅拥有 `Read`/`Grep` 权限——无 MCP、无 Write、无网络。它们返回长度受限的结构化 JSON。文件中嵌入的任何指令均是数据，而非命令。
2. **Analyzers**（分析层）从 readers 接收结构化 JSON，依据用户配置应用规则，拥有 MCP 读权限用于核实。无 Write。
3. **Writers**（输出层）生成最终产出，是唯一拥有 `Write` 权限的层级。它们从不接触原始文件。

编排器不持有 Write，也不读取原始文件。它只负责路由，不处理内容。

## 工作成果与特权保护

这些 agent 正常部署后产出的内容均为**律师工作成果**。每个清单的 headless append 会指示 agent 在输出前加上用户 plugin 配置中的工作成果标头。部署前请与你的法律团队确认标头内容。如果你的部署会处理不应被留存的材料，请在启用前检查 Anthropic 的数据留存设置以及你自己的存储留存策略。

## 你能得到什么，不能得到什么

- **能得到：** 可用的清单结构、具有合理安全层级的参考架构、在 Claude Code plugin 中经过验证的 skill，以及触发事件示例。
- **不能得到：** 生产就绪的 agent。你需要将 MCP 连接器对接到*你的*系统，设置执行节奏，配置通知路由，针对你的实践情况调整提示词，并在信任输出之前进行自有评估。
- **尤其不能得到：** 律师的替代品。这些 agent 负责监控、提取和起草。律师负责审阅、核实和决策。
