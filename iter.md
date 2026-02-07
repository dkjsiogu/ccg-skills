---
description: '轻量迭代：快速分析 + 精准分发'
---

# CCG Iter

$ARGUMENTS

---

## 规则

- 前端 → Gemini | 后端 → CC
- 重要修改（安全/权限）→ Codex 审核
- >200 行 → 建议 /ccg:major

---

## 执行

```bash
CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'Iter: $ARGUMENTS')
```

0. **配置加载**：Read `.ccg/workflow.json`（无则跳过）
1. **分析**（30s）：Grep 定位 + Read 上下文
2. **路由**：
   - 后端/逻辑/API → CC 自己写
   - 前端/UI/样式 → Gemini（>10KB 分批）
   - 重要修改 → CC 写 + Codex 审核
3. **验证**：运行测试

### 汇报（各模型完成后）

```bash
# CC 完成后端
~/.claude/bin/ccg-report msg claude WorkReport "后端修改完成：$变更描述 | 文件=N"

# Gemini 完成前端
~/.claude/bin/ccg-report msg gemini WorkReport "前端修改完成：$变更描述 | 文件=N"

# Codex 审核完成（如有）
~/.claude/bin/ccg-report msg codex ReviewReport "审核完成：$结论 | Critical=N, Major=N"
```

```bash
~/.claude/bin/ccg-report done "$CCG_TASK_ID" 'Iter 完成'
```

输出：变更描述 | 类型 | 执行者 | 审核状态
