# 老板端管理系统

> 状态：⬜ 待开发

---

## 当前进度

### 前端（admin/）

| 模块 | 状态 | 文件 |
|------|------|------|
| 项目初始化（Vite + React + TS） | ⬜ 待开发 | `admin/` |
| 登录页 | ⬜ 待开发 | `admin/src/pages/Login.tsx` |
| 布局组件（导航栏） | ⬜ 待开发 | `admin/src/components/Layout.tsx` |
| 房间管理页 | ⬜ 待开发 | `admin/src/pages/Rooms.tsx` |
| 退款管理页 | ⬜ 待开发 | `admin/src/pages/Deposits.tsx` |
| 入住记录页 | ⬜ 待开发 | `admin/src/pages/Checkins.tsx` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 管理员登录接口 | ⬜ 待开发 | `routes/admin.ts` + `controllers/admin.controller.ts` |
| 押金列表接口 | ⬜ 待开发 | `controllers/admin.controller.ts` |
| 入住记录列表接口 | ⬜ 待开发 | `controllers/admin.controller.ts` |
| 房间状态修改接口 | ⬜ 待开发 | `routes/rooms.ts` + `controllers/rooms.controller.ts` |
| adminOnly 中间件扩展 | ⬜ 待开发 | `middleware/auth.ts` |

### API

| 接口 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/api/admin/login` | POST | ⬜ 待开发 | 管理员密码登录 |
| `/api/admin/deposits` | GET | ⬜ 待开发 | 押金列表（支持状态筛选 + 分页） |
| `/api/admin/checkins` | GET | ⬜ 待开发 | 入住记录列表（支持日期/状态筛选 + 分页） |
| `/api/rooms/:roomNumber/status` | PUT | ⬜ 待开发 | 修改房间状态 |
| `/api/deposit/refund` | POST | ✅ 已有 | 退款（已加 auth + adminOnly） |
| `/api/rooms` | GET | ✅ 已有 | 房间列表 |
| `/api/rooms/sync` | POST | ✅ 已有 | 同步房间 |

---

## 概述

老板端是独立的 Web 管理系统，用于管理房间状态、处理退款、查看入住记录。独立部署，移动端适配，与用户端共用后端 API。

---

## 技术方案

- **前端**：根目录新建 `admin/` 项目，React + Vite + TypeScript
- **后端**：共用 `server/`，新增 `routes/admin.ts` 管理员专用路由
- **登录**：独立账号密码（环境变量 `ADMIN_USERNAME` + `ADMIN_PASSWORD`）
- **移动端适配**：CSS 媒体查询，移动优先

---

## API 接口

### 管理员登录

```
POST /api/admin/login
```

**请求**：
```json
{
  "username": "admin",
  "password": "xxx"
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "token": "JWT（含 role: admin，24h 过期）"
  }
}
```

**逻辑**：
- 比对环境变量 `ADMIN_USERNAME` + `ADMIN_PASSWORD`
- 用 `crypto.timingSafeEqual` 防止时序攻击
- 签发 JWT，payload 含 `{ role: 'admin' }`
- `adminOnly` 中间件扩展：接受 `role === 'admin'` 或 `phone in ADMIN_PHONES`

---

### 押金列表

```
GET /api/admin/deposits?status=paid&page=1&pageSize=20
Authorization: Bearer <admin token>
```

**查询参数**：
- `status`：筛选状态（created / paid / refunded），不传返回全部
- `page`：页码，默认 1
- `pageSize`：每页条数，默认 20

**响应**：
```json
{
  "success": true,
  "data": {
    "list": [
      {
        "_id": "xxx",
        "orderId": "ORD001",
        "amount": 1,
        "channel": "alipay",
        "status": "paid",
        "paidAt": "2026-03-21T10:00:00Z",
        "roomName": "301 豪华大床房",
        "phone": "138xxxx"
      }
    ],
    "total": 50,
    "page": 1,
    "pageSize": 20
  }
}
```

**逻辑**：
- 查 deposits 集合，按 `createdAt` 降序
- 应用层关联 check_in_records 拿 roomName 和 phone

---

### 入住记录列表

```
GET /api/admin/checkins?status=checked_in&page=1&pageSize=20
Authorization: Bearer <admin token>
```

**查询参数**：
- `status`：筛选状态（pending / checked_in / checked_out / checkout_pending）
- `startDate` / `endDate`：日期范围筛选
- `page` / `pageSize`：分页

**响应**：
```json
{
  "success": true,
  "data": {
    "list": [
      {
        "_id": "xxx",
        "hostexOrderId": "ORD001",
        "roomNumber": "301",
        "roomName": "301 豪华大床房",
        "phone": "138xxxx",
        "checkInDate": "2026-03-09",
        "checkOutDate": "2026-03-11",
        "status": "checked_in",
        "depositPaid": true,
        "ota": "meituan"
      }
    ],
    "total": 30,
    "page": 1,
    "pageSize": 20
  }
}
```

---

### 修改房间状态

```
PUT /api/rooms/:roomNumber/status
Authorization: Bearer <admin token>
```

**请求**：
```json
{
  "status": "available"
}
```

**响应**：
```json
{
  "success": true,
  "data": {
    "roomNumber": "301",
    "status": "available"
  }
}
```

**允许的状态变更**：
- `dirty` → `available`（打扫完成）
- `available` → `occupied`（手动标记）
- `occupied` → `dirty`（手动标记）

---

## 前端页面

### 登录页 `/login`
- 账号 + 密码输入框
- 登录按钮
- 登录后存 token 到 localStorage，跳转首页

### 房间管理 `/rooms`（默认首页）
- 房间卡片列表
- 每张卡片：房间号、房间名、状态标签（绿色 available / 红色 occupied / 黄色 dirty）
- dirty 状态显示「打扫完成」按钮
- 顶部同步按钮（调 POST /api/rooms/sync）

### 退款管理 `/deposits`
- Tab 切换：待退款（paid）/ 已退款（refunded）
- 列表：订单号、房间、金额、入住人、支付时间
- 操作：退款按钮 → 弹窗输入金额（默认全额）→ 确认退款

### 入住记录 `/checkins`
- 筛选栏：状态下拉 + 日期范围
- 列表：房间、住客手机、入住/退房日期、状态、OTA 来源

### 底部导航栏
- 三个 tab：房间 / 退款 / 记录

---

## 后端文件规划

| 文件 | 说明 | 新增/修改 |
|------|------|-----------|
| `routes/admin.ts` | 管理员路由 | 新增 |
| `controllers/admin.controller.ts` | 登录 + 列表查询 | 新增 |
| `services/admin.service.ts` | 登录验证 + 列表查询逻辑 | 新增 |
| `routes/index.ts` | 注册 admin 路由 | 修改 |
| `routes/rooms.ts` | 加 PUT /:roomNumber/status | 修改 |
| `controllers/rooms.controller.ts` | 加 updateStatus handler | 修改 |
| `middleware/auth.ts` | adminOnly 支持 role === 'admin' | 修改 |
| `services/auth.service.ts` | JwtPayload 加 role 字段 | 修改 |
| `db/deposits.ts` | 加 findAllDeposits + countDeposits | 修改 |
| `db/check_in_records.ts` | 加 findAllCheckInRecords + countCheckInRecords | 修改 |

---

## 环境变量

```bash
# 管理员登录
ADMIN_USERNAME=admin
ADMIN_PASSWORD=你的密码
```

---

## 待办事项

1. ⬜ 切片1：管理员登录（后端接口 + admin 项目初始化 + 登录页）
2. ⬜ 切片2：房间管理（房间状态修改接口 + 房间管理页）
3. ⬜ 切片3：退款管理（押金列表接口 + 退款管理页）
4. ⬜ 切片4：入住记录（记录列表接口 + 记录页）

切片 2/3/4 可并行开发，均依赖切片 1。
