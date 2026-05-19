# Embedded-Dev 7 Subagents

> 比赛模式 v2 的 7 角色，按 [VoltAgent](https://github.com/VoltAgent/awesome-claude-code-subagents) 标准格式拆成 7 个可独立调用的 Claude Code subagent。
>
> 优势：每个 subagent 在独立 context 中跑，主线 [ARCH] 收紧凑回传，可并行派多个，主上下文不爆。

---

## 安装方式

### 方式 1：本机全局（推荐 — Claude Code 立即识别）

把 7 个 .md 文件复制 / 软链到 `~/.claude/agents/`：

```bash
# Windows (Git Bash)
mkdir -p "$HOME/.claude/agents"
cp "$HOME/.claude/skills/embedded-dev/agents/embedded-"*.md "$HOME/.claude/agents/"

# 或软链（推荐，源文件改动自动同步）
ln -sf "$HOME/.claude/skills/embedded-dev/agents/embedded-arch.md" "$HOME/.claude/agents/"
# 重复其他 6 个
```

### 方式 2：项目本地

```bash
# 在你的工程目录下
mkdir -p .claude/agents
cp "$HOME/.claude/skills/embedded-dev/agents/embedded-"*.md .claude/agents/
```

### 方式 3：不安装

如果不想正式注册到 Claude Code，可以直接当作"prompt 模板"用：在 Claude Code 里用 Agent 工具时 `subagent_type=general-purpose`，prompt 复制对应 .md 文件的内容。

---

## 7 个 Subagent 速查

| Subagent | 用途 | 何时派 | 默认模型 |
|---|---|---|---|
| `embedded-arch` | 主控 / 路由 / 决策门 / 集成 | 任何竞赛题（必派）| opus |
| `embedded-drv` | 全部外设驱动（GPIO/UART/SPI/I2C/ADC/RTC/DMA/PWM）| 任何题（必派）| sonnet |
| `embedded-alg` | 应用层算法 / CLI / 状态机 / 编解码 | 任何题（必派）| sonnet |
| `embedded-qa` | 静态检查 / MIL/SIL/PIL / 5 元组验收 | 任何题（必派）| sonnet |
| `embedded-report` | 报告 / 答辩 why-evidence | 任何题（必派）| sonnet |
| `embedded-matlab` | 算法仿真 / `.h` 导出 | MAIN ≠ SYSTEM 或 SYSTEM 含 FFT/RF 标签 | opus |
| `embedded-vision` | 摄像头 / 视觉处理 | TAGS 含 `VISION` | sonnet |

最少 4 必派（ARCH/DRV/ALG/QA），最多 7 全派。按 `refs/competition-task-router.md` §2 决定。

---

## 用法示例

### 示例 1：派单个 Agent

```python
# Claude Code 主线里
Task(
    subagent_type="embedded-matlab",
    description="LQR 设计 + 仿真",
    prompt="""
    硬件：STM32F407 + IMU MPU6050
    任务：为两轮平衡车设计 4 状态 LQR。
    系统：x = [θ, θ̇, x, ẋ]，控制 u = 力矩
    指标：闭环带宽 5 Hz，超调 < 15%
    交付：lqr_gains.h（用 export_gains_to_c.py 导出）
    """
)
```

### 示例 2：N-Agent 并行（同一消息派多个）

```python
# 一条消息里同时派 5 个 Agent
派 Task(subagent_type="embedded-matlab", prompt=<MATLAB任务>)
派 Task(subagent_type="embedded-drv", prompt=<DRV任务>)
派 Task(subagent_type="embedded-alg", prompt=<ALG任务>)
派 Task(subagent_type="embedded-qa", prompt=<QA任务>)
派 Task(subagent_type="embedded-report", prompt=<REPORT任务>)
```

5 个 Agent 各自独立 context 跑，主线收齐紧凑回传后做决策门。

---

## 与 SKILL.md 协作

`refs/competition-ai-max-workflow.md` 的 Agent prompt 模板**仍然有效**（描述各 Agent 该干什么）。安装本目录后，你可以**直接用 subagent_type 调用**，不用每次手贴 prompt：

| 旧方式（手贴 prompt）| 新方式（subagent_type）|
|---|---|
| `Task(subagent_type=general-purpose, prompt=贴 §2.2 模板)` | `Task(subagent_type=embedded-matlab, prompt=具体任务)` |

新方式优势：
- prompt 模板封装在 .md 文件里，调用方只传具体任务
- 独立 context = 主线压力小
- VoltAgent 标准格式 = 可复用 / 可贡献回社区

---

## 卸载

```bash
rm -f "$HOME/.claude/agents/embedded-"*.md
```

不影响 `~/.claude/skills/embedded-dev/agents/` 源文件。

---

## 关联资源

- **主流程**：`modes/competition.md`
- **快速通道**：`refs/competition-quickstart-1page.md`
- **题型路由**：`refs/competition-task-router.md`
- **完整 prompt 模板**：`refs/competition-ai-max-workflow.md`
- **5 元组验收**：`refs/competition-scoring-checklist-template.md`
- **VoltAgent 仓库**（参考标准）：https://github.com/VoltAgent/awesome-claude-code-subagents
