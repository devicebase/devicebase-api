# Devicebase OpenAPI

Devicebase 开放 API 服务协议文档。

**API Version**: 2026.4.17

## 基础信息

| 项目 | 说明 |
|------|------|
| Base URL | `https://api.devicebase.cn` |
| 数据格式 | JSON |

所有接口均需在请求头中携带认证 API Key：

```http
Authorization: Bearer <your_api_key>
```

## 设备列表

### List Devices

获取当前用户可用的设备列表。

**过滤条件：**
- `deleted=0`
- `is_shared=1` 或用户私有的设备
- `state=free` 或 `state=busy` 但被当前用户占用的设备

```bash
curl -X GET "https://api.devicebase.cn/v1/devices?keyword=iPhone&state=free&limit=10" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| keyword | string | 否 | 关键词搜索（品牌/型号/序列号/名称/location/operator） |
| state | string | 否 | 设备状态筛选（busy/free/offline） |
| limit | integer | 否 | 返回设备数量，默认 10 |

**响应示例：**

```json
{
  "code": 200,
  "message": "success",
  "data": [
    {
      "id": 1,
      "user_id": 1,
      "serial": "ABC123456",
      "state": "free",
      "name": "iPhone 15 Pro",
      "alias_name": "测试机A",
      "udid": "auto-udid-string",
      "type": "ios",
      "brand": "Apple",
      "model": "iPhone 15 Pro",
      "os_type": "iOS",
      "os_version": "17.0",
      "display": "1170x2532",
      "server_url": "http://192.168.1.100:8080",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:00:00Z"
    }
  ],
  "trace_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

**设备信息字段说明：**

| 字段 | 类型 | 说明 |
|------|------|------|
| id | integer | 设备 ID |
| user_id | integer | 用户 ID |
| serial | string | 设备序列号 |
| state | string | 设备状态（busy/free/offline） |
| name | string | 设备名称 |
| alias_name | string | 设备别名 |
| udid | string | 设备 UDID |
| type | string | 设备类型（ios/adb/hdc） |
| brand | string | 设备品牌 |
| model | string | 设备型号 |
| os_type | string | 操作系统类型 |
| os_version | string | 操作系统版本 |
| display | string | 屏幕分辨率（宽x高） |
| server_url | string | 代理服务 URL |
| created_at | string | 创建时间 |
| updated_at | string | 更新时间 |

---

## 设备控制

### Tap

模拟点击操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/tap/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "x": 100,
  "y": 200
}
```

**响应示例：**

```json
{}
```

---

### Double Tap

模拟双击操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/double_tap/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "x": 100,
  "y": 200
}
```

**响应示例：**

```json
{}
```

---

### Long Press

模拟长按操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/long_press/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "x": 100,
  "y": 200,
  "duration": 1000
}
```

**响应示例：**

```json
{}
```

---

### Swipe

模拟滑动操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/swipe/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "x1": 100,
  "y1": 200,
  "x2": 300,
  "y2": 400,
  "duration": 500
}
```

**响应示例：**

```json
{}
```

---

### Back

模拟按返回键操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/back/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{}
```

---

### Home

模拟按 Home 键操作。

```bash
curl -X POST "https://api.devicebase.cn/v1/home/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{}
```

---

### Launch App

启动指定应用。

```bash
curl -X POST "https://api.devicebase.cn/v1/launch_app/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "package": "com.example.app"
}
```

**响应示例：**

```json
{}
```

---

### Input Text

向设备输入文本。

```bash
curl -X POST "https://api.devicebase.cn/v1/input/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**请求体：**

```json
{
  "text": "Hello World"
}
```

**响应示例：**

```json
{}
```

---

### Clear Text

清除设备输入框中的文本。

```bash
curl -X POST "https://api.devicebase.cn/v1/clear_text/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{}
```

---

### Current App

获取当前前台应用信息。

```bash
curl -X POST "https://api.devicebase.cn/v1/current_app/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{
  "package": "com.example.app",
  "activity": "MainActivity"
}
```

---

### Dump Hierarchy

获取设备 UI 层级结构。

```bash
curl -X POST "https://api.devicebase.cn/v1/dump_hierarchy/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{
  "hierarchy": {
    "node": "android.widget.FrameLayout",
    "children": [
      {
        "node": "android.widget.LinearLayout",
        "children": []
      }
    ]
  }
}
```

---

### Device Info

获取设备详细信息。

```bash
curl -X POST "https://api.devicebase.cn/v1/deviceinfo/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{
  "serial": "ABC123456",
  "brand": "Apple",
  "model": "iPhone 15 Pro",
  "os_version": "17.0",
  "display": "1170x2532"
}
```

---

### Screen Get

获取设备屏幕截图（GET 方式）。

```bash
curl -X GET "https://api.devicebase.cn/v1/screen/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{
  "image": "base64_encoded_image_data"
}
```

---

### Screen Post

获取设备屏幕截图（POST 方式）。

```bash
curl -X POST "https://api.devicebase.cn/v1/screen/{serial}" \
  -H "Authorization: Bearer <your_api_key>"
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

```json
{
  "image": "base64_encoded_image_data"
}
```

---

### Screenshot Download

下载设备屏幕截图（文件下载形式）。

```bash
curl -X GET "https://api.devicebase.cn/v1/screenshot/{serial}" \
  -H "Authorization: Bearer <your_api_key>" \
  -o screenshot.png
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |

**响应示例：**

返回 PNG 格式的图片文件。

---

## 设备控制 - WebSocket

### 设备屏幕流 WebSocket

通过 WebSocket 协议获取设备实时屏幕画面流。

**连接方式：**

```
wss://{host}/v1/minicap/{serial}?token={api_key}
```

**示例：**

```javascript
// JavaScript WebSocket 连接示例
const ws = new WebSocket('wss://api.devicebase.cn/v1/minicap/ABC123456?token=<your_api_key>');

ws.onopen = () => {
  console.log('Connected to device screen stream');
};

ws.onmessage = (event) => {
  // 接收屏幕画面数据
  const imageData = event.data;
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = () => {
  console.log('Disconnected from device screen stream');
};
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |
| token | string | query | API Key |

**前置条件：**
- 设备必须存在且状态不为 `offline`

**错误码：**

| 错误码 | 说明 |
|--------|------|
| 4001 | JWT 认证失败 |
| 4003 | 设备不存在或离线 |
| 1011 | 代理连接异常 |

---

### 设备触摸控制 WebSocket

通过 WebSocket 协议发送触摸控制指令。

**连接方式：**

```
wss://{host}/v1/minitouch/{serial}?token={api_key}
```

**示例：**

```javascript
// JavaScript WebSocket 连接示例
const ws = new WebSocket('wss://api.devicebase.cn/v1/minitouch/ABC123456?token=<your_api_key>');

ws.onopen = () => {
  console.log('Connected to device touch control');

  // 发送点击指令
  ws.send(JSON.stringify({
    action: 'tap',
    x: 100,
    y: 200
  }));

  // 发送滑动指令
  ws.send(JSON.stringify({
    action: 'swipe',
    x1: 100,
    y1: 200,
    x2: 300,
    y2: 400,
    duration: 500
  }));
};

ws.onmessage = (event) => {
  // 接收响应
  const response = JSON.parse(event.data);
};

ws.onerror = (error) => {
  console.error('WebSocket error:', error);
};

ws.onclose = () => {
  console.log('Disconnected from device touch control');
};
```

**参数说明：**

| 参数 | 类型 | 位置 | 说明 |
|------|------|------|------|
| serial | string | path | 设备序列号 |
| token | string | query | API Key |

**支持的触摸操作：**

| 操作 | 参数 | 说明 |
|------|------|------|
| tap | x, y | 单击 |
| double_tap | x, y | 双击 |
| long_press | x, y, duration | 长按 |
| swipe | x1, y1, x2, y2, duration | 滑动 |

**前置条件：**
- 设备必须存在且状态不为 `offline`

**错误码：**

| 错误码 | 说明 |
|--------|------|
| 4001 | JWT 认证失败 |
| 4003 | 设备不存在或离线 |
| 1011 | 代理连接异常 |

---

## 错误响应

所有 API 错误响应遵循统一格式：

```json
{
  "code": 422,
  "message": "Validation Error",
  "detail": [
    {
      "loc": ["body", "username"],
      "msg": "field required",
      "type": "value_error.missing"
    }
  ]
}
```

**HTTP 状态码说明：**

| 状态码 | 说明 |
|--------|------|
| 200 | 请求成功 |
| 422 | 参数验证错误 |

---

## 通用响应格式

### ResponseModel[dict]

```json
{
  "code": 200,
  "message": "success",
  "data": {},
  "trace_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### ResponseModel[DeviceInfo]

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "user_id": 1,
    "serial": "ABC123456",
    "state": "free",
    "name": "iPhone 15 Pro",
    "alias_name": "测试机A",
    "udid": "auto-udid-string",
    "type": "ios",
    "brand": "Apple",
    "model": "iPhone 15 Pro",
    "os_type": "iOS",
    "os_version": "17.0",
    "display": "1170x2532",
    "server_url": "http://192.168.1.100:8080",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  },
  "trace_id": "550e8400-e29b-41d4-a716-446655440000"
}
```

### ResponseModel[LoginResponse]

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "bearer",
    "expires_in": 3600
  },
  "trace_id": "550e8400-e29b-41d4-a716-446655440000"
}
```