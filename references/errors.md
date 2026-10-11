# 常见报错速查（建流期 / 运行期 / 落盘）

按需在「构建失败 / 运行失败」排查时读本文。以下条目均为实测踩坑，与具体业务平台无关。

## 建流期（fill / static 诊断）

| 报错 / 现象 | 原因 | 处理 |
|---|---|---|
| `variable_out_of_scope`（`变量 X 不存在 …produced earlier by (某产出积木), but that output is outside the consumer's scope`） | 产出实例/句柄的积木（各类 `*.launch` 之类）被放进了 `try` 或分支，而消费者在外层作用域 | **把产出实例的积木移到顶层（`try` 之前）**；`try` 只包住会抛异常的读/写/循环 |
| `output_id_used_as_variable_name`（`Use output name '...' instead of output id '...'`） | **文案有误导性**：看着像输出命名写错，实际常与上一条同源，仍是作用域问题 | 先按作用域排查（产出积木是否被 `try`/分支包住）；移回顶层后，本条与 `variable_out_of_scope` 一起消失 |
| `参数start_number类型错误`（同类：`参数 X 类型错误`） | 数值型参数收到了 float | 参数化时用 `value_type:'int'` + `int(max(...))`，或直接写整数字面量。**症状会放大**：异常会击穿整段 `try`，外显成无关环节的失败文案，别被文案带偏 |
| `invalid_choice_value` | 填了枚举以外的值 | 以积木 schema（`block_detail.md`）里的 `choices` 为准照抄 |
| `unknown_input` | 字段名不对 | 以 schema / `flow.blocks.list` 返回的 `fields` 为准，别按直觉拼字段名 |
| `字段"X"为必填项`（列名 / 模块名等） | 必填字段缺失，或把「显示名」填进了需要 id / 文件名的字段 | 按 schema 补齐必填项；需要 id 的位置填 flow_id / fileName，不填标题 |
| `变量名无效（字母数字下划线）` | 变量名含 `-`、空格等特殊字符 | 变量名只用字母 / 数字 / 下划线（下划线在表达式里可正常使用） |
| `target_flow_not_callable` | 调用了入口流程 | 把被调用逻辑放到非入口流程，入口流程只做编排 |
| fill 只报**前 1 条**错误、后面的看不到 | 构建封装截断了错误文本 | 已放宽（如 4000 字符）；若仍不够说明错误条数极多，**先修第一条再重建**，逐轮推进 |
| 编译/运行前报「存在静态错误」 | `diagnostics.snapshot` 是**应用级**返回，混入了其它流程的历史错误 | 只取**目标流程**的 `static_errors` 判定，别被别的流程带偏 |
| 本地 API 调用（列表/打开/创建环境）全部失败 | 端点端口不是默认值，或这些**本地请求被系统/规则代理拦截** | 端口从客户端设置里取并做成可配置；发本地请求时绕过系统代理（显式置空代理或用回环直连） |

## 运行期（引擎 / 流程执行）

| 报错 / 现象 | 原因 | 处理 |
|---|---|---|
| `already_running: A Studio flow is already running` | 多半是**上一次 run 仍在跑**，不一定是卡死 | **先看 `engine_status`**；确为卡死再 `app.stop_run`；仍无效再考虑重启客户端 / 机器。**危险**：引擎卡态下 `stop_run` 会返回 `no active run to stop`，而 `engine_status` 仍是 `running`——此时「遇 `already_running` 就先 stop」的自动重试会**杀掉正在跑的流程**（现象：流程跑一半「消失」、结果未落盘、日志停在半途）。判不准就别走自动 stop，直接用 `app.run` + `app.wait_run_finish` 驱动，或用日志轮询收尾 |
| `No Studio workspace is opened` | 没有应用在 Studio 中打开 | 人工在「我的应用」里打开目标应用；**客户端重启不会**恢复工作区 |
| `Code:120`（检测到多个用户环境）/ `Code:122`（未找到指定用户环境的浏览器窗口）/ `未找到目标环境` | 用 `env_name` 定位，而 `env_name` 匹配的是**窗口标题**（= 当前激活标签标题 + 浏览器后缀），随页面变化 → 多环境必然不稳；或环境根本没打开 | 改用**显式用户数据目录**定位：`web.set_user_environment{web_type:'BitBrowser', fingerprint_browser:false, specified_userdata:true, user_data_dir:'<缓存根>\<环境id>', profile_name:'Default'}` → `web.get{mode:'activated'}`。挂接前先用指纹浏览器**本地 API** 打开该环境（`POST /browser/open {"id":...}`）；缓存根取指纹浏览器自身配置（如 `global_configs.browser_cache_dir`），不要硬编码业务路径。细节见 `bitbrowser.md` |
| 指纹环境挂接失败/报"未找到环境"，但环境看起来是开着的 | ① 桥接扩展版本不匹配（多用户/多环境需要 V3 级扩展，V2 不支持）；② 环境是新建的、**用户数据目录还没生成**（新环境首次 `open` 之后才会生成 `<缓存根>\<环境id>`） | ① 查扩展状态：读 `%APPDATA%\BitBrowser\db.sqlite` 的 `local_extensions.is_use`（**不要**在 `global_configs` 里找），置 1 后**重开环境**生效；② 先用本地 API `open` 一次再挂接 |
| 想"重启"某个环境，反复 open/close | 频繁开关环境窗口会让窗口闪、变慢、并误报环境定位错误 | **环境一律常开**：开局各打开一次，之后只在环境之间挂接/切换；要"干净开始"就新建一个环境，而不是反复开关旧环境 |

## 数据落盘与写回

| 报错 / 现象 | 原因 | 处理 |
|---|---|---|
| 写回时抛 `ValueError: could not convert string to float ...` | 对**全表**做了强制类型转换，一行脏值（如 `'20万+'`）就整轮炸掉 | **只改目标行，其余行整行原样透传**，不要全表统一转型 |
| 目标表「数据变少但旧行还在」 | `override` 只覆盖写入区，尾部多余旧行没被覆盖 | 覆盖写之前，把多余的尾部行用空值补齐覆盖（或先清区再写） |
| 下次运行报「文件被占用 / 无法写入」 | 上次运行没关闭表格句柄 | 所有出口（正常与异常）都要**保存并关闭**已打开的表格实例 |
| 数据表里出现字符串 `'None'` | 空值被当成字符串写进去了 | 写入前把空值归一（按列语义写空字符串或数值 0），数值列尽量写数值类型 |
| 排名结果「一堆满分/高分挤在一起」 | 缺失数据的行（指标为 0/空）也参与了归一化排名 | **只让数据完整的行参与排名**；常量列（max == min）该维统一记 0 |
