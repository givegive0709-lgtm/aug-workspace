# AUG WorkSpace API 参考

> **SDK**：需要可安装客户端 + 全量方法表？见 [docs/SDK.md](SDK.md)（pip 包 `aug-sdk`，含语言无关 REST 端点表）。

AUG WorkSpace 的自动化控制面 = **两个协议层，同一能力面**（覆盖浏览器自动化全能力面）：

| 层 | 传输 | 用途 |
|----|------|------|
| **bridge** | HTTP JSON-RPC（127.0.0.1 短连接） | 每 profile 进程内嵌本地 server，脚本驱动导航 / JS 求值 / tab / cookie / 窗口 |
| **MCP** | stdio NDJSON（Model Context Protocol） | AI Agent 经 MCP 客户端操作 AUG WorkSpace；复用 bridge 能力 |

两者都不暴露 CDP / webdriver / devtools —— 协议层零反欺诈检测信号。

---

> ## 先分清两层（只读本页会以为「建档只能手工」）
>
> | 层 | 在哪 | 管什么 |
> |---|---|---|
> | **本机控制面** | `http://127.0.0.1:8311`（`Authorization: Bearer <AUG_API_KEY>`） | **生命周期**：建档 `POST /v1/profiles`、启动 `POST /v1/profiles/{id}/launch`、停止、删除（含回收站）、工作区、运行态 `GET /v1/profiles/{id}/status`。**其余端点见 `/docs/api.html`** |
> | ↳ **工作区的批量形态** | 同左 | 建：`POST /v1/workspaces` 带 `names`（数组，一次写盘；重名/空名/保留名逐条进 `skipped`）；删：`POST /v1/workspaces/batch_delete` 带 `workspace_ids`（数组；`default` 与不存在的 id 逐条进 `refused`）。**删组只摘标签，不动任何档案本体** |
> | **bridge**（本页讲的就是它） | `http://127.0.0.1:<该档案的端口>`（无鉴权，本机信任边界） | **页面级操作**：导航 / 求值 / 点击 / 键入 / cookie / 截图 —— 驱动的是**一个已经在跑的档案** |
>
> **权威接口清单是 `GET /v1/tools`**：由接口注册表自动生成（名字 / 说明 / 入参 schema），
> 永远不会落后于实现。本页是**人读**的说明，两者冲突时以它为准。

## 端口怎么拿（别自己算）

**端口是档案数据，不是可以自己推导的常量。** 每个档案的控制端口在启动时分配并落盘，此后固定不变。

**不要**自己按公式算端口去连一个已存在的档案 —— 档案一多就会撞车。撞了以后不止「谁连不上谁」，而是可能连到**别人的档案**上去。

正确做法只有一个：**用 SDK 取**。

```python
from aug_sdk import Client
a = Client().browser("你的档案ID")   # 端口由 SDK 从档案里读，不用你管
a.navigate("https://example.com")
```

任何语言都可以走 REST 直接发请求，但端口同样请从档案数据里读，不要硬编码、不要推导。

---

## bridge：本地 HTTP JSON-RPC

每 profile 浏览器进程启动时内嵌一个 HTTP server，**仅绑定 127.0.0.1**。
base URL = `http://127.0.0.1:<port>`（port 见上）。

响应均为 JSON；`ok: true` 成功，`ok: false` 时带 `error` 字段。**唯一例外 = `/screenshot`
返回 raw bytes**（image/png|jpeg，非 JSON）。短连接：响应后即关。

| 方法 | 路径 | 请求 body | 响应 |
|------|------|-----------|------|
| GET | `/tabs` | — | `{ok, tabs: [{title,url,active}]}` |
| GET | `/cookies/export` | — | `{ok, text: <Netscape cookies.txt>, count}` |
| POST | `/navigate` | `{url}` | `{ok}` |
| POST | `/evaluate` | `{js}` | `{ok, result}`（JS 求值结果，JSON 可序列化值；JS 抛错 → `{ok:false,error}`） |
| POST | `/tabs/new` | `{url?}`（缺省 example.com） | `{ok}` |
| POST | `/tabs/switch` | `{index}` | `{ok}` |
| POST | `/tabs/close` | `{index}` | `{ok}` |
| POST | `/cookies/import` | `{text: <Netscape cookies.txt>}` | `{ok, imported: <条数>}` |
| POST | `/cookies/replace` | `{text: <Netscape cookies.txt>}` | `{ok, imported: <条数>}`（先清空再注入；切换登录态不重启） |
| POST | `/window/frame` | `{x, y, width, height}` | `{ok}`（contentRect，AppKit 左下原点；宽高须 >0） |
| POST | `/screenshot` | `{format: png\|jpeg, max_width?: 点数, quality?: 0.1-1.0}` | **raw bytes**（`image/png` 或 `image/jpeg`；非 JSON）。native `WKWebView.takeSnapshot`：所见即真实渲染，**页面 canvas 加噪只作用于 JS 读回、native 天然免疫**。`max_width` 为逻辑点数（Retina 下 ~2×px） |
| POST | `/upload/arm` | `{paths: [绝对路径…]}` | `{ok, armed: <数>}`（须全部文件存在否则 400）。预置待上传文件：随后一次**可信点击** `<input type=file>` 唤起面板时自动完成选择（一次性消费即清）。未 arm 的真手点击 → 回退系统文件选择面板 |

### cookie 格式与不对称

Netscape cookies.txt 7 列：`domain  flag  path  secure  expires  name  value`。

- **导入**：`flag=TRUE` 自动补前导点；HttpOnly 恒置 YES（格式无 httpOnly 列）
- **导出**：如实包含 httpOnly
- round-trip 后 HttpOnly 标记丢失**是单向的**：导出写不出这一列，但**导入时每一条都会被置为 HttpOnly** —— 原本不是 HttpOnly 的 cookie 会被改成 HttpOnly，此后页面里 `document.cookie` 读不到它

---

## Python SDK

```python
from aug_sdk import Client
a = Client().browser("你的档案ID")     # 端口按档案数据取，无需手动给
a.navigate("https://example.com")
a.wait_for("h1")
print(a.read("h1"))
a.click("button.submit")
a.type("#email", "user@example.com")   # 脚本注入 value（非真人键入；零痕迹走 Human Paste）
a.export_cookies()                     # -> cookies.txt 文本
a.import_cookies(text)                 # -> 导入条数
a.set_window_frame(x, y, w, h)         # 移动/缩放窗口（contentRect）
png = a.screenshot()                   # native 截图 → PNG bytes（所见即真实渲染）
dom = a.page_map()                     # 结构化页面地图：骨架 + 可见交互元素（id/sel/rect）
a.click(dom["elements"][0]["sel"])     # 用 map 给的 sel 精确点击，不再猜选择器
```

方法 | 对应 endpoint | 说明
-----|--------------|------
`navigate(url)` / `evaluate(js)` / `tabs()` / `new_tab(url?)` / `switch_tab(i)` / `close_tab(i)` | bridge 同名 | 核心原语
`click(sel, index=0)` / `type(sel, text)` / `read(sel)` / `wait_for(sel, timeout)` | `/evaluate` 拼 JS | 派生原语（同步值）
`import_cookies(text, timeout=120)` | `/cookies/import` | timeout 放宽到 120s：引擎侧并发 setCookie + per-cookie 2s 兜底
`export_cookies(timeout=15)` | `/cookies/export` | 档案未运行 → `ConnectionRefusedError`
`set_window_frame(x,y,w,h)` | `/window/frame` | 一键排列用
`replace_cookies(text, timeout=120)` | `/cookies/replace` | 先清空再注入，登录态切换不重启引擎
`human_click(sel, i=0, pace=1)` / `human_type(sel, text, speed?)` / `human_hover(sel, i=0, pace=1)` | HID（CGEvent） | 拟人输入：真实 HID + 档案 persona 指纹，`isTrusted=true`；**单前台依赖**
`human_key(key, selector?)` | HID | 可信原生单键：仅 `Enter`（映射 Return，触发原生提交）；其余键 ValueError → 走 `press_key` JS dispatch
`arm_upload(paths)` | `/upload/arm` | 文件上传预置（配合可信点击自动完成；一次性消费）
`press_key(key, selector?)` | `/evaluate` 派发 KeyboardEvent | `key` 见工具描述；JS 注入 `isTrusted=false`
`scroll(dy)` / `scroll_state()` | `/evaluate` | `window.scrollBy` / 滚动高度读数
`page_map(...)` | `/evaluate`（只读 DOM 枚举） | 结构化页面感知：骨架 + ≤200 可见交互元素 `{id,tag,role,text,sel,rect}`；不写 DOM/canvas → 指纹零暴露
`resolve_sel(element_id)` | 重跑 `page_map` 枚举 | element_id → sel（现场重解析防 stale）
`wait_nav()` | SSE | 等导航加载完成（10s 超时）

---

## MCP：stdio server

MCP server 随 AUG WorkSpace 一起分发，位于应用包内：

```
「你的 AUG.app」/Contents/Resources/engine/core/mcp_server.py
```

自测（**必须用包内自带的运行时并带 `-B`**：`core/` 与运行时都住在**已签名**的应用包里，
CPython 缺省会把 `.pyc` 写回它导入的每个模块旁边 ⇒ **往签名包里新增文件会当场破坏封签**，
`codesign --verify --strict` 报 `a sealed resource is missing or invalid`；用系统 `python3`
指向包内也一样会写）：

```bash
"「你的 AUG.app」/Contents/Resources/engine/pyruntime/bin/python3" -B "<上面的路径>" --smoke
```

### framing

**newline-delimited JSON（NDJSON）**，非 Content-Length（那是 LSP）。客户端逐行读，
每行一个完整 JSON-RPC 2.0 消息。**日志一律 stderr**（stdout 是协议通道）。

### 协议

- 版本协商：`initialize` 回显 `protocolVersion`（目标 2025-03-26，兼容 2024-11-05）+ `capabilities.tools` + `serverInfo`
- `notifications/initialized` 不响应；`ping` → `{}`
- 未知 method → JSON-RPC `-32601`；解析失败 `-32700`
- **工具执行失败走 `result.isError=true`**（非 JSON-RPC error）

### 工具

| 工具 | 说明 |
|------|------|
| `list_profiles` | 全部 profile 摘要（代理密码脱敏） |
| `get_profile` | 单个 profile 完整数据（密码脱敏） |
| `launch_profile` | 启动 profile 浏览器；代理 profile 自 spawn 本地 socks5 转发器接上游（与 GUI 同路径，无需 GUI；`stop_profile` 可停） |
| `navigate` / `evaluate` / `tabs` / `click` / `read` | 复用 bridge 能力；档案未运行 → 提示「先 launch_profile」 |
| `stop_profile` | 优雅停**本 MCP 启动**的档：SIGTERM → 引擎自存 cookie jar 退出 + 释放并发槽；GUI 起档不受影响 |
| `upload` | file input 上传：arm 预置路径 + 可信 HID 点击自动完成（单前台/AX）；JS 点击开不了面板 |

> 完整工具清单以 `--smoke`（tools/list）为准。

---

## 安全注意

- bridge 只绑 127.0.0.1；`get_profile` / 摘要类接口一律脱敏代理 `username/password`
- 本机所有进程都可访问 bridge（无鉴权）——本机运行者即信任边界
