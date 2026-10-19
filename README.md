
# 事件合约 API 规范

提供服务器信息、Token 状态、实时预测、近期推单和订单历史查询，不提供交易提交、订单占用或成交上报接口。

[接入说明](#接入说明) · [接口目录](#接口目录) · [数据模型](#数据模型) · [错误码](#错误码)

## 接入说明

### 请求地址

```text
{base_url}/api/v1/{endpoint}
```

`base_url` 由服务提供方提供，例如 `https://api.example.com`。HTTP 与 HTTPS 使用相同的接口路径、请求头和 JSON 格式。

### 公共请求头

| 名称 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `X-Client-Version` | string | 是 | 调用方软件版本，使用 SemVer 格式，例如 `1.0.0`。服务端可限制支持的版本范围 |
| `Authorization` | string | 除 `/server` 外必填 | 格式为 `Bearer YOUR_TOKEN`，替换 `YOUR_TOKEN` 为服务端签发的 Token |

Token 只通过请求头传递，不放在 URL 中。对于有效路径的 GET 请求，服务端依次校验版本、Token 和查询参数。

### 请求与响应约定

| 项目 | 约定 |
| --- | --- |
| 请求方式 | 所有接口使用 `GET`，无请求体 |
| 查询参数 | 仅支持接口声明的参数；未知、重复、空值或格式错误返回 `400 INVALID_PARAMS` |
| 整数查询参数 | 使用不含前导零的正十进制整数，例如 `15`；`01`、`+15`、`15.0` 均无效 |
| 成功响应 | HTTP `200`，JSON 根对象直接包含业务字段，无 `data` 包装 |
| 失败响应 | 非 2xx 状态，返回 `code`、`message`，格式见[错误码](#错误码) |
| 响应编码 | `Content-Type: application/json; charset=utf-8` |
| 缓存 | `Cache-Control: no-store` |
| 时间 | Unix 时间戳，单位毫秒；除明确说明外均为正整数 |
| 价格 | 大于 0 的普通十进制字符串，例如 `"2550.25"`；不使用科学计数法 |
| 空值 | `null` 表示未知或不适用；可选字段可以省略，详见各字段定义 |

## 接口目录

| 接口 | 方法与路径 | Token | 用途 |
| --- | --- | --- | --- |
| [查询服务器信息](#查询服务器信息) | `GET /api/v1/server` | 不需要 | 获取服务器名称和版本 |
| [查询 Token 状态](#查询-token-状态) | `GET /api/v1/token` | 需要 | 获取绑定账号、到期时间和状态 |
| [查询实时预测](#查询实时预测) | `GET /api/v1/predict` | 需要有效 Token | 获取预测快照及当前价格 |
| [获取推单](#获取推单) | `GET /api/v1/push-orders` | 需要有效 Token | 获取指定时间范围内的待验证订单 |
| [查询订单历史](#查询订单历史) | `GET /api/v1/orders` | 需要有效 Token | 按验证状态分页查询完整历史 |

## 接口说明

### 查询服务器信息

`GET /api/v1/server`

返回服务器名称及软件版本。需要 `X-Client-Version`，无需 Token。

**请求参数**

无查询参数。

**请求示例**

```bash
curl 'https://api.example.com/api/v1/server' \
  -H 'X-Client-Version: 1.0.0'
```

**响应字段**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `name` | string | 非空服务器名称 |
| `version` | string | 非空服务器软件版本，独立于 URL 中的 API 版本 |

**响应示例**

```json
{
  "name": "Event Contract Server",
  "version": "1.0.0"
}
```

### 查询 Token 状态

`GET /api/v1/token`

返回请求头中 Token 的绑定账号、到期时间和状态。已识别的过期或停用 Token 也可查询自身状态。

**请求参数**

无查询参数。`Authorization` 中传入要查询的 Token。

**请求示例**

```bash
curl 'https://api.example.com/api/v1/token' \
  -H 'X-Client-Version: 1.0.0' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

**响应字段**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `account_id` | string | Token 绑定的账号 UID，非空；所属平台由下单客户端决定，详见下文 |
| `expires_at_ms` | integer 或 null | 到期时间，单位毫秒；不限期为 null |
| `status` | string | `ACTIVE` 有效、`EXPIRED` 过期、`DISABLED` 停用 |

**账号 UID 的含义**

`account_id` 是通用账号 UID 字段，API 不将它固定解释为某个交易平台的 UID。下单客户端应按自身对接的平台使用该值：

| 下单客户端 | `account_id` 的含义 |
| --- | --- |
| HIBT 下单程序（ Open Event Desktop） | HIBT 账号 UID |
| 币安下单程序（ Open-Hsaord ） | 币安账号 UID |

服务方签发 Token 时，应与客户端约定 UID 所属的账号体系。客户端不应根据 UID 的数字格式猜测平台，也不应将不同平台的 UID 混用。接口字段名仍为 `account_id`，无需增加平台前缀或修改请求格式。

**响应示例**

```json
{
  "account_id": "12345678",
  "expires_at_ms": 1791820800000,
  "status": "ACTIVE"
}
```

**接口规则**

- 不存在或认证格式无效的 Token 返回 `401 TOKEN_INVALID`。
- 已识别的 Token 返回 HTTP `200`，包括过期、停用状态；原到期时间保持不变。
- `expires_at_ms` 小于或等于当前服务器时间时视为过期；`EXPIRED` 状态的到期时间不能为 null。同时过期和停用时，状态为 `DISABLED`。
- 其它需要 Token 的查询接口只接受 `ACTIVE` 且未到期的 Token。

### 查询实时预测

`GET /api/v1/predict`

返回当前预测快照和对应标的的现价。需要有效 Token。

**请求参数**

无查询参数。

**请求示例**

```bash
curl 'https://api.example.com/api/v1/predict' \
  -H 'X-Client-Version: 1.0.0' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

**响应字段**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `server_time_ms` | integer | 本次响应的服务器时间 |
| `predict` | object | [预测对象](#预测对象)，不能为空 |

**响应示例**

```json
{
  "server_time_ms": 1791766801000,
  "predict": {
    "symbol": "ETHUSDT",
    "duration_sec": 1800,
    "direction": "DOWN",
    "confidence": 0.72,
    "price": "2550.25",
    "price_time_ms": 1791766800900
  }
}
```

**接口规则**

- 暂无方向预测时，仍返回对象及有效现价，`direction` 和 `confidence` 同时为 null。
- 无法提供完整预测对象或有效现价时返回 `500 SERVER_ERROR`，不以 `predict=null` 或无效价格代替。
- 预测与订单是独立数据，预测更新不创建订单记录。

### 获取推单

`GET /api/v1/push-orders`

返回指定时间范围内尚未验证的全部订单。需要有效 Token，不分页。

**请求参数**

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `max_age_sec` | integer | 否 | `15` | 允许的最大订单年龄，单位秒，范围 `1–300` |

**请求示例**

```bash
curl 'https://api.example.com/api/v1/push-orders?max_age_sec=15' \
  -H 'X-Client-Version: 1.0.0' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

**响应字段**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `server_time_ms` | integer | 本次响应的服务器时间，也是时间范围筛选的基准 |
| `orders` | array<object> | [推单对象](#推单对象)列表；服务端仅筛选尚未验证的订单，无匹配记录时为 `[]` |

**响应示例**

```json
{
  "server_time_ms": 1791766801000,
  "orders": [
    {
      "id": "ord-001",
      "symbol": "ETHUSDT",
      "duration_sec": 1800,
      "direction": "DOWN",
      "time_ms": 1791766800000,
      "payout_rate": 0.85
    }
  ]
}
```

**接口规则**

订单必须同时满足以下条件：

```text
result = PENDING
0 <= server_time_ms - time_ms < max_age_sec × 1000
```

- 使用同一快照的服务器时间与订单数据筛选，排除未来、达到或超过最大年龄、已验证的订单。
- 按 `time_ms` 降序、同时间 `id` 降序返回全部匹配记录 (方便客户端优先执行最近订单)，无 `page`、`page_size`、`total` 字段，不截断数量。
- 无匹配记录时仍返回 HTTP `200`、`server_time_ms` 和空数组。
- GET 不消费订单，连续查询可能返回同一条订单；身份与去重标识见[订单标识与状态](#订单标识与状态)。

### 查询订单历史

`GET /api/v1/orders`

按验证状态分页查询订单历史。需要有效 Token，不限制订单年龄。

**请求参数**

| 名称 | 类型 | 必填 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `filter` | string | 否 | `all` | `all` 全部、`pending` 待验证、`correct` 正确、`wrong` 错误；区分大小写 |
| `page` | integer | 否 | `1` | 页码，范围 `1–9999999999`，最多 10 位 |
| `page_size` | integer | 否 | `20` | 每页条数，范围 `1–100` |

**请求示例**

```bash
curl 'https://api.example.com/api/v1/orders?filter=all&page=1&page_size=20' \
  -H 'X-Client-Version: 1.0.0' \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

**响应字段**

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `server_time_ms` | integer | 本次响应的服务器时间 |
| `orders` | array<object> | 本页[历史订单对象](#历史订单对象)列表 |
| `page` | integer | 本次请求的页码 |
| `page_size` | integer | 本次请求的每页条数 |
| `total` | integer | 筛选后的总记录数，包含其它页，非负整数 |

**响应示例**

```json
{
  "server_time_ms": 1791766801000,
  "orders": [
    {
      "id": "ord-001",
      "symbol": "ETHUSDT",
      "duration_sec": 1800,
      "direction": "DOWN",
      "confidence": 0.72,
      "time_ms": 1791766800000,
      "result": "PENDING",
      "payout_rate": 0.85,
      "open_price": "2550.25",
      "close_price": null
    }
  ],
  "page": 1,
  "page_size": 20,
  "total": 1
}
```

**接口规则**

- 先按 `filter` 筛选，再按 `time_ms` 降序、同时间 `id` 降序分页。
- 本次响应的记录与 `total` 来自同一数据快照；不同分页请求之间不保证快照一致。
- 无匹配记录或页码超出末页时返回 HTTP `200`、空数组及准确的分页字段。
- `filter=pending` 包含较早的未验证订单。需要限制订单年龄时使用[获取推单](#获取推单)。

## 数据模型

### 公共字段取值

字段名称及枚举值区分大小写。

| 字段 | 类型 | 允许值 |
| --- | --- | --- |
| `symbol` | string | `BTCUSDT`、`ETHUSDT` |
| `duration_sec` | integer | `180`（3 分钟）、`300`（5 分钟）、`600`（10 分钟）、`900`（15 分钟）、`1800`（30 分钟）、`3600`（60 分钟）、`86400`（1 天） |
| `direction` | string | `UP`（看涨）、`DOWN`（看跌） |
| `confidence` | number | `0–1`，表示所选方向的置信度 |
| `result` | string | `PENDING`（待验证）、`CORRECT`（正确）、`WRONG`（错误） |

`duration_sec` 表示合约时长，与请求频率、订单年龄参数 `max_age_sec` 无关。

接口支持的周期不代表所有交易平台都提供该周期。若对应平台没有该周期值，可能导致下单失败；客户端应先确认平台支持对应周期，不应自动替换成其他周期下单。

### 预测对象

用于 `/predict` 响应中的 `predict` 字段。

| 字段 | 类型 | 是否必返 | 说明 |
| --- | --- | --- | --- |
| `symbol` | string | 是 | 当前标的，见公共字段取值 |
| `duration_sec` | integer | 是 | 合约时长，单位秒，见公共字段取值 |
| `direction` | string 或 null | 是 | 预测方向；暂无方向预测时为 null |
| `confidence` | number 或 null | 是 | 方向置信度；必须与 `direction` 同时为 null 或同时非 null |
| `price` | string | 是 | 当前标的有效现价，不能为 null、空字符串或 `"0"` |
| `price_time_ms` | integer | 是 | 行情更新时间，`0 < 值 <= server_time_ms` |

### 推单对象

用于 `/push-orders` 响应中的 `orders[]`，仅包含以下字段。

| 字段 | 类型 | 是否必返 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 是 | 非空订单标识，与原始 `time_ms` 组合识别订单 |
| `symbol` | string | 是 | 订单标的，见公共字段取值 |
| `duration_sec` | integer | 是 | 合约时长，单位秒，见公共字段取值 |
| `direction` | string | 是 | 订单方向，见公共字段取值 |
| `time_ms` | integer | 是 | 原始创建时间，创建后不变；用于订单年龄判断与身份识别 |
| `payout_rate` | number | 否 | 支付率比例，范围 `0 < 值 <= 1`；省略时下单软件按下文约定处理 |

### 历史订单对象

用于 `/orders` 响应中的 `orders[]`，保留完整决策记录及验证信息。

| 字段 | 类型 | 是否必返 | 说明 |
| --- | --- | --- | --- |
| `id` | string | 是 | 非空订单标识，与原始 `time_ms` 组合识别订单 |
| `symbol` | string | 是 | 订单标的，见公共字段取值 |
| `duration_sec` | integer | 是 | 合约时长，单位秒，见公共字段取值 |
| `direction` | string | 是 | 订单方向，见公共字段取值 |
| `confidence` | number | 是 | 决策时的方向置信度，范围 `0–1` |
| `time_ms` | integer | 是 | 原始创建时间，创建后不变 |
| `result` | string | 是 | 服务端验证的方向结果，见公共字段取值 |
| `payout_rate` | number | 否 | 支付率比例，范围 `0 < 值 <= 1`；`0.85` 表示 85%，省略时下单软件按下文约定处理 |
| `open_price` | string 或 null | 是 | 创建决策时的开仓参考价，未知为 null |
| `close_price` | string 或 null | 是 | 合约周期结束时的平仓参考价，尚未取得为 null |

推单对象与历史订单对象的 `payout_rate` 使用相同约定。存在时必须为有效数值；null、字符串、百分数 `85` 及范围外值均无效。已有周期采用下列默认值：

| `duration_sec` | 周期 | 默认支付率 |
| --- | --- | --- |
| `180` | 3 分钟 | `0.80` |
| `300` | 5 分钟 | `0.80` |
| `600` | 10 分钟 | `0.80` |
| `900` | 15 分钟 | `0.80` |
| `1800` | 30 分钟 | `0.85` |
| `3600` | 1 小时 | `0.85` |
| `86400` | 1 天 | `0.85` |

若客户端能取得平台实际订单支付率时，将实际值为准；
订单周期结束时间为 `time_ms + duration_sec × 1000`。开平仓参考价是订单快照，不随实时行情变化；补写参考价或验证结果不改变订单标识和创建时间，其余决策字段创建后保持不变。

### 订单标识与状态

- **数据范围**：预测、推单及订单历史为全服务器共享数据，不按 Token 绑定账号筛选。
- **订单标识**：同一服务器内，客户端将由 `id + 原始 time_ms` 组合识别订单；ID 本身不要求唯一，相同 ID、不同时间戳可表示不同订单。同一响应内相同组合不得重复。
- **查询语义**：GET 不消费、占用或删除订单，不修改创建时间，不确认实际成交，也不改变验证结果。
- **状态语义**：`PENDING` 表示方向尚未验证，与是否实际成交无关。超过推单时间窗口只影响是否被该查询返回，不会自动将结果改为 `WRONG`。

## 错误码

### 错误响应

失败响应使用非 2xx 状态，JSON 根对象固定为以下结构：

```json
{
  "code": "TOKEN_INVALID",
  "message": "Token 缺失或无效"
}
```

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | string | 错误码，用于程序判断 |
| `message` | string | 可读说明，具体文案可能随错误原因变化 |

所有 HTTP `401` 响应包含 `WWW-Authenticate: Bearer`。连接失败或超时且未收到 HTTP 响应时，没有服务端 JSON 错误体。

### 错误码列表

| HTTP 状态 | 错误码 | 说明 | 适用接口 |
| --- | --- | --- | --- |
| `400` | `INVALID_PARAMS` | 版本头缺失、格式无效，或查询参数未知、重复、为空、格式或范围错误 | 全部接口，按各接口参数约束判断 |
| `401` | `TOKEN_INVALID` | Token 缺失、不存在或认证格式无效 | 除 `/server` 外的接口 |
| `401` | `TOKEN_EXPIRED` | Token 已过期 | `/predict`、`/push-orders`、`/orders` |
| `403` | `TOKEN_DISABLED` | Token 已停用；同时过期时优先返回此错误 | `/predict`、`/push-orders`、`/orders` |
| `404` | `NOT_FOUND` | 请求路径不存在 | 未定义的 API 路径 |
| `405` | `METHOD_NOT_ALLOWED` | 已定义的公开接口使用了 GET 以外的方法 | 全部接口 |
| `409` | `CLIENT_VERSION_UNSUPPORTED` | 调用方软件版本不在服务端支持范围内 | 全部接口 |
| `500` | `SERVER_ERROR` | 服务端内部错误；`/predict` 也可能因预测或现价不可用返回此错误 | 全部接口 |
