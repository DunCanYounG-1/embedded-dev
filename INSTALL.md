# 安装与依赖

> 本 skill 的目标是"clone 到 `~/.claude/skills/embedded-dev/` 即可基本工作"，但完整功能依赖几个外部组件。本文按**必装 / 推荐 / 可选**三级列出全部依赖，并说明每项缺失时的降级方案。
>
> **重要原则**：缺失任何依赖时，**hooks 会 fail open**（静默 exit 0），**主协议规则仍由 Claude 自行遵守**，不会卡死任何工作流。

---

## 0. 快速验证

clone 之后想确认哪些功能可用，跑这一条：

```bash
bash ~/.claude/skills/embedded-dev/hooks/verify-deps
```

它会逐项探测 Python / Git Bash / 兄弟 skill / shared 工具 / 网络可达性，输出"必装/推荐/可选"三级状态报告。**不在乎缺失的项不影响其他功能**。

---

## 1. 安装本 skill

### Linux / macOS

```bash
git clone https://github.com/DunCanYounG-1/embedded-dev ~/.claude/skills/embedded-dev
```

### Windows (PowerShell)

```powershell
git clone https://github.com/DunCanYounG-1/embedded-dev "$env:USERPROFILE\.claude\skills\embedded-dev"
```

### Windows (Git Bash)

```bash
git clone https://github.com/DunCanYounG-1/embedded-dev ~/.claude/skills/embedded-dev
```

---

## 2. 必装依赖（缺了核心 hook 不工作）

| 依赖 | 用途 | 安装方法 | 缺失时降级 |
|---|---|---|---|
| **Python 3.8+** | `hooks/session-start.py` 注入协议引导；`shared/project_detect.py` 工程画像探测 | Windows: <https://www.python.org/downloads/> 或 `winget install Python.Python.3`；macOS: `brew install python@3.12`；Linux: 系统包管理器 | SessionStart hook 静默失效；协议主流程仍由 Claude 手动遵守 |
| **Bash** | 4 个 bash hook (`check-memory-files` / `inject-context` / `remind-update` / `run-hook.cmd` polyglot) | Linux/macOS 原生；Windows 装 **Git for Windows** <https://git-scm.com/download/win>（自带 Git Bash） | 4 个 bash hook 静默失效；polyglot 自动 `exit 0`，协议主流程仍生效 |
| **Git** | clone / commit / 自动存档 | <https://git-scm.com/downloads> | Git 快照与回档机制失效；其他不受影响 |

**Windows 用户特别注意**：装 Git for Windows 时勾选"Git Bash Here"和"Add Git to PATH"。Claude Code 在 Windows 下的 hook command 默认通过 Git Bash 执行。

---

## 3. 推荐依赖（缺了关键功能降级，但能用）

### 3.1 兄弟 skill（操作执行层 — 26 个）

EXECUTE 阶段的"操作执行层兄弟 skill 路由"表（见 `SKILL.md` 第 4 阶段）依赖这些 skill 真正执行编译/烧录/调试/通信。缺失时 Claude 只能给出"操作建议"，无法直接执行。

| 类别 | skill 列表 | 缺失时降级 |
|---|---|---|
| **构建** | `build-cmake` `build-keil` `build-iar` `build-platformio` `build-idf` `build-makefile` | Claude 给出命令但需用户手动跑 |
| **烧录** | `flash-openocd` `flash-keil` `flash-platformio` `flash-idf` `flash-jlink` | 同上 |
| **GDB 调试** | `debug-gdb-openocd` `debug-jlink` `debug-platformio` | 同上 |
| **串口/总线** | `serial-monitor` `modbus-debug` `can-debug` `visa-debug` | 同上 |
| **分析** | `memory-analysis` `rtos-debug` `static-analysis` | 同上 |
| **驱动适配** | `peripheral-driver` `stm32-hal-development` | 退回 Claude 手动按 `refs/driver-porting.md` 流程 |
| **流水线** | `workflow` | 用户分步调用各 skill |
| **代码质量** | `simplify` | REVIEW 阶段质量检查由 Claude 手动 |
| **外部协作** | `codex` | 失去 GPT 视角的"双模型擂台"能力 |

**安装方法**：这 26 个 skill 都是独立的 Claude Code skill。如果你已经装了 [MICU/AI-Embedded-Toolkit](https://github.com/example/ai-embedded-toolkit) 之类的整合包，它们应该一起来。否则按需 `git clone` 单个 skill 仓库到 `~/.claude/skills/<name>/`。

> ⚠ 本 skill 不会替你装这 26 个兄弟 skill。它们是 embedded-dev 的"被调用方"，不是"被依赖方"。**没装的话只是 EXECUTE 阶段的执行环节降级，研究/创新/计划/审查四个阶段不受影响**。

### 3.2 shared/ 工具（工程画像 + 工具路径管理）

| 文件 | 用途 | 安装方法 | 缺失时降级 |
|---|---|---|---|
| `~/.claude/skills/shared/project_detect.py` | RESEARCH 阶段自动探测构建系统/芯片/产物 | clone <https://github.com/example/claude-skills-shared> 到 `~/.claude/skills/shared/` | Claude 手动按文件包含/API 调用/项目结构识别（见 RESEARCH 步骤 2） |
| `~/.claude/skills/shared/tool_config.py` | OpenOCD/Keil UV4/arm-gcc/J-Link 工具路径登记 | 同上 | 兄弟 skill 自己探测工具路径，可能需要用户指定 |

> 这两个脚本是各操作执行层兄弟 skill 共享的契约层。如果你装了任何一个 `build-*` / `flash-*` skill，通常 shared/ 会一起来。

### 3.3 grok-search（联网检索）

| 依赖 | 用途 | 安装方法 | 缺失时降级 |
|---|---|---|---|
| `~/.claude/skills/grok-search/` | 所有联网检索（驱动/报错/数据手册入口）首选工具 | clone <https://github.com/example/grok-search> + 在 `config.json` 填 API key + base_url（如 `https://www.micuapi.ai/v1`） | 自动 fallback 到 Claude 内置 WebSearch / WebFetch |

### 3.4 Context7 MCP（库 API 即时文档）

| 依赖 | 用途 | 安装方法 | 缺失时降级 |
|---|---|---|---|
| Context7 MCP 服务器 | STM32/ESP-IDF/HAL 等库 API 即时文档；超出离线 refs 覆盖时使用 | `claude mcp add --transport http context7 https://mcp.context7.com/mcp` | 离线 refs 覆盖不到时降级到 grok-search 联网检索 |

### 3.5 gh CLI（GitHub 检索）

| 依赖 | 用途 | 安装方法 | 缺失时降级 |
|---|---|---|---|
| GitHub CLI `gh` | 检索 EmbedSummary 等开源驱动库 | <https://cli.github.com/> 后 `gh auth login` | grok-search + WebSearch site:github.com |

---

## 4. 可选依赖（特定场景才需要）

| 依赖 | 触发场景 | 安装方法 |
|---|---|---|
| **Sequential Thinking MCP** | 引脚冲突 / DMA 分配复杂推理 | `claude mcp add` 加入 |
| **Document Skills** (`pdf` / `docx` / `xlsx` / `pptx`) | 解析数据手册 PDF / 原理图 / 报告 | <https://github.com/example/document-skills> |
| **agent-browser** | 在线数据手册页面交互、厂商 Web 配置器、截图留档 | <https://github.com/example/agent-browser> |
| **Embedded Debugger MCP** | 硬件实时联调 / 烧录 / 串口交互的 MCP 接口 | 视厂商而定 |
| **Mermaid CLI** | 文档生成里的流程图渲染 | `npm i -g @mermaid-js/mermaid-cli` |

---

## 5. 平台特定注意事项

### Windows

- **Git Bash 必装**：所有 bash hook 通过它执行（`run-hook.cmd` polyglot 自动找）
- **路径分隔符**：Python hook 已自动处理 `/c/Users/` / `/cygdrive/c/` / `/mnt/c/` → `C:\` 规范化
- **Python 3 命令名**：建议用 `python`（Windows 标准）。`python3` 在 Windows 上经常指向 Windows Store stub（无实际 Python）— polyglot 会自动绕过
- **行尾符**：仓库已通过 `.gitattributes` 锁 LF；不要本地改 git autocrlf 配置

### macOS

- **Python 3**：macOS 自带的 `python3` 通常够用，但建议装 `python@3.12` 保证版本一致
- **Bash 版本**：macOS 自带 bash 3.2（很旧），但 hook 脚本兼容；如需 bash 4+ 用 `brew install bash`

### Linux

- Python / Bash / Git 通常预装。重点是装 27 个兄弟 skill 和 grok-search
- 串口权限：`sudo usermod -a -G dialout $USER`（Debian/Ubuntu）

---

## 6. 验证最小可用集

clone 完成 + 装好 Python + Git Bash（Windows）即可启动以下功能：

✅ 立即可用：
- RIPER-5 五阶段协议（RESEARCH / INNOVATE / PLAN / EXECUTE / REVIEW 决策框架）
- 四文件磁盘记忆（项目规划清单/编辑清单/硬件资源表/研究发现）
- SessionStart hook 注入 6 条铁律
- PLAN/EXECUTE 三件套（Iron Law + Red Flags + Rationalization Table）
- 嵌入式分层架构规范（HAL/BSP/Driver/Middleware/Service/App）
- 所有 `refs/*.md` 离线知识库（API 速查、引脚规划、IMU 检查、Mahony AHRS、故障排查等）
- 5 个扩展模式（competition / datasheet-lookup / gd32-board / netlist-lookup / seekfree-lib / mcp-healthcheck）

⚠ 装了对应兄弟 skill 后可用：
- 真正执行编译/烧录/调试/串口/总线/分析等操作
- 工程画像自动探测（`project_detect.py`）

⚠ 装了对应 MCP/CLI 后可用：
- Context7 即时库文档
- gh CLI 仓库检索
- grok-search 联网搜索

---

## 7. 完全 minimal install（只用核心协议）

如果你只想用本 skill 的"研究方法论 + 分层架构规范 + 引脚规划检查 + 反自欺协议"，**只需要装 Python + Git Bash（Windows）+ clone 本仓库**。其他全部可以不装。RESEARCH/INNOVATE/PLAN/REVIEW 四个阶段对外部依赖近乎为零；只有 EXECUTE 真正跑命令时才需要兄弟 skill。
