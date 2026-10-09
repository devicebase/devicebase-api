# Devicebase OpenAPI

Devicebase 开放 API 服务协议文档。

**API Version**: 2026.9.20

## 基础信息

| 项目 | 说明 |
|------|------|
| Base URL | `https://api.devicebase.cn`（自建环境由 `DEVICEBASE_BASE_URL` 指定） |
| 协议 | HTTP/1.1，HTTPS |
| 数据格式 | 请求与响应均为 `application/json`（截图接口除外，返回二进制图片） |
| 字符编码 | UTF-8 |

### 认证

所有接口都需要 API Key。以下三种携带方式**任选其一**，服务端按此顺序识别：

| 方式 | 示例 |
|------|------|
| `Authorization` 头 | `Authorization: Bearer <your_api_key>`（`Bearer ` 前缀可省略） |
| `X-API-Key` 头 | `X-API-Key: <your_api_key>` |
| 查询参数 | `?api_key=<your_api_key>` |

API Key 可从 <https://www.devicebase.cn/> 获取。缺失或无效时返回 **401**：

```json
{
  "code": 401,
  "message": "缺少 API Key，请通过 Authorization: Bearer、X-API-Key 头部或 ?api_key= 参数提供，登录 https://www.devicebase.cn/ 获取 API Key",
  "trace_id": "20c46a79-6a64-445b-b2cb-336bee7a00b7"
}
```

### 设备标识

**所有设备的标识字段是 `serialno`**，由平台签发，形如 `db-mttul4i41di8`。

设备行同时携带 `device_sn`（物理序列号，UUID）。网关按 `WHERE (serialno = ? OR device_sn = ?)` 查找设备，因此两者都能用，但 **`serialno` 是主键**，新代码应只使用它。

> 历史字段 `serial` 已废弃：设备表中不存在该列（只有 `serialno` 与 `device_sn`），使用它会直接报 `ER_BAD_FIELD_ERROR`。

### 三类设备

不同平台的接口位于不同的路径族，`serialno` 只在所属平台内有意义：

| 平台 | 覆盖 | 路径族 |
|------|------|--------|
| mobile | Android / HarmonyOS / iOS | `/v1/{action}/{serialno}` |
| browser | Chrome / Chromium / Edge（CDP） | `/api/browser/{serialno}/{action...}` |
| computer | macOS / Windows / Linux 桌面 | `/api/computer/{serialno}/{action}` |

移动端路径由控制服务注册后会 302/307 重定向到对应的 `/api/*` 处理器，方法与请求体在重定向后保持不变。

### 响应格式

平台自身合成的响应（设备列表、健康检查、登录、Agent 回调）统一为：

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "trace_id": null
}
```

经由 Agent 代理的设备操作（移动端动作、browser 族、computer 族）在上游成功时**原样透传**上游信封，其格式为：

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "timestamp": "2026-09-20 07:33:00"
}
```

两者的区别只在 `timestamp` / `trace_id` 字段：**客户端应以 `code` 为准，不要依赖这两个字段。**

### ⚠️ 错误判定：必须同时检查 HTTP 状态与信封 `code`

控制 API 在动作失败时**返回 HTTP 200，但信封内的 `code` 非 2xx**。只判断 HTTP 状态码会把失败当作成功。

```bash
curl -s -X POST "$BASE/v1/current_app/$SERIALNO"
# HTTP/1.1 200 OK
# {"code":502,"message":"Element not found: #definitely-missing-element"}
```

因此正确判定是：

1. HTTP 状态码非 2xx → 请求失败；
2. HTTP 状态码 2xx，但 `code` 不在 `[200, 300)` → **业务失败**。

以截图接口为例：当采集失败时，服务端同样返回 HTTP 200 + `{"code":503,...}` 的信封而非图片，客户端必须先嗅探内容再决定是否当作图片处理。

### HTTP 状态码

以下取值即网关的完整状态码文案表：

| 状态码 | 文案 | 说明 |
|--------|------|------|
| 200 | — | 成功，或上游业务失败（后者需检查信封 `code`） |
| 400 | 请求参数错误 | 网关对校验错误返回 400（历史实现曾返回 422） |
| 401 | 未授权 | API Key 缺失、格式错误或已停用 |
| 403 | 权限不足 | 设备被他人占用、余额不足、非本人私有设备等 |
| 404 | 资源不存在 | `message` 中会带上出错的 serialno |
| 405 | 不允许的请求方法 | |
| 408 | 请求超时 | |
| 409 | 请求冲突 | 设备注册等场景 |
| 422 | 无法处理的实体 | |
| 429 | 请求过于频繁 | 触发限流 |
| 500 | 服务器内部错误 | |
| 502 | 网关错误 | 驱动侧动作失败（通常以 HTTP 200 + `code:502` 出现，见下） |
| 503 | 服务暂不可用 | `请求失败: ...` 多为 Agent 侧不可达 |
| 504 | 网关超时 | |

---

## 设备列表

### List Devices

对当前 API Key 可见的设备列表。**这是所有其他接口的起点**：用它查出 `serialno`，再调用对应的平台接口。

```
GET /v1/devices
```

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| `keyword` | string | query | 否 | 关键词，大小写不敏感的子串匹配，覆盖 `name` / `alias_name` / `brand` / `model` / `serialno` / `device_sn` / `type` / `os_type` / `os_version` / `location` / `operator` |
| `state` | string | query | 否 | 设备状态：`busy` / `free` / `offline`。`busy` 只返回本人占用的设备，别人共享的 busy 设备不算 |
| `type` | string | query | 否 | 类别桶（`mobile` / `browser` / `computer`）或系统类型（见下） |
| `limit` | int | query | 否 | 返回条数上限，默认 10，取值 1–100 |

`type` 的系统类型取值：`android`、`harmonyos`、`ios`、`macos`、`windows`、`linux`、`chrome`、`chromium`、`edge`、`other`。

系统类型**匹配设备的 `os_type`**，因为设备行只携带粗粒度的 `type` —— 一台 Chrome 浏览器是 `type=browser` + `os_type=Chrome`。其中两个按排除法定义：

- `linux`：所有既非 macOS 也非 Windows 的 computer（含 Deepin / UOS / Kylin 及未知系统）；
- `other`：所有非 Chrome / Chromium / Edge 的 browser。

未识别的取值退化为对 `type` 的精确匹配。

**可见性**：返回 `is_shared = 1` 的共享设备加上当前用户自己的设备。

```bash
curl -s "$BASE/v1/devices?type=browser&state=free&limit=10" \
  -H "Authorization: Bearer $DEVICEBASE_API_KEY"
```

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "id": 10,
      "serialno": "db-mtthisv311f1",
      "device_sn": "f3ad1396-4fb9-4037-81e7-7496f261f3f4",
      "state": "free",
      "name": "Richie-Macbook-Air-7.local",
      "alias_name": "Richie-Macbook-Air-7.local",
      "udid": "b2:da:29:43:81:03",
      "type": "computer",
      "brand": "Apple",
      "model": "MacBook Air",
      "os_type": "macOS",
      "os_version": "26.0",
      "display": "1470x956",
      "location": "Beijing",
      "operator": "CMCC",
      "network": "wifi",
      "is_cloud": false,
      "updated_at": "2026-09-20T15:11:31"
    }
  ]
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | int | 设备主键 |
| `serialno` | string | **平台签发的设备标识**，所有控制接口使用该值 |
| `device_sn` | string | 物理序列号（UUID），网关同样可解析 |
| `state` | string | `busy` / `free` / `offline` |
| `name` / `alias_name` | string | 设备名与用户设置的别名 |
| `udid` | string | 设备 UDID |
| `type` | string | 粗粒度类型：`adb` / `hdc` / `ios` / `browser` / `computer` |
| `brand` / `model` | string | 品牌与型号 |
| `os_type` / `os_version` | string | 操作系统与版本，`type` 的系统类型过滤依据 |
| `display` | string | 分辨率，如 `1470x956` |
| `location` / `operator` / `network` | string | 位置、运营商、网络类型 |
| `is_cloud` | bool | 是否是**云浏览器**（平台在你的集群配额里替你建的那一种）。自己接进来的浏览器 `type` 同样是 `browser`，只有这个字段能把两者分开 —— 云浏览器才计入配额，也只有它能用下面的删除接口 |
| `updated_at` | string | 最后更新时间，**无时区偏移**（本地时间） |

> 该响应**没有 `trace_id`、`timestamp`、分页字段**，`data` 直接是设备数组。

---

## 设备控制 - 移动端

路径模板：`POST/GET /v1/{action}/{serialno}`。坐标单位为设备像素。

### Touch

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Tap | POST | `/v1/tap/{serialno}` | `{"x": int, "y": int}` |
| Double Tap | POST | `/v1/double_tap/{serialno}` | `{"x": int, "y": int}` |
| Long Press | POST | `/v1/long_press/{serialno}` | `{"x": int, "y": int}` |
| Swipe | POST | `/v1/swipe/{serialno}` | `{"x1": int, "y1": int, "x2": int, "y2": int}` |

```bash
curl -s -X POST "$BASE/v1/tap/$SERIALNO" \
  -H "Authorization: Bearer $DEVICEBASE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"x": 100, "y": 200}'
```

```json
{"code": 200, "message": "success", "data": {}, "timestamp": "2026-09-20 07:33:00"}
```

> `long_press` **不接受** `duration` 字段；`swipe` 的 `duration` 为可选，省略时由驱动决定。

### Navigation

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Back | POST | `/v1/back/{serialno}` | 无 |
| Home | POST | `/v1/home/{serialno}` | 无 |

### App

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Launch App | POST | `/v1/launch_app/{serialno}` | `{"app_name": string}` |
| Stop App | POST | `/v1/stop_app/{serialno}` | `{"app_name": string}` |
| Stop Current App | POST | `/v1/stop_current_app/{serialno}` | 无 |
| Current App | POST | `/v1/current_app/{serialno}` | 无 |

`app_name` 为 Android/HarmonyOS 的包名或 iOS 的 Bundle ID。`current_app` 返回：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "app_name": "荣耀桌面",
    "package_name": "com.hihonor.android.launcher"
  },
  "timestamp": "2026-09-20 07:32:59"
}
```

> 返回字段是 `app_name` 与 `package_name`，**没有 `activity` 字段**。

### Text

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Input Text | POST | `/v1/input/{serialno}` | `{"text": string}` |
| Clear Text | POST | `/v1/clear_text/{serialno}` | 无 |

### State

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Device Info | POST | `/v1/deviceinfo/{serialno}` | 无 |
| Dump Hierarchy | POST | `/v1/dump_hierarchy/{serialno}` | 无 |

`deviceinfo` 的设备信息包在 `device` 键下：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "device": {
      "id": "ABJKVB2705004635",
      "serial": "ABJKVB2705004635",
      "udid": "ABJKVB2705004635",
      "name": "FNE-AN00",
      "type": "adb",
      "is_connected": true,
      "display": { "width": 1080, "height": 2400 }
    }
  }
}
```

> `serial` 是 **Agent 上报的物理序列号**，与 `/v1/devices` 的 `serialno` 不是同一个值。

`dump_hierarchy` 的 `hierarchy` 是**字符串**（原始 XML/JSON 文本），不是嵌套对象：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "hierarchy": "[{\"bounds\":\"[0,0][1080,2400]\",\"type\":\"com.hihonor.uikit.hwscrollview.widget.HwScrollView\",\"desc\":\"第 1 屏，共 2 屏\"}]"
  }
}
```

### Shell

```
POST /v1/bash/{serialno}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `command` | string | 是 | 待执行的 shell 命令 |

仅 adb / hdc 平台支持。**命令自身的退出码在响应载荷里，不是 API 错误**：

```json
{
  "code": 200,
  "message": "success",
  "data": { "exitCode": 0, "stdout": "hello\n", "stderr": "" },
  "timestamp": "2026-09-20 07:33:00"
}
```

### Install

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Install App | POST | `/v1/install_app/{serialno}` | `{"app_path": string}` |
| Install Status | GET | `/v1/install_status/{serialno}?install_id={id}` | 无 |

`app_path` 是 **Agent 宿主机上的路径**，不是本地文件。安装是后台任务，`install_app` 返回 `installId`，用 `install_status` 轮询。

### Screen

```
POST /v1/screen/{serialno}      # 推荐
GET  /v1/screen/{serialno}
```

返回**原始图片字节**（非 JSON、非 base64），`Content-Type` 由上游决定，通常为 `image/jpeg`：

```bash
curl -s -X POST "$BASE/v1/screen/$SERIALNO" \
  -H "Authorization: Bearer $DEVICEBASE_API_KEY" \
  -o screen.jpg
```

这是**跨平台**接口：服务端按设备类型决定采集路径 —— computer 全屏采集、browser 走 CDP、其余走设备图像队列。因此 browser / computer 的 `serialno` 同样可用。

> 图片格式由**服务端**决定（JPEG），与输出文件名无关。把返回的 JPEG 存成 `.png` 会得到一个贴错标签的文件。

另有一个仅 GET 的附件下载变体，等价于同一张图，以附件形式返回：

```
GET /v1/screenshot/{serialno}
```

它不在命令行客户端暴露的接口集合内，新代码建议统一使用 `/v1/screen`。

---

## 设备控制 - 浏览器

路径模板：`POST/GET /api/browser/{serialno}/{action...}`。`serialno` 取自 `list-devices --type browser`。选择器为 **CSS 选择器**。

读操作（`state` / `tabs` / `text` / `attribute` / `exists`）为 **GET 且无请求体**，选择器通过查询参数传递。

### Navigation

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Navigate | POST | `/api/browser/{serialno}/navigate` | `{"url": string}` |
| Refresh | POST | `/api/browser/{serialno}/refresh` | 无 |
| Go Back | POST | `/api/browser/{serialno}/go_back` | 无 |
| Go Forward | POST | `/api/browser/{serialno}/go_forward` | 无 |

### DOM

| 接口 | 方法 | 路径 | 请求体 / 查询参数 |
|------|------|------|-------------------|
| Click | POST | `/api/browser/{serialno}/click` | `{"selector": string}` |
| Fill | POST | `/api/browser/{serialno}/fill` | `{"selector": string, "value": string}` |
| Select | POST | `/api/browser/{serialno}/select` | `{"selector": string, "value": string}` |
| Text | GET | `/api/browser/{serialno}/text` | `?selector=` |
| Attribute | GET | `/api/browser/{serialno}/attribute` | `?selector=&attribute=` |
| Exists | GET | `/api/browser/{serialno}/exists` | `?selector=` |
| Execute | POST | `/api/browser/{serialno}/execute` | `{"script": string}` |

`execute` 在页面上下文中求值 JavaScript，权限与页面相同 —— 等同于对浏览器配置的 shell 访问。

### Text and Keyboard

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Input | POST | `/api/browser/{serialno}/input` | `{"text": string}` |
| Hotkey | POST | `/api/browser/{serialno}/hotkey` | `{"keys": string[]}` |

`input` 使用 CDP `Input.insertText` 写入当前聚焦元素，对中日韩文本可靠（区别于合成的按键事件）。

`hotkey` 的编辑类快捷键（全选、剪切、复制、撤销、重做）作用于页面；**浏览器外壳快捷键（如 `Control+t`）不可达** —— CDP 驱动的是页面而非浏览器 UI。

### Tabs and State

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| State | GET | `/api/browser/{serialno}/state` | 无 |
| Tabs | GET | `/api/browser/{serialno}/tabs` | 无 |
| Tab Open | POST | `/api/browser/{serialno}/tab/open` | `{"url": string}` |
| Tab Close | POST | `/api/browser/{serialno}/tab/close` | `{"tab_id": string}` |
| Tab Close All | POST | `/api/browser/{serialno}/tab/close_all` | 无 |
| Tab Switch | POST | `/api/browser/{serialno}/tab/switch` | `{"tab_id": string}` |

```bash
curl -s "$BASE/api/browser/$SERIALNO/state" -H "Authorization: Bearer $DEVICEBASE_API_KEY"
```

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "url": "chrome://new-tab-page/",
    "title": "新标签页",
    "viewport": { "width": 1366, "height": 681 },
    "tab_count": 1,
    "connected": true
  },
  "timestamp": "2026-09-20 07:32:59"
}
```

### Lifecycle

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Launch | POST | `/api/browser/{serialno}/launch` | 无 |
| Close | POST | `/api/browser/{serialno}/close` | 无 |

### Screenshot

browser 的截图同样使用跨平台的 `POST /v1/screen/{serialno}`（见上文）。

---

## 云浏览器

上面那些接口驱动的是一台**已经存在**的浏览器。这一组**创建与销毁**浏览器本身，走
`/v1/browser/*`，路径里没有 `serialno` —— 创建的那一刻它还没有。

「云浏览器」是平台在自己的集群上替你跑的那一种。你自己接进来的 Chrome 不是：它不计入
配额，也不能用这里的删除接口（设备列表的 `is_cloud` 把两者分开）。

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Quota | GET | `/v1/browser/quota` | 无 |
| Create | POST | `/v1/browser/create` | `{"name"?: string, "window_size"?: string, "wait_seconds"?: int}`（`name` 落到 `alias_name`） |
| Status | GET | `/v1/browser/{device_sn\|serialno}/status` | 无 |
| Delete | DELETE | `/v1/browser/{device_sn\|serialno}` | 无 |

**Create** —— 平台自己选址、自己签发授权凭据、自己派发给集群节点，调用方不选机器，也不会
被告知落在哪一台。字段全部可选（显式的 `null` 与不传同义）；`window_size` 形如 `1366x768`；
`name` 最长 100 字符，不传则由平台的命名规则生成。

云浏览器**恒为 headless**：它跑在没有人看着的集群机器上，没有显示器可开窗口。所以没有
`headless` 字段 —— 传 `headless: false` 会被拒（400）而不是被忽略，静默忽略会让调用方以为
窗口被关掉了。

```bash
curl -s "$BASE/v1/browser/create" -X POST \
  -H "Authorization: Bearer $DEVICEBASE_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"name":"my-browser","window_size":"1366x768"}'
```

```json
{"code": 200, "message": "success",
 "data": {"device_sn": "3f2a1b4c-0000-1111-2222-333344445555",
          "serialno": "db-mtabc123",
          "name": "Browser-3f2a1b4c",
          "alias_name": "my-browser",
          "registered": true},
 "trace_id": null}
```

响应里有两个「名字」，别混：

| 字段 | 是什么 |
|------|--------|
| `name` | 平台的内部身份名，恒为 `Browser-` + serial 前 8 位。不由调用方决定，也不随节点变化 |
| `alias_name` | **你要的那个名字**，原样返回（没传就是 `null`）。它在创建时由平台定下来，之后每一次重新注册都不会改写它 |

之所以要分开：节点给自己的实例起名时会补上端口（`<请求名>-<端口>`，用于同机多实例去重），
那是它对内的称呼。若把它写进设备行，用户给的名字会在创建十几秒后自己变成
`my-browser-37377` 这样 —— 所以名字随授权凭据一起送到节点、再原样带回来，落进
`alias_name`；`name` 始终是平台的身份名。控制台显示的也是 `alias_name`（为空时回退到 `name`）。

**创建是异步的**（浏览器要启动，并在下一次注册里把设备报上来），所以这条接口会**等它注册**，
好把调用方真正要用的那个键 —— `serialno` —— 一并交出去。等多久由 `wait_seconds` 决定：
`0`–`60`，不传时服务端默认 15；`0` 表示立刻返回（批量创建时用）。

等不到**不是失败**：`registered` 为 `false`、`serialno` 为 `null`，同时给出一个仍然有效的
`device_sn`（`/status` 与删除接口都认它）。慢不等于坏 —— 机器可能在冷启动、可能正在下载
Chromium；把这个过程报成错误会诱使调用方重试，而重试会建出第二台浏览器、占掉第二个配额。

> 这条接口因此是唯一**会阻塞**的：服务端最坏情况 = 派发（20s）+ 等待（`wait_seconds`）。
> 客户端的 HTTP 期限要按这个量级放宽，别用默认的 30s。

**Status** —— 那件事发生了没有。`registered: false` 是**正常答案，不是错误**（HTTP 200），
所以轮询它是安静的；注册之后同时给出 `serialno`、`name`、`alias_name`、`state` 与 `server_url`。
按 `device_sn`（创建返回的）或 `serialno` 查都可以，只查本人的设备。

```bash
curl -s "$BASE/v1/browser/3f2a1b4c-0000-1111-2222-333344445555/status" \
  -H "Authorization: Bearer $DEVICEBASE_API_KEY"
```

**Quota** —— 还能建几台，`{"limit":10,"used":3,"remaining":7}`。与创建时的校验同源，
所以它给出的数不会与创建的结果打架；只有云浏览器计入，「已用」不含自己接进来的设备。

**Delete** —— 立碑、入回收队列、软删，**不可恢复**，且与 `close` 不是一回事（`close` 只是
停掉 CDP 引擎，浏览器还在）。它**不等节点**：指令先落库，节点下一拍心跳领走，所以节点离线
时删除照样成功，创建刚返回（`registered: false`）时也删得掉。标识可以是 `serialno`，
也可以是创建返回的 `device_sn`。

```bash
curl -s "$BASE/v1/browser/db-mtabc123" -X DELETE -H "Authorization: Bearer $DEVICEBASE_API_KEY"
```

失败时按 HTTP 状态码区分**该不该重试**：

| 状态码 | 含义 | 重试 |
|--------|------|------|
| `400` | 参数形状不对（如 `window_size` 不是 `WxH`） | 改参数 |
| `404` | 不存在 / 不属于你 / 已经被删（三者同一个回答，避免成为探测别人设备的探针） | 否 |
| `409` | 配额已满（创建），或这台不是云浏览器（删除） | 否 —— 删掉不用的，或找管理员调上限 |
| `502` | 平台与云节点之间的鉴权没通过（签名 / 密钥 / 时钟） | 否 —— 需要人处理 |
| `503` | 当前没有可用节点，或联系不上选中的节点 | 是，稍后重试 |

---

## 设备控制 - 电脑

路径模板：`POST/GET /api/computer/{serialno}/{action}`。`serialno` 取自 `list-devices --type computer`。坐标为**屏幕绝对像素**。

读操作（`position` / `screen_size` / `permissions`）为 **GET 且无请求体**。

### Mouse

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Click | POST | `/api/computer/{serialno}/click` | `{"x": int, "y": int, "button"?: "left"\|"right"\|"middle"}` |
| Double Click | POST | `/api/computer/{serialno}/double_click` | `{"x": int, "y": int}` |
| Long Click | POST | `/api/computer/{serialno}/long_click` | `{"x": int, "y": int, "duration"?: int}` |
| Move | POST | `/api/computer/{serialno}/move` | `{"x": int, "y": int}` |
| Drag | POST | `/api/computer/{serialno}/drag` | `{"x1": int, "y1": int, "x2": int, "y2": int}` |
| Scroll | POST | `/api/computer/{serialno}/scroll` | `{"direction": "up"\|"down"\|"left"\|"right", "amount"?: int}` |

`click` 的 `button` 省略时由服务端按 `left` 处理；`long_click` 的 `duration` 单位为**秒**，省略时由驱动决定；`scroll` 的 `amount` 单位为滚轮步数，省略时由驱动决定。

### Keyboard

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Type Text | POST | `/api/computer/{serialno}/type_text` | `{"text": string}` |
| Press | POST | `/api/computer/{serialno}/press` | `{"key": string}` |
| Hotkey | POST | `/api/computer/{serialno}/hotkey` | `{"keys": string[]}` |

### System

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Position | GET | `/api/computer/{serialno}/position` | 无 |
| Screen Size | GET | `/api/computer/{serialno}/screen_size` | 无 |
| Permissions | GET | `/api/computer/{serialno}/permissions` | 无 |
| Launch App | POST | `/api/computer/{serialno}/launch_app` | `{"app_name": string}` |

```json
// GET /api/computer/{serialno}/screen_size
{"code": 200, "message": "success", "data": {"width": 1470, "height": 956}, "timestamp": "2026-09-20 07:32:30"}

// GET /api/computer/{serialno}/position
{"code": 200, "message": "success", "data": {"x": 404, "y": 559}, "timestamp": "2026-09-20 07:32:30"}
```

`permissions` 返回桌面控制所需的系统授权状态：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "platform": "darwin",
    "accessibility": "granted",
    "screenRecording": "granted",
    "hasX11": false,
    "hasXdotool": false,
    "hasScrot": false,
    "hasPowershell": false,
    "allGranted": true,
    "missingItems": []
  }
}
```

### Wait and Shell

| 接口 | 方法 | 路径 | 请求体 |
|------|------|------|--------|
| Wait | POST | `/api/computer/{serialno}/wait` | `{"seconds": number}` |
| Bash | POST | `/api/computer/{serialno}/bash` | `{"command": string, "timeout"?: int}` |

`wait` 的字段是**秒**（可为小数），由服务端阻塞相应时长，上限 300 秒。调用方需保证 HTTP 超时大于该值。

`bash` 在**设备所在的宿主机**上以桌面用户身份执行命令，不受沙箱限制，使用平台默认 shell（macOS/Linux 为 `/bin/sh`，Windows 为 `cmd.exe`）—— 因此 `[[ ]]` 等 bash 专有语法可能不可用。**等同于 shell 访问权限。**

`timeout` 单位为**秒**，省略时服务端按 120 秒处理。与移动端 `bash` 一样，命令自身的退出码在载荷中返回：

```json
{
  "code": 200,
  "message": "success",
  "data": { "exitCode": 0, "stdout": "mcp-ok\n", "stderr": "" },
  "timestamp": "2026-09-20 07:32:30"
}
```

### Screenshot

computer 的截图同样使用跨平台的 `POST /v1/screen/{serialno}`，返回全屏采集结果。

---

## 设备控制 - WebSocket

### 设备屏幕流 WebSocket

```
wss://{host}/v1/minicap/{serialno}?token={jwt}
```

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| `token` | string | query | 是 | **JWT**，不是 API Key |

> `token` 是经由登录接口签发的 JWT，服务端会对其做 `decodeJwt` 校验。传入 API Key 会被拒绝。

连接建立后按 Minicap 协议推送二进制帧：首帧为 banner（版本、宽高、像素格式），其后每帧为 4 字节小端长度 + JPEG 数据。

### 设备触摸控制 WebSocket

```
wss://{host}/v1/minitouch/{serialno}?token={jwt}
```

| 参数 | 类型 | 位置 | 必填 | 说明 |
|------|------|------|------|------|
| `token` | string | query | 是 | **JWT**，不是 API Key |

连接建立后按 Minitouch 协议接收文本指令，并在首帧返回各触点支持的最大压力等能力信息：

```json
{"action": "tap", "x": 100, "y": 200}
{"action": "swipe", "x1": 100, "y1": 500, "x2": 100, "y2": 200, "duration": 300}
```

---

## 错误响应

### 认证与限流

| 状态码 | 场景 |
|--------|------|
| 401 | API Key 缺失、格式错误或已停用 |
| 403 | 设备被占用、余额不足、非本人私有设备、Agent 权限未授予 |
| 429 | 触发限流 |

### 参数校验

网关对校验失败返回 **400**，`detail` 为**字符串**（不是数组）：

```json
{
  "code": 400,
  "message": "参数错误",
  "detail": "body.username: Field required; body.password: Field required",
  "trace_id": null
}
```

### 设备不存在

```json
{
  "code": 404,
  "message": "设备不存在: db-does-not-exist-0000",
  "trace_id": "3034f482-fd07-4d20-b5fb-3e9e2a911d61"
}
```

### 上游 / Agent 失败

设备操作经由 Agent 代理，Agent 不可达时返回 **503**，`message` 为上游给出的原因：

```json
{
  "code": 503,
  "message": "请求失败: fetch failed",
  "trace_id": "cfaca31b-6bdf-406a-9cab-017a0c165c50"
}
```

### 业务失败（HTTP 200）

**这是最容易漏判的一类。** 动作在驱动侧失败时，控制服务返回 HTTP 200，信封内的 `code` 才是真实状态：

```json
{
  "code": 502,
  "message": "Element not found: #definitely-missing-element"
}
```

客户端必须依据 `code` 判定成败。仅检查 HTTP 状态码会把这类失败当作成功返回给上层。
