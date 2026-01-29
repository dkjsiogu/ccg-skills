---
description: 'CCG + Vibedev 整合工作流：结构化需求 → Codex审核文档 → 多模型执行 → 迭代审核直到满意'
---

# Workflow-Vibe - CCG + Vibedev 整合工作流

整合 vibedev MCP 的结构化流程与 CCG 多模型协作，每个文档阶段均经过 Codex 审核，支持模块级审查和最终联合审查的自动迭代。

## 使用方法

```bash
/ccg:workflow-vibe <功能描述>
```

## 流程概览

```
┌─────────────────────────────────────────────────────────┐
│              Vibedev 结构化 + Codex 审核                  │
├─────────────────────────────────────────────────────────┤
│  1. Goal        → 明确目标                               │
│  2. Requirements → 细化需求 → [Codex 审核需求]           │
│  3. Design      → 技术设计 → [Codex 审核设计]           │
│  4. Tasks       → 任务拆分 → [Codex 审核任务]           │
├─────────────────────────────────────────────────────────┤
│              CCG 多模型执行 + 迭代审核                    │
├─────────────────────────────────────────────────────────┤
│  5. Execute     → 逐模块执行 → 模块完成后审查 → 迭代     │
│  6. Final Review → 全部完成 → Codex+Claude 联合审查      │
│  7. Iterate     → 有问题则修复 → 重新审查 → 直到满意     │
└─────────────────────────────────────────────────────────┘
```

## 角色分工

| 角色 | 职责 |
|------|------|
| **Vibedev MCP** | 结构化需求、设计、任务 |
| **Claude** | 编排 + 后端开发 + 综合评估 + 代码修复 |
| **Copilot** | 前端开发（代码生成，不参与审查） |
| **Codex** | 审核文档（需求/设计/任务）+ 审核代码（主审） |

---

## 执行工作流

**功能描述**：$ARGUMENTS

### 📌 阶段 0：启动 Vibedev 工作流

`[模式：初始化]`

**首先向 Monitor 报告工作流启动**：

```
Bash({
  command: "curl -s -X POST http://127.0.0.1:3721/api/tasks -H 'Content-Type: application/json' -d '{\"cli_type\":\"claude\",\"prompt\":\"CCG Workflow-Vibe 启动\",\"workdir\":\"'\"$PWD\"'\"}'",
  timeout: 5000,
  description: "报告 CCG 启动到 Monitor"
})
```

**然后调用 vibedev MCP 启动工作流**：

```
mcp__vibedev-specs__vibedev_specs_workflow_start()
```

获取 session_id 后进入目标收集阶段。

---

### 🎯 阶段 1：目标确认 (Goal)

`[模式：目标]`

1. 与用户明确功能目标
2. 生成 feature_name（如 `user-auth`）
3. 确认目标：

```
mcp__vibedev-specs__vibedev_specs_goal_confirmed({
  session_id: "<session_id>",
  feature_name: "<feature_name>",
  goal_summary: "<目标描述>"
})
```

---

### 📋 阶段 2：需求收集 (Requirements) + Codex 审核

`[模式：需求]`

1. 启动需求收集：

```
mcp__vibedev-specs__vibedev_specs_requirements_start({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

2. 与用户细化需求（功能点、边界、约束）
3. 需求完整性评分（≥7分继续，<7分补充）

4. **Codex 审核需求文档**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - \"$PWD\" <<'CODEX_EOF'
## 需求文档审核

### 功能目标
<goal_summary>

### 需求列表
<requirements_list>

### 审核要点
1. 需求是否明确、可执行？
2. 是否有遗漏的功能点？
3. 边界条件是否清晰？
4. 是否有潜在的技术风险？
5. 是否有安全相关的需求需要考虑？

OUTPUT: 审核意见 + 补充建议（如有），用中文回复
CODEX_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审核需求文档"
})
```

5. 根据 Codex 审核意见补充需求
6. 确认需求：

```
mcp__vibedev-specs__vibedev_specs_requirements_confirmed({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

---

### 🏗️ 阶段 3：技术设计 (Design) + Codex 审核

`[模式：设计]`

1. 启动设计阶段：

```
mcp__vibedev-specs__vibedev_specs_design_start({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

2. Claude 设计技术方案（架构、数据流、接口设计）

3. **Codex 审核设计方案**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - \"$PWD\" <<'CODEX_EOF'
## 技术设计审核

### 功能目标
<goal_summary>

### 设计方案
<design_document>

### 审核要点
1. 架构设计是否合理？
2. 数据流是否清晰？
3. 接口设计是否符合规范？
4. 是否考虑了扩展性？
5. 是否有安全漏洞风险？
6. 是否与现有代码风格一致？

OUTPUT: 审核意见 + 改进建议（如有），用中文回复
CODEX_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审核设计方案"
})
```

4. 根据 Codex 审核意见调整设计
5. 用户确认后：

```
mcp__vibedev-specs__vibedev_specs_design_confirmed({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

---

### 📝 阶段 4：任务规划 (Tasks) + Codex 审核

`[模式：任务]`

1. 启动任务规划：

```
mcp__vibedev-specs__vibedev_specs_tasks_start({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

2. Claude 拆分任务列表（每个任务应是一个完整模块）

3. **Codex 审核任务安排**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex - \"$PWD\" <<'EOF'
ROLE_FILE: /home/dkjsiogu/.claude/.ccg/prompts/codex/reviewer.md
<TASK>
请审核以下任务拆分是否合理：

## 功能目标
<goal_summary>

## 任务列表
<task_list>

请评估：
1. 任务拆分粒度是否合适（每个任务应是完整模块）
2. 依赖关系是否正确
3. 是否有遗漏的任务
4. 优先级是否合理

OUTPUT: 审核意见 + 修改建议（如有），用中文回复
</TASK>
EOF",
  timeout: 600000,
  description: "Codex 审核任务安排"
})
```

4. 根据 Codex 审核意见调整任务
5. 用户确认后：

```
mcp__vibedev-specs__vibedev_specs_tasks_confirmed({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

---

### ⚡ 阶段 5：执行 + 模块级审查（迭代循环）

`[模式：执行]`

**核心原则**：每完成一个模块/任务，立即进行 Codex 审查，修复问题后再继续下一个模块。

1. 启动执行阶段：

```
mcp__vibedev-specs__vibedev_specs_execute_start({
  session_id: "<session_id>",
  feature_name: "<feature_name>"
})
```

2. **逐模块执行循环**：

```
FOR each task in task_list:

    ┌─────────────────────────────────────────┐
    │ 5.1 执行任务                             │
    ├─────────────────────────────────────────┤
    │ - 后端任务 → Claude 直接执行             │
    │ - 前端任务 → Copilot 生成 → Claude 写入  │
    └─────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────┐
    │ 5.2 模块完成后 Codex 审查               │
    ├─────────────────────────────────────────┤
    │ - 审查该模块的代码变更                   │
    │ - 返回 Critical/Major/Minor 问题        │
    └─────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────┐
    │ 5.3 Claude 综合 + 修复                  │
    ├─────────────────────────────────────────┤
    │ - 验证 Codex 审查结果                   │
    │ - 修复 Critical/Major 问题              │
    │ - 重新审查直到通过                       │
    └─────────────────────────────────────────┘
                    ↓
            继续下一个任务
```

3. **模块审查调用**（每个任务完成后）：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex - \"$PWD\" <<'EOF'
ROLE_FILE: /home/dkjsiogu/.claude/.ccg/prompts/codex/reviewer.md
<TASK>
审查模块: <task_name>

## 代码变更
<git diff for this module>

## 模块目标
<task_description>

请评估：
1. 代码质量与规范
2. 逻辑正确性
3. 错误处理
4. 安全性

OUTPUT: 按 Critical/Major/Minor 分类列出问题，用中文回复
</TASK>
EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审查模块: <task_name>"
})
```

4. **模块审查迭代**：
   - 若有 Critical/Major 问题 → Claude 修复 → 重新审查
   - 若只有 Minor/无问题 → 继续下一模块

---

### 🔍 阶段 6：最终联合审查（Codex + Claude）

`[模式：最终审查]`

**所有模块完成后，进行全局联合审查**：

1. **Codex 全局审查**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex - \"$PWD\" <<'EOF'
ROLE_FILE: /home/dkjsiogu/.claude/.ccg/prompts/codex/reviewer.md
<TASK>
最终全局审查

## 功能目标
<goal_summary>

## 完整代码变更
<git diff --stat>
<git diff>

## 已完成模块
<completed_tasks>

请进行全面评估：
1. 整体架构一致性
2. 模块间接口正确性
3. 全局安全性检查
4. 性能热点识别
5. 边界条件和错误处理
6. 代码重复和抽象机会

OUTPUT:
- Critical 问题（必须修复）
- Major 问题（强烈建议修复）
- Minor 问题（可选优化）
- 整体评分（1-10）
- 是否通过审查（是/否）

用中文回复
</TASK>
EOF",
  timeout: 600000,
  description: "Codex 最终全局审查"
})
```

2. **Claude 综合评估**：
   - 验证 Codex 审查结果
   - 补充架构/设计层面问题
   - 生成最终审查报告

---

### 🔄 阶段 7：迭代修复直到满意

`[模式：迭代]`

**自动迭代循环，直到 Codex + Claude 都满意**：

```
iteration_count = 0
max_iterations = 5

WHILE has_critical_or_major_issues AND iteration_count < max_iterations:

    ┌─────────────────────────────────────────┐
    │ 7.1 Claude 修复问题                     │
    ├─────────────────────────────────────────┤
    │ - 按优先级修复 Critical → Major         │
    │ - 每次修复后记录变更                     │
    └─────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────┐
    │ 7.2 Codex 重新审查                      │
    ├─────────────────────────────────────────┤
    │ - 审查修复后的代码                       │
    │ - 验证问题是否解决                       │
    │ - 检查是否引入新问题                     │
    └─────────────────────────────────────────┘
                    ↓
    ┌─────────────────────────────────────────┐
    │ 7.3 Claude 综合判断                     │
    ├─────────────────────────────────────────┤
    │ - 评估是否还有遗留问题                   │
    │ - 决定是否继续迭代                       │
    └─────────────────────────────────────────┘
                    ↓
            iteration_count++

IF iteration_count >= max_iterations:
    询问用户是否继续迭代

IF no_critical_or_major_issues:
    进入交付阶段
```

---

### ✅ 阶段 8：交付确认

`[模式：交付]`

审查通过后，向用户报告：

```markdown
## ✅ 功能开发完成

### 完成摘要
- 功能：<feature_name>
- 模块数：<task_count>
- 迭代次数：<iteration_count>

### 审查状态
- Codex 最终评分：<score>/10
- Critical 问题：0
- Major 问题：0
- Minor 问题：<count>（已记录，可后续优化）

### 变更文件
<file_list>

### 后续建议
- <suggestions>
```

---

## 关键规则

1. **文档级审查** - 需求、设计、任务文档各阶段完成后都经过 Codex 审核
2. **模块级审查** - 每完成一个模块/任务立即审查，不是写一点就审查
3. **最终联合审查** - 全部完成后 Codex + Claude 联合审查
4. **自动迭代** - 有 Critical/Major 问题自动修复并重新审查
5. **迭代上限** - 默认最多 5 次迭代，超过后询问用户
6. **Copilot 不审查** - Copilot 只负责前端代码生成
7. **外部模型零写入权限** - 所有文件操作由 Claude 执行
8. **满意标准** - Critical=0 且 Major=0 视为通过

## 审查时机总结

| 时机 | 审查者 | 范围 |
|------|--------|------|
| 需求文档完成后 | Codex | 需求完整性、边界条件、安全需求 |
| 设计文档完成后 | Codex | 架构合理性、接口规范、扩展性 |
| 任务列表完成后 | Codex | 任务覆盖度、依赖关系、前后端划分 |
| 每个模块完成后 | Codex | 该模块代码变更 |
| 全部模块完成后 | Codex + Claude | 全局审查 |
| 每次修复后 | Codex | 修复变更 |
