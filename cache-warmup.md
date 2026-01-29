# CCG Cache Warmup - 缓存预热

在开始任务前预热各CLI的缓存，提高后续操作的响应速度。

## 执行流程

### Step 1: 收集静态上下文

读取以下文件构建缓存基础：
1. 项目根目录的 `CLAUDE.md`
2. `.ccg/` 目录下的配置
3. 当前任务相关的设计文档

### Step 2: Claude 缓存预热

向 Claude 发送包含完整项目上下文的初始化请求：
- 使用 `codeagent-wrapper --backend claude`
- 发送: "分析项目结构，准备后续开发任务。回复OK即可。"
- 目的: 建立5分钟的 ephemeral cache

### Step 3: Codex 会话建立

创建 Codex 审查会话：
- 使用 `codeagent-wrapper --backend codex`
- 发送: "你是代码审查专家。接下来会有代码需要审查。回复OK。"
- 记录 session_id 用于后续复用

### Step 4: Copilot 上下文加载

通过 gemini wrapper 预热 Copilot：
- 使用 `codeagent-wrapper --backend gemini`
- 发送: "加载前端开发上下文，准备UI开发任务。回复OK。"

## 使用时机

1. **每日开始工作时** - 完整预热
2. **切换任务前** - 重新加载任务相关上下文
3. **缓存过期后** (约5分钟无操作) - 快速预热

## 缓存保持策略

为避免缓存过期，在长时间思考或等待时，可以发送心跳请求：
```
codeagent-wrapper --backend claude -p "继续"
```

## 输出

预热完成后报告各CLI的状态和session信息。
