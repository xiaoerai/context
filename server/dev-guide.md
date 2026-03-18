# 开发指南

## MVC 职责划分

| 层 | 职责 | 示例 |
|---|---|---|
| **routes** | 定义路由映射 | `router.post('/login', login)` |
| **controllers** | 参数验证、调用 service、返回响应 | 验证 phone 格式，调用 loginWithSmsCode() |
| **services** | 核心业务逻辑 | 验证码校验、用户创建、JWT 签发 |
| **db** | 数据库操作 | CRUD 操作，不含业务逻辑 |

---

## 错误处理

### asyncHandler

Express 的 async 函数抛出的错误不会自动传给全局 error handler，需要用 `asyncHandler` 包装：

```typescript
// middleware/error.ts
export const asyncHandler = (fn: AsyncFn): RequestHandler => {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next)
  }
}
```

### Controller 写法

```typescript
// ❌ 不用这样写
export async function login(req, res, next) {
  try {
    // ...
  } catch (err) {
    next(err)
  }
}

// ✅ 用 asyncHandler 包装，直接 throw
export const login = asyncHandler(async (req, res) => {
  if (!phone) {
    throw Errors.badRequest('缺少 phone 参数')
  }
  // ...
  res.json({ success: true, data })
})
```

### Errors 快捷方法

```typescript
import { Errors } from '../middleware/error'

Errors.badRequest('xxx')   // 400
Errors.unauthorized('xxx') // 401
Errors.forbidden('xxx')    // 403
Errors.notFound('xxx')     // 404
Errors.conflict('xxx')     // 409
Errors.internal('xxx')     // 500
```
