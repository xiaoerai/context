# 民宿小程序开发规范

## 目录结构

```
client/src/
├── app.ts                 # 应用入口
├── app.config.ts          # 路由配置
├── app.scss               # 全局样式
│
├── pages/                 # 页面目录
│   ├── index/             # 首页
│   │   ├── index.tsx      # 页面组件
│   │   ├── index.scss     # 页面样式
│   │   ├── index.config.ts # 页面配置
│   │   └── components/    # 页面私有组件
│   │       ├── Hero/
│   │       ├── StayCard/
│   │       └── FeatureGrid/
│   │
│   ├── checkin/           # 入住信息页
│   ├── deposit/           # 押金支付页
│   ├── success/           # 入住成功页
│   └── checkout/          # 退房申请页
│
├── components/            # 公共组件
│   ├── Icon/              # 图标组件
│   ├── TabBar/            # 底部导航
│   ├── NavBar/            # 顶部导航
│   └── Button/            # 按钮组件
│
├── stores/                # 状态管理 (Zustand)
│   └── useAppStore.ts
│
├── hooks/                 # 自定义 Hooks
│   └── useNavigate.ts
│
├── config/                # 配置文件
│   └── features.ts
│
├── locales/               # 国际化
│   ├── index.ts
│   ├── zh.ts
│   └── en.ts
│
├── styles/                # 全局样式系统
│   ├── _variables.scss    # 变量定义
│   └── _mixins.scss       # 混入函数
│
├── utils/                 # 工具函数
│   └── request.ts         # API 请求
│
└── types/                 # TypeScript 类型
    └── index.ts
```

---

## 组件拆分原则

### 1. 页面私有组件 vs 公共组件

| 类型 | 位置 | 使用场景 |
|------|------|----------|
| 页面私有组件 | `pages/xxx/components/` | 只在当前页面使用 |
| 公共组件 | `components/` | 多个页面复用 |

**判断标准：**
- 只在一个页面用 → 页面私有组件
- 两个以上页面用 → 提取为公共组件

### 2. 何时拆分组件

**需要拆分：**
- 代码超过 100 行
- 有独立的逻辑和状态
- 可复用的 UI 块
- 需要独立样式文件

**不需要拆分：**
- 简单的 UI 片段（< 30 行）
- 无独立逻辑
- 一次性使用

### 3. 组件文件结构

```
ComponentName/
├── index.tsx      # 组件代码
└── index.scss     # 组件样式
```

---

## 页面结构模板

### 基础页面

```tsx
import { useState } from 'react'
import { useTranslation } from 'react-i18next'
import { useAppStore } from '../../stores/useAppStore'
import { useNavigate } from '../../hooks/useNavigate'
import './index.scss'

function PageName() {
  const { t } = useTranslation()
  const { navigateTo } = useNavigate()

  return (
    <div className="page">
      {/* 页面内容 */}
    </div>
  )
}

export default PageName
```

### 页面样式模板

```scss
@use '../../styles/variables' as *;

.page {
  min-height: 100vh;
  background: $bg-dark;
  color: $text-primary;
  padding-bottom: 80px; // 为 TabBar 留空间
}
```

---

## 组件编写规范

### 1. Props 接口定义

```tsx
interface ComponentProps {
  title: string
  onClick?: () => void
  children?: React.ReactNode
}

export default function Component({ title, onClick }: ComponentProps) {
  // ...
}
```

### 2. 导出接口（如需外部引用）

```tsx
// 导出接口供外部使用
export interface Feature {
  id: string
  icon: string
}

export default function FeatureGrid({ features }: { features: Feature[] }) {
  // ...
}
```

### 3. 使用 i18n

```tsx
import { useTranslation } from 'react-i18next'

function Component() {
  const { t } = useTranslation()

  return <span>{t('home.title')}</span>
}
```

---

## 样式编写规范

### 1. 引入变量

```scss
@use '../../styles/variables' as *;
// 或
@use '../../../../styles/variables' as *; // 深层组件
```

### 2. 使用混入

```scss
@use '../../styles/variables' as *;
@use '../../styles/mixins' as *;

.my-card {
  @include card-dark;
}

.my-button {
  @include btn-primary;
}
```

### 3. 样式命名

- 使用 BEM 风格或简洁的嵌套
- 页面根元素用 `.page`
- 组件根元素用组件名（如 `.stay-card`）

```scss
.stay-card {
  // 根元素样式

  .stay-header {
    // 子元素
  }

  .stay-title {
    // 子元素
  }
}
```

---

## 状态管理规范

### Zustand Store 结构

```ts
import { create } from 'zustand'

interface AppState {
  // 状态
  data: DataType | null

  // 方法
  setData: (data: DataType) => void
}

export const useAppStore = create<AppState>((set) => ({
  data: null,
  setData: (data) => set({ data }),
}))
```

### 使用方式

```tsx
function Component() {
  // 获取状态和方法
  const { data, setData } = useAppStore()

  // 只获取部分状态（性能优化）
  const data = useAppStore((state) => state.data)
}
```

---

## 路由配置

### 添加新页面

1. 创建页面目录和文件
2. 在 `app.config.ts` 添加路由

```ts
export default defineAppConfig({
  pages: [
    'pages/index/index',
    'pages/checkin/index',  // 新页面
  ],
})
```

3. 在 `useNavigate.ts` 添加路由映射

```ts
const routes = {
  home: '/pages/index/index',
  checkin: '/pages/checkin/index',  // 新路由
}
```

---

## i18n 国际化

### 添加翻译

1. 在 `locales/zh.ts` 和 `locales/en.ts` 添加翻译

```ts
// zh.ts
export default {
  checkin: {
    title: '入住信息',
    submit: '提交',
  },
}
```

2. 在组件中使用

```tsx
const { t } = useTranslation()
<span>{t('checkin.title')}</span>
```

---

## 命名规范

| 类型 | 规范 | 示例 |
|------|------|------|
| 文件夹 | 小写 + 短横线 | `stay-card` 或 `StayCard` |
| 组件 | PascalCase | `StayCard` |
| 函数 | camelCase | `handleClick` |
| 变量 | camelCase | `currentStay` |
| 常量 | UPPER_SNAKE | `MAX_COUNT` |
| CSS 类 | kebab-case | `.stay-card` |
| SCSS 变量 | $kebab-case | `$bg-dark` |

---

## Git 提交规范

参考 `commit-convention.md`

```
feat: 添加入住信息页
fix: 修复表单验证问题
style: 调整按钮颜色
refactor: 重构状态管理
```
