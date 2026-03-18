# 客人端小程序

> 路径：`/client`
> 技术栈：Taro 4 + React 18 + TypeScript + Zustand + Vite

---

## 页面规划

| 页面 | 路径 | 状态 | 说明 |
|------|------|------|------|
| 首页 | `pages/index` | ✅ 完成 | 深色奢华风格，功能入口网格 |
| 入住信息 | `pages/checkin` | ✅ 完成 | 身份证OCR+表单填写 |
| 退房申请 | `pages/checkout` | ⬜ 待开发 | 申请退房 |

> 押金支付和入住成功不再是独立页面，改为首页统一弹窗的 step

---

## 核心架构

### 统一弹窗（useCheckinFlow）

所有入住相关的弹窗交互在一个 hook 里管理，首页只用一个 `<Modal>` 组件：

```typescript
const flow = useCheckinFlow(onNavigate)

<Modal visible={flow.visible} title={flow.title} contentKey={flow.contentKey} ...>
  {flow.content}
</Modal>
```

**Step 流转**：
```
phone → orders → loading → deposit → success
                    ↓
              跳转 checkin 页面（无入住记录时）
```

### 全局状态（Zustand Slice 模式）

```
stores/
  useAppStore.ts          # 合并所有 slice
  slices/
    userSlice.ts          # userPhone
    orderSlice.ts         # orders, selectedOrder
    checkinSlice.ts       # checkinRecord
    hotelSlice.ts         # hotelConfig, currentStay
    languageSlice.ts      # language
```

持久化字段：`userPhone`、`selectedOrder`、`language`

### Modal 组件

通用弹窗组件，支持：
- 进入/退出动画（淡入 + 缩放）
- 内容切换过渡（contentKey 变化时淡入 + 上滑）
- 父组件通过 props 控制 title、children、headerRight

---

## 用户流程

> 无需微信授权登录，用户通过"订单手机号 + 短信验证码"验证身份

```
扫码进入小程序
  ↓
首页（展示民宿信息）
  ↓
点击「办理入住」
  ↓
┌── 统一弹窗 ──────────────────────────┐
│  手机号验证 → 选择订单                 │
│      ↓ 无入住记录         有记录未付款  │
│  关弹窗→checkin页        显示押金弹窗   │
│  填表单→提交→返回              ↓       │
│      ↓                   支付成功      │
│  自动弹押金弹窗                ↓       │
│      ↓                   显示成功弹窗   │
│  支付成功→成功弹窗                     │
└──────────────────────────────────────┘
  ↓
首页显示门锁密码、WiFi（待做）
```

---

## 关键文件

| 文件 | 说明 |
|------|------|
| `hooks/useCheckinFlow.tsx` | 入住流程 hook（统一弹窗逻辑） |
| `hooks/useOrderAuth.ts` | 订单路由保护（从 store 读取） |
| `hooks/useNavigate.ts` | 路由导航 |
| `components/Modal/` | 通用弹窗组件 |
| `components/Input/` | 通用输入框 |
| `api/deposit.ts` | 押金支付 API |
| `api/checkin.ts` | 入住 API |
| `api/orders.ts` | 订单 API |
| `stores/useAppStore.ts` | 全局状态（slice 模式） |

---

## 多平台编译

- **H5 / 微信**：main 分支，Vite 编译
- **支付宝**：feat/webpack-alipay 分支，Webpack5 编译（Taro + Vite 支付宝有 bug）

详见 [multi-platform.md](./multi-platform.md)

---

## 开发命令

```bash
# H5 开发（浏览器预览）
pnpm dev:h5

# 微信小程序开发
pnpm dev:weapp

# 构建支付宝小程序（需切到 feat/webpack-alipay 分支）
pnpm build:alipay
```

---

## 测试账号

手机号 `15290500792`，验证码随便填，返回两条模拟订单。
