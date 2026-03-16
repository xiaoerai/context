# 登录认证

> 状态：✅ 完成

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 登录页 | ✅ 完成 | `pages/login/index.tsx` |
| 办理入住弹窗 | ✅ 完成 | `pages/index/components/CheckInModal/index.tsx` |
| 状态管理 | ✅ 完成 | `stores/useAppStore.ts` |
| 认证 Hook | ✅ 完成 | `hooks/useAuth.ts` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 短信路由 | ✅ 完成 | `routes/sms.ts` |
| 认证路由 | ✅ 完成 | `routes/auth.ts` |
| 短信控制器 | ✅ 完成 | `controllers/sms.controller.ts` |
| 认证控制器 | ✅ 完成 | `controllers/auth.controller.ts` |
| 短信服务 | ✅ 完成 | `services/sms.service.ts` |
| 认证服务 | ✅ 完成 | `services/auth.service.ts` |
| 阿里云短信 SDK | ⏳ 模拟中 | `services/sms.ts` |
| 微信 API | ✅ 完成 | `services/wechat.ts` |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/sms/send` | ✅ 完成 |
| `POST /api/auth/login` | ✅ 完成 |

---

## 概述

用户通过「订单手机号 + 短信验证码」验证身份，无需微信授权登录。

### 为什么用短信验证码？

| 原因 | 说明 |
|------|------|
| 安全 | 防止他人偷看订单信息 |
| 灵活 | 用户可能用别人手机号订房（帮家人订、公司号等） |
| 简单 | 无需微信认证费用（短信费约 0.045元/条） |

---

## 认证流程

```
小程序                      阿里云短信              微信              后端
  │                           │                    │                  │
  │── 1. 打开小程序，wx.login ─────────────────────▶│                  │
  │◀── 2. 返回 code ──────────────────────────────│                  │
  │                           │                    │                  │
  │   [用户点击办理入住，输入手机号]                  │                  │
  │                           │                    │                  │
  │── 3. POST /api/sms/send { phone } ────────────────────────────────▶│
  │                           │◀── 4. 发送验证码 ──────────────────────│
  │◀── 5. 返回 { success } ───────────────────────────────────────────│
  │                           │                    │                  │
  │   [用户输入验证码]          │                    │                  │
  │                           │                    │                  │
  │── 6. POST /api/auth/login { code, phone, smsCode } ───────────────▶│
  │                           │                    │◀─ 7. code→openid─│
  │                           │                    │                  │
  │                           │      8. 验证 smsCode + 保存用户        │
  │                           │                    │                  │
  │◀── 9. 返回 { token, orders } ─────────────────────────────────────│
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
  "code": "wx.login返回的code",
  "phone": "13800138000",
  "smsCode": "123456"
}
```

**响应**：
```json
{
  "token": "JWT token",
  "orders": [
    {
      "orderId": "ORD20260309001",
      "roomNumber": "301",
      "checkInDate": "2026-03-09",
      "checkOutDate": "2026-03-11"
    }
  ]
}
```

---

## 数据表

### users 集合

```typescript
{
  _id: string,
  openid: string,        // 微信openid（唯一索引）
  phone: string,         // 手机号（索引）
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

## 前端实现

### 相关文件

| 文件 | 说明 |
|------|------|
| `pages/login/index.tsx` | 登录页 |
| `pages/index/components/CheckInModal/index.tsx` | 办理入住弹窗（包含验证） |
| `stores/useAppStore.ts` | 存储用户手机号 |
| `hooks/useAuth.ts` | 认证相关 hook |

### 状态管理

```typescript
// stores/useAppStore.ts
interface AppState {
  phone: string | null
  setPhone: (phone: string) => void
}
```

---

## 后端实现

### 相关文件

| 文件 | 说明 |
|------|------|
| `routes/sms.ts` | 短信路由 |
| `routes/auth.ts` | 认证路由 |
| `controllers/sms.controller.ts` | 短信控制器 |
| `controllers/auth.controller.ts` | 认证控制器 |
| `services/sms.service.ts` | 短信业务逻辑 |
| `services/auth.service.ts` | 认证业务逻辑 |
| `services/sms.ts` | 阿里云短信 SDK |
| `services/wechat.ts` | 微信 API |

---

## 测试数据

```
手机号：13800138000
验证码：1234
```
