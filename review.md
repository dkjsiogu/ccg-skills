---
description: '代码审查：Codex 审核 + Claude 综合，支持迭代修复'
---

# Review - 代码审查

Codex 深度审查 + Claude 综合评估。无参数时自动审查当前 git 变更。

## 使用方法

```bash
/review [代码或描述]
```

- **无参数**：自动审查 `git diff HEAD`
- **有参数**：审查指定代码或描述

---

## 调用规范

**Codex 审查调用**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex - \"$PWD\" <<'EOF'
ROLE_FILE: /home/dkjsiogu/.claude/.ccg/prompts/codex/reviewer.md
<TASK>
审查以下代码变更：
<git diff 内容>
</TASK>
OUTPUT: 按 Critical/Major/Minor/Suggestion 分类列出问题，用中文回复
EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 代码审查"
})
```

**等待后台任务**：

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**重要**：
- 必须指定 `timeout: 600000`（10 分钟）
- 若超时未完成，继续用 `TaskOutput` 轮询，**绝对不要 Kill 进程**
- 若需要中断，必须调用 `AskUserQuestion` 询问用户

---

## 执行工作流

### 🔍 阶段 1：获取待审查代码

`[模式：研究]`

**无参数时**：执行 `git diff HEAD` 和 `git status --short`

**有参数时**：使用指定的代码/描述

### 🔬 阶段 2：Codex 深度审查

`[模式：审查]`

发起 Codex 后台审查（参照上方调用规范）：
- ROLE_FILE: `/home/dkjsiogu/.claude/.ccg/prompts/codex/reviewer.md`
- 需求：审查代码变更（git diff 内容）
- OUTPUT：按 Critical/Major/Minor/Suggestion 分类

用 `TaskOutput` 等待 Codex 审查结果。

### 🧠 阶段 3：Claude 综合评估

`[模式：综合]`

Claude 基于 Codex 的审查结果：
1. 验证问题的准确性
2. 补充 Codex 可能遗漏的架构/设计问题
3. 按严重程度重新分类

### 📊 阶段 4：呈现审查结果

`[模式：总结]`

```markdown
## 📋 代码审查报告

### 审查范围
- 变更文件：<数量> | 代码行数：+X / -Y

### 关键问题 (Critical)
> 必须修复才能合并
1. <问题描述> - [来源: Codex/Claude]

### 主要问题 (Major)
...

### 次要问题 (Minor)
...

### 建议 (Suggestions)
...

### 总体评价
- 代码质量：[优秀/良好/需改进]
- 是否可合并：[是/否/需修复后]
```

### 🔄 阶段 5：迭代修复（如有问题）

`[模式：迭代]`

如果发现 Critical 或 Major 问题：

1. **询问用户**：是否自动修复？
2. **修复问题**：Claude 直接修复代码
3. **重新审查**：回到阶段 2，直到无 Critical/Major 问题
4. **最终确认**：所有问题解决后，提交代码

---

## 关键规则

1. **无参数 = 审查 git diff** – 自动获取当前变更
2. **Codex 主审 + Claude 综合** – 深度审查 + 架构把控
3. **迭代直到满意** – Critical/Major 问题必须修复
4. 外部模型对文件系统**零写入权限**（只有 Claude 可以修改代码）
