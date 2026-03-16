# 百居易 (Hostex) API 对接文档

> 官方文档：https://apidoc.myhostex.com (密码: realpartner)
> API 基础地址：`https://api.myhostex.com`

---

## 认证方式

采用 HTTP Basic 认证，需要 AppID 和 SecretKey（联系百居易客服获取）。

```
Authorization: Basic Base64(AppID:SecretKey)
```

**示例**：
```
AppID: myapp
SecretKey: mysecret
认证字串: Basic bXlhcHA6bXlzZWNyZXQ=
```

---

## Token 机制

```
Basic Auth
    ↓
refresh_token（长期有效）
    ↓
access_token（短期有效）
    ↓
调用业务 API
```

### 获取 refresh_token

```http
POST /refresh_token/refresh
Authorization: Basic {credentials}
```

**响应**：
```json
{
  "error_code": 0,
  "data": {
    "refresh_token": "W7NmYI1XcqLrcK4nqiGjIrfhfUofZCET1N5A1jh2",
    "refresh_token_expire": "2024-05-05 12:15:00"
  }
}
```

### 获取 access_token

```http
POST /access_token/refresh
Authorization: Basic {credentials}
Content-Type: application/json

{"refresh_token": "W7NmYI1XcqLrcK4nqiGjIrfhfUofZCET1N5A1jh2"}
```

**响应**：
```json
{
  "error_code": 0,
  "data": {
    "access_token": "89ed05bd6a2907706e80fdaa83c32d302325f30b",
    "expire_time": "2019-05-05 14:28:03"
  }
}
```

---

## 房东授权

需要房东用百居易账号密码授权，授权后获得 `operator_id`。

### 新增房东授权

```http
POST /manage_operator/join
Authorization: Basic {credentials}
Content-Type: application/json

{"account": "18811772881", "password": "xxx"}
```

**响应**：
```json
{
  "error_code": 0,
  "data": {
    "operator_id": 10035,
    "account": "18811772881"
  }
}
```

---

## 订单 API

> 请求头需要携带：
> - `Hostex-Operator-Id`: 房东 ID
> - `Hostex-Access-Token`: 访问令牌

### 检索订单

```http
GET /reservation/query?page=1&page_size=10&status=accepted
Hostex-Operator-Id: 10008
Hostex-Access-Token: {access_token}
```

**请求参数**：

| 参数 | 类型 | 必须 | 说明 |
|------|------|------|------|
| page | int | Y | 页码 |
| page_size | int | Y | 每页数量 |
| status | string | N | 订单状态 |
| begin | date | N | 离店日期起始 |
| end | date | N | 离店日期截止 |

**订单状态**：
- `wait_accept` - 待确认
- `wait_pay` - 待支付
- `accepted` - 已确认
- `cancelled` - 已取消
- `denied` - 已拒绝
- `timeout` - 已超时

**响应示例**：
```json
{
  "error_code": 0,
  "data": {
    "list": [
      {
        "code": "1-801252832144",
        "uniq_code": "1-801252832144-AJS82",
        "house_id": 2866,
        "check_in": "2018-10-10",
        "check_out": "2018-10-11",
        "status": "accepted",
        "staying_status": "wait_stay",
        "lock_password": "123456",
        "guests": [
          {
            "name": "小黑",
            "full_name": "小黑 韩",
            "phone": "+86 186 2961 6237",
            "email": "user@guest.airbnb.com"
          }
        ]
      }
    ]
  }
}
```

### 订单详情

```http
GET /reservation/detail?reservation_code=1-801252832144
Hostex-Operator-Id: 10008
Hostex-Access-Token: {access_token}
```

### 关键字段说明

**reservation**:

| 字段 | 说明 |
|------|------|
| code | 订单编码（已废弃） |
| uniq_code | 订单唯一编码 |
| house_id | 房间 ID |
| check_in | 入住日期 |
| check_out | 离店日期 |
| status | 订单状态 |
| staying_status | 入住状态：wait_stay / staying / checkout |
| lock_password | 门锁密码 |
| guests | 房客信息数组 |

**guests[]**:

| 字段 | 说明 |
|------|------|
| name | 名字 |
| full_name | 全名 |
| phone | 手机号（格式：`+86 186 2961 6237`）|
| email | 邮箱 |

---

## 事件回调 (Webhook)

### 注册回调地址

```http
POST /callback/register
Hostex-Access-Token: {access_token}
Content-Type: application/json

{
  "notify_url": "https://your-server.com/api/webhook/hostex",
  "headers": [
    {"name": "Authorization", "value": "Bearer xxx"}
  ]
}
```

### 回调消息格式

**订单变化 (biz_type: 2)**：
```json
{
  "biz_type": 2,
  "biz_id": "0-XJHSFNAS",
  "operator_id": 10008,
  "send_time": "2019-08-21 18:58:57",
  "content": {
    "reservation_uniq_code": "0-XJHSFNAS-ADJ91JS",
    "thirdparty_type": 0,
    "origin_code": "XJHSFNAS",
    "attributes": ["house_id", "status"]
  }
}
```

**回调响应格式**：
```json
{
  "status": 0,
  "message": "success"
}
```

- 失败重试次数：3 次
- 请求超时时间：3 秒

### 业务类型 (biz_type)

| 值 | 说明 |
|----|------|
| 1 | 新消息 |
| 2 | 订单变化 |
| 3 | 房态价格变化 |
| 4 | 同步任务状态 |
| 5 | 基础价格变化 |

---

## 对接方案

### 我们需要做的

1. **获取凭证**：联系百居易客服获取 AppID 和 SecretKey
2. **房东授权**：用房东百居易账号调用 `/manage_operator/join`
3. **注册 Webhook**：接收订单变更推送

### 订单查询流程

```
用户输入手机号
    ↓
调用 /reservation/query 获取订单列表
    ↓
匹配 guests[].phone（需格式化：+86 186 → 186）
    ↓
返回匹配的订单
```

### 手机号格式处理

API 返回格式：`+86 186 2961 6237`
用户输入格式：`18629616237`

```typescript
function normalizePhone(phone: string): string {
  return phone.replace(/[\s\+\-]/g, '').replace(/^86/, '')
}
```

---

## 环境变量

```bash
# 百居易 API
HOSTEX_APP_ID=xxx
HOSTEX_SECRET_KEY=xxx
HOSTEX_OPERATOR_ID=xxx          # 房东 ID（授权后获得）
```

---

## 相关文件

| 文件 | 说明 |
|------|------|
| `services/hostex.service.ts` | 百居易 API 调用（待创建）|
| `routes/webhook.ts` | Webhook 路由 |
| `controllers/webhook.controller.ts` | Webhook 处理（待创建）|
