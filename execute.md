---
description: '多模型协作执行 - 根据计划获取原型 → Claude 重构实施 → 多模型审计交付'
---

# Execute - 多模型协作执行

$ARGUMENTS

---

## 核心协议

- **语言协议**：与工具/模型交互用**英语**，与用户交互用**中文**
- **代码主权**：外部模型对文件系统**零写入权限**，所有修改由 Claude 执行
- **审查后应用**：Claude 审查 Codex/Gemini 的 Unified Diff，高质量直接应用，有问题按需修正
- **止损机制**：当前阶段输出通过验证前，不进入下一阶段
- **前置条件**：仅在用户对 `/ccg:plan` 输出明确回复 "Y" 后执行（如缺失，必须先二次确认）

---

## Call Spec

**调用规范详见 `~/.claude/.ccg/docs/callspec.md`**（语法、超时、会话复用、角色提示词表）。

**本命令特有角色**：

| 阶段 | Codex | Gemini |
|------|-------|--------|
| 实施 | `~/.claude/.ccg/prompts/codex/architect.md` | `~/.claude/.ccg/prompts/gemini/frontend.md` |
| 审查 | `~/.claude/.ccg/prompts/codex/reviewer.md` | `~/.claude/.ccg/prompts/gemini/reviewer.md` |

---

## 执行工作流

**执行任务**：$ARGUMENTS

### 📖 Phase 0：读取计划

`[模式：准备]`

1. **识别输入类型**：
   - 计划文件路径（如 `.claude/plan/xxx.md`）
   - 直接的任务描述

2. **读取计划内容**：
   - 若提供了计划文件路径，读取并解析
   - 提取：任务类型、实施步骤、关键文件、SESSION_ID

3. **执行前确认**：
   - 若输入为"直接任务描述"或计划中缺失 `SESSION_ID` / 关键文件：先向用户确认补全信息
   - 若无法确认用户是否已对计划回复 "Y"：必须二次询问确认后再进入下一阶段

4. **任务类型判断**：

   | 任务类型 | 判断依据 | 路由 |
   |----------|----------|------|
   | **前端** | 页面、组件、UI、样式、布局 | Gemini |
   | **后端** | API、接口、数据库、逻辑、算法 | Codex |
   | **全栈** | 同时包含前后端 | Codex ∥ Gemini 并行 |

---

### 🔍 Phase 1：上下文快速检索

`[模式：检索]`

**⚠️ 必须使用 MCP 工具快速检索上下文，禁止手动逐个读取文件**

根据计划中的"关键文件"列表，调用 `mcp__ace-tool__search_context` 检索相关代码：

```
mcp__ace-tool__search_context({
  query: "<基于计划内容构建的语义查询，包含关键文件、模块、函数名>",
  project_root_path: "$PWD"
})
```

**检索策略**：
- 从计划的"关键文件"表格提取目标路径
- 构建语义查询覆盖：入口文件、依赖模块、相关类型定义
- 若检索结果不足，可追加 1-2 次递归检索
- **禁止**使用 Bash + find/ls 手动探索项目结构

**检索完成后**：
- 整理检索到的代码片段
- 确认已获取实施所需的完整上下文
- 进入 Phase 3

---

### 🎨 Phase 3：原型获取

`[模式：原型]`

**根据任务类型路由**：

#### Route A: 前端/UI/样式 → Gemini

**限制**：上下文 < 32k tokens

1. 调用 Gemini（使用 `~/.claude/.ccg/prompts/gemini/frontend.md`）
2. 输入：计划内容 + 检索到的上下文 + 目标文件
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Gemini 是前端设计的权威，其 CSS/React/Vue 原型为最终视觉基准**
5. ⚠️ **警告**：忽略 Gemini 对后端逻辑的建议
6. 若计划包含 `GEMINI_SESSION`：优先 `resume <GEMINI_SESSION>`

#### Route B: 后端/逻辑/算法 → Codex

1. 调用 Codex（使用 `~/.claude/.ccg/prompts/codex/architect.md`）
2. 输入：计划内容 + 检索到的上下文 + 目标文件
3. OUTPUT: `Unified Diff Patch ONLY. Strictly prohibit any actual modifications.`
4. **Codex 是后端逻辑的权威，利用其逻辑运算与 Debug 能力**
5. 若计划包含 `CODEX_SESSION`：优先 `resume <CODEX_SESSION>`

#### Route C: 全栈 → 并行调用

1. **并行调用**（`run_in_background: true`）：
   - Gemini：处理前端部分
   - Codex：处理后端部分
2. 用 `TaskOutput` 等待两个模型的完整结果
3. 各自使用计划中对应的 `SESSION_ID` 进行 `resume`（若缺失则创建新会话）

**务必遵循 Call Spec 的等待规则**

---

### ⚡ Phase 4：编码实施

`[模式：实施]`

**Claude 审查并应用外部模型输出**：

1. **读取 Diff**：解析 Codex/Gemini 返回的 Unified Diff Patch

2. **快速审查**（替代旧版"全量重构"）：
   - 检查逻辑一致性、安全性、副作用
   - **高质量输出**（无明显问题）→ 直接 `git apply` 或等效操作应用
   - **需要修正**（逻辑错误/风格不一致/安全隐患）→ 针对性修正后应用
   - 去除明显冗余，确保符合项目代码规范

3. **最小作用域**：
   - 变更仅限需求范围
   - **强制审查**变更是否引入副作用

4. **应用变更**：
   - 使用 Edit/Write 工具执行实际修改
   - **仅修改必要的代码**，严禁影响用户现有的其他功能

5. **自检验证**（强烈建议）：
   - 运行项目既有的 lint / typecheck / tests（优先最小相关范围）
   - 若失败：优先修复回归，再继续进入 Phase 5

---

### ✅ Phase 5：审计与交付

`[模式：审计]`

#### 5.1 自动审计

**变更生效后，强制立即并行调用** Codex 和 Gemini 进行 Code Review：

1. **Codex 审查**（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/codex/reviewer.md`
   - 输入：变更的 Diff + 目标文件
   - 关注：安全性、性能、错误处理、逻辑正确性

2. **Gemini 审查**（`run_in_background: true`）：
   - ROLE_FILE: `~/.claude/.ccg/prompts/gemini/reviewer.md`
   - 输入：变更的 Diff + 目标文件
   - 关注：可访问性、设计一致性、用户体验

用 `TaskOutput` 等待两个模型的完整审查结果。优先复用 Phase 3 的会话（`resume <SESSION_ID>`）以保持上下文一致。

#### 5.2 整合修复

1. 综合 Codex + Gemini 的审查意见
2. 按信任规则权衡：后端以 Codex 为准，前端以 Gemini 为准
3. 执行必要的修复
4. 修复后按需重复 Phase 5.1（直到风险可接受）

#### 5.3 交付确认

审计通过后，向用户报告：

```markdown
## ✅ 执行完成

### 变更摘要
| 文件 | 操作 | 说明 |
|------|------|------|
| path/to/file.ts | 修改 | 描述 |

### 审计结果
- Codex：<通过/发现 N 个问题>
- Gemini：<通过/发现 N 个问题>

### 后续建议
1. [ ] <建议的测试步骤>
2. [ ] <建议的验证步骤>
```

---

## 关键规则

1. **代码主权** – 所有文件修改由 Claude 执行，外部模型零写入权限
2. **审查后应用** – Codex/Gemini 输出经 Claude 审查，高质量直接用，有问题按需修正
3. **信任规则** – 后端以 Codex 为准，前端以 Gemini 为准
4. **最小变更** – 仅修改必要的代码，不引入副作用
5. **强制审计** – 变更后必须进行多模型 Code Review

---

## 使用方法

```bash
# 执行计划文件
/ccg:execute .claude/plan/功能名.md

# 直接执行任务（适用于已在上下文中讨论过的计划）
/ccg:execute 根据之前的计划实施用户认证功能
```

---

## 与 /ccg:plan 的关系

1. `/ccg:plan` 生成计划 + SESSION_ID
2. 用户确认 "Y" 后
3. `/ccg:execute` 读取计划，复用 SESSION_ID，执行实施
