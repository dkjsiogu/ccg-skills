---
description: 'GitHub DevOps：使用 Copilot CLI 管理 PR/Issue/CI/Release'
---

# DevOps - GitHub 自动化

使用 GitHub Copilot CLI (`copilot`) 执行 DevOps 任务。Copilot 有原生 GitHub MCP 集成，擅长 PR/Issue/CI 操作。

## 使用方法

```bash
/devops <task>
```

## 常见任务

| 任务 | 示例 |
|------|------|
| 创建 PR | `/devops create pr for current branch` |
| 查看 PR 状态 | `/devops check pr #123 status` |
| 处理 Issue | `/devops close issue #45 with comment` |
| 查看 CI 状态 | `/devops show ci status for main` |
| 创建 Release | `/devops draft release v1.2.0` |
| 管理 Actions | `/devops fix failing workflow` |

---

## 执行工作流

### 🔍 阶段 1：任务分析

`[模式：分析]`

1. 解析用户请求
2. 确定 GitHub 操作类型（PR/Issue/CI/Release）
3. 收集必要上下文（当前分支、仓库状态等）

### 🤖 阶段 2：Copilot 执行

`[模式：执行]`

使用 Copilot CLI 执行任务：

```bash
copilot -p "<task description>" --allow-all-tools --model gpt-5.1-codex
```

**注意**：Copilot CLI 有自己的工具执行能力，可以直接操作 GitHub API。

### ✅ 阶段 3：结果确认

`[模式：验证]`

1. 检查操作结果
2. 提供操作链接（PR URL、Issue URL 等）
3. 报告任何错误或需要手动处理的情况

---

## Copilot CLI 能力

| 能力 | 说明 |
|------|------|
| GitHub MCP | 原生 GitHub API 集成 |
| Shell 执行 | 可运行 `gh` 命令 |
| 文件编辑 | 可修改 workflow 文件 |
| 多模型 | 支持 GPT-5/Gemini/Claude |

## 模型选择

| 模型 | 用途 |
|------|------|
| `gpt-5.1-codex` | 复杂 DevOps 任务（默认） |
| `gpt-5.2` | 最新能力 |
| `gemini-3-pro-preview` | 快速任务 |

---

## 示例

### 创建 PR

```bash
/devops create a PR from current branch to main with:
- title: "feat: add user authentication"
- description: summarize recent commits
- add labels: enhancement, needs-review
```

### 处理 Issue

```bash
/devops look at issue #42, understand the bug, and create a fix branch
```

### CI 调试

```bash
/devops the CI is failing on main, diagnose and suggest fixes
```

### Release 管理

```bash
/devops create release v2.0.0 with changelog from commits since v1.9.0
```

---

## 与其他角色的协作

```
代码开发 (CC/GM/CX) → 完成后 → /devops create pr
PR 审查需求 → /review → 通过后 → /devops merge pr
CI 失败 → /devops diagnose → 修复 → /commit → /devops rerun
```

## 关键规则

1. **Copilot 专属** – 不使用 codeagent-wrapper，直接调用 `copilot` CLI
2. **权限验证** – 确保已登录 GitHub (`copilot` 自动处理)
3. **安全操作** – 危险操作（force push、delete branch）需确认
4. **结果链接** – 总是提供操作结果的 URL
