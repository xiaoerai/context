# 订单同步

> 状态：⬜ 待开发

---

## 概述

从百居易 (Hostex) PMS 系统同步订单到本地数据库，供客人入住时查询。

### 为什么不用官方 API？

| 方案 | 费用 | 说明 |
|------|------|------|
| 官方 API | 8000 元/年 | 需要购买专业版 |
| Session 请求 | 免费 | 模拟浏览器请求，用登录后的 Cookie |

我们采用 **Session 请求** 方案，直接用房东账号的登录态获取订单。

---

## 技术方案

### 架构

```
┌─────────────────┐     定时触发(5-10分钟)     ┌─────────────────┐
│   腾讯云函数     │ ◀────────────────────────  │   定时器        │
│  (sync-service) │                            └─────────────────┘
└────────┬────────┘
         │
         │ 1. HTTP 请求 + Cookie
         ▼
┌─────────────────┐
│   百居易 API     │
│  myhostex.com   │
└────────┬────────┘
         │
         │ 2. 返回订单列表
         ▼
┌─────────────────┐
│   同步服务       │  3. 解析 + 转换格式
└────────┬────────┘
         │
         │ 4. 写入数据库
         ▼
┌─────────────────┐
│  CloudBase 数据库│
│   orders 集合    │
└────────┬────────┘
         │
         │ 5. 客人查询订单
         ▼
┌─────────────────┐
│   主服务 server  │
└─────────────────┘
```

### 认证方式

百居易使用 Cookie + Session 认证（不是 JWT）：

| Cookie | 说明 | 有效期 |
|--------|------|--------|
| `hostex_session` | 会话令牌 | 约 1 年 |
| `operator_id` | 房东 ID | 长期 |

**获取方式**：
1. 浏览器登录 https://www.myhostex.com
2. F12 → Application → Cookies
3. 复制 `hostex_session` 和 `operator_id` 的值

### API 端点

```
GET https://www.myhostex.com/api/reservation_order/list
```

**请求参数**：

| 参数 | 类型 | 说明 |
|------|------|------|
| page | int | 页码 |
| page_size | int | 每页数量 |
| date_type | string | 日期类型：check_in |
| is_exclude_pending | bool | 排除待确认订单 |
| opid | string | 房东 ID |
| opclient | string | 客户端标识 |

**请求示例**：

```javascript
fetch("https://www.myhostex.com/api/reservation_order/list?page=1&page_size=50&date_type=check_in&is_exclude_pending=true&opid=120002&opclient=Web-Linux-Chrome", {
  headers: {
    "accept": "application/json",
    "cookie": "hostex_session=xxx; operator_id=120002"
  }
})
```

**响应示例**：

```json
{
  "request_id": "RT2026031616162054907",
  "error_code": 0,
  "error_msg": "操作成功",
  "data": {
    "list": [
      {
        "code": "1-801252832144",
        "uniq_code": "1-801252832144-AJS82",
        "house_id": 2866,
        "house_name": "悦享民宿",
        "room_name": "大床房 301",
        "check_in": "2026-03-16",
        "check_out": "2026-03-18",
        "status": "accepted",
        "staying_status": "wait_stay",
        "lock_password": "123456",
        "guests": [
          {
            "name": "张三",
            "full_name": "张三",
            "phone": "+86 138 0013 8000",
            "email": "user@example.com"
          }
        ]
      }
    ]
  }
}
```

---

## 数据映射

### 百居易 → 本地数据库

| 百居易字段 | 本地字段 | 说明 |
|-----------|---------|------|
| uniq_code | orderId | 订单唯一编码 |
| room_name | roomNumber | 房间名称 |
| check_in | checkInDate | 入住日期 |
| check_out | checkOutDate | 离店日期 |
| staying_status | status | 入住状态 |
| lock_password | lockPassword | 门锁密码 |
| guests[0].phone | phone | 客人手机号 |
| guests[0].full_name | guestName | 客人姓名 |

### 手机号格式化

百居易返回格式带空格和国际区号，需要处理：

```typescript
// 百居易格式：+86 138 0013 8000
// 用户输入格式：13800138000

function normalizePhone(phone: string): string {
  return phone.replace(/[\s\+\-]/g, '').replace(/^86/, '')
}
```

---

## 同步策略

### 定时同步

| 配置 | 值 | 说明 |
|------|-----|------|
| 频率 | 每 5-10 分钟 | 云函数定时触发 |
| 范围 | 近 30 天订单 | 避免数据量过大 |
| 去重 | 按 orderId | 已存在则更新 |

### 同步逻辑

```
1. 获取百居易订单列表
2. 遍历每个订单：
   - 格式化手机号
   - 检查本地是否存在
   - 存在 → 更新
   - 不存在 → 插入
3. 记录同步日志
```

---

## 环境变量

```bash
# 百居易 Session（从浏览器 Cookie 获取）
HOSTEX_SESSION=g63nTvEWOpevc3LMVxR4LPbEdNXwVqLraLrrYTTS
HOSTEX_OPERATOR_ID=120002

# 云开发数据库
TCB_ENV_ID=xiaoer-dev-3g5g7pw9482967c9
TCB_SECRET_ID=xxx
TCB_SECRET_KEY=xxx
```

---

## 风险与应对

### Session 过期

| 风险 | 应对 |
|------|------|
| Session 过期 | 当前有效期约 1 年，过期后重新登录获取 |
| 被封禁 | 百居易是小公司（400万 Pre-A），风控较弱；保持 5-10 分钟间隔 |

### 监控

- 同步失败时发送告警（待实现）
- 记录每次同步的订单数量和耗时

---

## 目录结构

```
/sync-service
├── src/
│   ├── index.ts           # 云函数入口
│   ├── hostex.ts          # Hostex API 请求
│   ├── db.ts              # 数据库操作
│   └── types.ts           # 类型定义
├── package.json
├── tsconfig.json
└── .env.example
```

---

## 开发计划

| 任务 | 状态 |
|------|------|
| 创建 sync-service 项目 | ⬜ 待开发 |
| 实现 Hostex 请求逻辑 | ⬜ 待开发 |
| 实现数据库写入 | ⬜ 待开发 |
| 云函数部署配置 | ⬜ 待开发 |
| 定时触发器配置 | ⬜ 待开发 |
| Session 过期监控 | ⬜ 待开发 |
