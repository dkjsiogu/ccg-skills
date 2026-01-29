---
description: 'CCG 首次大生成：Vibedev 完整流程 + 多模型协作执行'
---

# CCG Major - 首次大生成

$ARGUMENTS

---

## 强制规则 (必须遵守)

**你（Claude Code）是主控，但你绝对不能自己做完所有事。**

1. **必须调用 Codex 审核文档** - 需求、设计、任务文档各阶段完成后经过 Codex 审核
2. **必须调用 Copilot 执行前端任务** - 使用 Bash 工具执行 codeagent-wrapper
3. **必须调用 Codex 审核代码** - 模块完成后 + 最终全局审核
4. **禁止跳过任何一个 Stage** - 每个 Stage 完成后向用户报告
5. **⛔ 强制检查点** - 每个 Stage 开始前必须检查工作流状态，有待审查则不能继续

**违反以上任何一条，流程无效。**

---

## 工作流状态管理

**每个 Stage 开始前必须执行检查**：

```
Bash({
  command: "~/.claude/bin/ccg-workflow check || exit 1",
  timeout: 5000,
  description: "检查工作流状态"
})
```

**如果返回 BLOCKED，必须先完成审查才能继续。**

**状态转换规则**：
```
Stage 开始 → in_progress
Stage 代码完成 → pending_review
Codex 审查通过 → reviewed
进入下一 Stage → in_progress
```

---

## 多模型调用规范

### 调用 Codex (审查)

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - \"$PWD\" <<'CODEX_EOF'\n<在这里填入审核任务>\nCODEX_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审核"
})
```

### 调用 Copilot (前端执行)

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend gemini --lite - \"$PWD\" <<'COPILOT_EOF'\n<在这里填入前端任务>\nCOPILOT_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Copilot 执行前端任务"
})
```

### 等待结果

```
TaskOutput({ task_id: "<task_id>", block: true, timeout: 600000 })
```

**超时后继续轮询，禁止 Kill 进程。**

---

## 执行流程

### Stage 0: 自动初始化检查

`[模式：初始化]`

**首先初始化工作流状态**：

```
Bash({
  command: "~/.claude/bin/ccg-workflow init major '$ARGUMENTS' '功能目标待确认'",
  timeout: 5000,
  description: "初始化工作流状态"
})
```

**向 Monitor 报告 CCG 工作流启动**：

```
Bash({
  command: "CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'CCG Major: $ARGUMENTS') && echo \"CCG_TASK_ID=$CCG_TASK_ID\" && ~/.claude/bin/ccg-workflow set monitor_task_id \"$CCG_TASK_ID\"",
  timeout: 5000,
  description: "报告 CCG 启动到 Monitor"
})
```

**保存返回的 CCG_TASK_ID，后续阶段需要用它来更新状态。**

```
Bash({
  command: "~/.claude/bin/ccg-report running '$CCG_TASK_ID'",
  timeout: 5000,
  description: "更新状态为运行中"
})
```

**然后检查项目是否已初始化**：

1. **检查 `.ccg/` 目录是否存在**（使用 Glob 或 LS）
2. **如果不存在** → 执行快速初始化：
   - 用 Glob 扫描项目结构（`**/*.{json,toml,yaml}`, `src/**/*`）
   - 用 Grep 识别技术栈（`import`/`require` 语句、框架特征）
   - 生成 `.ccg/project.toml`（项目元信息）
   - 使用全局默认 prompt（`~/.claude/.ccg/prompts/`）
   - 向用户报告："项目未初始化，已使用默认配置。建议后续执行 `/ccg:init` 获取更精准的角色配置。"
3. **如果已存在** → 读取 `.ccg/project.toml` 获取项目上下文

**快速初始化生成的最小配置**：

```
.ccg/
├── project.toml          # 项目类型、技术栈、基本信息
└── context/
    └── quick-scan.md     # 快速扫描结果摘要
```

**project.toml 最小模板**：

```toml
[project]
name = "<folder_name>"
type = "<auto_detected>"    # web-app / api / cli / library / ...
tech_stack = "<detected>"   # rust / node / python / ...

[prompts]
# 项目级 prompt 未生成时，使用全局默认
use_global_defaults = true
```

---

### Stage 1: Vibedev 需求收集 + 设计 + 任务分解 (CC 主导 + Codex 审核)

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 1 in_progress", timeout: 5000 })
```

**更新 Monitor 阶段**：
```
Bash({ command: "~/.claude/bin/ccg-report stage '$CCG_TASK_ID' 'Stage 1: 需求/设计/任务分解'", timeout: 5000 })
```

CC 亲自执行，使用 MCP 工具。**每个文档阶段完成后必须经过 Codex 审核。**

#### 1.1 Goal 确认
- 引导用户 → 调用 `vibedev_specs_goal_confirmed`

#### 1.2 Requirements + Codex 审核需求
- 用 exa 查技术方案, 用 Glob/Grep 了解项目 → 生成需求文档
- **Codex 审核需求文档**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - \"$PWD\" <<'CODEX_EOF'\n## 需求文档审核\n\n### 功能目标\n<goal_summary>\n\n### 需求列表\n<requirements>\n\n### 审核要点\n1. 需求是否明确、可执行？\n2. 是否有遗漏的功能点？\n3. 边界条件是否清晰？\n4. 是否有安全相关需求需要考虑？\n\nOUTPUT: 审核意见 + 补充建议，用中文回复\nCODEX_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审核需求文档"
})
```

- 根据审核意见补充 → 调用 `vibedev_specs_requirements_confirmed`

#### 1.3 Design + Codex 审核设计
- 设计技术方案（架构、数据流、接口）
- **Codex 审核设计方案**：

```
Bash({
  command: "/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - \"$PWD\" <<'CODEX_EOF'\n## 技术设计审核\n\n### 功能目标\n<goal_summary>\n\n### 设计方案\n<design_document>\n\n### 审核要点\n1. 架构设计是否合理？\n2. 接口设计是否符合规范？\n3. 是否考虑了扩展性？\n4. 是否有安全漏洞风险？\n5. 是否与现有代码风格一致？\n\nOUTPUT: 审核意见 + 改进建议，用中文回复\nCODEX_EOF",
  run_in_background: true,
  timeout: 600000,
  description: "Codex 审核设计方案"
})
```

- 根据审核意见调整 → 调用 `vibedev_specs_design_confirmed`

#### 1.4 Tasks 分解 + Codex 审核任务
- 每个任务标注角色（后端/前端）和模块名
- **Codex 审核任务安排**（见 Stage 2）
- 调用 `vibedev_specs_tasks_confirmed`

**Stage 0 生成的 project.toml 现在用来补充上下文**，如果之前是快速初始化，在此阶段更新：
- 根据需求细化技术栈信息
- 更新 `.ccg/project.toml` 中的 `use_global_defaults = false`（如果有足够信息）

**完成后**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 1 pending_review", timeout: 5000 })
```

**向用户报告**: "Stage 1 完成，需求/设计/任务均通过 Codex 审核，共 N 个任务，后端 X 个，前端 Y 个。"

**标记审查通过**（所有文档均已在子阶段审核通过）：
```
Bash({ command: "~/.claude/bin/ccg-workflow review 1 passed", timeout: 5000 })
```

---

### Stage 2: Codex 最终审核任务列表

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 2 in_progress", timeout: 5000 })
```

**更新 Monitor 阶段**：
```
Bash({ command: "~/.claude/bin/ccg-report stage '$CCG_TASK_ID' 'Stage 2: Codex 审核任务'", timeout: 5000 })
```

**Stage 1 各阶段已逐步审核，此处做最终综合审核。你必须执行以下 Bash 命令，不能跳过：**

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - "$PWD" <<'CODEX_EOF'
## 最终任务列表审核

### 功能目标
（goal_summary）

### 需求摘要
（已审核通过的需求要点）

### 设计摘要
（已审核通过的设计要点）

### 待审核任务列表
（这里粘贴你在 Stage 1 生成的完整任务列表）

### 审核要点
1. 任务列表是否覆盖了所有需求和设计要点？
2. 任务分解是否合理？有没有遗漏？
3. 任务之间的依赖关系是否正确？
4. 前端/后端划分是否合适？
5. 有没有安全风险需要注意？

请逐条审核，输出审核意见。用中文回复。
CODEX_EOF
```

**使用 `run_in_background: true` 和 `timeout: 600000`**

**处理结果**：
- Codex 提出问题 → CC 调整任务后重新提交审核
- Codex 全部通过 → 标记审查通过，进入 Stage 3

**审查通过后**：
```
Bash({ command: "~/.claude/bin/ccg-workflow review 2 passed", timeout: 5000 })
```

**向用户报告**: "Stage 2 完成，Codex 审核通过/有 N 处建议。"

---

### Stage 3: 逐模块执行 + 模块级审查

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 3 in_progress", timeout: 5000 })
```

**更新 Monitor 阶段**：
```
Bash({ command: "~/.claude/bin/ccg-report stage '$CCG_TASK_ID' 'Stage 3: 逐模块执行'", timeout: 5000 })
```

**CC 作为执行代理**：CC 先执行命令收集上下文，再传给 Copilot/Codex。
**项目级 Prompt**：如果 `.ccg/prompts/` 存在项目级提示词，读取后附加到调用中。

#### 3.1 后端任务 → CC 自己写

CC 直接写后端/逻辑代码，但要：
- 先用 Grep/Read 分析现有代码
- 为前端提供清晰接口定义

#### 3.2 前端任务 → 必须调用 Copilot

**CC 先读取相关代码和上下文，然后传给 Copilot**：

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend gemini --lite - "$PWD" <<'COPILOT_EOF'
## 项目上下文
（如有 .ccg/prompts/copilot.md，读取并粘贴在这里）

## 前端任务

### 任务描述
（这里填写具体的前端任务）

### 相关代码上下文
（CC 预先读取的相关代码，粘贴在这里）

### 接口规范
（后端提供的 API 接口定义）

请生成完整实现。
COPILOT_EOF
```

**使用 `run_in_background: true` 和 `timeout: 600000`**

#### 3.3 Copilot 辅助任务（测试/文档）

**除前端开发外，Copilot 还应承担以下任务**：

**测试生成**：CC 读取已完成模块的代码，传给 Copilot 生成测试

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend gemini --lite - "$PWD" <<'COPILOT_EOF'
## 任务：为以下代码生成单元测试

### 代码
（CC 预先 Read 的源代码）

### 项目测试规范
（CC 预先读取的现有测试文件风格）

### 要求
- 覆盖主要路径和边界条件
- 保持项目现有测试风格
- 输出完整可运行的测试代码

OUTPUT: 测试代码
COPILOT_EOF
```

**文档生成**：CC 执行命令获取结构，传给 Copilot 生成文档

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend gemini --lite - "$PWD" <<'COPILOT_EOF'
## 任务：为以下模块生成 API 文档

### 模块结构
（CC 预先执行 find/grep 获取的文件列表和公开接口）

### 代码摘要
（CC 预先读取的关键代码片段）

OUTPUT: Markdown 文档
COPILOT_EOF
```

#### 3.4 模块完成后 → Codex 审查（每个模块）

**每完成一个模块/任务，立即调用 Codex 审查**：

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - "$PWD" <<'CODEX_EOF'
## 项目上下文
（如有 .ccg/prompts/codex.md，读取并粘贴在这里）

## 模块审查: <task_name>

### 代码变更
<git diff for this module>

### 模块目标
<task_description>

### 审查要点
1. 代码质量与规范
2. 逻辑正确性
3. 错误处理
4. 安全性

请按 Critical/Major/Minor 分类列出问题。用中文回复。
CODEX_EOF
```

**模块审查迭代**：
- 有 Critical/Major → Claude 修复 → 重新提交 Codex 审查
- 只有 Minor 或无问题 → 继续下一模块

#### 3.5 并行优化

后端和前端无依赖时可并行：
- CC 开始写后端
- 同时 `run_in_background: true` 让 Copilot 写前端
- 同时 Copilot 生成已完成模块的测试
- 完成后各自审查

**完成后**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 3 pending_review", timeout: 5000 })
```

**向用户报告**: "Stage 3 完成，CC 完成 X 个后端任务，Copilot 完成 Y 个前端/测试/文档任务，全部通过模块审查。"

**标记模块审查通过**：
```
Bash({ command: "~/.claude/bin/ccg-workflow review 3 passed", timeout: 5000 })
```

---

### Stage 4: Codex + Claude 最终联合审查

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 4 in_progress", timeout: 5000 })
```

**更新 Monitor 阶段**：
```
Bash({ command: "~/.claude/bin/ccg-report stage '$CCG_TASK_ID' 'Stage 4: 最终联合审查'", timeout: 5000 })
```

**所有模块完成后，进行全局审查**：

```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite - "$PWD" <<'CODEX_EOF'
## 最终全局审查

### 功能目标
<goal_summary>

### 完整变更
<git diff --stat>
<git diff>

### 已完成模块
<task_list with status>

### 全面评估
1. 整体架构一致性
2. 模块间接口正确性
3. 全局安全性检查
4. 性能热点识别
5. 代码重复和抽象机会

OUTPUT: Critical/Major/Minor 分类 + 整体评分(1-10) + 是否通过。用中文回复。
CODEX_EOF
```

**使用 `run_in_background: true` 和 `timeout: 600000`**

**Claude 综合评估**：
- 验证 Codex 审查结果
- 补充架构/设计层面问题
- 生成最终审查报告

**审查完成后**：
- 如果有 Critical/Major → 标记为 pending_review，进入 Stage 5 迭代修复
- 如果全部通过 → 标记为 reviewed，跳过 Stage 5

```
Bash({ command: "~/.claude/bin/ccg-workflow stage 4 pending_review", timeout: 5000 })
```

---

### Stage 5: 迭代修复直到满意

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 5 in_progress", timeout: 5000 })
```

**更新 Monitor 阶段**：
```
Bash({ command: "~/.claude/bin/ccg-report stage '$CCG_TASK_ID' 'Stage 5: 迭代修复'", timeout: 5000 })
```

**自动迭代循环**：

```
iteration = 0

WHILE has_critical_or_major AND iteration < 5:
    1. Claude 修复 Critical → Major
    2. Codex 重新审查修复内容
    3. Claude 综合判断
    4. iteration++

IF iteration >= 5:
    询问用户是否继续
```

**满意标准**：Critical=0 且 Major=0

**迭代修复完成后**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 4 pending_review && ~/.claude/bin/ccg-workflow review 4 passed && ~/.claude/bin/ccg-workflow stage 5 pending_review && ~/.claude/bin/ccg-workflow review 5 passed", timeout: 5000 })
```

---

### Stage 6: 整合交付

**⛔ 强制检查点**：
```
Bash({ command: "~/.claude/bin/ccg-workflow check || exit 1", timeout: 5000 })
```

**更新工作流状态**：
```
Bash({ command: "~/.claude/bin/ccg-workflow stage 6 in_progress", timeout: 5000 })
```

审查全部通过后：

1. 整合所有变更
2. 运行测试（如有）
3. **更新 Monitor 和工作流状态为完成**：
   ```
   Bash({
     command: "~/.claude/bin/ccg-report done '$CCG_TASK_ID' 'CCG Major 完成: {feature_name}' && ~/.claude/bin/ccg-workflow stage 6 completed",
     timeout: 5000,
     description: "报告 CCG 完成到 Monitor"
   })
   ```
4. 生成完成报告

```markdown
## CCG Major 完成

### 功能: {feature_name}

### 流程追踪
- [x] Stage 0: 项目初始化检查
- [x] Stage 1: Vibedev 需求收集 + 设计 + 任务分解（各阶段 Codex 审核）
- [x] Stage 2: Codex 最终任务列表审核
- [x] Stage 3: 逐模块执行 + 模块审查
- [x] Stage 4: 最终联合审查
- [x] Stage 5: 迭代修复
- [x] Stage 6: 整合交付

### 工作分配
| 角色 | 任务数 | 内容 |
|-----|-------|------|
| CC | {n} | {后端任务列表} |
| Copilot | {n} | {前端任务列表} |
| Codex | {n}次审核 | 需求审核 + 设计审核 + 任务审核 + 模块审查 + 最终审查 |

### 审查结果
- Codex 最终评分：{score}/10
- 迭代次数：{iteration_count}

### 后续
- `/ccg:iter` 继续迭代
```

---

## 再次提醒

**你必须在 Stage 1.2/1.3/1.4/2/3.2/3.3/4 执行 Bash 调用 codeagent-wrapper 进行审核。**
**需求文档完成后审核，设计文档完成后审核，任务列表完成后审核。**
**如果你发现自己在写前端代码，停下来，调用 Copilot。**
**如果你发现自己跳过了审核，停下来，调用 Codex。**
**每个模块完成后必须审查，不要等全部写完才审查。**
