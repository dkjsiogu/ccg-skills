# CCG 项目类型检测

智能检测当前项目类型，自动加载对应的角色配置。

## 执行流程

### Step 1: 扫描项目特征

检查以下文件/目录来判断项目类型：

| 特征文件 | 项目类型 |
|---------|---------|
| `build.gradle` + `src/main/resources/mcmod.info` | minecraft-mod (Forge) |
| `fabric.mod.json` | minecraft-mod (Fabric) |
| `*.unity` / `ProjectSettings/` | game-unity |
| `project.godot` | game-godot |
| `requirements.txt` + `*.ipynb` | data-science |
| `platformio.ini` / `*.ino` | embedded |
| `package.json` + `src/` | web-app (默认) |

### Step 2: 加载配置

从 `~/.claude/.ccg/project-types.toml` 加载对应配置。

### Step 3: 生成项目配置

在项目根目录创建 `.ccg/project.toml`：

```toml
[project]
type = "detected-type"
detected_at = "2026-01-29"

[roles]
# 从 project-types.toml 继承
```

### Step 4: 报告配置

输出检测结果和角色分配：

```
📁 项目类型: Minecraft Mod (Forge)
📋 角色分配:
   - Claude: mod-logic, events, commands, api-integration
   - Gemini: animation, texture-config, model-json, recipes
   - Codex: review, compatibility, performance
```

## 手动覆盖

如果自动检测不准确，可以手动指定：

```bash
/ccg:detect-project --type minecraft-mod --framework forge
```

## 输出

检测完成后，后续的 CCG 命令会自动使用正确的角色配置。
