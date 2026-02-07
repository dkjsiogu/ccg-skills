---
description: '问题诊断：多模型分析原因 → 记录 → 路由修复'
---

# CCG Diagnose

$ARGUMENTS

---

## 规则

- 前端问题 → Gemini 诊断 | 后端问题 → Codex 诊断
- 综合结论由 CC 汇总
- 诊断结果持久化到 `.ccg/issues/`
- 修复自动路由：≤200 行 → `/ccg:iter` | >200 行 → `/ccg:major`

---

## 执行

```bash
CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'Diagnose: $ARGUMENTS')
```

### Phase 1: 收集上下文

0. Read `.ccg/workflow.json`（无则跳过）
1. Grep/Read 定位问题相关文件
2. 收集错误日志、git diff、堆栈信息

### Phase 2: 多模型诊断（并行）

| 问题类型 | 诊断者 | 关注点 |
|---------|--------|--------|
| 后端/逻辑/API | Codex | 逻辑错误、安全漏洞、性能瓶颈 |
| 前端/UI/样式 | Gemini | 渲染问题、交互缺陷、兼容性 |
| 混合型 | 两者并行 | 各自视角，CC 交叉验证 |

### Phase 3: 综合结论

CC 汇总诊断结果：
- **根因**（Root Cause）
- **影响范围**（Impact Scope）
- **修复方案**（Suggested Fix）
- **预估规模**（行数/文件数）

### Phase 4: 记录

```bash
mkdir -p .ccg/issues
```

写入 `.ccg/issues/{timestamp}-{slug}.json`：

```json
{
  "timestamp": "ISO-8601",
  "description": "用户原始描述",
  "diagnosis": {
    "codex": "后端诊断结论",
    "gemini": "前端诊断结论"
  },
  "root_cause": "综合根因",
  "impact": "影响范围",
  "suggested_fix": "修复方案",
  "estimated_scope": "small|large",
  "related_files": [],
  "status": "diagnosed"
}
```

### Phase 5: 路由修复

- `estimated_scope: small`（≤200 行）→ 直接调用 `/ccg:iter` 执行修复
- `estimated_scope: large`（>200 行）→ 调用 `/ccg:major` 完整流程

修复完成后更新 issue 的 `status` 为 `fixed`。

```bash
~/.claude/bin/ccg-report done "$CCG_TASK_ID" 'Diagnose 完成'
```

输出：问题描述 | 根因 | 修复方式（iter/major） | 状态
