# 后端服务

> 路径：`/server`
> 技术栈：Node.js + Express + TypeScript + 云开发数据库 + 阿里云短信

## 相关文档

- [开发指南](./dev-guide.md) - MVC 约定、错误处理
- [数据表设计](./database.md) - 集合结构、初始化

---

## 架构概览

```
┌──────────────┐         ┌─────────────────────────────┐
│ 阿里云函数计算 │──SDK──▶ │ 腾讯云 CloudBase 数据库       │
│   (FC)       │         │  (NoSQL，上海节点)            │
│  Express     │         └─────────────────────────────┘
└──────┬───────┘
       │ HTTP 触发器（自带域名）
       │
┌──────┴───────┐          ┌──────────────────┐
│ 支付宝小程序  │          │    PMS 系统       │
│              │          │  （Webhook推送）   │
└──────────────┘          └────────┬─────────┘
                                   │
                                   ▼
                          POST /api/webhook/pms
```

### 部署方案

- **计算**：阿里云函数计算 FC（上海区域，custom.debian10 运行时 + Node.js 18 层，HTTP 触发器）
- **数据库**：腾讯云 CloudBase 云开发数据库（仅数据库，计算不在 CloudBase；数据在上海）
- **日志**：阿里云日志服务 SLS（project: xiaoer-logs，logstore: fc-logs）
- **CI/CD**：GitHub Actions → `s deploy`（Serverless Devs CLI）
- **域名**：`https://xiaoer-server-dodtzpbsbz.cn-shanghai.fcapp.run`

> 历史：先后尝试过 CloudBase 云托管（CI 插件不兼容）、CloudBase 云函数（免费套餐不支持 HTTP），最终选择阿里云 FC。

---

## 目录结构

```
/server
├── src/
│   ├── index.ts                    # 入口文件
│   ├── app.ts                      # Express 应用
│   ├── routes/                     # 路由层
│   ├── controllers/                # 控制器层
│   ├── services/                   # 服务层
│   ├── db/                         # 数据库层
│   └── middleware/                 # 中间件
├── Dockerfile
├── package.json
└── tsconfig.json
```

---

## API 接口

| 接口 | 方法 | 说明 |
|------|------|------|
| `/api/sms/send` | POST | 发送验证码 |
| `/api/auth/login` | POST | 登录 |
| `/api/orders` | GET | 订单列表 |
| `/api/checkin` | POST | 创建入住记录 |
| `/api/checkin/:orderId` | GET | 查询入住记录（无记录返回 null） |
| `/api/checkin/:orderId` | PUT | 更新入住记录 |
| `/api/deposit/create` | POST | 创建押金支付订单 |
| `/api/deposit/notify` | POST | 支付宝异步通知（验签 → 更新状态） |
| `/api/deposit/:orderId/status` | GET | 查询押金状态 |
| `/api/checkin/checkout` | POST | 退房（状态改 checked_out，房间改 dirty） |
| `/api/rooms/sync` | POST | 从 PMS 同步房间数据 |
| `/api/rooms` | GET | 获取房间列表 |
| `/api/user/guests` | GET | 获取用户历史住客列表（待开发） |

---

## 环境变量

```bash
# 云开发数据库
TCB_ENV_ID=xxx
TCB_SECRET_ID=xxx
TCB_SECRET_KEY=xxx

# 支付宝
ALIPAY_APP_ID=xxx
ALIPAY_PRIVATE_KEY=xxx
ALIPAY_PUBLIC_KEY=xxx
ALIPAY_NOTIFY_URL=xxx

# 百居易 PMS
HOSTEX_SESSION=xxx
HOSTEX_OPERATOR_ID=xxx

# JWT
JWT_SECRET=xxx
JWT_EXPIRES_IN=7d

# 服务
PORT=7001
```

---

## 部署

### 阿里云函数计算 FC

```bash
# 本地构建 + 部署
pnpm build
s deploy -y

# 或推送 main 分支，CI/CD 自动部署
git push origin main
```

- 运行时：`custom.debian10` + Node.js 18 层
- Express 直接 `app.listen(9000)`，不需要 `serverless-http`
- HTTP 触发器自带公网域名
- 环境变量通过 `s.yaml` 配置，CI 从 GitHub Secrets 注入

### CI/CD

GitHub Actions 自动部署，配置文件：`.github/workflows/deploy.yml`
推送 `main` 分支自动触发构建和部署。

Secrets 需要配置：
- `ALIYUN_ACCESS_KEY_ID`
- `ALIYUN_ACCESS_KEY_SECRET`
- 所有 `.env` 中的环境变量
