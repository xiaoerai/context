# 老板端入住记录优化

> 状态：✅ 已完成

---

## 技术选型

| 类别 | 选择 | 理由 |
|------|------|------|
| 图片预览 | antd Image 组件 | 自带缩略图 + 点击放大（preview），无需额外依赖；antd 已在项目中使用 |
| 响应式方案 | antd Grid.useBreakpoint | 现有页面已使用此方案，保持一致 |
| 新增依赖 | 无 | antd Image + Image.PreviewGroup 满足所有需求 |

不需要引入 lightbox 等第三方图片预览库，antd 的 Image 组件已经足够。

---

## 当前进度

### 前端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住记录页（改造） | ✅ 已完成 | `admin/src/pages/Checkins.tsx` |

### 后端

| 模块 | 状态 | 文件 |
|------|------|------|
| 入住记录接口关联 guests | ✅ 已完成 | `server/src/controllers/admin.controller.ts` |

### API

| 接口 | 方法 | 状态 | 说明 |
|------|------|------|------|
| `/api/admin/checkins` | GET | ✅ 已完成 | 返回数据增加 guests 字段 |

---

## 概述

### 业务场景

老板端入住记录页主要用于**应付公安检查**。公安检查时需要出示每位住客的身份信息（姓名、身份证号、身份证照片）。

当前页面只显示房间号、手机号、日期、状态，**缺少住客身份信息**，无法满足公安检查要求。

### 改造范围

- 后端：`GET /api/admin/checkins` 增加 guests 关联查询
- 前端：`Checkins.tsx` 展示住客信息 + 身份证照片预览

---

## 数据流设计

```
前端 Checkins.tsx
  │
  │── GET /api/admin/checkins?status=xxx&page=1
  │
  ▼
后端 admin.controller.ts → getCheckins()
  │
  │── 1. findAllCheckInRecords() 查入住记录列表
  │── 2. 收集所有记录的 guestIds，去重合并
  │── 3. findGuestsByIds() 批量查 guests 表（一次查询）
  │── 4. 按 guestIds 映射，将 guests 挂到每条记录上
  │
  ▼
返回 { list: [..., guests: [{ name, idNumber, idImageUrl }]], total, page, pageSize }
  │
  ▼
前端渲染
  ├── 桌面端：卡片内直接展示住客列表
  └── 移动端：列表项下方展示住客信息
```

### 关键设计决策：批量查询而非逐条关联

一页最多 50 条记录，每条记录可能有 1-3 个住客。如果逐条查 guests，最坏情况需要 50 次数据库查询。

采用批量查询方案：
1. 从所有记录中收集 guestIds，去重
2. 一次 `findGuestsByIds()` 查出所有住客（用 `_.in(ids)`）
3. 在内存中映射回各条记录

这样无论多少条记录，guests 表只查一次。

---

## API 接口

### GET /api/admin/checkins（改造）

**改动**：响应数据的每条记录增加 `guests` 字段。

**请求参数**：不变（status / startDate / endDate / page / pageSize）

**响应**（变更部分加粗标注）：

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
        "ota": "meituan",
        "guestIds": ["g1", "g2"],
        "guests": [
          {
            "_id": "g1",
            "name": "张三",
            "idNumber": "110101199001011234",
            "idImageUrl": "https://xxx/id-photo-1.jpg"
          },
          {
            "_id": "g2",
            "name": "李四",
            "idNumber": "110101199002022345",
            "idImageUrl": null
          }
        ]
      }
    ],
    "total": 30,
    "page": 1,
    "pageSize": 20
  }
}
```

**guests 字段说明**：
- 类型：`Guest[]`，按 guestIds 顺序排列
- 如果 guestIds 为空数组，guests 也为空数组
- 如果某个 guestId 在 guests 表中找不到（脏数据），跳过该条
- `idImageUrl` 可能为 null（用户未上传身份证照片）

**后端实现伪代码**：

```typescript
// admin.controller.ts → getCheckins()
const [list, total] = await Promise.all([
  findAllCheckInRecords({ status, startDate, endDate, page, pageSize }),
  countCheckInRecords({ status, startDate, endDate }),
])

// 批量查询 guests
const allGuestIds = [...new Set(list.flatMap(r => r.guestIds || []))]
const allGuests = await findGuestsByIds(allGuestIds)
const guestMap = new Map(allGuests.map(g => [g._id, g]))

// 挂载到每条记录
const enriched = list.map(record => ({
  ...record,
  guests: (record.guestIds || [])
    .map(id => guestMap.get(id))
    .filter(Boolean)
    .map(g => ({
      _id: g._id,
      name: g.name,
      idNumber: g.idNumber,
      idImageUrl: g.idImageUrl || null,
    })),
}))

res.json({ success: true, data: { list: enriched, total, page, pageSize } })
```

---

## 页面规格

### 入住记录页（Checkins.tsx 改造）

**业务目的**：展示入住记录及住客身份信息，满足公安检查要求。

**展示字段**：

| 字段 | 来源 | 说明 |
|------|------|------|
| 房间号 | check_in_records.roomNumber | 粗体，18px |
| 房间名 | check_in_records.roomName | 灰色辅助文字 |
| 入住状态 | check_in_records.status | Tag 颜色区分（同现有逻辑） |
| 手机号 | check_in_records.phone | UserOutlined 图标前缀 |
| 入住日期 | check_in_records.checkInDate | CalendarOutlined 图标前缀 |
| 退房日期 | check_in_records.checkOutDate | 与入住日期用 ~ 连接 |
| OTA 来源 | check_in_records.ota | Tag 展示，无则不显示 |
| 押金状态 | check_in_records.depositPaid | 绿色"已付" / 红色"未付" |
| 住客姓名 | guests.name（通过 guestIds） | 每位住客一行，IdcardOutlined 图标前缀 |
| 身份证号 | guests.idNumber（通过 guestIds） | 紧跟姓名后面 |
| 身份证照片 | guests.idImageUrl（通过 guestIds） | 缩略图 40x40，点击放大；无照片显示灰色占位 |

**操作按钮**：无（纯展示页面）

**筛选/搜索**：
- 按状态筛选（全部/待入住/已入住/待退房/已退房）——保留现有逻辑不变

**空状态**：显示"暂无入住记录"（Empty 组件，同现有）

#### 布局

**桌面端（md 及以上）**：

卡片网格布局（Row + Col），每张卡片结构：

```
┌─────────────────────────────────────┐
│ 301  301 豪华大床房        [已入住]  │  ← Card title + extra
├─────────────────────────────────────┤
│ 📱 138xxxx                          │
│ 📅 2026-03-09 ~ 2026-03-11         │
│ [meituan]              [已付押金]   │
│─────────────────────────────────────│
│ 住客信息                            │  ← Divider
│ 🪪 张三  110101xxxx1234  [照片]     │  ← 每位住客一行
│ 🪪 李四  110101xxxx2345  [照片]     │
└─────────────────────────────────────┘
```

- 住客区域用 antd Divider 分隔，标题"住客信息"
- 身份证照片用 antd Image 组件，width=40，fallback 占位图
- 多张照片包在 Image.PreviewGroup 中，支持左右翻页浏览
- 没有住客信息时显示灰色文字"暂无住客信息"

**移动端（md 以下）**：

List 布局，每个 List.Item 结构：

```
┌─────────────────────────────────────┐
│ 301  301 豪华大床房  [已入住]        │
│ 📱 138xxxx                          │
│ 📅 2026-03-09 ~ 2026-03-11         │
│                                     │
│ 🪪 张三  110101xxxx1234  [照片]     │
│ 🪪 李四  110101xxxx2345  [照片]     │
│                    [meituan] [已付]  │
└─────────────────────────────────────┘
```

- 住客信息直接展示在描述区域内（不需要折叠/展开）
- 身份证照片同样用 Image 组件，width=36

#### 身份证照片交互

1. 缩略图：40x40px 圆角方形，object-fit: cover
2. 点击放大：使用 antd Image 内置 preview 功能，自动弹出遮罩层全屏查看
3. 多位住客照片：包在 `<Image.PreviewGroup>` 中，支持左右切换
4. 无照片时：显示灰色占位方块，内部显示 antd `FileImageOutlined` 图标
5. 照片加载失败：使用 Image 的 `fallback` 属性显示占位图

---

## 数据模型

### 无 schema 变更

本次改造不涉及数据库 schema 变更。所有字段均已存在：

- `check_in_records.guestIds: string[]` — 已有
- `guests.name: string` — 已有
- `guests.idNumber: string` — 已有
- `guests.idImageUrl?: string` — 已有

### 已有可复用代码

| 代码 | 文件 | 说明 |
|------|------|------|
| `findGuestsByIds(ids)` | `server/src/db/guests.ts` | 已实现，用 `_.in(ids)` 批量查询，直接可用 |
| `findAllCheckInRecords()` | `server/src/db/check_in_records.ts` | 已实现，返回含 guestIds 的记录 |
| `getCheckins()` | `admin/src/api/admin.ts` | 已实现，前端 API 调用层不需要改动（返回类型用 unknown[]） |

---

## 风险评估

### 性能风险

| 风险 | 影响 | 规避方案 |
|------|------|----------|
| guestIds 总量过大导致 `_.in()` 查询慢 | 低。每条记录 1-3 个住客，50 条记录最多 150 个 ID | 腾讯云 `_.in()` 对数组大小无特别限制，150 个 ID 不构成问题 |
| 身份证照片加载慢 | 中。列表页同时加载多张图片 | Image 组件默认 lazy load；缩略图尺寸小（40x40）；照片存在腾讯云存储有 CDN |

### 安全风险

| 风险 | 影响 | 规避方案 |
|------|------|----------|
| 身份证信息泄露 | 高。身份证号是敏感数据 | 接口已有 adminOnly 中间件保护；前端身份证号可考虑脱敏显示（如 110101****1234），但公安检查需要完整号码，暂不脱敏 |
| 身份证照片 URL 直接暴露 | 中。URL 如果可猜测则可能被外部访问 | 照片存储在腾讯云 CloudBase，URL 含随机 token，不可猜测 |

### 技术风险

| 风险 | 影响 | 规避方案 |
|------|------|----------|
| 部分记录 guestIds 为空或 undefined | 低。早期记录可能没有住客 | 代码中做防御处理：`record.guestIds \|\| []` |
| guestId 对应的 guest 已被删除 | 低。正常不会删除 guest | `.filter(Boolean)` 过滤掉找不到的 guest |

---

## 待办事项

按优先级排列：

1. **后端改造**：`GET /api/admin/checkins` 关联 guests 表返回住客信息
   - 文件：`server/src/controllers/admin.controller.ts`
   - 改动：import `findGuestsByIds`，在 getCheckins 中批量查询并挂载
   - 预计改动量：~15 行

2. **前端改造**：Checkins.tsx 展示住客姓名 + 身份证号 + 照片
   - 文件：`admin/src/pages/Checkins.tsx`
   - 改动：CheckinRecord 接口增加 guests 字段；桌面卡片和移动列表增加住客区域
   - 新增 import：`Image, Divider` from antd，`IdcardOutlined, FileImageOutlined` from @ant-design/icons
   - 预计改动量：~60 行
