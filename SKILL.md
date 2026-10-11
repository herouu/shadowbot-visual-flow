---
name: shadowbot-visual-flow
description: Use when building, modifying, or debugging a 影刀(ShadowBot) 可视化流程(VisualFlow) programmatically via MCP — 建流四步(edit_blocks/fill_blocks/diagnostics/save)、payload 语义值、积木作用域与隐式输出编号、Excel 积木读写、循环/条件/异常模式、引擎卡态与报错速查。Trigger on: 影刀, ShadowBot, 可视化流程, 可视化积木, MCP建流, flow.edit_blocks, flow.fill_blocks, VisualFlow, 积木作用域, variable_out_of_scope, output_id_used_as_variable_name, 参数start_number类型错误, already_running, No Studio workspace, 工作区丢失, 影刀CLI, 指纹浏览器, BitBrowser, 比特浏览器, 多环境, 多账号, 代理隔离, Clash Verge, listener, user_data_dir, env_name, Code:120, Code:122. 不适用：具体业务平台的反爬/风控/账号策略（那属于各项目自己的文档），也不适用影刀 codeflow（纯 Python 流程）。
---

# 影刀（ShadowBot）可视化流程 × MCP 程序化建流

> **定位**：只沉淀「怎么用 MCP 写 / 运行 / 调试影刀可视化流程」的通用经验；**不含**任何业务平台专属的对抗策略与表单结构（那属于各项目自己的文档）。

## §0 适用与边界
- 适用：影刀 **6.x 可视化流程（VisualFlow）**；Windows；经 **MCP** 程序化建流 / 改流 / 运行 / 排障。
- 不适用：影刀 **codeflow**（纯 Python 流程）；具体业务平台的对抗策略（站点限流、渠道/账号策略、某个接口的字段语义）——那些属于各项目自己的文档。
- **包含**「指纹浏览器多环境 + 代理隔离」这类**工具层**经验（多环境定位、多身份轮换的结构、一个身份一个出口），见 §6；仍**不包含**业务平台自身的对抗策略（站点限流 / 账号策略 / 接口字段语义）。
- 前置：目标应用已在影刀 Studio 里打开（见 §7 工作区丢失）。

## §1 关键路径 / 端点
- MCP 端点：`http://127.0.0.1:42500/api/v1/mcp` —— 裸 fetch JSON-RPC，**无需 initialize / session**。
  头：`Content-Type: application/json` + `Accept: application/json, text/event-stream` + `MCP-Protocol-Version: 2025-03-26`。
- **积木 schema（权威）**：`<影刀安装目录>\Resources\Copilot\knowledge\visual-blocks\{block_catalog.md,block_detail.md}`
  → 查原型名、字段名、类型、`choices`、输出名的唯一权威来源（catalog 是「原型名 | 说明」清单，detail 是 `<原型名>{JSON}` 定义）。
- 应用目录：`%LOCALAPPDATA%\ShadowBot\users\<uid>\apps\<uuid>\xbot_robot\`
- 影刀 CLI：`shadowbot.shell-cli.exe`（在安装目录下）

## §2 建流四步（必须按此顺序）
```
flow.edit_blocks(insert) → flow.fill_blocks → diagnostics.snapshot(断言) → app.save
```
- 插入用 `prototype_name`；返回值取 `ins.block_region?.inserted || ins.inserted`，并**断言插入条数 == blocks 条数**（不相等说明有块没插进去）。
- fill 的值是**语义值对象**：`{text:str}` / `{variable:name}` / `{expression:code}` / `{array:[...]}` / `{object:{...}}`；**禁止 `10:` / `13:` 前缀**（写了会被整块丢弃）。
- 运行：`app.run{flow_id, wait_timeout}` → 未结束用 `app.wait_run_finish{run_id}`；日志用 `app.logs`。
- **顺序铁律**：父流程通过 `process.run` 引用子流程，**必须先建子、后建父**（建父时要解析子流程 id，子不存在则父建不出来）。
- **破坏性提醒**：通用封装多为 `clear → insert → fill`。**fill 失败会把目标流程留在「已清空」状态，不是回滚**。改流程前先留**回滚点**（备份 `build_*.mjs` / 先导出流程）；一旦 fill 失败，必须修到成功才算恢复。

## §3 积木硬约束（现象 → 原因 → 正确做法）
1. **隐式输出名不可自定义**：不少积木输出名是固定的（如打开表格 → `excel_instance`、读表 → `excel_data`）。**改了输出名，整块 fill 会被丢弃**（连输入也不落）。需要稳定名 → 用赋值类积木（`programing.variable`）复制一份。
2. **隐式输出是「全流程全局递增编号」，不按对象隔离**：第 N 个同类产出积木 → 名字带 N 后缀（第 1 个无后缀，第 2 个 `...2`，第 3 个 `...3`…）。实测跨不同对象实例仍连续编号。**代价**：中间插入一个同类积木，会让其后所有引用整体后移 → **加/删块后必须复核全流程引用**。
3. **读区域常「仍含表头行」**：`read_way=used_range` + `has_header_row=true` 实测仍返回表头行 → 消费时切片（如 `data[1:]`）。
4. **写区域模式必填 `row_num` + `column_name`**（`write_range=area`，不是 `begin_*`）；缺则报「字段"列名"为必填项」。
5. **HTTP 请求积木（`web_service.rest_request`）的 `headers` 是 `名: 值` 文本**（不是 JSON）；`body` 要序列化时用表达式 `__import__('json').dumps({...})`。
6. **`web.browser.execute_javascript` 的 `code` 必须是同步函数表达式**：`function(element, input){...}`；入参走 `argument`；返回值在输出变量 `web_js_result`。
7. **循环必须配对**：`workflow.forin` / `workflow.for` / `workflow.infinite_loop` 一律以 `workflow.endloop` 收尾。索引输出名**以 schema 为准**：`workflow.for` → `loop_index`；`workflow.forin` → `loop_item` + `loop_item_index`（后者需 `output_with_index=true`，且是**窗口内**的 0 基下标，不是源列表下标）。条件用 `operand1/operator/operand2`。
8. **数值参数类型严格**：如 `programing.sleep` 的 `start_number`/`stop_number` **必须是 int**（`seconds` 才是 float）。传 float 变量报「参数start_number类型错误」，**且该异常会击穿整段 `programing.try`**，外显成与真实意图无关的文案。参数化延时用 `value_type:'int'` + `int(max(...))`。
9. **表达式沙箱**：`programing.variable` 的 `expression` 编译成 `lambda: <expr>`，只自动 import `xbot`/`xbot_visual`/`time`/`glv`；其它库写 `__import__('re')` 这种内联导入（裸写 `re.match` 会报「变量 re 未被定义」）。表达式里可放心用列表推导/生成器/lambda 做聚合；多语句需求改用 `programing.snippet`。
10. **调用子流程**：`process.run` 的 `process` 字段要填**流程 id / 文件名**（显示名不行）；**主流程（入口）不能被其它流程调用**。
11. **静态诊断是应用级**：`diagnostics.snapshot` 返回整个应用的错误 → 判定时**只取目标流程的 `static_errors`**，别被其它流程的历史错误阻断。

## §4 作用域铁律（最容易翻车的一类）
- **`try` / `for` / `if` 等块内「产出」的变量不会逃逸到外层**。跨作用域传状态：在外层声明**可变容器**（dict / list），块内**原地修改**（不重新赋值、不重新产出）。
- **产出「实例/句柄」的积木要放在 `try` 之前（顶层）**：打开表格、启动浏览器、建立连接这类积木的输出是**隐式产出**；一旦放进 `try`，正常出口与 `catch` 都取不到 → fill 报 `variable_out_of_scope`，有时报文案极具误导性的 `output_id_used_as_variable_name`（看着像命名写错，实际是作用域）。
- **正确形状**：顶层产出句柄 → `try` 只包住会抛异常的读/写/循环 → `catch` 里做收尾/告警。
- 取舍：顶层产出的代价是「第 2 个产出失败时，第 1 个句柄不会进 catch 收尾」；必要时在 catch 里用**状态字典**判断「哪些还开着」再逐个收尾。

## §5 可视化数据流设计模式（通用）
- **幂等键**：写入前先剔除「同一键（业务主键 + 批次/日期）」的旧记录再写，保证重复运行不产生重复数据。
- **只写目标行、其余整行透传**：改写表格时只替换需要改的行/列，其它行**原样回传**。切忌对全表做强制类型转换——**一行脏值就会抛异常让整轮作废**。
- **哨兵值区分「未处理」与「真 0」**：用 `-1` / 空值表示「没拿到」，避免把合法的 `0` 当「没数据」。写盘 gate 用「**实际命中/变更的行数 > 0**」，不要用「容器长度 > 0」（后者会在全部取到 0 时白改一遍数据并打印成功日志）。
- **防空覆盖**：本轮结果为空 → **跳过写入**（保住上一轮好数据），只写告警/日志。覆盖写只覆盖前 N 行时，**旧尾部残留要用空值补齐覆盖**。
- **外部受限资源的节流/熔断**：请求计数器（单轮预算）+ 失败/异常计数（阈值）→ 达阈值就**停发 + 写告警文件 + 结束本轮**。换渠道（账号/出口/实例）时计数复位，但要保留一个**全局**熔断计数，防止「所有渠道都被限还继续打」。
- **失败判定要区分形态**：外部接口「软降级」通常表现为**返回体结构不对**（如首字符不是 `{`/`[`）；而「结构正常但结果为空」**不是失败**——混为一谈会白退避、误切换渠道。
- **增益环节 best-effort**：非关键的增强步骤（补数据/富化）失败只写告警 + 错误日志，**不要 re-raise 阻断主链**；关键步骤则要 fail-fast。
- **告警要落盘**：把「时间/环节/原因/已发请求数」等写进**告警文件**，别只打日志（日志会被滚掉，人也看不到）。

## §6 指纹浏览器多环境 × 多身份轮换 × 出口隔离（工具层）
### 6.1 前提
- 影刀要操作指纹浏览器，需要**匹配版本的桥接扩展**（多用户/多环境场景尤其如此）；版本不匹配会直接表现为「定位不到环境」。检查与启用见 `references/bitbrowser.md`。
- 每个环境有独立用户数据目录：`<缓存根>\<环境id>`（缓存根取指纹浏览器自身配置，不是业务的路径）。

### 6.2 定位单个环境：用「用户数据目录」，不要用「窗口标题」
- 正确形状：`web.set_user_environment{web_type:'<指纹浏览器类型>', fingerprint_browser:false, specified_userdata:true, user_data_dir:'<缓存根>\<环境id>', profile_name:'Default'}` → 再 `web.get{mode:'activated'}`。
- 为什么不用 `env_name`：它匹配的是**窗口标题**（= 当前激活标签的标题 + 浏览器后缀），**随页面变化** → 多环境必然不稳（典型报错就是 Code:120 / Code:122）。
- 定位失败的另一半原因常常是「环境根本没开」：`web.create` **不能冷启动**指纹环境 → 必须先用指纹浏览器的**本地 API** 把环境打开，再挂接。

### 6.3 多身份轮换：一个身份 = 账号 + 环境 + 出口（三者绑定）
- 建模：用一张**账号表**驱动循环（`账号名 | 环境id | 启用 | 备注`），**账号 ↔ 指纹环境 ↔ 独立出口** 三者一一绑定。这样「被限/被封」才落到**某个身份**上，而不是互相污染。
- 开局：所有环境**各开一次并保持常开**，之后只挂接/切换，**不要反复 close/open**（窗口狂闪、变慢，还会误报环境定位错误）。
- 循环结构：**外层身份、内层任务**；已完成的任务记入 `done` 集合，后续身份自动跳过 → 天然支持「某身份被限后，由下一个身份接管剩余任务」（failover）。
- 失败切换：命中限流/异常 → 记录原因 → 切下一个身份；**局部计数复位、全局熔断计数保留**（防止「所有身份都被限还继续打」）。
- 写盘：所有身份的产出**汇总后统一写一次**（别每个身份各写一遍、互相覆盖）。

### 6.4 出口隔离：一个身份一个出口
- 做法：用支持**多入站监听**的本地代理客户端，为每个身份开一个 listener（如 socks5），**端口钉死到不同上游节点**，且每个 listener 配**独立用户名/密码**（防身份之间串用）；指纹环境里把代理指向自己那个 listener。
- 收益：一个身份超限 → 换下一个身份即**同时换了 IP 与会话**，无需重启任何程序。
- 校验：挂接后**实测出口**（用 HTTP 探测出口 IP）确认每个身份走的是预期节点，别只信配置。
- 反例：多个身份共用一个出口 → 一个被限全员被限，「切换身份」失去意义；或在运行中临时更换出口（身份与出口不匹配，行为更像异常）。
- 配置结构与排查清单见 `references/proxy-isolation.md`。

## §7 运维与引擎坑
- **单应用约束**：所有 `app.*` / `flow.*` 只作用于 **Studio 当前打开的那一个应用**，不会自动切到别的应用。同一时刻只开一个目标应用；**不要新建 / 保留重名应用**。
- **Studio 会自动刷新**：改块、新增流程都会即时显示，一般**不需要重开应用**。
- **工作区丢失**：MCP 需要目标应用在 Studio 里打开；客户端重启**不会**自动恢复工作区；CLI 的 `studio open` **只会新建**（开不了已有），需人工在「我的应用」里打开。
- **`already_running` ≠ 真卡死（实测踩坑）**：它常表示「**上一次 run 仍在跑**」。而 `app.stop_run` 在引擎卡态下返回 `no active run to stop`、同时 `engine_status` 仍是 `running` —— 此时若脚本「遇 `already_running` 就先 stop 再重试」，会**把正在跑的流程停掉**（实测：已处理到一半、结果未落盘）。
  → 遇 `already_running` **先看 `engine_status`**；必要时直接 `app.run` + `app.wait_run_finish` 驱动，或用 `app.logs` 轮询「执行结束」，**别盲目 stop**。
- **`app.get_info` 可核验身份与状态**：返回应用 id，以及 `has_unsaved_changes`（建流/删流程并 save 后应为 false）。

## §8 排障方法论（分层 + 最小探针）
1. **分层定位，逐层证伪**：MCP 通联 → 应用/流程是否存在 → fill/static 诊断 → 运行日志 → 落盘数据。
2. **报错文本不要只看第一层语义**：先怀疑「上层基础设施」而非业务逻辑（例：真因是某积木参数类型不对，却被外显成另一个环节的失败文案）。
3. **异常必须可见**：`programing.catch` 的输出 `exception` 要打进日志（`str(exception)`），否则 try 内任何异常都会被吞成固定文案。
4. **最小探针**：临时建一个「只保留必要积木」的流程，把关键变量实值、异常文本打出来；比在完整流程里推断快得多。用完删除临时流程，保持应用流程清单干净。
5. **日志轮询**：`app.logs` 拉日志后按「执行结束/失败」关键字判定完成，**不要只看一次快照**。
6. **报错截断**：构建封装里的错误信息常被截断；若只看得到第一条错误，**先修第一条再重建**（第一层错误常掩盖后续错误）。

## §9 最小闭环（通用骨架）
块序（`col_a`/`col_b` 为占位字段名）：
```
1  [顶层] 打开表格/资源        → 产出句柄（必须在 try 之前）
2  [顶层] 初始化状态 dict      → 如 {"written": 0, "skipped": 0}
3  try
4     读区域                     → data（含表头 → data[1:]）
5     forin data[1:]             → 逐行构造
6       if 该行键命中本批次
7         只改目标列，其余整行透传 → 收集到 out_rows
8       endloop
9     if len(out_rows) > 0        ← 空结果跳过写入（防空覆盖）
10       写回目标区域（只覆盖前 N 行，尾部用空值补齐）
11       状态 dict 里 written += len(out_rows)
12    else 只记告警/日志
13 catch
14    打印 str(exception)；写告警文件（时间/环节/原因）
15 关闭/收尾（读状态 dict 判断哪些还开着）
```
要点：**顶层产出 → try 只包异常面 → 状态走 dict → 只改目标行 → 空结果不写**。

## §10 References（按需读取，勿一次全读）
- `references/mcp-payloads.md` — MCP 通联细节、建流四步、语义值、运行/日志、易错字段
- `references/blocks-scope.md` — 原型/字段查询方法、隐式输出编号规则、作用域与异常模式
- `references/excel-blocks.md` — Excel / 表格类积木的读写语义与坑
- `references/patterns.md` — 可视化数据流设计模式（幂等 / 防空覆盖 / 节流熔断 / best-effort）
- `references/bitbrowser.md` — 指纹浏览器本地 API（打开/创建/删除环境）、影刀侧多环境定位、桥接扩展检查与启用
- `references/proxy-isolation.md` — 多入站 listener 代理隔离（一个身份一个出口）与出口校验
- `references/errors.md` — 报错速查（症状 → 原因 → 处理）
