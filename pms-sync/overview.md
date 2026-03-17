# PMS 订单同步服务

> 路径：`/pms-sync`
> 技术栈：Node.js + TypeScript + 腾讯云函数 + 云开发数据库
>
> **状态：暂不使用** - 第一版采用纯实时查询，不做本地同步。本项目保留供后续参考。

---

## 概述

独立微服务，定时从百居易 (Hostex) 获取订单数据，同步到本地数据库。

### 为什么独立服务？

| 对比 | 放在主服务 | 独立服务 |
|------|----------|---------|
| 职责 | 混杂用户请求和同步任务 | 职责单一 |
| 部署 | 必须 24 小时运行 | 云函数按需触发，几乎免费 |
| 风险 | 同步出问题影响用户 | 隔离，互不影响 |

---

## 架构

```
┌─────────────────┐     每 5-10 分钟       ┌─────────────────┐
│   腾讯云函数     │ ◀───────────────────  │   定时触发器     │
│   (pms-sync)    │                        └─────────────────┘
└────────┬────────┘
         │
         │ 1. HTTP + Cookie
         ▼
┌─────────────────┐
│   百居易 API     │
│  myhostex.com   │
└────────┬────────┘
         │
         │ 2. 订单列表 JSON
         ▼
┌─────────────────┐
│   同步服务       │  3. 转换格式 + 去重
└────────┬────────┘
         │
         │ 4. upsert
         ▼
┌─────────────────┐
│  CloudBase 数据库│
│   orders 集合    │
└─────────────────┘
```

---

## 认证方式

百居易使用 Cookie + Session（不是 JWT），不用官方 API（8000 元/年）。

| Cookie | 说明 | 有效期 |
|--------|------|--------|
| `hostex_session` | 会话令牌 | 约 1 年 |
| `operator_id` | 房东 ID | 长期 |

**获取方式**：
1. 浏览器登录 https://www.myhostex.com
2. F12 → Application → Cookies
3. 复制两个值到环境变量

---

## API

### 获取订单列表

```
GET https://www.myhostex.com/api/reservation_order/list
```

**请求**：

```javascript
fetch("https://www.myhostex.com/api/reservation_order/list?page=1&page_size=100&start_date=2026-03-17&end_date=2026-03-24&opid=120002&opclient=Web-Linux-Chrome", {
  headers: {
    "accept": "application/json",
    "cookie": "hostex_session=xxx; operator_id=120002"
  }
})
```

**响应**（嵌套结构，入住信息在 reservations 数组）：

```json
{
  "error_code": 0,
  "error_msg": "操作成功",
  "data": {
    "list": [
      {
        "code": "22-1092253481431381639-ib96sakgw5",
        "status": "accepted",
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
              "title": "301 轻旅｜投影大床房"
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

| 百居易字段 | 本地字段 | 说明 |
|-----------|---------|------|
| uniq_code | orderId | 订单唯一编码 |
| room_name | roomNumber | 房间名称 |
| check_in | checkInDate | 入住日期 |
| check_out | checkOutDate | 离店日期 |
| staying_status | status | 入住状态 |
| lock_password | lockPassword | 门锁密码 |
| guests[0].phone | phone | 客人手机号（需格式化） |
| guests[0].full_name | guestName | 客人姓名 |

### 手机号格式化

```typescript
// 百居易：+86 138 0013 8000
// 本地：  13800138000

function normalizePhone(phone: string): string {
  return phone.replace(/[\s\+\-]/g, '').replace(/^86/, '')
}
```

---

## 目录结构

```
/pms-sync
├── src/
│   ├── index.ts        # 入口（云函数 main 导出）
│   ├── hostex.ts       # Hostex API 请求
│   ├── db.ts           # 数据库操作
│   └── types.ts        # 类型定义
├── package.json
├── tsconfig.json
├── .env.example
└── .gitignore
```

---

## 环境变量

```bash
# 百居易 Session
HOSTEX_SESSION=g63nTvEWOpevc3LMVxR4LPbEdNXwVqLraLrrYTTS
HOSTEX_OPERATOR_ID=120002

# 云开发数据库
TCB_ENV_ID=xiaoer-dev-3g5g7pw9482967c9
TCB_SECRET_ID=xxx
TCB_SECRET_KEY=xxx
```

---

## 同步策略

### 核心思路

业务场景是**办理入住**，只需同步**即将入住**的订单。客人什么时候下单不重要，入住日期在范围内就会被同步到。

| 配置 | 值 | 说明 |
|------|-----|------|
| 频率 | 每 5-10 分钟 | 云函数定时触发 |
| 范围 | **今天 ~ 今天+7天** | 只拉即将入住的订单 |
| 去重 | 按 orderId | 存在则更新（包括状态变化） |

### API 参数（经测试验证）

| 参数 | 有效 | 说明 |
|------|------|------|
| `start_date` / `end_date` | ✅ | 按入住日期筛选 |
| `page` / `page_size` | ✅ | 分页 |
| 按创建时间筛选 | ❌ | **不支持** |

### 同步逻辑

```
1. 请求：start_date=今天，end_date=今天+7天
2. 遍历每条订单：
   - 从 reservations[0] 提取入住信息
   - 从 client 提取客人信息
   - 格式化手机号
   - 按 orderId 查本地：
     - 不存在 → 插入
     - 存在 + update_time 变化 → 更新（包括取消状态）
3. 输出统计：新增 X，更新 Y
```

### 实时验证

本地同步只是缓存，加速查询。办理入住时**实时验证**百居易：

```
本地查询 → 展示候选订单
    ↓
用户点击"办理入住"
    ↓
后端用 orderId 实时查百居易
    ↓
存在且有效 → 继续
不存在/已取消 → 拒绝
```

这样取消/修改的订单不怕漏同步。

### 漏单兜底

远期订单在入住前会被同步到。万一漏了，提供手动录入入口。

---

## 运行方式

### 本地开发

```bash
cd pms-sync
pnpm install
cp .env.example .env  # 填写环境变量
pnpm sync             # 运行一次同步
```

### 云函数部署

```bash
pnpm build
# 上传 dist/ 到腾讯云函数
# 配置定时触发器：0 */5 * * * *（每5分钟）
```

---

## 风险与监控

| 风险 | 应对 |
|------|------|
| Session 过期 | 有效期约 1 年，过期重新登录获取 |
| 被封禁 | 百居易是小公司（400万 Pre-A），5-10 分钟间隔风险低 |
| 同步失败 | 记录日志，后续加告警通知 |

---

## 开发状态

| 任务 | 状态 |
|------|------|
| 项目初始化 | ⬜ 待开发 |
| Hostex 请求逻辑 | ⬜ 待开发 |
| 数据库写入 | ⬜ 待开发 |
| 云函数部署 | ⬜ 待开发 |
| 定时触发器 | ⬜ 待开发 |
