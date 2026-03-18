# 后端服务

> 路径：`/server`
> 技术栈：Node.js + Express + TypeScript + 微信云托管 + 云开发数据库 + 阿里云短信

## 相关文档

- [开发指南](./dev-guide.md) - MVC 约定、错误处理
- [数据表设计](./database.md) - 集合结构、初始化

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
| `/api/deposit/confirm` | POST | 确认押金支付（mock，真实环境由收钱吧回调） |
| `/api/user/guests` | GET | 获取用户历史住客列表（开发中） |

---

## 环境变量

```bash
# 云开发数据库
TCB_ENV_ID=xxx
TCB_SECRET_ID=xxx
TCB_SECRET_KEY=xxx

# 百居易 PMS
HOSTEX_SESSION=xxx
HOSTEX_OPERATOR_ID=xxx

# 微信小程序
WECHAT_APPID=xxx
WECHAT_APP_SECRET=xxx

# JWT
JWT_SECRET=xxx
JWT_EXPIRES_IN=7d

# 服务
PORT=7001
```

---

## 部署

```bash
# 构建
pnpm build

# 部署到云托管
tcb framework deploy
```
