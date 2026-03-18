# 押金支付

> 状态：✅ 完成（收钱吧接口 mock 中，待接入真实支付）

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 押金支付弹窗 | ✅ 完成 | `hooks/useCheckinFlow.tsx`（deposit step） |
| 入住成功弹窗 | ✅ 完成 | `hooks/useCheckinFlow.tsx`（success step） |
| 押金 API | ✅ 完成 | `api/deposit.ts` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 支付路由 | ✅ 完成 | `routes/deposit.ts` |
| 支付控制器 | ✅ 完成 | `controllers/deposit.controller.ts` |
| 支付服务 | ✅ 完成（mock） | `services/deposit.service.ts` |
| 收钱吧支付 SDK | ⬜ 待接入 | 待商户号申请完成 |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/deposit/create` | ✅ 完成（mock tradeNO） |
| `POST /api/deposit/confirm` | ✅ 完成（mock，真实环境由收钱吧回调） |

### 数据表

| 集合 | 状态 |
|------|------|
| deposits | ✅ 已创建 |
| rooms | ⬜ 待创建数据 |

---

## 概述

用户提交入住信息后，需支付押金才能获取门锁密码和WiFi信息。

---

## 账号申请流程

共需申请 3 个账号，前 2 个由营业执照法人（朋友）操作，第 3 个可自行申请。

### 前置条件

- 个体工商户营业执照（法人是朋友）
- 法人的个人支付宝账号

### 第 1 步：支付宝商家号

1. 法人登录 [支付宝商家中心](https://b.alipay.com)
2. 选择「个体工商户」主体类型（不要选企业）
3. 上传营业执照，完成认证（不需要对公账户）
4. 免费

### 第 2 步：支付宝小程序

1. 法人登录 [支付宝开放平台](https://open.alipay.com)
2. 点击「创建小程序」
3. 绑定第 1 步的商家账号（填自己的支付宝，同一个人直接通过）
4. 创建成功后，在小程序设置中把开发者（我）加为**开发成员**
5. 上线前需完成小程序备案

### 第 3 步：收钱吧商户

1. 登录 [收钱吧](https://www.shouqianba.com) 申请商户入驻
2. 提供营业执照等资料
3. 获取服务商序列号（vendor_sn）、终端序列号等密钥
4. 将小程序 AppID 提供给收钱吧进行配置
5. 费率 0.38%

### 申请状态

| 账号 | 状态 |
|------|------|
| 个体工商户营业执照 | ⬜ 待办理 |
| 支付宝商家号 | ⬜ 待申请 |
| 支付宝小程序 | ⬜ 待创建 |
| 收钱吧商户 | ⬜ 待申请 |

---

## 支付方案

### 平台：支付宝小程序

选择支付宝小程序而非微信小程序，因为：
- 支付宝个人主体可免费认证
- 微信需企业认证（300元/年）才能使用支付功能

### 支付通道：收钱吧（Shouqianba）

选择收钱吧作为支付聚合服务，因为：
- 支持支付宝、微信等多种支付方式
- 费率 0.38%
- 无需分别对接各支付平台 SDK
- 个人商户可接入

### 支付方式

在支付宝小程序中，通过收钱吧 SDK 调起支付宝支付。

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
│       [支付宝支付]              │
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
- 点击「支付宝支付」通过收钱吧调起支付

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
小程序                         后端                        收钱吧
  │                             │                            │
  │── 1. POST /api/deposit/create ──▶│                        │
  │      { orderId }            │                            │
  │                             │──── 2. 预下单 ────────────▶│
  │                             │◀─── 3. 返回支付参数 ────────│
  │◀── 4. 返回支付参数 ──────────│                            │
  │                             │                            │
  │── 5. my.tradePay ──────────────────────────────────────▶│
  │◀── 6. 支付结果 ────────────────────────────────────────│
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
  "tradeNo": "xxx",
  "orderStr": "xxx"
}
```

> 返回收钱吧的交易号和支付宝调起参数

---

### 支付回调

```
POST /api/deposit/notify
```

> 收钱吧服务器回调，更新订单押金状态

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
  transactionId: string,  // 收钱吧交易号
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
| `services/shouqianba.ts` | 收钱吧支付 SDK 封装 |

---

## 环境变量

```bash
# 收钱吧
SQB_VENDOR_SN=xxx          # 服务商序列号
SQB_VENDOR_KEY=xxx          # 服务商密钥
SQB_TERMINAL_SN=xxx         # 终端序列号
SQB_TERMINAL_KEY=xxx        # 终端密钥
```
