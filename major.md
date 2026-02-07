---
description: '大型功能开发：Vibedev 规范 + 多模型协作'
---

# CCG Major

$ARGUMENTS

---

## 规则

1. **Codex 审核**：需求/设计/任务/代码
2. **Gemini 执行**：前端任务
3. **CC 执行**：后端任务

---

## Stage 0-1: 初始化 + 规范

```bash
~/.claude/bin/ccg-workflow init major '$ARGUMENTS' '待确认'
CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'Major: $ARGUMENTS')
~/.claude/bin/ccg-report running "$CCG_TASK_ID"
```

1. 检查 `.ccg/` → 无则快速扫描
2. Read `.ccg/workflow.json` 获取项目配置
3. Vibedev: Goal → Requirements → Design → Tasks
4. 每阶段 Codex 审核后 `vibedev_specs_*_confirmed`

**汇报**：
```bash
~/.claude/bin/ccg-report msg codex ReviewReport "规范审核完成：Goal/Req/Design/Tasks 已确认"
```

---

## Stage 2: 任务审核

Codex 审核完整任务列表，确认无遗漏。

**汇报**：
```bash
~/.claude/bin/ccg-report msg codex ReviewReport "任务审核完成：共 N 个任务 | 无遗漏确认"
```

---

## Stage 3: 执行

| 任务类型 | 执行者 | 备注 |
|---------|-------|------|
| 后端/API | CC | 直接写 |
| 前端/UI | Gemini | >30KB 分批 |
| 测试/文档 | Gemini | >30KB 分批 |

**自适应审查策略**：
- 前 3 个模块：`codex review --uncommitted` 强制审查
- 连续 3 个模块 Critical=0 且 Major=0 → 后续改为**抽查**（每 3 个模块审查 1 个）
- **关键模块例外**（安全/认证/支付/数据库迁移）：始终强制审查
- 任何模块出现 Critical → 恢复强制审查直到再次连续 3 个通过

**汇报**（各模块完成后）：
```bash
~/.claude/bin/ccg-report msg claude WorkReport "后端模块完成：$模块名 | 文件=N"
~/.claude/bin/ccg-report msg gemini WorkReport "前端模块完成：$模块名 | 文件=N"
~/.claude/bin/ccg-report msg codex ReviewReport "模块审核：$模块名 | Critical=N, Major=N"
```

---

## Stage 4-5: 审查 + 修复

1. `codex review --uncommitted`（Codex 自己读 diff）
2. Gemini 分批审查（<15KB/批）
3. Claude 综合：交叉验证 + 去重
4. 迭代修复直到 Critical=0 且 Major=0

**汇报**（每轮审查后）：
```bash
~/.claude/bin/ccg-report msg codex ReviewReport "全量审查 Round N：Critical=N, Major=N"
~/.claude/bin/ccg-report msg gemini ReviewReport "前端审查 Round N：UX=N, 性能=N, 设计=N"
~/.claude/bin/ccg-report msg claude TaskSummary "综合审查 Round N：待修复=N 项 | 可交付=是/否"
```

---

## Stage 6: 交付

**汇报**：
```bash
~/.claude/bin/ccg-report msg claude TaskSummary "Major 完成：$功能名 | 迭代=N 轮 | Codex评分=N"
```

```bash
~/.claude/bin/ccg-report done "$CCG_TASK_ID" 'Major 完成'
```

输出：功能名 | 工作分配 | Codex 评分 | 迭代次数
