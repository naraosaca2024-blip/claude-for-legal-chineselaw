<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# Docket Watcher — 托管 Agent 模板

## 概述

监控活跃诉讼案件组合中的法院案卷动态。Trellis 覆盖州初审法院；CourtListener / PACER 覆盖联邦法院。针对每个活跃案件，Agent 拉取自上次检查以来的新提交文件，将文件类型映射到候选期限规则，与案件历史及未决事项进行交叉比对，并生成案卷状态报告和结构化期限清单。

与诉讼法律 Claude Code 插件中的 [`docket-watcher`](../../litigation-legal/agents/docket-watcher.md) Agent 同源——本目录是 `POST /v1/agents` 的托管 Agent Cookbook。

## ⚠️ 部署前须知

- **计算出的期限是线索，不是日历条目。** 法院期限规则因管辖区、法院、法官和本地规则而异，并可能因常设命令或案件特定的案件管理命令而被修改。错过法院期限会产生职业过失后果。执照律师在将每个计算期限正式登记入档之前，必须对照法院的实际规则及任何案件特定命令进行核实。本 Agent 处于该决定的上游，不是其替代品。
- **文件分类是启发式的。** Agent 误分类的文件——将行政动议读作实质性动议，将和解协议读作发现争议——可能产生错误的期限规则。请阅读原始文件，不要只信赖标签。
- **未知法院不等于默认值。** 如果管辖区规则表不覆盖某个法院，映射器必须产出 `confidence: low` + `needs_verification: true`，而绝不能使用静默默认值。如果您看到某个冷僻法院出现高置信度期限，请将规则表视为过时，直至另行证实。
- **安静的案卷不等于干净的案卷。** 书记员有时会延迟登记。庭审记录有时在事件发生数天后才到达。"无新提交文件"是关于数据源的陈述，不是关于案件本身的陈述。

## 部署

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export TRELLIS_MCP_URL=...
export COURTLISTENER_MCP_URL=...
export GDRIVE_MCP_URL=...
../../scripts/deploy-managed-agent.sh docket-watcher
```

## 引导事件

参见 [`steering-examples.json`](./steering-examples.json)。

## 安全与交接

法院文件是公开记录，但同时也是不可信输入。提交人控制文本内容，可以在其中嵌入针对 Agent 的提示词、URL 和指令。三层隔离机制：

| 层级 | 接触文件？ | 工具 | 连接器 |
|---|---|---|---|
| **`docket-reader`** | **是** | 仅 `Read`、`Grep` | trellis、courtlistener（只读） |
| `deadline-mapper` / 编排器 | 否——只看结构化 JSON | `Read`、`Grep`、`Glob`、`Agent` | gdrive（管辖区配置，只读） |
| **`tracker-writer`**（写入端） | 否 | `Read`、`Write`、`Edit` | 无 |

`docket-reader` 返回长度受限、经 schema 验证的 JSON。`deadline-mapper` 无 MCP、无网络——它仅应用部署团队已配置的规则。`tracker-writer` 生成 `./out/docket-report-<date>.md` 和 `./out/deadlines.yaml`，且不接触原始文件。

## 适配说明

本 Cookbook 是起点。在完成以下工作之前，它无法在生产环境中使用：

- **设置 MCP URL。** `TRELLIS_MCP_URL` 和 `COURTLISTENER_MCP_URL` 必须指向您部署环境的端点，并配置您平台所需的身份认证。`GDRIVE_MCP_URL`（或替代方案）指向您的管辖区规则表所在位置。
- **加载案件组合。** Agent 读取 `matters/_log.yaml` 以及部署团队诉讼法律配置中每个案件的 `docket_id` 和 `court`。如果您的案卷系统是唯一数据源，请通过 MCP 或定时同步到配置路径来接入它。
- **配置管辖区规则。** 为案件组合中的每个法院为 deadline-mapper 提供本地规则表。联邦规则可以一次性配置；州初审法院和个别法官才是风险所在。未知法院应产出 `confidence: low` + `needs_verification: true`，而绝不能使用静默默认值。
- **接入交付渠道。** 决定输出流向：您的案卷系统接收 `./out/deadlines.yaml`；叙述性报告发送到 Slack、邮件或案件管理工作区；关键标记路由到需要紧急响应的人员。
- **设置执行周期。** 大多数案件每周一次；任何 14 天内有庭审的案件、处于 `trial` 或后期 `discovery` 阶段的案件，或任何 `risk: critical` 的案件，则每天执行。

## 计算期限是线索，不是日历条目

**本 Agent 生成的计算期限在登记入日历之前，必须由人工对照控制性本地规则、常设命令和案件管理命令进行核实。错过法院期限会产生职业过失后果。本 Agent 呈现期限；由人工核实并正式登记。**

每个期限都携带 `confidence` 和 `needs_verification` 字段。报告将低置信度条目单独列出，并对任何非源自明确联邦规则的内容添加核实提示。将这视为人工审查的最低要求——而非上限。法官可通过个人命令覆盖默认值，本地规则会变动，书记员实际登记送达的日期也可能与案卷显示的日期不同。

**不提供保证：** 本 Agent 推荐期限；登记律师对照控制性规则进行确认并正式录入。
