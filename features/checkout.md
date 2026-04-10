# 退房与退款

> 状态：🔄 基本完成（退房+退款接口已完成，退款缺权限校验）

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| StayCard 退房按钮 | ✅ 完成 | `pages/index/components/StayCard/index.tsx` |
| 退房确认弹窗 | ✅ 完成 | `pages/index/index.tsx`（Taro.showModal） |
| StayCard 改造 | ✅ 完成 | 用 checkinRecord，支持 checkout_pending 状态 |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 退房接口 | ✅ 完成 | `services/checkin.service.ts`（checkout 方法） |
| 退款接口 | ✅ 完成 | `controllers/deposit.controller.ts`（refund） |
| 支付宝退款 | ✅ 完成 | `services/alipay.service.ts`（refundTrade） |
| 退款权限校验 | ⬜ 待开发 | 退款接口未加 auth 中间件，任何人可调 |

### API

| 接口 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/api/checkin/:orderId` | GET | ✅ 完成 | 查询入住状态，StayCard 数据来源 |
| `/api/checkin/checkout` | POST | ✅ 完成 | 退房（状态改 checked_out，房间状态改 dirty） |
| `/api/deposit/refund` | POST | ✅ 完成 | 退款（缺 auth + 管理员权限校验） |

---

## 概述

用户入住后，在首页 StayCard 点「退房」提交退房申请，老板在管理端确认后退还押金（支持部分退款）。

---

## StayCard（已完成）

- 数据来源：`checkinRecord`（从 useAppStore 获取）
- 按钮：「开门密码」+「退房」
- 显示逻辑：
  - `status = checked_in` → 显示 StayCard
  - `status = checkout_pending` → 显示「待确认退房」badge
  - 其他状态 → 不显示

---

## 用户流程

```
首页 StayCard
  ├── 开门密码按钮 → 显示门锁密码
  └── 退房按钮 → 退房确认弹窗
        ↓ 确认
  POST /api/checkout
        ↓
  状态变为 checkout_pending
  StayCard 显示「待确认退房」
        ↓ 老板确认并退款
  押金原路退回支付宝
  状态变为 refunded
```

---

## 退房状态流转

```
checked_in (已入住)
      ↓ 用户点「退房」
checkout_pending (待检查)
      ↓ 老板确认退房
checked_out (已退房)
      ↓ 老板操作退款（可调整金额）
refunded (已退款)
```

---

## API 接口

### 提交退房

```
POST /api/checkout
Authorization: Bearer <token>
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
  "data": {
    "status": "checkout_pending"
  }
}
```

**逻辑**：
- 验证订单存在且 status = checked_in
- 更新 status 为 checkout_pending
- 记录 checkoutAppliedAt

---

### 退款（老板端）

```
POST /api/deposit/refund
Authorization: Bearer <token>  (需要管理员权限)
```

**请求**：
```json
{
  "orderId": "ORD20260309001",
  "refundAmount": 50000
}
```

**说明**：
- `refundAmount`：退款金额（分），可以小于押金（部分退款）
- 不传 `refundAmount` 默认全额退款

**响应**：
```json
{
  "success": true,
  "data": {
    "refundAmount": 50000,
    "status": "refunded"
  }
}
```

**逻辑**：
1. 从押金记录取 `transactionId`（支付宝交易号）
2. 调 `alipay.trade.refund` 原路退款
3. 更新押金记录状态为 refunded
4. 更新入住记录状态为 checked_out

---

## 数据模型

### 入住记录 check_in_records（更新字段）

```typescript
{
  // ... 已有字段
  status: 'pending' | 'checked_in' | 'checkout_pending' | 'checked_out'
  checkoutAppliedAt?: Date    // 申请退房时间
}
```

### 押金记录 deposits（更新字段）

```typescript
{
  // ... 已有字段
  status: 'created' | 'paid' | 'refunded' | 'partial_refund'
  payerUserId?: string        // 支付者平台用户ID（已有）
  refundAmount?: number       // 实际退款金额（分）
  refundedAt?: Date           // 退款时间
  refundTradeNo?: string      // 支付宝退款交易号
}
```

---

## 待办事项

1. ⬜ 退款接口加 auth 中间件 + 管理员权限校验（安全问题）
2. ⬜ 老板端退款管理页面（查看待退款列表、操作退款、支持部分退款）
3. ⬜ 打扫完成接口 `PUT /api/rooms/:roomNumber/status`（老板端标记房间已打扫）
