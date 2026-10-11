# shadowbot-visual-flow

> Platform-agnostic field notes for building, running and debugging 影刀（ShadowBot）**可视化流程（VisualFlow）** programmatically via MCP.

一个 **Agent Skill**：把「怎么用 MCP 程序化编写 / 运行 / 调试影刀可视化流程」的实操经验沉淀成可复用的技能包。

它**刻意排除业务平台耦合**——不包含任何具体站点的对抗策略、账号策略或接口字段语义（那属于各项目自己的文档）。这里只讲**怎么把可视化流程写对、跑稳、查得清**。

## 安装

克隆到宿主的 skills 目录，**目录名须与 `SKILL.md` 中的 `name` 一致**：

```sh
# opencode / 通用
git clone https://github.com/herouu/shadowbot-visual-flow.git ~/.config/opencode/skills/shadowbot-visual-flow

# Claude Code 等
git clone https://github.com/herouu/shadowbot-visual-flow.git ~/.claude/skills/shadowbot-visual-flow
```

## 内容

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 总纲：适用边界、端点与 schema 路径、建流四步、积木硬约束 11 条、作用域铁律、可视化数据流设计模式、指纹浏览器多环境 × 多身份轮换 × 出口隔离、运维与引擎坑、分层排障方法论、最小闭环骨架 |
| `references/mcp-payloads.md` | MCP 通联细节、建流四步、payload 语义值、运行/等待/日志、易错字段、错误文本截断 |
| `references/blocks-scope.md` | 原型字段查询纪律、隐式输出全局递增编号、作用域与异常模式、结构配对自检脚本 |
| `references/excel-blocks.md` | Excel / 表格类积木的读写语义与「写回五条口径」 |
| `references/patterns.md` | 数据流设计模式：幂等键 / 只改目标行 / 哨兵值 / 防空覆盖 / 名次归一 / 节流熔断 / 失败形态判定 / best-effort / 回滚点 |
| `references/bitbrowser.md` | 指纹浏览器本地 API（打开/创建/删除环境）、`user_data_dir` 多环境定位（**不用窗口标题**）、桥接扩展体检与排查清单 |
| `references/proxy-isolation.md` | 逐入站 listener 的代理出口隔离：端口钉上游节点 + 独立认证、配置结构、出口实测校验、排查清单 |
| `references/errors.md` | 报错速查：建流期 / 运行期 / 数据落盘，均为「症状 → 原因 → 处理」 |

## 几条最有价值的经验（节选）

- **积木隐式输出是「全流程全局递增编号」**，不按对象隔离：中间插一个同类积木，会让其后所有引用整体后移 —— 加/删块后必须复扫全流程引用。
- **`try` 内产出的变量不会逃逸**；产出「实例/句柄」的积木（打开表格/浏览器等）必须放在 `try` 之前，否则正常出口与 `catch` 都取不到（报 `variable_out_of_scope`，有时报文案极具误导性的 `output_id_used_as_variable_name`）。
- **数值型参数（如 sleep 的 `start_number`）必须是 int**：传 float 会报「参数类型错误」，而且该异常会**击穿整段 `try`**，外显成与真实原因无关的文案。
- **`app.stop_run` 在引擎卡态下会误杀正在跑的流程**（返回 `no active run to stop` 而 `engine_status` 仍是 `running`）：遇 `already_running` 先看 `engine_status`，别盲目 stop。
- **建流不是事务**：封装通常是 `clear → insert → fill`，**fill 失败会留下「已清空」的流程**，必须修到成功才算恢复 —— 改动前先留回滚点。
- **数据落盘三件套**：只改目标行、其余整行透传；用哨兵值区分「未处理」与「真 0」；结果为空则跳过写入（防空覆盖）。

## 使用边界（重要）

- 本技能包只沉淀**工程实现经验**：MCP 建流、积木语义与作用域、数据流设计、环境与代理的工程组织方式。
- **请仅用于你拥有或已获授权的账号、环境与系统**，并遵守目标平台的服务条款与当地法律法规。
- 本仓库**不提供**任何绕过平台规则、规避风控或批量滥用账号的方法；由此产生的判断与责任由使用者自行承担。
- 「多身份轮换 / 代理隔离」是**通用的工程隔离模式**（多环境测试、灰度发布、故障切换同样适用），请把它用在合规场景。

## 环境与版本

- 影刀（ShadowBot）**6.x** + Windows；通过 MCP（默认 `http://127.0.0.1:42500/api/v1/mcp`）驱动。
- 文中所述的积木字段名、报错文本来自实测；影刀版本升级后可能变化，**以本机 `Resources\Copilot\knowledge\visual-blocks\block_detail.md` 为准**。

## 许可

MIT — 见 [LICENSE](./LICENSE)。
