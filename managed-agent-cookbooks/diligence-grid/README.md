<!--
This file is a Chinese translation of the original by Anthropic PBC.
Original: https://github.com/anthropics/claude-for-legal
Licensed under Apache License 2.0
-->

# Diligence Grid — 托管 Agent 模板

## 概述

针对虚拟数据室（VDR）的批量文档审查。支持两种模式：

- **watch（监控模式）** — 监控 VDR 中自截止时间以来的新上传文件，对照部署团队的尽调请求清单类别对每份文件进行分类，并标记高优先级类别（重大合同、诉讼、知识产权）中的上传文件。
- **grid（表格模式）** — 按照列 schema 对某个文件夹中的文档进行表格化审查。每份文档占一行，每个数据点占一列，每个单元格均引用原文来源。这是并购尽调的核心工具。

与 [`corporate-legal`](../../corporate-legal) 插件同源——本目录是 `POST /v1/agents` 的托管 Agent Cookbook。表格模式即 `tabular-review` skill，以无头方式在一批提取器 worker 上并行运行。

## ⚠️ 部署前须知

- **每个单元格是线索，不是结论。** 尽调表格在律师阅读底层文件之前，不是陈述、披露附表或尽调备忘录。每个单元格中的原文引用是供审查人快速核实用的——请务必使用。
- **重要性筛选器和列分类采用启发式规则，而非法律判断。** schema 认定为不重要的合同，可能恰恰是终止交易的那份。"已回答"的单元格如果提取器误读了条款，仍然可能是错的。审查人的时间应分配在 `unclear`、`needs_review` 和 `answered` 这三类上——而不仅仅是被标记的那些。
- **监控模式对元数据和预览内容进行分类，而非完整文档。** 分类器标记为"低优先级"的新上传文件，仍可能是改变交易的补充函。将监控报告视为待处理队列，而非过滤器。
- **交易对手上传的文档对工具链来说也是不可信输入。** grid-writer 的 CSV 公式注入防御是强制要求，不是可选项——详见下方安全章节。

## 部署

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export BOX_MCP_URL=...
export GDRIVE_MCP_URL=...
export IMANAGE_MCP_URL=...          # 可选；如使用，请将工具集默认值设为 enabled
export DEFINELY_MCP_URL=...         # 可选；用于 normalizer 阶段的条款结构质量检查
../../scripts/deploy-managed-agent.sh diligence-grid
```

## 引导事件

参见 [`steering-examples.json`](./steering-examples.json)。

## 安全与交接

VDR 文档——合同、董事会会议纪要、补充函、交易对手上传文件——均为**不可信输入**。交易对手上传的合同可能包含旨在操控审查人或下游工具链的字符串。四层隔离机制将写入端和 MCP 端与文档分开：

| 层级 | 接触不可信文档？ | 工具 | 连接器 |
|---|---|---|---|
| **`doc-reader`** | **是**（只读） | `Read`、`Grep` | Box、Google Drive、iManage（读取） |
| **`extractor`** | **是**（只读） | `Read`、`Grep` | 无 |
| `normalizer` / 编排器 | 否 | `Read`、`Grep`、`Glob`、`Agent` | 无（definely 可选，只读） |
| **`grid-writer`**（写入端） | 否 | `Read`、`Write` | 无 |

`doc-reader` 和 `extractor` 返回长度受限、经 schema 验证的 JSON。编排器和 `normalizer` 只看到结构化数据。`grid-writer` 生成 `./out/diligence-grid-<date>.csv`、`./out/diligence-grid-<date>_sources.csv` 和 `./out/diligence-grid-<date>-summary.md`。

**CSV 公式注入防御。** `grid-writer` 写入任意单元格之前——包括值、原文引用、位置、文档名称、列标签——都会检查首字符是否为 `=`、`+`、`-`、`@`、制表符或回车符。匹配的单元格会在写入 CSV 前添加单引号前缀。交易对手上传的合同通常包含 Excel 和 Sheets 在审查人打开文件时会执行为公式的字符串（`=HYPERLINK(...)` 数据窃取、`=cmd|...` 旧版 Excel 的 DDE 执行）。来源 CSV 是更大的攻击面——原文引用是攻击者可控的内容。

**xlsx 是部署层关注事项。** 本 Cookbook 仅生成 CSV。部署团队通过 [`corporate-legal/skills/tabular-review/references/excel-output.md`](../../corporate-legal/skills/tabular-review/references/excel-output.md) 中的工作簿结构将其转换为 `.xlsx`——隐藏的 `_source` 列、悬停显示引用的单元格注释、基于状态的填充色、每列的 `Verified` 下拉菜单、`_schema` 和 `_summary` 工作表。该转换在部署团队的 Excel 环境中进行（Claude in Excel、openpyxl 或通过 Sheets API 操作 Google Sheets）。从无头 Agent 生成 xlsx 需要可信运行时和宏环境，本 Cookbook 故意不作此假设。

**不提供保证：** 本 Agent 生成的每个单元格是**需要核实的线索**，而非结论。审查人阅读来源文件、核对引用内容、填写 `Verified` 列。由律师决定哪些内容进入陈述、附表或备忘录。

## 适配说明

- **VDR URL。** 设置 `BOX_MCP_URL` / `GDRIVE_MCP_URL` / `IMANAGE_MCP_URL` 以匹配您的数据室。默认启用 Box 和 Google Drive；如果使用 iManage 或 Datasite 作为主要数据室，请在 [`agent.yaml`](./agent.yaml) 中修改 `default_config`。如果您的 VDR 是 Intralinks 或 Datasite，请在 `mcp_servers` 和 `tools` 中添加对应的 MCP URL 条目。
- **列 schema。** 默认使用 [`corporate-legal/skills/tabular-review/references/ma-diligence-columns.md`](../../corporate-legal/skills/tabular-review/references/ma-diligence-columns.md) 中的并购尽调标准列集。根据您的交易类型定制——科技/知识产权、医疗健康、房地产、政府承包商、受监管金融机构——参考该参考文件中的补充说明。
- **输出目录。** 输出文件保存在 `./out/`。通过部署流水线将其接入您的交易文件夹、Google Drive、iManage 工作区或 Box 文件夹。不要给 `grid-writer` 添加上传 MCP；交接给您的上传步骤更简洁，也能保持写入层的隔离。
- **默认模式。** watch 和 grid 通过引导事件选择。如果您的工作流几乎总是使用其中一种，请在编排器中预置相应的引导事件模板。
- **请求清单类别。** 监控模式按部署团队在 corporate-legal `CLAUDE.md` 配置中的类别进行分类。在将监控模式接入实际交易之前，请先在该配置中重新运行 `/corporate-legal:cold-start-interview`。
- **工作成果抬头。** `grid-writer` 会在报告前添加部署团队 `## Outputs` 配置中的抬头。部署前请与您的法律团队确认抬头内容——审查人角色不同（律师与非律师），抬头也不同。
- **Slack 路由。** 本 Agent 不直接发送消息。报告以文件形式输出；`handoff_request` 通知您的编排器将消息路由到哪个频道。请在部署团队的 `CLAUDE.md` 团队风格章节中配置交易频道。
