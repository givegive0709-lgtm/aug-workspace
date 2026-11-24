# AUG WorkSpace SDK — 完整参考

AUG WorkSpace 反检测浏览器（macOS）客户端 SDK。**独立 pip 包 `aug-sdk`**，Python 3.9+。**纯标准库面**：HTTP bridge 驱动/本机自动化（自动化面不引第三方）；**唯一硬依赖 = `cryptography>=41.0`**，仅云端同步的端到端加密路径使用。

能力 = **本机控制面（生命周期）+ 本机驱动（页面）**。

> 本页只写**驱动浏览器**需要的东西：装 SDK → 起控制面 → 建档/启动 → 驱动页面 → 收尾。
> 与本机驱动无关的功能都在 **AUG WorkSpace 应用里**，**不在本页范围**。

> ## 先看这一页的「哪一段与我有关」
>
> | 你是 | 读这几节 |
> |---|---|
> | **企业接入**（Java/Go/Node/Python 驱动浏览器） | 安装 → 快速开始 → `ControlClient`（建档/启动/停止/删除/工作区/cookie/运行态）→ `Automator`（页面驱动）→〈REST 端点参考〉（语言无关） |
> | **要从 Selenium 迁移** | 再加〈Selenium 兼容层〉一节（那里逐条写了会踩的差异） |
>
> **一句话分工**：`ControlClient` 管**档案的生死**（`http://127.0.0.1:8311`，`Bearer`），
> `Automator` 管**页面里发生什么**（档案自己的 bridge 端口，无鉴权）。两者地址、鉴权、用途都不同，别混用。

---

## 安装

```bash
pip install <SDK 目录路径>     # SDK 由我们提供，未随 AUG.app 打包
```

如果拿到的是 wheel 文件，也可以直接装：

```bash
pip install https://aug.jokerdao.win/updates/aug_sdk-<版本>-py3-none-any.whl
```

## 快速开始

```python
from aug_sdk import Client

# ① 控制面（本机 127.0.0.1:8311，Bearer）：管档案的生死
c = Client()                       # key 取自 $AUG_API_KEY；不设则服务端只读
p = c.control.create_profile(name="s1", geo="US", no_proxy=True, group_id="batch-1")
pid = p["created"][0]["id"]
c.control.launch_profile(pid, url="https://example.com")
print(c.control.profile_status(pid))        # 启动是**异步**的：要轮询到 running=True

# ② bridge（档案自己的端口，无鉴权）：管页面里发生什么
c.browser(pid).navigate("https://example.com")
```

---

## 安全边界（读前必读）

| 能力 | 传输 | 鉴权 | 说明 |
|------|------|------|------|
| **控制面**（`ControlClient`） | HTTP `127.0.0.1:8311` | **`Authorization: Bearer <AUG_API_KEY>`** | 建档/启动/停止/删除/工作区/运行态。不设 key = **本机只读**（执行类回 `403 local_readonly`）。**远程监听**须同时给 key + 限流 + `AUG_API_ALLOWED_HOSTS`（未声明的 Host 一律 403，防 DNS rebinding） |
| 自动化（bridge） | HTTP 127.0.0.1 | **无**（本机信任边界） | 每 profile 进程内嵌 server；仅本机进程可访问。本机运行者即信任边界 |
| 远程命令中继 | — | — | **不提供** |

档位门控（**自动化接口 = Team 及以上**）在**服务端判**，客户端不判：
不带 key 时控制面只允许**本机只读**，执行类调用会被明确拒（`403 local_readonly`）。
**API key 在 AUG WorkSpace 应用内创建与吊销**（设置页），把它交给你的服务端/CI 即可。

---

## Python SDK 方法表

### 入口 `aug_sdk.Client`

| 成员 | 说明 |
|------|------|
| `Client(api_key=None)` | `api_key` 缺省读 `AUG_API_KEY`；不设 ⇒ 控制面处于**本机只读**模式 |
| `.control` | `ControlClient` 实例（懒创建）—— 本机控制面，管档案生命周期 |
| `.browser(profile_id, port=None)` | `Automator`（驱动本机已运行档案） |
| `.browser(profile_id, port=None)` | `Automator`（驱动本机**已运行**档案） |

### 控制面 `aug_sdk.ControlClient`（**企业接入的主入口**）

构造：`ControlClient(base=None, api_key=None, timeout=120, retry=False)`。
`base` 缺省 `http://127.0.0.1:8311`（可用 `AUG_API_BASE` 覆盖）；`api_key` 缺省读 `AUG_API_KEY`。
失败**抛异常**（`AugApiError(code, message, status)`），429 单独抛 `RateLimited(retry_after, queue_depth)`；
`retry=True` 时按 `Retry-After` 自动退避重试。

| 方法 | 说明 |
|------|------|
| `health()` / `tools()` | 探活 / **权威**接口清单（名字+说明+入参 schema，注册表生成） |
| `create_profile(**kwargs)` | 建档。键名与 schema 同源：`name` `geo` `group_id` `browser_version` `location` `probe_egress` `no_proxy` `proxy` `count` …（拼错 → **400 并列出可接受键**） |
| `list_profiles(**kw)` / `get_profile(id, include_secret=False)` | 列表 / 单档（含密钥时 `secrets` 单独一层） |
| `update_profile(id, **kw)` | 回写 `notes` / `totp_secret` / `passkey` / `save_password`（给哪个键改哪个；`save_password`：true=总是问、false=从不问、null=回默认） |
| `profile_status(id)` | 运行态：**真连一次 bridge**（连不上＝没在跑）。支持批量探测 |
| `launch_profile(id, url=None, runtime_proxy=None)` | 启动；`runtime_proxy` 是**本次会话**的出口覆盖（不改档案） |
| `stop_profile(id)` | 停**本进程启动**的档（GUI 起的档不受影响） |
| `delete_profile(id)` / `restore_profile(id)` / `list_trash()` / `purge_profile(id)` | **软删**（进回收站，可恢复）→ 恢复 / 列回收站 / **彻底删**（清 datastore 与密钥，不可恢复） |
| `navigate(id, url)` / `current_url(id)` / `page_source(id, max_bytes=None)` | 页面导航与内容 |
| `evaluate(id, js, await_promise=False)` | 任意 JS 求值；`await_promise=True` 等 Promise 结算 |
| `screenshot(id, fmt="png", max_width=None, quality=None)` | **返回原始图片字节**（不是 JSON 信封） |
| `click(id, **kw)` / `type_text(id, **kw)` / `press_key(id, key)` / `hover(id, **kw)` / `wait_for(id, selector)` / `read(id, selector=None)` / `page_map(id, **kw)` | 拟人操作与页面感知。**并发**要在 `click`/`type_text` 里显式写 `human=True, input="inline"`（缺省 `hid` 是全机串行） |
| `tabs(id)` / `new_tab(id, url=None)` / `switch_tab(id, i)` / `close_tab(id, i)` | 标签页 |
| `fill_credential(id)` / `save_credential(id, password, username="")` | **凭据库**：`fill` 把本档案在**当前页 host** 下保存的第一条凭据填进登录表单；`save` 把凭据写进该 host 的钥匙串条目（同 host 同用户名合并）。两者**都不回密码**（`fill` 只回 `filled/username/reason`）；`save` 的 host 由引擎取当前页，不接受调用方指定 |
| `export_cookies(id)` / `import_cookies(id, cookies)` / `replace_cookies(id, cookies)` | cookie **JSON 全字段**通道（类型错**逐字段**进 `dropped`，不静默丢） |
| `clear_cache(id, origin=None)` / `clear_data(id, origin=None)` | 不依赖 CDP 的清缓存/清源数据（不给 origin = 全清） |
| `workspaces(**kw)` / `create_workspace(name)` / `create_workspaces(names)` / `rename_workspace(id, name)` / `delete_workspace(id)` / `delete_workspaces(ids)` | 工作区（＝分组；与界面同一份数据）。**批量**：`create_workspaces([...])` 一次写盘、重名/空名/保留名逐条进 `skipped`；`delete_workspaces([...])` 一次写盘、`default` 与不存在的 id 逐条进 `refused`。**删组只摘标签，不动任何档案本体**（回执里每条 `deleted` 带 `untagged_profiles`） |
| `profile_workspaces(id, add=None, remove=None)` | 归属读写（只给 id = 只读；**多标签**语义） |
| `call(name, arguments=None)` | 逃生口：任何工具都可达（`tools()` 看清单） |

> ⚠️ **三条使用前提**（门 `sdk_control` / `lease` 钉住）：
> ① **驱动类与 cookie 类端点要求档案已在运行** —— 先 `launch_profile`，否则回
>   `400 浏览器未运行：先调用 launch_profile`；
> ② **启动是异步的** —— `launch_profile` 返回 ≠ 已就绪，判「起来了没有」要**轮询** `profile_status`；
> ③ **拟人操作也是异步的** —— `click` / `type_text`（`human:true`）**立刻**回 `started:true`，
>   动作在后台逐字/逐帧完成 ⇒ **判有没有生效要回页面读值**（`evaluate` 轮询），不能只看回执。
>
> ⚠️ **自动化接口需要授权**：`AUG_API_KEY` 必须在云端校验通过且档位 **Team 及以上**
> （不是 → `403 tier_required`）。首次校验通过会在本机落下**一份 24 小时离线租约**，期间断网照常；
> 过期或没有租约时，**本地引擎**会在自动化接口上回 `403 lease_required`
> （只读接口——查标签页、导出 cookie——**不受影响**）。
>
> ⚠️ **资源不存在回 `404 not_found`**（2026-09-25 起）。此前一律折成 `400 tool_error`，
> 客户端容易读成「我请求写错了」。错误码由**抛出点**决定（`ToolError(code=…)`），不按文案匹配。

### 本机驱动 `aug_sdk.Automator`

构造：`Automator(profile_id, port=None)`（port 缺省**从档案数据里读**，见「端口怎么拿」）。

| 方法 | 说明 |
|------|------|
| `navigate(url)` | 当前 tab 导航 |
| `evaluate(js)` | 任意 JS 求值，返回同步 JSON 可序列化值 |
| `tabs()` / `new_tab(url)` / `switch_tab(i)` / `close_tab(i)` | tab 管理 |
| `click(selector, index=0)` | 脚本点击（非真人输入） |
| `type(selector, text)` | 注入 `value` + 触发 input 事件（零痕迹键入走 Human Paste） |
| `read(selector)` | 首元素 `textContent`（无匹配 → None） |
| `wait_for(selector, timeout=10, interval=0.3)` | 轮询等待，超时 False |
| `import_cookies(text, timeout=120)` | 注入 Netscape cookies.txt，返回成功条数 |
| `export_cookies(timeout=15)` | 导出全部 cookie 为 cookies.txt |
| `set_window_frame(x, y, w, h)` | 窗口 contentRect（AppKit 左下原点） |
| `human_click(sel, i=0, pace=1)` / `human_type(sel, text, speed?)` / `human_hover(sel, i=0, pace=1)` | 真实 HID 拟人输入（CGEvent isTrusted=true + persona 指纹；单前台） |
| `human_key(key, selector?)` | 可信原生单键：仅 `Enter`（真实 HID Return 触发原生提交）；其余 ValueError |
| `arm_upload(paths)` | 文件上传预置（`/upload/arm`）：随后一次可信点击 file input 自动完成 |

> 本表为常用子集，完整清单（含 `screenshot` / `page_map` / `scroll` / `replace_cookies` / `wait_nav` 等）见 `python3 -c "import aug_sdk; help(aug_sdk.Automator)"`。

异常：`RuntimeError`（bridge 返回 `ok:false`）、`urllib.error.URLError` / `ConnectionRefusedError`（档案未运行）。

### Selenium 兼容层 `aug_sdk.selenium_adapter`

让 Selenium 生态脚本最小改动驱动 AUG WorkSpace。

```python
from aug_sdk.selenium_adapter import AUGDriver, By
driver = AUGDriver("PROFILE-ID")            # 档案须已运行（GUI/MCP 启动）
driver.get("https://example.com/login")
driver.find_element("id", "email").send_keys("user@example.com")
driver.find_element("css selector", "button[type=submit]").click()
driver.wait_for("h1", timeout=15)          # WebDriverWait 的等价物
```

> ⚠️ **`By` 是普通 dict，不是类**（`selenium_adapter.py:41` = `By = {"ID": "id", …}`）。
> 写 `By.ID` 会抛 `AttributeError: 'dict' object has no attribute 'ID'`。
> 要用就写 `By["ID"]`；不过 `find_element(by, value)` 收的本来就是字符串，
> 像上面那样直接传 `"id"` / `"css selector"` 最省事。

| 兼容接口 | 说明 |
|---------|------|
| `get(url)` / `title` / `current_url` / `page_source` | 导航 + 页面状态（get 内部 wait_nav） |
| `find_element(by, value)` / `find_elements` | by 支持 id/name/class name/tag name/css selector/xpath/link text/partial link text |
| `element.click()` / `send_keys(text)` / `clear()` / `.text` / `get_attribute(name)` | 元素操作 |
| `execute_script(js, *args)` | 等价 Selenium execute_script（args 替换 `arguments[0..n]`） |
| `wait_for(css, timeout)` | WebDriverWait.until 等价物 |
| `switch_to(i)` / `new_tab` / `close` | tab 用 index（无稳定 window handle） |
| `set_window_size(w, h)` | 设置窗口尺寸（x/y 缺省 100,100） |
| `send_keys(..., human=False)` | 退化为 JS 注入 value |

**五条必须先知道的差异**（是架构选择，不是功能缺失）：

1. **等待写法不报错，但基本等不到 —— 请用 `driver.wait_for(...)`**。`WebDriverWait` 本身只是轮询器，`expected_conditions` 内部就是调 `find_element`，所以写法能接上。但：
   - `presence_of_element_located`：元素已在时能返回；**元素不在时**本适配器抛的是 `RuntimeError`（见下条），而 Selenium 的等待**默认只忽略 `NoSuchElementException`** —— 于是**第一轮就抛出去，不再轮询**
   - `visibility_of_element_located`、`element_to_be_clickable`：**直接报 `TypeError: 'bool' object is not callable`**。原因是本适配器的 `is_displayed` 是**属性**，而 Selenium 是按**方法**调用
2. **异常类型不同**：`find_element` 找不到元素时抛的是 **`RuntimeError`**，不是 `NoSuchElementException` —— 如果你写了 `except NoSuchElementException`，要改成捕获 `RuntimeError`
3. **元素方法不全，而且有的是属性**：`AUGElement` 只有 `click` / `send_keys` / `clear` / `text` / `get_attribute` / `is_displayed` / `tag_name`。其中 `text`、`is_displayed`、`tag_name` 是**属性**（读的时候不加括号）。另外**没有 `is_enabled` / `is_selected`** —— 所以 `element_to_be_selected` 这类会报 `AttributeError`
4. **`send_keys` 默认真实 HID 拟人键入**（CGEvent 逐字 + 档案 persona 打字指纹）—— 超 Selenium 的地方。XPath 定位的元素自动退化为 JS 注入
5. **`quit()` 无操作**：停档案走 GUI 或 MCP（bridge 无 stop 端点）


## REST 端点参考（语言无关）

任何语言可自行实现。要点复述：

### bridge（本机，127.0.0.1）

base = `http://127.0.0.1:<port>`，port 取法见「端口怎么拿」。POST JSON，响应 JSON，`ok:true/false`。
cookie 格式：Netscape cookies.txt。


---

## 端口怎么拿（硬约束）

**端口是档案数据，不是可以自己推导的常量。** 每个档案的控制端口在启动时分配（避撞）并落盘，此后固定不变。

`Automator(profile_id)` 已内建「先读档案字段」的顺序 —— 正常情况无需关心端口，直接构造即可。

- ⚠️ **不要**自己按公式算端口去连一个已存在的档案 —— 档案一多就会撞车。撞了以后不止「谁连不上谁」，而是可能连到**别人的档案**上去。
- 任何语言走 REST 时，端口同样请从档案数据里读。

---

## 与 AUG.app 的分工

| 事 | 谁做 |
|----|------|
| 启动档案（代理转发器 spawn + preflight + cookie jar + inject-dir + disconnectOnIPChange） | AUG WorkSpace 界面或 MCP `launch_profile`。**SDK 不做**（防双实现漂移） |
| 驱动已运行档案 | SDK `Automator` / MCP |
| API key 的创建与吊销 | **AUG WorkSpace 应用内**（设置页；不在本 SDK 文档范围） |
| 档位判定 | 服务端（SDK/MCP 客户端都不判） |
| 远程命令中继 | 不提供 |
