# 数据表设计

> 腾讯云开发数据库（NoSQL / MongoDB 风格），仅使用数据库服务，计算层已迁移至阿里云 FC

## 集合列表

- `sms_codes` - 短信验证码
- `users` - 用户
- `check_in_records` - 入住记录
- `guests` - 住客
- `rooms` - 房间配置
- `deposits` - 押金记录

---

## users 集合

```typescript
{
  _id: string,
  phone: string,            // 手机号（唯一索引，跨端唯一标识）
  openid?: string,          // 微信 openid
  alipayUserId?: string,    // 支付宝用户ID
  guestIds?: string[],      // 关联的住客ID列表（按最近使用排序）
  createdAt: Date,
  lastLoginAt: Date
}
```

## sms_codes 集合

```typescript
{
  _id: string,
  phone: string,            // 手机号（唯一索引）
  code: string,             // 6位验证码
  expireAt: Date,           // 过期时间（5分钟后）
  createdAt: Date
}
```

## check_in_records 集合

> 入住记录（订单从百居易实时查询，不存本地）

```typescript
{
  _id: string,
  hostexOrderId: string,    // 百居易订单ID（唯一索引）
  roomId: string,           // 百居易房型ID
  roomName: string,         // 房间名称
  phone: string,            // 入住人手机号（索引）
  checkInDate: string,      // YYYY-MM-DD
  checkOutDate: string,
  guestIds: string[],       // 住客ID列表
  depositId?: string,       // 关联押金记录ID
  depositPaid: boolean,     // 是否已支付押金
  status: string,           // pending / checked_in / checked_out
  createdAt: Date,
  updatedAt: Date
}
```

## guests 集合

```typescript
{
  _id: string,
  name: string,             // 姓名
  idNumber: string,         // 身份证号（按此去重）
  idImageUrl?: string,      // 身份证照片
  createdAt: Date
}
```

## deposits 集合

```typescript
{
  _id: string,
  orderId: string,          // 关联的订单ID
  amount: number,           // 金额（分）
  channel: string,          // 支付渠道：alipay / wechat
  status: string,           // created / paid / refunded
  tradeNO?: string,         // 收钱吧/支付宝交易号
  transactionId?: string,   // 支付回调的交易流水号
  paidAt?: Date,            // 支付时间
  refundedAt?: Date,        // 退款时间
  createdAt: Date,
  updatedAt: Date
}
```

## rooms 集合

```typescript
{
  _id: string,
  hotelId: string,          // 民宿ID（索引）
  roomNumber: string,
  doorCode: string,
  wifiName: string,
  wifiPassword: string,
  deposit: number           // 押金（分）
}
```

---

## 初始化

```bash
cd server
npx tsx src/db/init.ts
```

## 数据库连接

```typescript
// db/client.ts
import tcb from '@cloudbase/node-sdk'

const app = tcb.init({
  env: process.env.TCB_ENV_ID!,
  secretId: process.env.TCB_SECRET_ID,
  secretKey: process.env.TCB_SECRET_KEY,
})

export const db = app.database()
```
