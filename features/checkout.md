# 退房申请

> 状态：⬜ 待开发

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 退房申请页 | ⬜ 待开发 | `pages/checkout/index.tsx` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 退房路由 | ⬜ 待开发 | `routes/checkout.ts` |
| 退房控制器 | ⬜ 待开发 | `controllers/checkout.controller.ts` |
| 退房服务 | ⬜ 待开发 | `services/checkout.service.ts` |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/checkout` | ⬜ 待开发 |
| `GET /api/orders/:id/checkout-status` | ⬜ 待开发 |
| `POST /api/admin/checkout/confirm` | ⬜ 待开发 |
| `POST /api/deposit/refund` | ⬜ 待开发 |

### 数据表

| 集合 | 状态 |
|------|------|
| orders (退房/退款字段) | ⬜ 待扩展 |

---

## 概述

用户入住后可申请退房，等待房东确认后退还押金。

---

## 用户流程

```
入住成功页 / 首页「当前入住」
      ↓ 点击「申请退房」
┌─────────────────────────────────┐
│         退房申请页               │
│  ┌─────────────────────────┐    │
│  │ 房间：301室              │    │
│  │ 入住：2026-03-09        │    │
│  │ 退房：2026-03-11        │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │ 押金：¥500.00           │    │
│  │ 状态：待检查             │    │
│  └─────────────────────────┘    │
│                                 │
│       [确认申请退房]             │
└─────────────────────────────────┘
      ↓ 提交申请
┌─────────────────────────────────┐
│         等待确认                 │
│                                 │
│  房东正在检查房间...             │
│                                 │
│  押金将在确认后原路退回           │
└─────────────────────────────────┘
      ↓ 房东确认
押金退还，订单完成
```

---

## 退房状态流转

```
checked_in (已入住)
      ↓ 用户申请退房
checkout_pending (待检查)
      ↓ 房东确认
checked_out (已退房)
      ↓ 系统退款
refunded (已退款)
```

---

## 页面设计

### 退房申请页 (checkout)

**路径**：`pages/checkout`

**展示内容**：
| 内容 | 说明 |
|------|------|
| 房间信息 | 房间号 |
| 入住日期 | YYYY-MM-DD |
| 退房日期 | YYYY-MM-DD |
| 押金金额 | 已支付的押金 |
| 退房状态 | 待检查 / 已退房 |

**状态展示**：

| 状态 | 显示 | 操作 |
|------|------|------|
| checked_in | 可申请退房 | 显示「申请退房」按钮 |
| checkout_pending | 待检查 | 显示等待提示 |
| checked_out | 已退房 | 显示退款进度 |
| refunded | 已退款 | 显示完成信息 |

---

## API 接口

### 申请退房

```
POST /api/checkout
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
  "success": true,
  "status": "checkout_pending"
}
```

---

### 查询退房状态

```
GET /api/orders/:id/checkout-status
```

**响应**：
```json
{
  "status": "checkout_pending",
  "depositAmount": 50000,
  "refundStatus": null
}
```

或（已退款）：
```json
{
  "status": "refunded",
  "depositAmount": 50000,
  "refundStatus": "success",
  "refundedAt": "2026-03-11T14:00:00Z"
}
```

---

### 房东确认退房（管理后台调用）

```
POST /api/admin/checkout/confirm
```

**请求**：
```json
{
  "orderId": "ORD20260309001",
  "approved": true,
  "deduction": 0
}
```

**说明**：
- `approved`: 是否同意退房
- `deduction`: 扣款金额（分），如有损坏

---

### 退款（系统自动）

房东确认后，系统自动调用微信退款接口：

```
POST /api/deposit/refund
```

---

## 数据模型

### 退房状态

```typescript
type CheckoutStatus =
  | 'checked_in'        // 已入住
  | 'checkout_pending'  // 申请退房，待检查
  | 'checked_out'       // 已退房
  | 'refunded'          // 已退款
```

### 退款信息

```typescript
interface RefundInfo {
  refundAmount: number    // 实际退款金额（分）
  deduction: number       // 扣款金额（分）
  refundStatus: string    // pending / success / failed
  refundedAt: Date        // 退款时间
  refundId: string        // 微信退款单号
}
```

---

## 数据表

### orders 集合（更新字段）

```typescript
{
  // ... 其他字段
  status: string,           // checked_in → checkout_pending → checked_out
  checkoutAppliedAt: Date,  // 申请退房时间
  checkoutConfirmedAt: Date,// 房东确认时间

  // 退款相关
  refundStatus: string,     // pending / success / failed
  refundAmount: number,     // 实际退款金额（分）
  refundDeduction: number,  // 扣款金额（分）
  refundedAt: Date,         // 退款时间
  refundId: string,         // 微信退款单号
}
```

---

## 前端实现（待开发）

### 文件规划

| 文件 | 说明 |
|------|------|
| `pages/checkout/index.tsx` | 退房申请页 |

### 入口

1. 入住成功页 → 「申请退房」按钮
2. 首页 StayCard（当前入住卡片） → 「申请退房」

---

## 后端实现（待开发）

### 文件规划

| 文件 | 说明 |
|------|------|
| `routes/checkout.ts` | 退房路由 |
| `controllers/checkout.controller.ts` | 退房控制器 |
| `services/checkout.service.ts` | 退房业务逻辑 |

---

## 通知机制（可选）

### 房东通知

用户申请退房时，通知房东：
- 微信模板消息
- 或短信通知

### 用户通知

押金退还后，通知用户：
- 微信模板消息
