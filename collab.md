# CCG 跨角色协作 - 规范驱动开发

处理需要一个角色为另一个角色提供规范/指导的场景。

典型场景：
- Minecraft mod: Claude 定义动画 API → Copilot 生成动画配置
- Unity 游戏: Claude 设计系统接口 → Copilot 实现着色器
- Web 应用: Claude 定义 API 契约 → Copilot 实现前端调用

## 参数

```
$ARGUMENTS: <source-role>:<target-role> <task-description>

示例:
/ccg:collab claude:copilot "为自定义生物实体生成行走和攻击动画"
/ccg:collab claude:copilot "根据 API 规范实现登录表单"
```

## 执行流程

### Phase 1: 规范生成 (Source Role)

使用 **$1** (如 claude) 生成目标任务的规范：

**Prompt 模板**:
```
你是 {project_type} 项目的 {source_role} 专家。

任务: {task_description}

请为 {target_role} 生成详细的实现规范，包括：

1. **接口定义**: 需要实现的具体接口/格式
2. **约束条件**: 必须遵守的规则和限制
3. **数据结构**: 输入/输出的数据格式
4. **示例**: 至少一个完整的实现示例
5. **注意事项**: 常见陷阱和最佳实践

输出格式: Markdown 规范文档
```

调用: `codeagent-wrapper --backend {source_backend} -p "{prompt}"`

### Phase 2: Codex 规范审查

使用 **codex** 审查生成的规范：

**审查要点**:
- 规范是否完整清晰
- 是否有歧义或遗漏
- 是否符合项目约定

调用: `codeagent-wrapper --backend codex -p "审查以下规范..."`

### Phase 3: 实现生成 (Target Role)

使用 **$2** (如 copilot/gemini) 根据规范生成实现：

**Prompt 模板**:
```
你是 {project_type} 项目的 {target_role} 专家。

## 规范文档
{spec_from_phase1}

## 任务
根据上述规范，生成完整的实现代码/配置。

要求:
1. 严格遵循规范中的接口定义
2. 包含所有必需的文件
3. 添加必要的注释
```

调用: `codeagent-wrapper --backend {target_backend} -p "{prompt}"`

### Phase 4: Codex 实现审查

使用 **codex** 审查实现是否符合规范：

**审查要点**:
- 实现是否符合规范
- 代码质量和最佳实践
- 潜在问题和改进建议

---

## Minecraft Mod 示例

```
/ccg:collab claude:copilot "为自定义僵尸实体生成动画"
```

### Phase 1 输出 (Claude):
```markdown
# 僵尸实体动画规范

## 1. 动画状态
- `idle`: 待机状态，20帧循环
- `walk`: 行走状态，40帧循环，速度 > 0 时触发
- `attack`: 攻击状态，15帧单次，攻击事件触发

## 2. 文件结构
animations/
└── custom_zombie.animation.json

## 3. JSON 格式
{
  "format_version": "1.8.0",
  "animations": {
    "animation.custom_zombie.idle": { ... },
    "animation.custom_zombie.walk": { ... },
    "animation.custom_zombie.attack": { ... }
  }
}

## 4. 动画曲线
- 使用 "linear" 插值
- 骨骼: head, body, left_arm, right_arm, left_leg, right_leg

## 5. 触发条件 (在 Java 代码中)
registerAnimationController(new AnimationController<>(this, "controller", 0, state -> {
    if (state.isMoving()) return PlayState.walkAnim;
    if (this.isAttacking) return PlayState.attackAnim;
    return PlayState.idleAnim;
}));
```

### Phase 3 输出 (Copilot):
```json
// animations/custom_zombie.animation.json
{
  "format_version": "1.8.0",
  "animations": {
    "animation.custom_zombie.idle": {
      "loop": true,
      "animation_length": 1.0,
      "bones": {
        "head": {
          "rotation": {
            "0.0": [0, 0, 0],
            "0.5": [5, 0, 0],
            "1.0": [0, 0, 0]
          }
        }
        // ... 完整实现
      }
    }
  }
}
```

---

## 输出

1. 规范文档 (保存到 `.ccg/specs/`)
2. 实现文件 (保存到项目对应目录)
3. 审查报告 (保存到 `.ccg/reviews/`)
