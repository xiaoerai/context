# 登录认证

> 状态：✅ 基本完成（短信验证码待接入真实服务）

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住流程（含登录） | ✅ 完成 | `hooks/checkinFlow/usePhoneStep.ts` |
| 状态管理 | ✅ 完成 | `stores/useAppStore.ts` |
| 登录 API | ✅ 完成 | `api/auth.ts`（支付宝用 `my.getAuthCode`） |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 短信路由 | ✅ 完成 | `routes/sms.ts` |
| 认证路由 | ✅ 完成 | `routes/auth.ts` |
| 认证服务 | ✅ 完成 | `services/auth.service.ts`（多端 platform 路由已实现） |
| 支付宝 OAuth | ✅ 完成 | `services/alipay.service.ts`（`getAlipayUserId` 已实现） |
| 短信服务 | ✅ 完成 | `services/sms.service.ts`（阿里云短信未接入，开发阶段 mock）|
| 数据库 | ✅ 完成 | `db/users.ts`（phone 为唯一标识，存 phone + guestIds） |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/sms/send` | ✅ 已联调 |
| `POST /api/auth/login` | ✅ 已联调（支持 `platform` + `code` 参数） |
| `GET /api/orders` | ✅ 已联调 |

---

## 概述

用户通过「订单手机号 + 短信验证码」验证身份，同时获取平台授权（支付宝 user_id / 微信 openid），用于后续支付。

### 多端设计

`phone` 作为跨端统一标识，各平台 ID 作为附属字段：

| 平台 | 授权方式 | 获取的用户标识 | 存储字段 |
|------|----------|---------------|----------|
| 支付宝小程序 | `my.getAuthCode({ scopes: 'auth_base' })` | user_id | `alipayUserId` |
| 微信小程序 | `wx.login()` | openid | `openid` |
| H5 | 无 | 无 | 无 |

同一手机号在不同平台登录，关联到同一个用户。

---

## 认证流程

### 支付宝小程序（当前重点）

```
小程序                      阿里云短信              支付宝              后端
  │                           │                    │                  │
  │── 1. my.getAuthCode ──────────────────────────▶│                  │
  │◀── 2. 返回 authCode ─────────────────────────│                  │
  │                           │                    │                  │
  │   [用户点击办理入住，输入手机号]                  │                  │
  │                           │                    │                  │
  │── 3. POST /api/sms/send { phone } ────────────────────────────────▶│
  │                           │◀── 4. 发送验证码 ──────────────────────│
  │◀── 5. 返回 { success } ───────────────────────────────────────────│
  │                           │                    │                  │
  │   [用户输入验证码]          │                    │                  │
  │                           │                    │                  │
  │── 6. POST /api/auth/login ────────────────────────────────────────▶│
  │      { authCode, phone, smsCode, platform: 'alipay' }             │
  │                           │                    │                  │
  │                           │        7. alipay.system.oauth.token ──▶│
  │                           │           authCode → user_id           │
  │                           │                    │                  │
  │                           │      8. 验证 smsCode + 保存用户        │
  │                           │         存 alipayUserId                │
  │                           │                    │                  │
  │◀── 9. 返回 { token } ────────────────────────────────────────────│
  │        (JWT 含 alipayUserId，支付时使用)                           │
  │                           │                    │                  │
  │── 10. GET /api/orders?phone=xxx ─────────────────────────────────▶│
  │◀── 11. 返回订单列表 ─────────────────────────────────────────────│
```

---

## API 接口

### 发送验证码

```
POST /api/sms/send
```

**请求**：
```json
{
  "phone": "13800138000"
}
```

**响应**：
```json
{
  "success": true
}
```

**验证规则**：
- 手机号：11位数字
- 频率限制：60秒内只能发送一次

---

### 登录

```
POST /api/auth/login
```

**请求**：
```json
{
  "code": "平台授权码（authCode）",
  "phone": "13800138000",
  "smsCode": "123456",
  "platform": "alipay"
}
```

`platform` 取值：`alipay` | `wechat` | `h5`

**后端根据 platform 走不同逻辑**：
- `alipay` → `alipay.system.oauth.token` 换 `user_id`
- `wechat` → `jscode2session` 换 `openid`
- `h5` → 不需要平台授权

> JWT 中包含 phone + 各平台 ID（alipayUserId 等），平台 ID 不存数据库，仅在 JWT 中携带，支付时使用。

**响应**：
```json
{
  "success": true,
  "data": {
    "token": "JWT token"
  }
}
```

---

## 数据表

### users 集合

```typescript
{
  _id: string,
  phone: string,              // 手机号（跨端唯一标识，索引）
  guestIds?: string[],        // 关联的住客ID列表
  createdAt: Date,
  lastLoginAt: Date
}
```

### sms_codes 集合

```typescript
{
  _id: string,
  phone: string,         // 手机号（唯一索引）
  code: string,          // 6位验证码
  expireAt: Date,        // 过期时间（5分钟后）
  createdAt: Date
}
```

---

## 待办事项

1. 接入阿里云短信真实服务（替换 mock 验证码）
2. 微信小程序端 `wx.login()` 联调验证

---

## 测试数据

开发阶段：
- 测试手机号 `15290500792` 可跳过验证码，但正常走平台授权（拿真实 alipayUserId）
- 其他手机号需真实验证码（阿里云短信未接入前查数据库或后端日志）

```
手机号：15290500792
验证码：任意（mock 模式）
```
