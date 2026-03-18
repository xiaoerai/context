# 多平台编译

## 编译命令

| 平台 | 开发 | 构建 |
|-----|------|------|
| H5 | `pnpm dev:h5` | `pnpm build:h5` |
| 微信小程序 | `pnpm dev:weapp` | `pnpm build:weapp` |
| 支付宝小程序 | - | 见下方 |

## 支付宝编译

Taro 4.1.11 + Vite 编译支付宝有 bug，需要切换到 Webpack。

**解决方案**：使用独立分支 `feat/webpack-alipay`

```bash
# 切换到支付宝分支
git checkout feat/webpack-alipay

# 编译
pnpm build:alipay

# 用支付宝开发者工具打开 dist/ 目录
```

## 分支策略

| 分支 | 编译器 | 用途 |
|-----|-------|------|
| `main` | Vite | H5 / 微信开发（编译快） |
| `feat/webpack-alipay` | Webpack5 | 支付宝编译 |

## 已知问题

- Taro + Vite 编译支付宝报错：`TypeError: Cannot read properties of undefined (reading 'type')`
- 相关 issue: https://github.com/NervJS/taro/issues/16991

## 请求适配

`src/api/request.ts` 根据环境自动选择请求方式：

- H5：`Taro.request` → 直连后端
- 微信小程序：`wx.cloud.callContainer` → 云托管内网调用
- 支付宝小程序：`Taro.request` → 直连后端
