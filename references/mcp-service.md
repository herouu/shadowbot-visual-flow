# 本地 MCP 服务的性质、能力探测与风险

> 面向：用**任意 MCP 客户端**（含平台内置的 AI 助手、opencode 等）驱动影刀可视化流程的人。

## 1. 它是什么

- 影刀客户端的**本地 MCP 服务**：由**客户端进程自己**在本机回环上监听（默认 `127.0.0.1:42500`，路径 `/api/v1/mcp`）。
- **标准 MCP JSON-RPC**：无需 `initialize` / session、无 API key；请求体 `{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<tool>","arguments":{...}}}`；**工具清单与参数 schema 由 `tools/list` 自描述**。
- 结论：**不需要逆向**（不必抓包/反编译猜协议），但它**不是文档化的对外 API** —— 按「**客户端自带、未文档化的本地服务**」来对待最准确。

## 2. 别和另外两种「影刀 MCP」混用

| 形态 | 部署形态 | 能力面 | 公开文档 |
|---|---|---|---|
| **客户端内置本地 MCP**（本 skill 针对的） | 客户端进程内的本地服务 | **编辑级**：流程/块的增删改、静态诊断、运行、日志 | 未文档化 |
| 官方开源 MCP Server（`yingdao_mcp_server`，SSE/Stdio 两种模式） | 独立进程 | **调度级**：列应用、查参数、跑应用 | 有 |
| 控制台 / 云端 MCP（SSE + Key） | 云端 | 远程调度应用 | 有 |

→ **拿「调度级」的官方文档去找 `flow.edit_blocks` / `fill_blocks` / `diagnostics.snapshot` 是找不到的** —— 那是内置本地服务独有的工具面。

## 3. 客户端内的相关组件（解释「为什么本机有积木知识库」）

- 客户端随包分发 AI 组件：内部 AI 运行时 + **积木知识库**（`Resources\Copilot\knowledge\visual-blocks\{block_catalog.md,block_detail.md}`）+ 若干官方技能包（`Resources\Skills\*.zip`）。
- 实操意义：**积木原型名 / 字段名 / 输出名 / `choices` 一律以本机那份 `block_detail.md` 为准**（= `SKILL.md` §1 的权威来源）；版本升级后该文件会变，**不要凭记忆写字段**。
- 另一层含义：把「平台内的 AI 助手」理解为「**本地 MCP + 积木知识库 + 一个 MCP 客户端**」的组合；用外部客户端接同一个服务，属于顺着既有架构使用，而不是绕过它。

## 4. 必须做的防御（无文档 ⇒ 无兼容性承诺）

- **启动先探测**：用 `tools/list` 枚举能力集，检查所需工具是否在列（尤其编辑级工具）；**缺了就立刻显式报错**，别等 fill / run 半途才失败。
- **端点与工具名可配置**：端口在客户端设置里可改，别把 `42500` 硬编码进业务代码；工具名集中在一处映射。
- **把探测结果写进日志**：记录「工具总数 / 关键工具是否存在」，升级后能一眼定位差异。
- **能力降级**：把编辑级能力当作增强路径；关键链路要么有替代通道（见 §5），要么配监控与告警。
- **探测失败不动数据**：初始化/探测失败时**不写盘、不改数据**（遵循 `patterns.md` 的防空覆盖与 best-effort）。

## 5. 另一条「有文档」的通道：官方 CLI

- 影刀自带 `shadowbot.shell-cli.exe`，官方有文档与对应技能包，命令族覆盖：`auth login|current|account`、`console app`、`console task run|stop|history|status|logs`、`studio open|create|save|sync`、`system health|state|version`。
- 分工建议：
  - **CLI** → 应用生命周期与调度，以及跑之前的「健康 / 登录」自检（`system health`、`auth current`）。
  - **本地 MCP** → 流程 / 块的**编辑与诊断**（本 skill 的主题）。
- 两者互补：CLI 把「环境与应用」准备好，MCP 把「流程改对、跑稳、查清」。

## 6. 速查

```
# 探测能力（动手前都该先做）
{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}
# 调用工具
{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"<tool>","arguments":{...}}}
```
- 现象：工具不存在 / 字段不认 → 先查 `tools/list` 与 `block_detail.md`，而不是猜。
- 现象：升级后原本正常的流程突然报「缺工具 / 缺字段」→ 按 §4 的探测日志定位差异。
