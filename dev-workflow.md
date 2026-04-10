# AI 小二 — Claude Code 开发体系方案

## 一、现状分析

### 当前问题
- 每次新会话需要重新解释项目背景
- 开发流程串行，一步步讨论确认，效率低
- 手动部署，没有自动化
- 没有利用 Claude 的并行能力
- context 文档和代码状态经常不同步

### 当前已有
- MEMORY.md：少量跨会话记忆
- context/：项目文档（功能设计、开发规范）
- GitHub Actions：CI/CD（待修复）
- 后端部署在阿里云 FC
- 前端用 Taro 跨端（支付宝/微信/H5）

---

## 二、体系架构

```
┌─────────────────────────────────────────────────────┐
│                    CLAUDE.md                         │
│            项目指令 + @context 文档引用               │
│              每次会话自动加载                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐           │
│  │  Skills  │  │  Hooks  │  │   MCP    │           │
│  │ 可复用   │  │ 自动化  │  │ 外部连接  │           │
│  │ 工作流   │  │ 质量守护│  │ 数据库等  │           │
│  └─────────┘  └─────────┘  └──────────┘           │
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │           Sub-agents（子代理）            │       │
│  │  并行执行独立任务，worktree 隔离开发       │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
│  ┌─────────────────────────────────────────┐       │
│  │           Plan Mode（规划模式）           │       │
│  │  复杂功能先出方案，确认后一口气执行         │       │
│  └─────────────────────────────────────────┘       │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## 三、各层详细方案

### 1. CLAUDE.md — 项目大脑

放在项目根目录 `/Users/jojo/Documents/小二/CLAUDE.md`，每次会话自动加载。

**内容结构：**
```markdown
# AI 小二 — 酒店自助入住系统

## 项目架构
- 前端：Taro（React）跨端小程序，支付宝分支 feat/webpack-alipay
- 后端：Express + TypeScript，部署在阿里云 FC
- 数据库：腾讯云 CloudBase（NoSQL）
- 支付：支付宝直连（alipay.trade.create）
- PMS：百居易 Hostex

## 仓库结构
- server/：后端代码（GitHub: xiaoerai/server）
- client/：前端代码（GitHub: xiaoerai/client）
- context/：项目文档和功能设计

## 开发规范
详见 @context/commit-convention.md
详见 @context/server/dev-guide.md

## 常用命令
- 后端启动：cd server && pnpm dev
- 前端 H5：cd client && pnpm dev:h5
- 前端支付宝：cd client && pnpm build:alipay
- 部署后端：cd server && pnpm build && s deploy -y
- 同步房间：curl -X POST http://localhost:7001/api/rooms/sync

## 功能文档
- @context/features/auth.md — 登录系统
- @context/features/checkin-flow.md — 入住流程
- @context/features/deposit.md — 押金支付
- @context/features/checkout.md — 退房退款
- @context/server/database.md — 数据库设计

## 注意事项
- git commit 不加 Co-Authored-By
- 测试手机号 15290500792 跳过验证码
- 押金当前为 0.01 元（测试），正式为 500 元
- 前端支付宝相关改动只在 feat/webpack-alipay 分支
```

**要点：**
- 控制在 200 行以内
- 用 `@path` 引用 context 文档，不在 CLAUDE.md 里写详细内容
- 包含常用命令，省去每次查找

---

### 2. Skills — 可复用工作流

存放路径：`.claude/skills/` 或 `~/.claude/skills/`

#### Skill: /deploy — 一键部署后端

```yaml
# .claude/skills/deploy/SKILL.md
---
name: deploy
description: 构建并部署后端到阿里云 FC
disable-model-invocation: true
allowed-tools: Bash Read
---

执行以下步骤部署后端：

1. cd 到 server 目录
2. 运行 `pnpm build`，确认编译通过
3. 加载 .env 环境变量
4. 设置 ALIPAY_NOTIFY_URL
5. 运行 `s deploy -y`
6. 验证部署成功（curl health 接口）
7. 报告结果
```

#### Skill: /sync-rooms — 同步房间

```yaml
# .claude/skills/sync-rooms/SKILL.md
---
name: sync-rooms
description: 从 Hostex 同步房间数据到本地数据库
disable-model-invocation: true
allowed-tools: Bash
---

调用 POST /api/rooms/sync 接口同步房间数据。
先检查后端是否在运行，没运行则提示启动。
同步后显示所有房间列表和状态。
```

#### Skill: /test-flow — 测试完整入住流程

```yaml
# .claude/skills/test-flow/SKILL.md
---
name: test-flow
description: 端到端测试入住支付流程
disable-model-invocation: true
allowed-tools: Bash Read
---

使用 curl 模拟完整入住流程：
1. 登录（测试手机号）
2. 查询订单
3. 创建入住记录
4. 创建押金订单
5. 查询押金状态
6. 报告每步结果
```

#### Skill: /feature — 完整功能开发

```yaml
# .claude/skills/feature/SKILL.md
---
name: feature
description: 规划并实现一个完整功能
argument-hint: [功能描述]
---

按以下流程开发功能 $ARGUMENTS：

1. 读取相关 context 文档了解现状
2. 制定实现方案（列出要改的文件和改动内容）
3. 等用户确认方案
4. 按方案实现（后端 → 前端 → 文档）
5. 编译检查
6. 提交代码
7. 更新 context 文档
```

#### Skill: /update-docs — 同步文档

```yaml
# .claude/skills/update-docs/SKILL.md
---
name: update-docs
description: 根据代码现状更新 context 文档
allowed-tools: Read Grep Glob Write Edit
---

扫描后端代码，和 context/ 文档对比，找出不一致的地方并更新：
1. 检查 API 接口是否和文档一致
2. 检查数据库 schema 是否和文档一致
3. 检查功能状态标记是否正确
4. 更新不一致的文档
5. 提交
```

---

### 3. Hooks — 自动化守护

配置在 `.claude/settings.local.json`：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "FILE=$(echo $CLAUDE_TOOL_INPUT | jq -r '.file_path // .new_string // empty'); if [[ \"$FILE\" == *.ts ]]; then cd $(dirname \"$FILE\") && npx prettier --write \"$FILE\" 2>/dev/null; fi"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "if": "Bash(git push*)",
        "hooks": [
          {
            "type": "command",
            "command": "echo '请确认是否要推送到远端' >&2; exit 0"
          }
        ]
      }
    ]
  }
}
```

**作用：**
- 编辑 .ts 文件后自动 prettier 格式化
- git push 前提醒确认

---

### 4. MCP Server — 外部连接

配置在 `.claude/.mcp.json`：

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "$GH_TOKEN"
      }
    }
  }
}
```

**推荐接入：**

| MCP Server | 用途 | 优先级 |
|---|---|---|
| GitHub | PR/Issue 管理 | 高 |
| CloudBase | 直接查改数据库 | 高（需自己写） |
| Sentry | 线上错误监控 | 中（上线后） |

**自定义 CloudBase MCP Server（后续开发）：**
- 查询/修改数据库记录
- 查看房间状态
- 查看入住记录
- 不用每次写 curl 或 node 脚本

---

### 5. Sub-agents — 并行开发

创建专用子代理：

#### Agent: 后端开发

```yaml
# .claude/agents/backend-dev/AGENT.md
---
name: backend-dev
description: 后端功能开发，包括接口、服务层、数据库操作
model: opus
tools: Read Write Edit Grep Glob Bash
isolation: worktree
---

你是后端开发代理，负责：
- Express 路由和控制器
- 业务逻辑（services）
- 数据库操作（db）
- TypeScript 类型定义

开发完后运行 pnpm build 确认编译通过。
```

#### Agent: 前端开发

```yaml
# .claude/agents/frontend-dev/AGENT.md
---
name: frontend-dev
description: 前端功能开发，Taro React 小程序
model: opus
tools: Read Write Edit Grep Glob Bash
isolation: worktree
---

你是前端开发代理，负责：
- Taro React 组件
- 页面逻辑和状态管理（zustand）
- API 调用层
- 样式（SCSS）

开发完后运行 pnpm build:alipay 确认编译通过。
```

#### Agent: Code Reviewer

```yaml
# .claude/agents/reviewer/AGENT.md
---
name: reviewer
description: 代码审查，检查质量和安全问题
model: opus
tools: Read Grep Glob
---

审查代码变更，关注：
- 安全问题（SQL注入、XSS、敏感信息泄露）
- 错误处理是否完整
- 类型安全
- 边界情况
```

---

### 6. 开发工作流

#### 小功能（< 30 分钟）

```
你：描述需求
Claude：直接实现 → 编译 → 提交
```

#### 中等功能（30 分钟 - 2 小时）

```
你：描述需求
Claude：/plan 出方案
你：确认或调整
Claude：TodoWrite 拆任务 → 逐步实现 → 编译 → 提交 → 更新文档
```

#### 大功能（跨前后端）

```
你：描述需求
Claude：/plan 出方案
你：确认
Claude：
  ├── Agent A（worktree）→ 后端接口
  ├── Agent B（worktree）→ 前端页面
  └── 主线程 → 协调 + 文档更新
合并 → 联调 → 提交
```

---

## 四、实施计划

### 第一阶段：基础搭建（立即）
1. [ ] 创建 CLAUDE.md
2. [ ] 创建 /deploy skill
3. [ ] 修复 CI/CD（s deploy -y）

### 第二阶段：自动化（本周）
4. [ ] 配置 PostToolUse hook（自动格式化）
5. [ ] 创建 /sync-rooms、/test-flow skills
6. [ ] 接入 GitHub MCP Server

### 第三阶段：并行开发（下周）
7. [ ] 创建 backend-dev、frontend-dev 子代理
8. [ ] 创建 reviewer 子代理
9. [ ] 用 worktree 并行开发老板端

### 第四阶段：深度集成（后续）
10. [ ] 开发 CloudBase MCP Server
11. [ ] 创建 /feature 完整功能开发 skill
12. [ ] 配置更多 hooks（编译检查、文档同步提醒）

---

## 五、预期效果

| 指标 | 现在 | 优化后 |
|---|---|---|
| 新会话启动 | 5-10 分钟解释背景 | 0 分钟（CLAUDE.md 自动加载） |
| 部署 | 手动 3 步命令 | /deploy 一键 |
| 前后端并行 | 串行开发 | worktree 并行 |
| 代码格式化 | 手动或忘记 | hook 自动格式化 |
| 文档同步 | 经常忘记 | /update-docs 一键同步 |
| 数据库查询 | 写 curl/node 脚本 | MCP 直接查 |
