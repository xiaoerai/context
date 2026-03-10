# 后端服务

> 路径：`/server`
> 技术栈：Node.js + Express + TypeScript + 微信云托管 + 云开发数据库 + 阿里云短信

---

## 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                     微信生态                             │
│                                                         │
│  ┌─────────────┐              ┌─────────────┐          │
│  │  微信云托管  │──SDK连接──▶ │ 云开发数据库  │          │
│  │  (Express)  │              │  (NoSQL)    │          │
│  └──────┬──────┘              └─────────────┘          │
│         │                                               │
└─────────┼───────────────────────────────────────────────┘
          │
          │ wx.cloud.callContainer()
          │ （内网通信，免域名免备案）
          │
┌─────────┴─────────┐          ┌──────────────────┐
│   微信小程序       │          │    PMS 系统       │
│                   │          │  （Webhook推送）   │
└───────────────────┘          └────────┬─────────┘
                                        │
                                        ▼
                               POST /api/webhook/pms
```

---

## 费用说明

| 服务 | 费用 | 说明 |
|------|------|------|
| 微信云托管 | 按量付费 | 没请求不花钱，前3个月有免费额度 |
| 云开发数据库 | 19.9 元/月 | 包含 3GB 数据库、20万次调用 |
| 阿里云短信 | 按条付费 | 约 0.04 元/条 |

**开发阶段免费**：云开发有免费体验版（3000点/月，6个月）

---

## 认证流程

> 手机号短信验证码 + wx.login 获取 openid

```
小程序                      阿里云短信              微信              后端
  │                           │                    │                  │
  │── 1. 打开小程序，wx.login ─────────────────────▶│                  │
  │◀── 2. 返回 code ──────────────────────────────│                  │
  │                           │                    │                  │
  │   [用户点击办理入住，输入手机号]                  │                  │
  │                           │                    │                  │
  │── 3. POST /api/sms/send { phone } ────────────────────────────────▶│
  │                           │◀── 4. 发送验证码 ──────────────────────│
  │◀── 5. 返回 { success } ───────────────────────────────────────────│
  │                           │                    │                  │
  │   [用户输入验证码]          │                    │                  │
  │                           │                    │                  │
  │── 6. POST /api/auth/login { code, phone, smsCode } ───────────────▶│
  │                           │                    │◀─ 7. code→openid─│
  │                           │                    │                  │
  │                           │      8. 验证 smsCode + 保存用户        │
  │                           │                    │                  │
  │◀── 9. 返回 { token, orders } ─────────────────────────────────────│
```

---

## 目录结构（MVC）

```
/server
├── src/
│   ├── index.ts                    # 入口文件
│   ├── app.ts                      # Express 应用
│   │
│   ├── routes/                     # 路由层（只定义路由映射）
│   │   ├── index.ts                # 路由汇总
│   │   ├── auth.ts                 # /api/auth/*
│   │   ├── sms.ts                  # /api/sms/*
│   │   ├── orders.ts               # /api/orders/*
│   │   ├── checkin.ts              # /api/checkin/*
│   │   └── webhook.ts              # /api/webhook/*
│   │
│   ├── controllers/                # 控制器层（参数验证、调用service、返回响应）
│   │   ├── auth.controller.ts      # 登录
│   │   ├── sms.controller.ts       # 短信验证码
│   │   ├── orders.controller.ts    # 订单
│   │   └── checkin.controller.ts   # 入住
│   │
│   ├── services/                   # 服务层（核心业务逻辑）
│   │   ├── auth.service.ts         # 登录业务逻辑
│   │   ├── sms.service.ts          # 短信业务逻辑
│   │   ├── db.ts                   # 数据库操作
│   │   ├── sms.ts                  # 阿里云短信 SDK
│   │   └── wechat.ts               # 微信 API
│   │
│   ├── middleware/                 # 中间件
│   │   ├── auth.ts                 # JWT 验证
│   │   ├── error.ts                # 错误处理
│   │   └── logger.ts               # 日志
│   │
│   └── utils/                      # 工具函数
│       └── jwt.ts                  # JWT 签发/验证
│
├── Dockerfile                      # 容器配置
├── package.json
└── tsconfig.json
```

### MVC 职责划分

| 层 | 职责 | 示例 |
|---|---|---|
| **routes** | 定义路由映射 | `router.post('/login', login)` |
| **controllers** | 参数验证、调用 service、返回响应 | 验证 phone 格式，调用 loginWithSmsCode() |
| **services** | 核心业务逻辑 | 验证码校验、用户创建、JWT 签发 |

---

## 数据表设计

> 云开发数据库（NoSQL / MongoDB 风格）
> 验证码直接存数据库，不需要 Redis

### users 集合

```typescript
{
  _id: string,
  openid: string,           // 微信openid（唯一索引）
  phone: string,            // 手机号（索引）
  createdAt: Date,
  lastLoginAt: Date
}
```

### sms_codes 集合

```typescript
{
  _id: string,
  phone: string,            // 手机号（唯一索引）
  code: string,             // 6位验证码
  expireAt: Date,           // 过期时间（5分钟后）
  createdAt: Date
}
```

### orders 集合

```typescript
{
  _id: string,
  orderId: string,          // 订单ID（唯一索引）
  hotelId: string,
  phone: string,            // 预订人手机号（索引）
  roomNumber: string,
  checkInDate: string,      // YYYY-MM-DD
  checkOutDate: string,
  depositStatus: string,    // unpaid / paid / refunded
  status: string,           // pending / checked_in / checked_out
  createdAt: Date,
  updatedAt: Date
}
```

### guests 集合

```typescript
{
  _id: string,
  orderId: string,          // 订单ID（索引）
  name: string,
  idNumber: string,
  phone: string,
  createdAt: Date
}
```

### rooms 集合

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

## API 接口

### 认证

| 接口 | 方法 | 说明 | 鉴权 |
|------|------|------|------|
| `/api/sms/send` | POST | 发送验证码 | 否 |
| `/api/auth/login` | POST | 登录 | 否 |

### 订单

| 接口 | 方法 | 说明 | 鉴权 |
|------|------|------|------|
| `/api/orders` | GET | 订单列表 | 是 |
| `/api/orders/:id` | GET | 订单详情 | 是 |

### 入住

| 接口 | 方法 | 说明 | 鉴权 |
|------|------|------|------|
| `/api/checkin` | POST | 提交入住信息 | 是 |

### 支付

| 接口 | 方法 | 说明 | 鉴权 |
|------|------|------|------|
| `/api/deposit/create` | POST | 创建支付 | 是 |
| `/api/deposit/notify` | POST | 支付回调 | 否 |

### Webhook

| 接口 | 方法 | 说明 | 鉴权 |
|------|------|------|------|
| `/api/webhook/pms` | POST | PMS 订单推送 | 签名验证 |

---

## 环境变量

```bash
# 云开发数据库
TCB_ENV_ID=xxx                      # 云开发环境ID
TCB_SECRET_ID=xxx                   # 腾讯云 SecretId
TCB_SECRET_KEY=xxx                  # 腾讯云 SecretKey

# 阿里云短信
ALIYUN_ACCESS_KEY_ID=xxx
ALIYUN_ACCESS_KEY_SECRET=xxx
ALIYUN_SMS_SIGN_NAME=xxx
ALIYUN_SMS_TEMPLATE_CODE=xxx

# 微信小程序
WECHAT_APPID=xxx
WECHAT_APP_SECRET=xxx

# 微信支付
WECHAT_MCH_ID=xxx
WECHAT_API_KEY=xxx

# JWT
JWT_SECRET=xxx
JWT_EXPIRES_IN=7d

# 服务
PORT=7001
```

---

## 数据库连接

```typescript
// services/db.ts
import tcb from '@cloudbase/node-sdk'

const app = tcb.init({
  env: process.env.TCB_ENV_ID,
  secretId: process.env.TCB_SECRET_ID,
  secretKey: process.env.TCB_SECRET_KEY,
})

export const db = app.database()

// 示例：存验证码
await db.collection('sms_codes').add({
  phone: '13800138000',
  code: '123456',
  expireAt: new Date(Date.now() + 5 * 60 * 1000),
  createdAt: new Date()
})

// 示例：查验证码
const { data } = await db.collection('sms_codes')
  .where({ phone, code })
  .get()
```

---

## 部署

### 1. 安装 CLI

```bash
npm install -g @cloudbase/cli
tcb login
```

### 2. Dockerfile

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY dist ./dist
EXPOSE 7001
CMD ["node", "dist/index.js"]
```

### 3. 部署命令

```bash
# 构建
pnpm build

# 部署到云托管
tcb framework deploy
```

---

## 小程序端调用

```typescript
// 不需要配置域名，直接用 wx.cloud.callContainer

// 初始化
wx.cloud.init({
  env: 'your-env-id'
})

// 调用后端 API
const res = await wx.cloud.callContainer({
  config: {
    env: 'your-env-id'
  },
  path: '/api/sms/send',
  method: 'POST',
  data: { phone: '13800138000' }
})
```
