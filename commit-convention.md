# Git 提交规范

## Commit Message 格式

```
<type>: <description>
```

## Type 类型

| 类型 | 说明 |
|------|------|
| feat | 新功能 |
| fix | 修复 bug |
| chore | 构建/工具/依赖变动 |
| style | 样式调整（不影响逻辑） |
| refactor | 重构（非新功能、非修复） |
| docs | 文档更新 |
| test | 测试相关 |

## 示例

```
feat: 添加扫码入住功能
fix: 修复押金支付失败问题
chore: 升级 NutUI 版本
style: 调整入住表单布局
refactor: 重构订单查询逻辑
docs: 更新 README
```

## 注意事项

- 使用中文描述
- 一句话简洁说明改动
- 不加句号
- 不要添加机器人签名信息
