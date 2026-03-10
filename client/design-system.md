# 民宿小程序设计规范

## 设计风格

**Dark Luxury（深色奢华）**

- 深邃黑色背景，营造高端酒店氛围
- 金色点缀，提升品质感
- 简洁留白，突出内容
- 细腻质感，注重细节

---

## 颜色系统

### 背景色
| 变量 | 色值 | 用途 |
|------|------|------|
| `$bg-dark` | `#0a0a0a` | 主背景 |
| `$bg-card` | `#1a1816` | 深色卡片 |
| `$bg-card-light` | `#2a2724` | 浅色卡片 |
| `$bg-overlay` | `rgba(10,10,10,0.95)` | 底部栏/遮罩 |

### 主题色（金色）
| 变量 | 色值 | 用途 |
|------|------|------|
| `$accent` | `#a08c6a` | 主金色，按钮/图标/高亮 |
| `$accent-light` | `#c9b896` | 浅金色 |

### 文字色
| 变量 | 色值 | 用途 |
|------|------|------|
| `$text-primary` | `#ffffff` | 主文字 |
| `$text-secondary` | `rgba(255,255,255,0.7)` | 次要文字 |
| `$text-muted` | `rgba(255,255,255,0.5)` | 弱化文字 |

### 边框色
| 变量 | 色值 | 用途 |
|------|------|------|
| `$border-color` | `#2a2520` | 深色卡片边框 |
| `$border-light` | `#4a443b` | 浅色卡片边框 |
| `$border-subtle` | `rgba(255,255,255,0.1)` | 微弱边框 |

---

## 字体系统

### 字号层级
| 用途 | 字号 | 变量 |
|------|------|------|
| 大标题 Hero | 36px | `$font-size-3xl` |
| 标题 | 24px | `$font-size-2xl` |
| 房间名/卡片标题 | 18px | `$font-size-xl` |
| 正文/按钮 | 14px | `$font-size-lg` |
| 描述文字 | 13px | `$font-size-md` |
| 副标题 | 12px | `$font-size-base` |
| 标签 | 10px | `$font-size-sm` |
| 极小英文 | 8px | `$font-size-xs` |

### 字重
| 用途 | 字重 |
|------|------|
| 大标题 | 200 (极细) |
| 正文 | 300 (细) |
| 按钮/强调 | 500 (中等) |

### 字间距
- 英文标签：`letter-spacing: 2px`
- Welcome 文字：`letter-spacing: 3px`

---

## 组件规范

### 按钮

**主按钮（金色）**
```scss
background: $accent;
color: $bg-dark;
height: 44px;
border-radius: 12px;
font-weight: 500;
```

**次要按钮（透明边框）**
```scss
background: rgba(255, 255, 255, 0.05);
border: 1px solid rgba(255, 255, 255, 0.1);
color: $text-primary;
height: 44px;
border-radius: 12px;
```

### 卡片

**深色卡片（主信息卡）**
```scss
background: linear-gradient(135deg, #1a1816 0%, #141210 100%);
border: 1px solid #2a2520;
border-radius: 16px;
padding: 20px;
```

**浅色卡片（功能入口）**
```scss
background: #2a2724;
border: 1px solid #4a443b;
border-radius: 16px;
padding: 20px;
```

### 输入框
```scss
background: rgba(255, 255, 255, 0.05);
border: 1px solid rgba(255, 255, 255, 0.1);
border-radius: 12px;
height: 48px;
padding: 0 16px;
color: #ffffff;

&:focus {
  border-color: $accent;
}

&::placeholder {
  color: rgba(255, 255, 255, 0.5);
}
```

### 上传区域
```scss
background: rgba(255, 255, 255, 0.03);
border: 1px dashed #4a443b;
border-radius: 16px;

&:active {
  border-color: $accent;
}
```

### 徽章/标签
```scss
background: $accent;
color: $bg-dark;
font-size: 10px;
letter-spacing: 1px;
padding: 4px 12px;
border-radius: 16px;
```

### 分隔线
```scss
// 水平分隔线
height: 1px;
background: rgba(255, 255, 255, 0.25);

// 垂直分隔线
width: 1px;
background: rgba(255, 255, 255, 0.2);
```

---

## 间距系统

| 变量 | 值 | 用途 |
|------|------|------|
| `$spacing-xs` | 4px | 紧凑间距 |
| `$spacing-sm` | 8px | 小间距 |
| `$spacing-md` | 12px | 中间距 |
| `$spacing-lg` | 16px | 大间距/页面边距 |
| `$spacing-xl` | 20px | 卡片内边距 |
| `$spacing-2xl` | 24px | 区块间距 |
| `$spacing-3xl` | 32px | 大区块间距 |

---

## 布局规范

### 页面结构
```
┌─────────────────────────┐
│      Hero (200px)       │
├─────────────────────────┤
│                         │
│     Main Content        │
│     padding: 0 16px     │
│                         │
├─────────────────────────┤
│   TabBar (固定底部)      │
│   padding-bottom: 80px  │
└─────────────────────────┘
```

### 底部固定按钮
```scss
position: fixed;
bottom: 0;
left: 0;
right: 0;
background: rgba(10, 10, 10, 0.95);
backdrop-filter: blur(20px);
border-top: 1px solid rgba(255, 255, 255, 0.15);
padding: 16px;
padding-bottom: calc(16px + env(safe-area-inset-bottom));
```

---

## 图标规范

- 使用 Lucide 风格线性图标
- 默认颜色：当前文字色 `currentColor`
- 强调颜色：金色 `$accent`
- 常用尺寸：20px / 22px / 24px

---

## 动效规范

- 过渡时间：0.2s
- 按钮点击：`:active` 状态背景加深
- 卡片点击：边框变金色

---

## 使用方式

在 SCSS 文件中引入：

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
