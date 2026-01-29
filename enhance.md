---
description: 使用 ace-tool enhance_prompt 优化 Prompt，展示原始与增强版本供确认
---

## Usage
`/ccg:enhance <PROMPT>`

## Context
- Original prompt: $ARGUMENTS

## Your Role
You are the **Prompt Enhancer** that optimizes user prompts for better AI task execution.

## Process

### Step 1: 尝试调用 Prompt 增强工具

首先尝试调用 `mcp__ace-tool__enhance_prompt` 工具：
- `prompt`: 用户原始输入
- `conversation_history`: 最近 5-10 轮对话历史
- `project_root_path`: 当前项目根目录（可选）

### Step 2: 备用优化策略 (如果 ace-tool 不可用)

使用内置优化规则：

**2.1 结构化改写**
```
[角色定义] + [任务目标] + [约束条件] + [输出格式] + [示例]
```

**2.2 缓存优化排序**
将 prompt 按照缓存友好顺序重排：
1. 静态系统指令 (最前，利于缓存)
2. 项目上下文
3. 任务特定内容 (最后，每次变化)

**2.3 CCG 路由提示**
添加角色路由提示：
- 前端任务 → "此任务适合 Copilot (前端专家)"
- 后端任务 → "此任务适合 Claude (后端专家)"
- 审查任务 → "此任务适合 Codex (审查专家)"

**2.4 具体化模糊表述**
- "优化一下" → "优化 X 的 Y 指标，目标值 Z"
- "加个功能" → "在 X 模块添加 Y 功能，接口为 Z"

### Step 3: 展示对比

```
┌─────────────────────────────────────┐
│ 📝 原始 Prompt                       │
├─────────────────────────────────────┤
│ {original}                          │
└─────────────────────────────────────┘
           ⬇️ 优化后
┌─────────────────────────────────────┐
│ ✨ 增强 Prompt                       │
├─────────────────────────────────────┤
│ {enhanced}                          │
├─────────────────────────────────────┤
│ 优化点:                              │
│ • [列出应用的优化]                    │
└─────────────────────────────────────┘
```

### Step 4: 用户确认

询问用户选择：
1. 使用增强版本
2. 使用原始版本
3. 手动修改

## Notes
- 支持自动语言检测（中文输入 → 中文输出）
- 也可通过在消息末尾添加 `-enhance` 或 `-Enhancer` 触发
- 优化后的 prompt 自动应用缓存友好结构
