# AI 虚拟工程团队 — 多 Agent 自循环开发架构

## 一、核心理念

用 Claude Code 的 Agent 能力模拟一个完整的工程团队：

```
需求进来 → PM 拆解 → 架构师设计 → 开发者并行实现 → 视觉验证 → QA 自检 → 审查 → 部署
         ↑                                                       ↓
         └──────────────────── 发现问题，回到上游 ────────────────┘
```

整个流程尽量减少人工干预，只在关键节点（需求确认、合并审批）需要人。

---

## 二、团队角色

| 角色 | 职责 | 实现方式 | 状态 |
|---|---|---|---|
| PM（产品经理） | 接收需求、拆解任务、分配优先级 | Skill `/pm` | ✅ 已建 |
| Architect（架构师） | 设计方案、定义接口、评审设计 | Subagent（只读） | ✅ 已建 |
| Fullstack Dev | 按功能切片开发（前后端+文档） | Subagent + Worktree | ✅ 已建 |
| Visual QA | 浏览器视觉检查前端页面 | Subagent + chrome 工具 | ✅ 已建 |
| QA（质量保证） | 编译检查、自循环修复 | Stop Hook（prompt） | ✅ 已建 |
| Reviewer（代码审查） | 审查代码质量和安全 | Subagent（只读） | ✅ 已建 |
| DevOps | 部署、环境管理 | Skill `/deploy` | ✅ 已建 |

> 不拆前后端，按功能切片分配。每个 Dev Agent 负责一个完整功能的前后端+文档，减少协调成本。

---

## 三、架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    用户（你）                              │
│              只做两件事：提需求、审批合并                    │
└───────────────────────┬─────────────────────────────────┘
                        │ 需求
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  主 Agent（协调者）                        │
│                                                          │
│  读取 CLAUDE.md → 理解项目 → 调度下游 Agent               │
│                                                          │
│  ┌──────────┐                                           │
│  │ PM Skill │ 拆解需求 → 生成任务列表                     │
│  └──────────┘                                           │
│        │                                                 │
│        ▼                                                 │
│  ┌──────────────┐                                       │
│  │ Architect     │ 读代码 → 设计方案 → 定义接口            │
│  │ (Subagent)    │ 只读，不改代码                         │
│  └──────────────┘                                       │
│        │ 方案确认                                         │
│        ▼                                                 │
│  ┌───────────────┐  ┌───────────────┐                    │
│  │ Fullstack A   │  │ Fullstack B   │  ← 按功能切片并行   │
│  │ (功能切片1)    │  │ (功能切片2)    │  各自 worktree 隔离 │
│  └──────┬────────┘  └──────┬────────┘                    │
│         │ 合并回主分支       │                              │
│         ▼                  ▼                              │
│  ┌──────────────────────────────┐                        │
│  │      Visual QA（主 Agent 调用）│                        │
│  │  浏览器打开 H5 → 检查渲染效果  │                        │
│  │  有问题 → 返回开发者修复       │                        │
│  └──────────────────────────────┘                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────────────────────┐                        │
│  │      QA（Stop Hook 自动触发）  │                        │
│  │  判断是否改了代码 → 编译检查    │                        │
│  │  失败 → block → Claude 修复   │                        │
│  └──────────────────────────────┘                        │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                       │
│  │ Reviewer      │ 审查代码 → 发现问题 → 返回修复          │
│  │ (Subagent)    │ 通过 → 准备合并                       │
│  └──────────────┘                                       │
│         │                                                │
│         ▼                                                │
│  ┌──────────────┐                                       │
│  │ DevOps Skill │ /deploy → 自动部署                     │
│  └──────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

### 自循环机制

```
开发 → 视觉检查（前端）→ QA 编译检查
                           │
         ├── 编译失败 → 返回开发者，附上错误信息 → 修复 → 重新 QA
         │
         ├── 页面异常 → 返回开发者，附上问题描述 → 修复 → 重新验证
         │
         ├── Review 不通过 → 返回开发者，附上审查意见 → 修复 → 重新 QA
         │
         └── 全部通过 → 合并 → 部署
```

---

## 四、各组件详细定义

### 1. PM Skill — 需求拆解

**文件**：`.claude/skills/pm/SKILL.md`

用 `/pm <需求描述>` 调用。分析需求影响范围，按功能切片拆解任务，标注依赖关系和可并行项。只做分析，不写代码。

### 2. Architect Agent — 架构设计

**文件**：`.claude/agents/architect/AGENT.md`

只读代码不改代码。设计技术方案、定义 API 接口和数据库变更、识别风险点。

### 3. Fullstack Dev Agent — 全栈开发

**文件**：`.claude/agents/fullstack-dev/AGENT.md`

在 worktree 中隔离开发，负责一个完整功能切片的前后端代码和文档。完成后自行编译验证。

**注意**：前端视觉验证由主 agent 在 dev 完成后调用 visual-qa agent 执行，fullstack-dev 不操作浏览器。

### 4. Visual QA Agent — 视觉检查

**文件**：`.claude/agents/visual-qa/AGENT.md`

使用 claude-in-chrome 浏览器工具，打开 H5 页面（localhost:10086），检查渲染效果、控制台报错、交互行为。返回检查结果给主 agent。

**配套 Skill**：`/visual-check` 可手动触发视觉检查。

### 5. QA — Stop Hook 自检

**配置**：`.claude/settings.local.json` → `hooks.Stop`

使用 `prompt` 类型，由 Haiku 模型判断本次对话是否修改了代码：
- 没改代码 → 直接 approve，不浪费时间编译
- 改了 server/ → 跑 `pnpm build`
- 改了 client/ → 跑 `pnpm build:alipay`
- 编译失败 → block，Claude 继续修复

### 6. Reviewer Agent — 代码审查

**文件**：`.claude/agents/reviewer/AGENT.md`

只读代码，检查安全问题、代码质量、一致性、边界情况。输出按严重程度分级的审查报告。

### 7. Deploy Skill — 部署

**文件**：`.claude/skills/deploy/SKILL.md`

用 `/deploy` 调用。编译 → 加载环境变量 → s deploy -y → curl health 验证。

### 8. PostToolUse Hook — 自动格式化

**配置**：`.claude/settings.local.json` → `hooks.PostToolUse`

编辑 .ts/.tsx 文件后自动运行 prettier 格式化。

---

## 五、工作流模式

### 模式 A：小功能（你只说一句话）

```
你："加个退房按钮"
主 Agent → 判断规模小 → 直接开发 → Stop Hook 自检 → 提交
```

### 模式 B：中等功能（PM + 开发）

```
你："做退款功能"
主 Agent → /pm 拆解任务 → 你确认 →
  → Fullstack Dev：退款接口 + 退款页面
  → Visual QA 检查页面
  → Stop Hook 编译检查 → 有问题自动修复
  → 提交
```

### 模式 C：大功能（全团队协作）

```
你："做老板端管理系统"
主 Agent → /pm 拆解 → Architect 设计方案 → 你确认 →
  → Fullstack A（worktree）→ 房间管理（前后端+文档）
  → Fullstack B（worktree）→ 退款审批（前后端+文档）
  → 各自合并 → Visual QA 逐个检查
  → Reviewer 审查
  → 有问题 → 返回开发者修复 → 再审查
  → 全部通过 → /deploy
```

---

## 六、能力边界

### 能做到的
- PM 自动拆解任务
- 架构师出方案
- 前后端并行开发（worktree 隔离）
- 浏览器视觉验证前端页面
- 编译检查自循环修复
- 代码审查
- 一键部署
- 文档自动更新

### 做不到的（需要人）
- 需求确认（PM 拆完你要看一眼）
- 方案确认（架构师设计完你要确认）
- 合并审批（代码合并前你要确认）
- 线上问题判断（部署后需要人验证业务逻辑）
- 真机测试（支付宝小程序必须人扫码）

---

## 七、文件清单

```
.claude/
├── settings.local.json          # Hooks（PostToolUse 格式化 + Stop QA 自检）
├── launch.json                  # 后端 + H5 启动配置
├── skills/
│   ├── pm/SKILL.md              # /pm 需求拆解
│   ├── deploy/SKILL.md          # /deploy 一键部署
│   └── visual-check/SKILL.md    # /visual-check 手动视觉检查
└── agents/
    ├── architect/AGENT.md       # 架构师（只读）
    ├── fullstack-dev/AGENT.md   # 全栈开发（worktree）
    ├── reviewer/AGENT.md        # 代码审查（只读）
    └── visual-qa/AGENT.md       # 视觉 QA（chrome 工具）
```
