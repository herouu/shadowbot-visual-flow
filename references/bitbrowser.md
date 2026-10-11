# 指纹浏览器（BitBrowser / 比特浏览器）多环境对接

工具层通用经验：**同一类"本地 API 管理多指纹环境 + 影刀挂接"的浏览器**均可套用，平台无关。
按需在「多环境间精确切换 / 排查 Code:120/122」时读本文。

## 1. 本地 API 基础

- 端点默认 `http://127.0.0.1:54345`；**端口可在客户端设置里改** → 脚本里做成**可配置**，调用前先做一次**健康检查**。
- **无鉴权**：`POST` + `Content-Type: application/json`，body 直接是 JSON，**不需要 API key / 签名头**。
- **必须绕过系统代理**发这些本地请求（现象：请求超时/连接被拒；原因：被系统或规则的代理拦截；做法：本地地址走 `no_proxy`）。
- 返回风格**不统一**：有的接口回 `{"success":true,"data":...}`，有的回 `{"code":0,...}`
  → **判成功要兼容两种**，否则会误判失败。

常用端点：

| 端点 | 用途 |
|---|---|
| `POST /health` | 预检（端口/客户端是否活着） |
| `POST /browser/list` `{"page":0,"pageSize":50}` | 分页环境列表 |
| `POST /browser/open` `{"id":"<环境id>"}` | 打开，返回 `pid` / `http`(CDP 调试端口) / `ws` |
| `POST /browser/close` `{"id":...}` | 关闭 |
| `POST /browser/delete` `{"id":...}` | 删除（**先 close 更稳**） |
| `POST /browser/update` | **创建与修改共用**（见下） |

- `browser/open` 是**异步**的：首次可能回 `"浏览器正在打开中"`，需轮询几次直到拿到 `ws`。
- **对已打开的环境，`browser/open` 是幂等的**：重复调用返回**同一 `ws`/`pid`**，不会新开窗口 → 与「环境一律常开」不冲突。
- **坑**：`/browser/create` 之类路径**不存在**（返回 404）——那是**别家指纹浏览器的命名**（如 RoxyBrowser、AdsPower 各有前缀），**别混用不同家的端点**。
- **新环境的用户数据目录在首次 `open` 之后才生成**（`<缓存根>\<环境id>`）→ **挂接前必须先 `open`**，否则目录不存在、挂接必失败。

### 创建 / 修改：`browser/update`（靠有无 `id` 区分）

- **不传 `id` = 创建，传 `id` = 修改**，同一个端点，语义靠 `id` 有无切换。
- 最小 body 示例（字段名以官方 schema 为准，代理与凭据用占位）：

```json
{"name":"env-demo","remark":"","proxyMethod":2,"proxyType":"socks5",
 "host":"127.0.0.1","port":20001,"proxyUserName":"<user>","proxyPassword":"<pass>",
 "browserFingerPrint":{"coreVersion":"150","ostype":"PC","os":"Win32"}}
```

  返回 `data.id`，**取它作为环境 id**（后续 `open` / `user_data_dir` 都用这个 id）。

## 2. 影刀侧：多环境定位（核心）

**推荐形状（按目录定位，免标题、免开关）**：

```
web.set_user_environment{ web_type:'BitBrowser', fingerprint_browser:false,
                          specified_userdata:true,
                          user_data_dir:'<缓存根>\<环境id>',
                          profile_name:'Default' }
→ web.get{ web_type:'BitBrowser', mode:'activated' }
```

- 缓存根怎么拿：读指纹浏览器自身配置（如 SQLite `global_configs` 里的 `browser_cache_dir`），或直接在客户端设置里看；**不要硬编码到业务路径**。
- **不要用 `env_name`**：它按**窗口标题**匹配（= 当前激活标签标题 + 浏览器后缀），**随页面变化**，静态写死只在恰好停在那页时成立 → 多环境必不稳。环境名 / 环境 id / 序号**都不能**用于该字段。
- 现象：多环境同开时不指定 → **Code:120**；指定错标题 → **Code:122**。原因：按标题定位。做法：改 `specified_userdata` + `user_data_dir`。
- **`web.create` 不能冷启动指纹环境**（报"未找到目标环境"）→ 必须先用本地 API `open`，再 `web.get` 挂接；`set_user_environment` **只激活已打开的环境，不负责打开**。
- **环境一律常开**：开局各环境**各打开一次**，之后只在环境之间**挂接/切换**，**不要逐环境 close/open**（现象：窗口狂闪、整体变慢、易误报定位错误）。

## 3. 桥接扩展 / 版本匹配

- 影刀操作指纹浏览器依赖**桥接扩展**；**多用户/多环境场景需要 V3 级扩展**（V2 不支持多用户，表现为 Code:120/122）。
- 检查（**查 `local_extensions` 表，不要**去 `global_configs` 找扩展信息——那里查不到）：
  `%APPDATA%\BitBrowser\db.sqlite` → 表 `local_extensions(id,name,desc,extension_id,icon,version,is_use,cloud_id,main_user_id)`，看目标扩展 `is_use` 是否为 1、版本号。
- 启用：把目标扩展 `is_use` 置 1，**改动前先备份 db**，**重开环境窗口后生效**（无需重启客户端）；再用下列命令确认扩展已出现在 `--load-extension`：

```powershell
Get-CimInstance Win32_Process -Filter "Name='BitBrowser.exe'" |
  Where-Object { $_.CommandLine -match 'load-extension' } | ForEach-Object {
    $eid=[regex]::Match($_.CommandLine,'--user-data-dir="?([^"\s]+)"?').Groups[1].Value -replace '.*\\',''
    $l=[regex]::Match($_.CommandLine,'--load-extension=([^\s]+)').Groups[1].Value
    "[$eid] " + (($l -split ',') -join ' ; ') }
```

- 原生桥接与扩展**配套，缺一不可**（安装器 `--kind=... --version=V3 --path="<浏览器exe>"`；注册表宿主在 `HKCU:\Software\Google\Chrome\NativeMessagingHosts\*`）。
- 现象：**所有环境都"定位不到"或扩展报错** → 先怀疑**扩展/桥接版本不匹配**，而不是先改流程。

## 4. 排查清单（现象 → 原因 → 做法）

- **找不到目标环境窗口（Code:120/122），多开时必现** → 用了窗口标题定位 → 改 `specified_userdata` + `user_data_dir=<缓存根>\<环境id>`。
- **单个环境也挂不上** → 环境没开，或用户数据目录还不存在（新环境从未 open 过）→ 先用本地 API `open`。
- **本地 API 调用全部失败** → 端口不是默认值，或请求被系统代理拦截 → 查客户端设置里的端口、让本地地址绕过代理。
- **挂接成功但页面空白/未加载完** → `web.get` 的 `open_page` / `wait_load_completed` 参数按需调整。
- **目标环境停在 `chrome://` 等系统页**，JS 注入报"不支持对系统页面进行该操作" → 这是**页面限制，不是定位失败**，切到普通页面即可。
- **想"重新开始"某环境** → 不要靠反复 open/close，改用**新环境**（新指纹 + 新用户数据目录），旧环境保持不动。
- **扩展体检三步**：① 环境**实际加载**了哪些扩展（上面的进程命令，按 `user-data-dir` 区分）→ ② 磁盘已装扩展与版本（`%APPDATA%\BitBrowser\BitExtensions\<extId>\<ver>_0\manifest.json`）→ ③ 核对扩展 ID 是否为本轮要求的多用户级版本。
