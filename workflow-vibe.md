---
description: 'CCG + Vibedev 整合工作流：结构化需求 → Codex审核任务 → 多模型执行 → Codex审核代码'
---

# CCG + Vibedev 整合工作流

$ARGUMENTS

---

## 概述

> **参考 CLAUDE.md 中的 CCG 协作规范**（角色定义、调用模板、审查格式）

整合 Vibedev MCP 结构化流程与 CCG 多模型协作。

---

## 阶段 0: 初始化

```bash
CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'CCG Vibedev: $ARGUMENTS')
~/.claude/bin/ccg-report running "$CCG_TASK_ID"
```

调用 `vibedev_specs_workflow_start()` 获取 session_id。

---

## 阶段 1: 目标确认

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 1: 目标确认'
```

1. 明确功能目标
2. 调用 `vibedev_specs_goal_confirmed(session_id, feature_name, goal_summary)`

---

## 阶段 2: 需求收集 + Codex 审核

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 2: 需求收集'
```

1. `vibedev_specs_requirements_start()`
2. 细化需求
3. **Codex 审核需求文档**（审核要点：明确性、完整性、边界、安全）
4. 根据审核补充后 `vibedev_specs_requirements_confirmed()`

---

## 阶段 3: 技术设计 + Codex 审核

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 3: 技术设计'
```

1. `vibedev_specs_design_start()`
2. Claude 设计（架构、数据流、接口）
3. **Codex 审核设计**（审核要点：架构合理性、接口规范、扩展性、安全）
4. 调整后 `vibedev_specs_design_confirmed()`

---

## 阶段 4: 任务规划 + Codex 审核

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 4: 任务规划'
```

1. `vibedev_specs_tasks_start()`
2. 拆分任务，标注角色（后端/前端）
3. **Codex 审核任务列表**（审核要点：覆盖度、依赖、粒度）
4. 调整后 `vibedev_specs_tasks_confirmed()`

---

## 阶段 5: 逐模块执行 + 模块审查

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 5: 模块执行'
```

1. `vibedev_specs_execute_start()`
2. 按任务类型分发：
   - 后端 → CC 自己写
   - 前端 → Gemini（**分批，每批 ≤10KB**）
   - 测试/文档 → Gemini（**分批，每批 ≤10KB**）
3. **每个模块完成后 → `codex review --uncommitted`**（Codex 自己读 diff）
4. 有 Critical/Major → 修复 → 重新审查

**Gemini 分批规则**：
- 单次 prompt ≤10KB（超过会返回空）
- 大任务拆分为多次调用
- CC 负责汇总结果

---

## 阶段 6: 最终联合审查

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 6: 最终审查'
```

1. `codex review --uncommitted`（Codex 全局审查，自己读 diff）
2. Claude 综合评估
3. 输出 Critical/Major/Minor + 评分(1-10)

---

## 阶段 7: 迭代修复

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 7: 迭代修复'
```

```
WHILE has_critical_or_major AND iteration < 5:
    Claude 修复 → Codex 重审 → 评估
IF iteration >= 5: 询问用户
```

**满意标准**：Critical=0 且 Major=0

---

## 阶段 8: 交付

```bash
~/.claude/bin/ccg-report stage "$CCG_TASK_ID" '阶段 8: 交付'
~/.claude/bin/ccg-report done "$CCG_TASK_ID" 'CCG Vibedev 完成'
```

生成完成报告（功能、模块数、迭代次数、评分）。

---

## 关键规则

1. **文档级审查** - 需求/设计/任务各阶段 Codex 审核
2. **模块级审查** - 每完成一个模块立即审查
3. **最终联合审查** - Codex + Claude
4. **自动迭代** - 有问题自动修复重审
5. **迭代上限** - 默认 5 次
6. **Gemini 不审查** - 只负责代码生成
7. **外部模型零写入权限** - 文件操作由 Claude 执行
