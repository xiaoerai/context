# 办理入住

> 状态：✅ 完成（前端） / ⬜ 待开发（后端API）

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 办理入住弹窗 | ✅ 完成 | `pages/index/components/CheckInModal/index.tsx` |
| 入住信息页 | ✅ 完成 | `pages/checkin/index.tsx` |
| 顶部导航 | ✅ 完成 | `pages/checkin/components/NavBar/index.tsx` |
| 表单区域 | ✅ 完成 | `pages/checkin/components/FormSection/index.tsx` |
| 上传区域 | ✅ 完成 | `pages/checkin/components/UploadSection/index.tsx` |
| OCR 识别 | ✅ 完成 | `utils/ocr.ts` |
| 表单验证 | ✅ 完成 | `utils/schemas.ts` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住路由 | ⬜ 待开发 | `routes/checkin.ts` |
| 入住控制器 | ⬜ 待开发 | `controllers/checkin.controller.ts` |
| 入住服务 | ⬜ 待开发 | `services/checkin.service.ts` |

### API

| 接口 | 状态 |
|------|------|
| `POST /api/checkin` | ⬜ 待开发 |

### 数据表

| 集合 | 状态 |
|------|------|
| guests | ⬜ 待创建 |

---

## 概述

用户办理入住的完整流程，包括：
1. 验证身份（手机号+验证码）
2. 选择当日订单
3. 填写入住信息（身份证OCR）
4. 提交入住信息

---

## 用户流程

```
┌─────────────────────────────────┐
│  Step 1: 验证手机号              │
│  ┌─────────────────────────┐    │
│  │ 手机号: [___________]   │    │
│  │ 验证码: [____] [获取]   │    │
│  │                         │    │
│  │      [下一步]           │    │
│  └─────────────────────────┘    │
└─────────────────────────────────┘
              ↓ 验证通过
┌─────────────────────────────────┐
│  Step 2: 选择订单                │
│  ┌─────────────────────────┐    │
│  │ 🏠 301室                │    │
│  │ 3月9日 - 3月11日        │    │
│  └─────────────────────────┘    │
│  ┌─────────────────────────┐    │
│  │ 🏠 502室                │    │
│  │ 3月9日 - 3月10日        │    │
│  └─────────────────────────┘    │
└─────────────────────────────────┘
              ↓ 点击选择
┌─────────────────────────────────┐
│  Step 3: 入住信息页              │
│  ┌─────────────────────────┐    │
│  │ [上传身份证正面]         │    │
│  │     (OCR自动识别)        │    │
│  └─────────────────────────┘    │
│  ┌─────────────────────────┐    │
│  │ 姓名: [___________]     │    │
│  │ 身份证号: [___________] │    │
│  │ 手机号: [___________]   │    │
│  └─────────────────────────┘    │
│        [提交入住信息]            │
└─────────────────────────────────┘
              ↓ 提交成功
        跳转押金支付页
```

---

## 功能模块

### 1. 办理入住弹窗 (CheckInModal)

**位置**：首页 → 点击「办理入住」

**Step 1 - 手机号验证**：
- 手机号输入框（11位）
- 验证码输入框（4-6位）
- 获取验证码按钮（60秒倒计时）

**Step 2 - 选择订单**：
- 验证通过后显示当日订单列表
- 订单卡片：房间号、入住日期、退房日期
- 无订单时提示"该手机号今日无待办理订单"
- 可返回 Step 1 更换手机号

---

### 2. 入住信息页 (checkin)

**路径**：`pages/checkin`

**表单字段**：
| 字段 | 必填 | 验证规则 |
|------|------|----------|
| 姓名 | 是 | 非空 |
| 身份证号 | 是 | 18位有效格式 |
| 手机号 | 是 | 11位数字 |

**上传内容**：
- 身份证正面（可选，OCR自动识别填充表单）

**OCR 识别**：
- 拍照/选择图片后自动调用 OCR
- 识别成功自动填充姓名和身份证号
- 识别失败可手动填写

---

## 数据模型

### 入住信息

```typescript
interface GuestInfo {
  name: string           // 姓名
  idNumber: string       // 身份证号
  phone: string          // 手机号
  idCardFrontUrl?: string // 身份证正面照片URL
}
```

### 订单信息

```typescript
interface Order {
  orderId: string        // 订单ID
  roomNumber: string     // 房间号
  checkInDate: string    // 入住日期 YYYY-MM-DD
  checkOutDate: string   // 退房日期 YYYY-MM-DD
  depositStatus: string  // 押金状态: unpaid / paid / refunded
  status: string         // 订单状态: pending / checked_in / checked_out
}
```

---

## API 接口

### 获取订单列表

> 已在登录接口返回，无需单独请求

---

### 提交入住信息

```
POST /api/checkin
```

> 状态：⬜ 待开发

**请求**：
```json
{
  "orderId": "ORD20260309001",
  "guests": [
    {
      "name": "张三",
      "idNumber": "110101199001011234",
      "phone": "13800138000",
      "idCardFrontUrl": "https://..."
    }
  ]
}
```

**响应**：
```json
{
  "success": true,
  "depositAmount": 50000,
  "depositStatus": "unpaid"
}
```

---

## 前端实现

### 相关文件

| 文件 | 说明 |
|------|------|
| `pages/index/components/CheckInModal/index.tsx` | 办理入住弹窗 |
| `pages/checkin/index.tsx` | 入住信息页 |
| `pages/checkin/components/NavBar/index.tsx` | 顶部导航 |
| `pages/checkin/components/FormSection/index.tsx` | 表单区域 |
| `pages/checkin/components/UploadSection/index.tsx` | 上传区域 |
| `utils/ocr.ts` | 身份证OCR识别 |
| `utils/schemas.ts` | Zod验证规则 |

### 验证规则

```typescript
// utils/schemas.ts
import { z } from 'zod'

export const guestSchema = z.object({
  name: z.string().min(1, '请输入姓名'),
  idNumber: z.string().regex(/^\d{17}[\dXx]$/, '请输入有效身份证号'),
  phone: z.string().regex(/^1\d{10}$/, '请输入有效手机号'),
})
```

---

## 后端实现

### 待开发文件

| 文件 | 说明 |
|------|------|
| `routes/checkin.ts` | 入住路由 |
| `controllers/checkin.controller.ts` | 入住控制器 |
| `services/checkin.service.ts` | 入住业务逻辑 |

### 数据表

**guests 集合**：
```typescript
{
  _id: string,
  orderId: string,       // 订单ID（索引）
  name: string,
  idNumber: string,
  phone: string,
  idCardFrontUrl: string,
  createdAt: Date
}
```

---

## 路由保护

入住信息页需要验证用户已选择订单：

```typescript
// hooks/useOrderAuth.ts
// 检查是否有选中的订单，无则跳回首页
```
