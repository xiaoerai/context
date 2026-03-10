# API 接口设计

## 客人端接口

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 查询订单 | GET | `/api/orders/query` | 根据手机号查询 |
| 提交入住信息 | POST | `/api/checkin/submit` | 提交登记信息 |
| 上传图片 | POST | `/api/upload/image` | 上传证件/自拍 |
| 创建押金支付 | POST | `/api/deposit/create` | 创建支付订单 |
| 支付回调 | POST | `/api/deposit/callback` | 微信支付回调 |
| 获取入住信息 | GET | `/api/checkin/{id}` | 获取门锁密码等 |
| 申请退房 | POST | `/api/checkout/apply` | 申请退房 |

---

## 管理后台接口

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 登录 | POST | `/api/admin/login` | 管理员登录 |
| 今日入住 | GET | `/api/admin/checkin/today` | 今日入住列表 |
| 待退房 | GET | `/api/admin/checkout/pending` | 待退房列表 |
| 确认退房 | POST | `/api/admin/checkout/confirm` | 确认退房 |
| 入住记录 | GET | `/api/admin/checkin/records` | 历史记录查询 |
| 房间列表 | GET | `/api/admin/rooms` | 获取房间 |
| 房间操作 | POST/PUT/DELETE | `/api/admin/rooms` | 房间增删改 |

---

## 数据模型

### 订单
```typescript
interface Order {
  orderId: string
  roomNumber: string
  checkInDate: string
  checkOutDate: string
  totalAmount: number
  guestName: string
  guestPhone: string
  status: 'pending' | 'checked_in' | 'checked_out'
}
```

### 入住登记
```typescript
interface GuestInfo {
  name: string
  idNumber: string
  phone: string
  idCardFrontUrl: string
  idCardBackUrl: string
  selfieUrl: string
}
```

### 押金支付
```typescript
interface DepositPayment {
  orderId: string
  depositAmount: number
  roomNumber: string
  checkInDate: string
  paymentMethod: 'wechat'
}
```

### 入住结果
```typescript
interface CheckInResult {
  roomNumber: string
  doorCode: string
  wifiName: string
  wifiPassword: string
  checkInTime: string
  checkOutTime: string
}
```

### 入住记录
```typescript
interface CheckInRecord {
  id: string
  orderId: string
  roomNumber: string
  guestName: string
  idNumber: string
  phone: string
  idCardFrontUrl: string
  idCardBackUrl: string
  selfieUrl: string
  checkInTime: string
  checkOutTime: string | null
  status: 'checked_in' | 'pending_checkout' | 'checked_out'
  depositAmount: number
  depositStatus: 'paid' | 'refunded' | 'deducted'
}
```

### 房间
```typescript
interface Room {
  id: string
  roomNumber: string
  roomType: string
  doorCode: string
  wifiName: string
  wifiPassword: string
  depositAmount: number
  status: 'available' | 'occupied' | 'cleaning'
}
```

---

## 环境配置

```env
# 开发环境
VITE_API_BASE_URL=http://localhost:3000/api

# 生产环境
VITE_API_BASE_URL=https://api.example.com/api
```
