# 订单查询

> 状态：✅ 实时查询已完成（本地同步暂不使用）

---

## 概述

客人办理入住时，通过手机号**实时查询**百居易 (Hostex) 订单，不做本地同步。

### 为什么不用官方 API？

| 方案 | 费用 | 说明 |
|------|------|------|
| 官方 API | 8000 元/年 | 需要购买专业版 |
| Session 请求 | 免费 | 模拟浏览器请求，用登录后的 Cookie |

我们采用 **Session 请求** 方案，直接用房东账号的登录态获取订单。

### 为什么不做本地同步？

| 方案 | 复杂度 | 说明 |
|------|--------|------|
| 本地同步 | 高 | 需要同步服务、云函数、数据库、处理取消/修改 |
| **实时查询** | **低** | 直接调 API，数据永远最新 |

民宿场景一天就几个客人，实时查询完全够用。

---

## 技术方案

### 架构

```
┌─────────────────┐
│   小程序前端     │
│   输入手机号     │
└────────┬────────┘
         │
         │ POST /api/orders/search
         ▼
┌─────────────────┐
│   主服务 server  │
│  hostex.service │
└────────┬────────┘
         │
         │ HTTP + Cookie
         ▼
┌─────────────────┐
│   百居易 API     │
│  myhostex.com   │
└────────┬────────┘
         │
         │ 订单列表
         ▼
┌─────────────────┐
│   格式化 + 返回  │
└─────────────────┘
```

**简单直接，不需要额外服务。**

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
| start_date | string | 入住日期起始（YYYY-MM-DD） |
| end_date | string | 入住日期截止（YYYY-MM-DD） |
| opid | string | 房东 ID |
| opclient | string | 客户端标识 |

**请求示例**：

```javascript
fetch("https://www.myhostex.com/api/reservation_order/list?page=1&page_size=100&start_date=2026-03-17&end_date=2026-03-24&opid=120002&opclient=Web-Linux-Chrome", {
  headers: {
    "accept": "application/json",
    "cookie": "hostex_session=xxx; operator_id=120002"
  }
})
```

**响应结构**（嵌套结构，入住信息在 reservations 数组中）：

```json
{
  "error_code": 0,
  "error_msg": "操作成功",
  "data": {
    "list": [
      {
        "code": "22-1092253481431381639-ib96sakgw5",
        "create_time": "2026-03-16 23:05:13",
        "update_time": "2026-03-16 23:05:13",
        "client": {
          "name": "周粥",
          "phone": "15072851529"
        },
        "reservations": [
          {
            "check_in": "2026-03-20 00:00:00",
            "check_out": "2026-03-21 00:00:00",
            "staying_status": "staying_wait",
            "house": {
              "id": 12556371,
              "title": "301 轻旅｜投影大床房"
            },
            "guest": {
              "name": "周粥",
              "phone": "15072851529"
            }
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

注意：订单是**嵌套结构**，入住信息在 `reservations[0]` 中。

| 百居易字段 | 本地字段 | 说明 |
|-----------|---------|------|
| code | orderId | 订单编码 |
| client.phone | phone | 客人手机号 |
| client.name | guestName | 客人姓名 |
| reservations[0].check_in | checkInDate | 入住日期 |
| reservations[0].check_out | checkOutDate | 离店日期 |
| reservations[0].staying_status | status | 入住状态 |
| reservations[0].house.title | roomNumber | 房间名称 |
| create_time | - | 订单创建时间（仅日志用） |
| update_time | - | 订单更新时间（判断是否需要更新） |

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

> **暂不使用**：第一版采用纯实时查询，不做本地同步。以下内容保留供后续参考。

### 核心思路

我们的业务场景是**办理入住**，只需要同步**即将入住**的订单：

```
预订时间 ≠ 入住时间

客人3月1日下单，订了4月1日的房
    ↓
我们不用立即同步
    ↓
等到3月25日同步时（4月1日已在7天范围内）
    ↓
自然就拉到了
    ↓
4月1日客人来办入住，订单已经在本地
```

### 定时同步

| 配置 | 值 | 说明 |
|------|-----|------|
| 频率 | 每 5-10 分钟 | 云函数定时触发 |
| 范围 | **今天 ~ 今天+7天** | 只拉即将入住的订单 |
| 去重 | 按 orderId | 已存在则更新 |

### API 参数（经测试验证）

| 参数 | 有效 | 说明 |
|------|------|------|
| `start_date` / `end_date` | ✅ | 按入住日期筛选 |
| `page` / `page_size` | ✅ | 分页 |
| `staying_status` | ✅ | 入住状态筛选 |
| 按创建时间筛选 | ❌ | **不支持** |

### 同步逻辑

```
1. 请求百居易订单：start_date=今天，end_date=今天+7天
2. 遍历每条订单：
   - 从 reservations[0] 提取入住信息
   - 从 client 提取客人信息
   - 格式化手机号
   - 按 orderId 查本地：
     - 不存在 → 插入
     - 存在 → 对比 update_time，有变化则更新
3. 输出统计：新增 X，更新 Y
```

### 实时验证（防止取消/修改不同步）

本地同步只是为了**快速展示**候选订单，最终办理入住时**实时验证**百居易：

```
客人输入手机号
    ↓
本地查询 → 快速展示候选订单列表
    ↓
客人选择订单，点击"办理入住"
    ↓
后端调用百居易 API，用 orderId 查询订单详情
    ↓
订单存在且 status=accepted → 继续入住流程
订单不存在或已取消 → "订单无效，请联系房东"
```

**优点**：
- 取消/修改的订单不怕漏同步
- 最终以百居易实时数据为准
- 本地数据只是缓存，加速查询

### 漏单兜底

如果客人查不到订单（比如远期预订还没同步到），提供手动录入入口：

```
客人输入手机号
    ↓
本地查询
    ↓
找到 → 正常入住流程
找不到 → "未找到订单，请联系房东" 或 手动录入
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

## 当前进度

| 任务 | 状态 | 说明 |
|------|------|------|
| 实时查询 Hostex 订单 | ✅ 完成 | 通过 `GET /api/orders` 实时从 PMS 拉取 |
| 订单数据含来源信息 | ✅ 完成 | 返回 ota（OTA: meituan/ctrip/douyin/manual）、pms("hostex")、pmsRoomId |
| 测试手机号 mock 订单 | ✅ 完成 | 15290500792 返回 mock 订单 |
| 本地同步服务 | ⬜ 暂不需要 | 当前实时查询够用 |
| Session 过期监控 | ⬜ 待开发 | |
