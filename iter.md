---
description: 'CCG 轻量迭代：快速分析 + 精准分发，不走完整 Vibedev'
---

# CCG Iter - 后续迭代

$ARGUMENTS

---

## ⛔ 强制规则

1. **前端任务必须调用 Copilot** - 使用 Bash 执行 codeagent-wrapper
2. **重要修改必须调用 Codex 审核** - 安全/权限/支付相关
3. **CC 可以自己写后端代码** - 这是最快的

---

## 快速调用模板

### 调用 Copilot (前端)
```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend gemini --lite -p "任务描述" "$PWD"
```

### 调用 Codex (审核)
```bash
/home/dkjsiogu/.claude/bin/codeagent-wrapper --backend codex --lite -p "审核请求" "$PWD"
```

---

**前置条件**: 已运行 `/ccg:init` (最好也运行过 `/ccg:major`)

轻量级迭代流程，CC 快速分析变更需求，精准分发给对应角色。

## 使用方法

```bash
/ccg:iter <变更描述>
```

## 与 Major 的区别

| 方面 | Major | Iter |
|-----|-------|------|
| Vibedev | 完整流程 | 跳过 |
| 需求收集 | 结构化引导 | 快速确认 |
| 任务分解 | 详细规划 | 简单拆分 |
| 审核 | 任务+代码双审 | 按需审核 |
| 适用场景 | 新功能开发 | Bug修复/小改动 |

---

## 执行流程

### Step 0: 向 Monitor 报告

**向 Monitor 报告迭代启动**：

```
Bash({
  command: "CCG_TASK_ID=$(~/.claude/bin/ccg-report start 'CCG Iter: $ARGUMENTS') && echo \"CCG_TASK_ID=$CCG_TASK_ID\"",
  timeout: 5000,
  description: "报告 CCG Iter 到 Monitor"
})
```

**保存返回的 CCG_TASK_ID，完成后需要用它来更新状态。**

```
Bash({
  command: "~/.claude/bin/ccg-report running '$CCG_TASK_ID'",
  timeout: 5000,
  description: "更新状态为运行中"
})
```

### Step 1: CC 快速分析 (30秒内)

**CC 使用自己的工具快速理解变更**：

```
1. 解析用户描述，提取关键词
2. Grep 搜索相关代码位置
3. Read 相关文件了解上下文
4. 判断变更类型和影响范围
```

**变更类型判定**：
| 类型 | 特征 | 路由 |
|-----|------|------|
| Bug 修复 | 错误、崩溃、异常 | CC 自己修 或 Copilot |
| 样式调整 | UI、CSS、布局 | → Copilot |
| 逻辑修改 | 业务逻辑、算法 | CC 自己做 |
| 性能优化 | 慢、卡、内存 | CC 分析 + Codex 审核 |
| 重构 | 结构、模式 | CC 设计 + Codex 审核 |
| 新增小功能 | 添加、增加 | 按前后端分发 |

### Step 2: 快速决策

**CC 根据分析结果决定**：

```
┌─ 后端/逻辑/API/配置
│   └─ CC 自己写 (最快)
│
├─ 前端/UI/样式
│   └─ 分发给 Copilot
│
├─ 重要修改 (安全/权限/支付)
│   └─ CC 写完 → Codex 审核
│
└─ 复杂修改 (> 200 行 或 架构变更)
    └─ 建议升级到 /ccg:major
```

**CC 是主力**: 大部分代码 CC 自己写，只有前端任务分发给 Copilot。

### Step 3: 执行 (按需分发)

#### 3.1 CC 自己执行 (简单后端修改)
```
CC 直接:
1. 定位文件
2. 修改代码
3. 验证修改
```

#### 3.2 分发给 Copilot (前端任务)
```
codeagent-wrapper --backend gemini --lite -p "
## 快速任务

### 上下文
{CC 预读的相关代码片段}

### 任务
{具体修改描述}

### 约束
- 保持现有代码风格
- 不改动无关代码
- 输出完整修改后的代码
"
```

#### 3.3 请求 Codex 审核 (重要修改)
```
codeagent-wrapper --backend codex --lite -p "
## 快速审核

### 变更
{git diff 或修改内容}

### 审核重点
{根据变更类型指定}

简洁回复：通过/问题列表
"
```

### Step 4: 快速验证

- 运行相关测试 (如有)
- 确认修改生效
- **更新 Monitor 任务状态为完成**：
  ```
  Bash({
    command: "~/.claude/bin/ccg-report done '$CCG_TASK_ID' 'CCG Iter 完成: $ARGUMENTS'",
    timeout: 5000,
    description: "报告 CCG Iter 完成到 Monitor"
  })
  ```
- 报告完成

---

## 输出格式

```markdown
## CCG Iter 完成

### 变更: {描述}
### 类型: {bug修复/样式/逻辑/...}
### 执行者: {CC/Copilot/协作}
### 审核: {跳过/Codex通过/...}

### 修改文件
- {file1}: {变更摘要}
- {file2}: {变更摘要}

### 验证
- {测试结果或手动验证}
```

---

## CC 快速迭代原则

1. **速度优先**: 简单任务直接做，不要过度流程
2. **按需分发**: 不是所有任务都需要多模型协作
3. **按需审核**: 小修改可跳过审核，重要修改必须审核
4. **升级意识**: 发现复杂度超预期时，建议升级到 Major

---

## 示例

### 示例 1: 简单 Bug
```
用户: /ccg:iter 登录按钮点击没反应

CC 分析:
→ 前端问题，UI 交互
→ Grep 搜索 "登录" 相关代码
→ 定位到 LoginButton.tsx

决策: 分发给 Copilot
执行: codeagent-wrapper --backend gemini --lite ...
审核: 跳过 (简单修复)
完成: 2分钟
```

### 示例 2: 逻辑修改
```
用户: /ccg:iter 用户注册时需要验证邮箱格式

CC 分析:
→ 后端验证逻辑
→ 定位到 user.service.ts
→ 简单正则验证

决策: CC 自己做
执行: CC 直接添加验证代码
审核: 跳过 (简单逻辑)
完成: 1分钟
```

### 示例 3: 重要修改
```
用户: /ccg:iter 修改用户权限判断逻辑

CC 分析:
→ 安全相关，重要逻辑
→ 影响多处鉴权

决策: CC 做 + Codex 审核
执行: CC 修改权限逻辑
审核: 必须经过 Codex
完成: 5分钟
```
