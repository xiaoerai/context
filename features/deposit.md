# 押金支付

> 状态：⬜ 待开发

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 押金支付页 | ⬜ 待开发 | `pages/deposit/index.tsx` |
| 入住成功页 | ⬜ 待开发 | `pages/success/index.tsx` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 支付路由 | ⬜ 待开发 | `routes/deposit.ts` |
| 支付控制器 | ⬜ 待开发 | `controllers/deposit.controller.ts` |
| 支付服务 | ⬜ 待开发 | `services/deposit.service.ts` |
| 微信支付 SDK | ⬜ 待开发 | `services/wechat-pay.ts` |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/deposit/create` | ⬜ 待开发 |
| `POST /api/deposit/notify` | ⬜ 待开发 |
| `GET /api/orders/:id/room-info` | ⬜ 待开发 |

### 数据表

| 集合 | 状态 |
|------|------|
| rooms | ⬜ 待创建 |
| orders (支付字段) | ⬜ 待扩展 |

---

## 概述

用户提交入住信息后，需支付押金才能获取门锁密码和WiFi信息。

---

## 用户流程

```
提交入住信息
      ↓
┌─────────────────────────────────┐
│         押金支付页               │
│  ┌─────────────────────────┐    │
│  │ 房间：301室              │    │
│  │ 入住：2026-03-09        │    │
│  │ 退房：2026-03-11        │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │ 押金金额                 │    │
│  │      ¥500.00            │    │
│  └─────────────────────────┘    │
│                                 │
│       [微信支付]                │
└─────────────────────────────────┘
      ↓ 支付成功
跳转入住成功页（获取门锁密码）
```

---

## 页面设计

### 押金支付页 (deposit)

**路径**：`pages/deposit`

**展示内容**：
| 内容 | 说明 |
|------|------|
| 房间信息 | 房间号 |
| 入住日期 | YYYY-MM-DD |
| 退房日期 | YYYY-MM-DD |
| 押金金额 | 金额（元） |

**操作**：
- 点击「微信支付」调起微信支付

---

### 入住成功页 (success)

**路径**：`pages/success`

**展示内容**：
| 内容 | 说明 |
|------|------|
| 房间号 | 大字醒目 |
| 门锁密码 | 大字醒目，可复制 |
| WiFi 名称 | 可复制 |
| WiFi 密码 | 可复制 |
| 入住时间 | 入住日期 |
| 退房时间 | 退房日期 |

**操作**：
- 联系房东
- 申请退房

---

## 支付流程

```
小程序                         后端                        微信支付
  │                             │                            │
  │── 1. POST /api/deposit/create ──▶│                        │
  │      { orderId }            │                            │
  │                             │──── 2. 统一下单 ───────────▶│
  │                             │◀─── 3. 返回支付参数 ────────│
  │◀── 4. 返回支付参数 ──────────│                            │
  │                             │                            │
  │── 5. wx.requestPayment ─────────────────────────────────▶│
  │◀── 6. 支付结果 ─────────────────────────────────────────│
  │                             │                            │
  │                             │◀─── 7. 支付回调 ───────────│
  │                             │      POST /api/deposit/notify
  │                             │                            │
  │── 8. 查询支付状态 ──────────▶│                            │
  │◀── 9. 返回门锁密码、WiFi ───│                            │
```

---

## API 接口

### 创建支付

```
POST /api/deposit/create
```

**请求**：
```json
{
  "orderId": "ORD20260309001"
}
```

**响应**：
```json
{
  "timeStamp": "1234567890",
  "nonceStr": "xxx",
  "package": "prepay_id=xxx",
  "signType": "RSA",
  "paySign": "xxx"
}
```

---

### 支付回调

```
POST /api/deposit/notify
```

> 微信支付服务器回调，更新订单押金状态

---

### 获取入住信息

```
GET /api/orders/:id/room-info
```

**响应**：
```json
{
  "roomNumber": "301",
  "doorCode": "123456",
  "wifiName": "Hotel-301",
  "wifiPassword": "welcome123",
  "checkInDate": "2026-03-09",
  "checkOutDate": "2026-03-11"
}
```

---

## 数据模型

### 入住结果

```typescript
interface CheckInResult {
  roomNumber: string      // 房间号
  doorCode: string        // 门锁密码
  wifiName: string        // WiFi 名称
  wifiPassword: string    // WiFi 密码
  checkInDate: string     // 入住日期
  checkOutDate: string    // 退房日期
}
```

---

## 数据表

### orders 集合（更新字段）

```typescript
{
  // ... 其他字段
  depositStatus: string,  // unpaid → paid → refunded
  depositAmount: number,  // 押金金额（分）
  depositPaidAt: Date,    // 支付时间
  transactionId: string,  // 微信支付订单号
}
```

### rooms 集合

```typescript
{
  _id: string,
  hotelId: string,        // 民宿ID（索引）
  roomNumber: string,     // 房间号
  doorCode: string,       // 门锁密码
  wifiName: string,       // WiFi 名称
  wifiPassword: string,   // WiFi 密码
  deposit: number         // 押金金额（分）
}
```

---

## 前端实现（待开发）

### 文件规划

| 文件 | 说明 |
|------|------|
| `pages/deposit/index.tsx` | 押金支付页 |
| `pages/success/index.tsx` | 入住成功页 |

---

## 后端实现（待开发）

### 文件规划

| 文件 | 说明 |
|------|------|
| `routes/deposit.ts` | 支付路由 |
| `controllers/deposit.controller.ts` | 支付控制器 |
| `services/deposit.service.ts` | 支付业务逻辑 |
| `services/wechat-pay.ts` | 微信支付 SDK |

---

## 环境变量

```bash
# 微信支付
WECHAT_MCH_ID=xxx          # 商户号
WECHAT_API_KEY=xxx         # API密钥
WECHAT_CERT_PATH=xxx       # 证书路径（退款用）
```
