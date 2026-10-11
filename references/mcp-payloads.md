# MCP 建流载荷要点（通用·实测）

用 MCP 程序化建/改影刀(ShadowBot) **可视化流程**时读本文。每条尽量给「现象 → 原因 → 做法」。

## 1. 通联：裸 JSON-RPC，不需要握手
- 端点 `http://127.0.0.1:42500/api/v1/mcp`；**直接 POST**，**不要**发 `initialize`、不要维护 session。
- 必需请求头：`Content-Type: application/json`、`Accept: application/json, text/event-stream`、`MCP-Protocol-Version: 2025-03-26`（缺 Accept 会拿到空的 SSE 流或直接 4xx）。
```js
const res = await fetch(URL, {method:'POST', headers:H, body:JSON.stringify({
  jsonrpc:'2.0', id:1, method:'tools/call',
  params:{ name:'flow.blocks.list', arguments:{flow_id} }})});
const out = JSON.parse(res.result.content[0].text);   // text 常是 JSON 字符串，需二次 parse
if (res.result.isError) throw new Error(res.result.content[0].text); // isError 当异常处理
```
- 现象：返回体有 `result.content[]` 却取不到字段 → 原因：`content[0].text` 未被 parse。做法：统一包一层 `mcp()`，parse + `isError` 抛错。

## 2. 建流四步（顺序不可换）
`flow.edit_blocks(insert)` → `flow.fill_blocks` → `diagnostics.snapshot` → `app.save`
```js
const ins = await mcp('flow.edit_blocks',
  {flow_id, op:'insert', block_index:-1,
   blocks:[{prototype_name:'programing.comment', comment:'label'}, /* ... */]});
const ids = ins.block_region?.inserted || ins.inserted;   // 两个位置都出现过
if (ids.length !== blocks.length) throw new Error('inserted 数量不符'); // 必断言
await mcp('flow.fill_blocks', {flow_id,
  fills: ids.map((id,i)=>({block_id:id, inputs:blocks[i].inputs, outputs:blocks[i].outputs}))});
// diagnostics.snapshot 是应用级 → 只取目标流程的 static_errors
await mcp('app.save');
```
- 现象：`insert` 成功但后续按 `ins.inserted` 取到 undefined → 原因：不同版本返回在 `block_region.inserted`。做法：双取兼容 + 数量断言。

## 3. fill 的语义值（禁用 `10:` / `13:` 前缀）
| 构造 | 含义 | 用途 |
|---|---|---|
| `{text:'s'}` | 字面文本（可空串） | 字符串常量 |
| `{variable:'name'}` | **引用已存在的变量** | 读上一步产出 |
| `{expression:'code'}` | **求值一段 Python 表达式** | 计算/聚合 |
| `{array:[...]}` / `{object:{...}` | 真列表 / 真字典 | 结构化入参 |
| JSON 数字 / 布尔 | int / bool / float 字面量 | `value_type:'int'` 等 |

- 现象：fill 报 `10:`/`13:` 前缀相关错误 → 原因：写了旧版内部编码。做法：一律用上表语义值。
- `variable` 是「引用」，`expression` 是「求值」：把变量名写进 expression 字符串没意义（除非确实想按名取）。

## 4. 运行与等待
```js
const r = await mcp('app.run', {flow_id, wait_timeout: 60});   // 未跑完返回需续等
let run = r; if (!r?.finished) run = await mcp('app.wait_run_finish', {run_id: r.run_id, ...});
await mcp('app.logs', {limit: 200});                            // 日志是最终判据
```
- 现象：`app.run` 返回 `already_running`(run_id 为 null)。做法：**先看 `engine_status`**（见 §7），别盲目 stop；必要时直接 `app.run` + `app.wait_run_finish` 驱动，不经任何「先清场」封装。
- 收尾判据：轮询 `app.logs` 出现「执行结束」比只看 run 状态更可靠。

## 5. 常用工具清单（**不全**，以实际 `tools/list` 为准）
- 建流/改流：`flow.edit_blocks`、`flow.fill_blocks`、`flow.blocks.list`、`flow.toggle_blocks`、`flow.list_variables`
- 应用/流程管理：`app.list_flows`、`app.create_flow`、`app.delete_flow`（**复数 `flow_ids`**）、`app.rename_flow`、`app.save`、`app.get_info`、`app.reload`
- 运行：`app.run`、`app.stop_run`、`app.run_result`、`app.wait_run_finish`、`app.logs`
- 诊断/查询：`diagnostics.snapshot`、`wiki.search`、`wiki.read`、`selector.*`

## 6. 易错字段（实测）
- **输出名固定、不可自定义**：某些积木的输出名是引擎写死的，改输出名会导致**整块 fill 被丢弃**（不报错，静默少字段）。做法：用默认名，需要稳定名时再用赋值类积木复制一份。
- **隐式输出全局递增编号**：同族产出积木的第 N 个实例名字带 N 后缀（第 1 个无后缀），且**跨不同实例仍连续**（实测 `excel.launch` → `excel_instance`/`excel_instance2`…，`excel.read_data_from_workbook` → `excel_data`/`excel_data2`…）。
  - 现象：在中间插入一个同类积木后，运行期取到错误数据或报变量未定义。原因：其后所有 `*N` 引用整体后移。
  - 做法：**加/删同类积木后必须复核全流程引用**，不能只看被改的那段。
- **请求型积木**：`web_service.rest_request.headers` 是 `名: 值` 的**多行文本**（不是 JSON）；`body` 需序列化时用表达式 `__import__('json').dumps({...})`；`send_by_web=false` 时勿绑 `browser`。
- **浏览器执行 JS**：`web.browser.execute_javascript.code` 必须是**同步函数表达式** `function(element,input){...}`（不能是箭头/语句块），入参走 `argument`，结果在 `web_js_result`。
- **Excel 写入的区域模式**：`write_range='area'` 必填 `row_num` + `column_name`（**不是** `begin_*`）；`read_way='used_range'` + `has_header_row=true` **仍含表头行**，用 `[1:]` 去掉。
- **`diagnostics.snapshot` 是应用级**：判定时必须只取**目标流程**的 `static_errors`，否则会被无关流程的错误卡住。
- **`app.get_info`** 返回当前 Studio 打开的应用标识与 `has_unsaved_changes`（保存后应为 `false`）——建流前先核验「是不是目标应用」。所有 MCP 调用只作用于 Studio 当前打开的那一个应用；**既有应用里追加流程**时不要新建应用。

## 7. 错误信息可见性与 stop_run 陷阱
- 现象：`fill_blocks` 只报 1 条错、看不到第 2 条。原因：封装层按字符数截断。做法：把截断阈值放宽（实测 **4000** 字符才够；500 会吃掉第 2 条）；**看到多少就先把第一条修掉再整体重建**，逐条推进。
- 现象：`app.stop_run` 返回 `no active run to stop`，但 `engine_status` 仍是 `running`。原因：引擎卡态下 stop 语义失效。做法：**不要盲目 stop——它会误杀正在跑的流程**（实测某次采集已跑到一半被停、结果未落盘）。判据顺序：先读 `engine_status`，再决定续等还是重启。

## 8. 表达式两条高频用法
- 跨作用域状态：顶层 `programing.variable` 建 dict/list，在循环/if/try 内**原地 mutate**（不要重新赋值），即可跨作用域可见。
- 整表聚合：在单条 expression 里用列表推导 + lambda 一次算完，比「循环 + 数据表格」可靠。
```js
{expression:"[0.0]*len(v) if min(v)==max(v) else [round((x-min(v))/(max(v)-min(v)),4) for x in v]"}
```
- 注意仍受 try 内产出不外泄限制：算完要落到**顶层容器**才能在 try 外读（见 `blocks-scope.md`）。
