# 退房与退款

> 状态：🔄 开发中

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| StayCard 改造 | ⬜ 待开发 | `pages/index/components/StayCard/index.tsx` |
| 删除 currentStay，改用 checkinRecord | ⬜ 待开发 | `stores/slices/hotelSlice.ts`、`checkinSlice.ts` |
| 退房确认弹窗 | ⬜ 待开发 | StayCard 内弹窗 |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 退房接口 | ⬜ 待开发 | `controllers/checkout.controller.ts` |
| 退款接口 | ⬜ 待开发 | `controllers/deposit.controller.ts` |
| 支付宝退款 | ⬜ 待开发 | `services/alipay.service.ts` |

### API

| 接口 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/api/checkin/:orderId` | GET | ✅ 已有 | 查询入住状态，StayCard 数据来源 |
| `/api/checkout` | POST | ⬜ 待开发 | 客户提交退房 |
| `/api/deposit/refund` | POST | ⬜ 待开发 | 老板端退款操作 |

---

## 概述

用户入住后，在首页 StayCard 点「退房」提交退房申请，老板在管理端确认后退还押金（支持部分退款）。

---

## StayCard 改造

### 现状
- 数据来源：`currentStay`（mock 硬编码）
- 按钮：「详情」+「开门密码」

### 改造后
- 数据来源：`checkinRecord`（从后端 `/api/checkin/:orderId` 拉取）
- 按钮：「开门密码」+「退房」
- 房间名右侧显示 Wi-Fi 信息
- 删除 `currentStay` 和 `StayInfo` 类型，全部用 `CheckinRecord`

### 显示逻辑
- 有 `checkinRecord` 且 `status = checked_in` → 显示 StayCard
- `status = checkout_pending` → StayCard 显示「待确认退房」
- `status = checked_out` / `refunded` → 不显示或显示已退房
- 无记录 → 不显示 StayCard

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

## 后端文件规划

| 文件 | 说明 |
|------|------|
| `routes/checkout.ts` | 退房路由 |
| `controllers/checkout.controller.ts` | 退房控制器 |
| `services/alipay.service.ts` | 新增 refundTrade 函数 |
| `controllers/deposit.controller.ts` | 新增 refund handler |

---

## 前端改动

| 文件 | 改动 |
|------|------|
| `stores/slices/hotelSlice.ts` | 删除 `currentStay`、`StayInfo` |
| `stores/slices/checkinSlice.ts` | StayCard 数据来源 |
| `pages/index/components/StayCard/index.tsx` | 改用 checkinRecord，改按钮，加 Wi-Fi |
| `pages/index/index.tsx` | 去掉 mock 数据，加载时拉 checkinRecord |
