# 办理入住

> 状态：✅ 基本完成（入住 + 押金支付已联调通过）

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住流程 Hook | ✅ 完成 | `hooks/checkinFlow/index.tsx` |
| 手机号验证步骤 | ✅ 完成 | `hooks/checkinFlow/usePhoneStep.ts` |
| 订单选择步骤 | ✅ 完成 | `hooks/checkinFlow/useOrdersStep.ts` |
| 押金支付步骤 | ✅ 完成 | `hooks/checkinFlow/useDepositStep.ts` |
| 入住信息页 | ✅ 完成 | `pages/checkin/index.tsx` |
| 住客选择 | ✅ 完成 | `pages/checkin/components/GuestPicker/index.tsx` |
| 表单区域 | ✅ 完成 | `pages/checkin/components/FormSection/index.tsx` |
| 上传区域 | ✅ 完成 | `pages/checkin/components/UploadSection/index.tsx` |
| OCR 识别 | ✅ 完成 | `utils/ocr.ts` |
| 表单验证 | ✅ 完成 | `utils/schemas.ts` |
| StayCard | ✅ 完成 | `pages/index/components/StayCard/index.tsx` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住路由 | ✅ 完成 | `routes/checkin.ts` |
| 入住控制器 | ✅ 完成 | `controllers/checkin.controller.ts` |
| 入住服务 | ✅ 完成 | `services/checkin.service.ts` |
| 退房接口 | ✅ 完成 | `services/checkin.service.ts` (checkout) |
| 房间同步 | ✅ 完成 | `controllers/rooms.controller.ts` |

### API

| 接口 | 状态 | 说明 |
|------|------|------|
| `POST /api/checkin` | ✅ 完成 | 创建入住记录 |
| `GET /api/checkin/:orderId` | ✅ 完成 | 查询入住记录 |
| `PUT /api/checkin/:orderId` | ✅ 完成 | 更新入住记录 |
| `POST /api/checkin/checkout` | ✅ 完成 | 退房 |
| `POST /api/rooms/sync` | ✅ 完成 | 从 PMS 同步房间 |
| `GET /api/rooms` | ✅ 完成 | 获取房间列表 |

### 已完成的改造

| 改动 | 说明 |
|------|------|
| 入住记录 `roomId` → `roomNumber` | ✅ 用房间号关联 rooms 表，不依赖第三方 ID |
| 入住时传 `pmsRoomId` + `pms` | ✅ 后端查 rooms 表 pms 映射拿 roomNumber |
| 入住记录加 `source` 字段 | ✅ 记录订单来源 OTA 平台 |
| 支付成功 → 房间状态改 `occupied` | ✅ 在 handleAlipayNotify 中联动 |
| 退房 → 房间状态改 `dirty` | ✅ 在 checkout 中联动 |
| rooms 表改用 `pms` 数组映射 | ✅ 支持多 PMS 平台 |
| 订单数据加 `source`/`pms`/`pmsRoomId` | ✅ 从 Hostex 返回平台信息 |

### 待开发

| 改动 | 说明 |
|------|------|
| 前端入住时传 `pmsRoomId` + `pms` | 前端需要从订单数据取 houseId 传给后端 |
| 前端退房确认弹窗 | StayCard 退房按钮的交互逻辑 |
| 退款接口 `POST /api/deposit/refund` | 老板端操作，调支付宝退款 |
| 打扫完成接口 `PUT /api/rooms/:roomNumber/status` | 老板端标记房间已打扫 |

---

## 完整流程

```
1. 用户登录（手机号 + 平台授权）

2. 查询订单
   → 后端从 PMS 拉订单，返回 orderId、roomName、houseId、日期、source

3. 用户选订单
   → 查 GET /api/checkin/:orderId
   → 有记录：跳到押金或成功页
   → 没记录：跳到入住页

4. 填写住客信息，提交入住
   → POST /api/checkin { orderId, pmsRoomId, pms, roomName, phone, guest, source }
   → 后端查 rooms 表 pms 映射 → 拿到 roomNumber
   → 创建入住记录（status: pending）
   → 创建住客记录

5. 支付押金
   → POST /api/deposit/create { orderId, channel: "alipay" }
   → 返回 tradeNO → 前端调 my.tradePay 唤起支付

6. 支付宝回调
   → POST /api/deposit/notify
   → 入住记录 → checked_in
   → 房间状态 → occupied

7. 退房
   → POST /api/checkin/checkout { orderId }
   → 入住记录 → checked_out
   → 房间状态 → dirty

8. 打扫完成（老板端）
   → 房间状态 → available

9. 退款（老板端）
   → POST /api/deposit/refund
```

---

## 房间状态流转

```
available → occupied（支付成功回调）→ dirty（退房）→ available（打扫完成）
```

---

## 数据模型

### 入住记录

```typescript
{
  hostexOrderId: string,    // PMS 订单ID
  roomNumber: string,       // 房间号（关联 rooms 表）
  roomName: string,         // 房间名（冗余显示）
  phone: string,            // 入住人手机号
  checkInDate: string,      // YYYY-MM-DD
  checkOutDate: string,
  source?: string,          // OTA 来源（meituan / ctrip / douyin / manual）
  guestIds: string[],       // 住客ID列表
  depositId?: string,       // 押金记录ID
  depositPaid: boolean,
  status: string,           // pending / checked_in / checked_out
}
```

### 房间

```typescript
{
  roomNumber: string,       // 唯一标识
  roomName: string,
  doorCode: string,
  wifiName: string,
  wifiPassword: string,
  deposit: number,          // 押金（分）
  status: string,           // available / occupied / dirty
  pms: [                    // 多 PMS 映射
    { platform: string, roomId: string }
  ]
}
```

### 住客

```typescript
{
  name: string,
  idNumber: string,         // 身份证号（去重依据）
  idImageUrl?: string,
  createdAt: Date
}
```

---

## 开发环境

- `SKIP_DEPOSIT` 变量可跳过押金支付（`config/dev.ts` 中配置）
- 测试手机号 `15290500792` 跳过验证码
- mock 订单在测试手机号下自动返回

---

## 安全待办

- 入住不强制要求先付押金（pending 状态不检查押金），后续需改为押金支付后才算入住成功
- 短信验证码服务待接入（目前测试手机号跳过）
