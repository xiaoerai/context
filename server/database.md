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
  guestIds?: string[],      // 关联的住客ID列表（按最近使用排序）
  createdAt: Date,
  lastLoginAt: Date
}
```

> 注：平台用户ID（alipayUserId / openid）只存在 JWT 中，不存数据库

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

> 入住记录（订单从 PMS 实时查询，不存本地）

```typescript
{
  _id: string,
  hostexOrderId: string,    // PMS 订单ID（唯一索引）
  roomNumber: string,       // 房间号（关联 rooms 表，如 "301"）
  roomName: string,         // 房间名称（冗余，方便显示）
  phone: string,            // 入住人手机号（索引）
  checkInDate: string,      // YYYY-MM-DD
  checkOutDate: string,
  source?: string,          // 订单来源 OTA 平台（meituan / ctrip / douyin / manual）
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
  tradeNO?: string,         // 支付宝交易号
  transactionId?: string,   // 支付回调的交易流水号
  payerUserId?: string,     // 支付者的平台用户ID（退款用）
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
  roomNumber: string,       // 房间号（唯一标识，如 "301"）
  roomName: string,         // 房间名（如 "301 轻旅｜投影大床房"）
  doorCode: string,         // 门锁密码
  wifiName: string,         // WiFi 名称
  wifiPassword: string,     // WiFi 密码
  deposit: number,          // 押金金额（分）
  status: string,           // available / occupied / dirty

  // PMS 平台关联（支持多 PMS）
  pms: [
    { platform: string, roomId: string }
    // 如 { platform: "hostex", roomId: "12556371" }
    // 如 { platform: "fliggy", roomId: "FL_98765" }
  ]
}
```

> 房间状态流转：available → occupied（支付成功）→ dirty（退房）→ available（打扫完成）
>
> 入住时：前端传 pmsRoomId + pms，后端查 pms 映射拿到 roomNumber

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
