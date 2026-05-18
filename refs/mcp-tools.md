# MCP 工具详细用法参考

> 本文件由主协议按需加载。两张总表（何时用哪个工具）在主协议中，此处提供**具体调用方式、降级策略和恢复要点**。

---


## Grok-Search（Python CLI Skill）— 主要网络搜索工具（最高优先级）

**适用场景**：所有需要联网检索的场景，包括搜索开源驱动、查报错解决方案、查竞赛经验帖、搜索数据手册下载链接、查询最新版本信息等。

**实现**：本地 Python CLI 脚本 `~/.claude/skills/grok-search/scripts/grok_search.py`，通过 OpenAI 兼容端点调用 Grok 模型。**不是 MCP 服务器**，只通过命令行调用。默认配置读自 `~/.claude/skills/grok-search/config.json`。

**唯一入口**：`--query "<检索词>"`。可选覆盖：`--model` / `--base-url` / `--api-key` / `--config` / `--timeout-seconds` / `--extra-body-json` / `--extra-headers-json`。

**使用方式**（Bash 直接调用）：

```bash
# 基本搜索
python ~/.claude/skills/grok-search/scripts/grok_search.py --query "STM32F103 SSD1306 OLED driver StdPeriph site:github.com"

# 临时切换模型（例如当前 channel 只剩某个模型可用）
python ~/.claude/skills/grok-search/scripts/grok_search.py --query "..." --model grok-4.20-expert
```

**输出格式**：stdout 打印一行 JSON，关键字段：
- `ok`：布尔，是否成功
- `content`：模型归纳后的答案（解析失败时为空字符串）
- `sources`：URL 列表（含 title/snippet 可能为空）
- `raw`：原始模型输出 — 当 `content` 解析失败时作为兜底，**务必读 raw**
- `usage` / `elapsed_ms`：诊断
- `error` / `detail`：失败时的错误信息

**推荐搜索关键词模板**：
- 驱动搜索：`STM32F103 SSD1306 OLED driver StdPeriph site:github.com`
- 报错排查：`STM32 HardFault handler cause and solution`
- 竞赛经验：`Chinese Electronic Design Contest PID motor control STM32`
- 数据手册：`STM32F103C8T6 Reference Manual PDF download site:st.com`
- 版本查询：`STM32CubeIDE latest version 2026`

**调用原则**（八荣八耻）：
- 以瞎猜接口为耻，以认真查询为荣 → 遇到不确定的信息，先用 grok-search 搜索
- 以臆想业务为耻，以人类确认为荣 → 搜索结果需引用来源（看 `sources`），不凭空杜撰

**容灾备份**：
- 返回 `ok=false` 且 `error=HTTP 503`（`No available channel under group ...`）时：先用 `--model` 切换同分组的其他可用模型重试；若仍失败再走下一级降级
- grok-search 不可用时：第一降级 Claude WebSearch → 第二降级用户手动搜索 → Sequential Thinking 仅用于整理已获取证据（禁止用于事实检索）

---
## Context7 MCP — 固件库文档查询

适用：不确定 HAL / StdPeriph / ESP-IDF / Arduino 某个函数的参数含义、返回值、初始化顺序时

```
使用流程：
1. mcp__context7__resolve-library-id  libraryName="STM32 HAL"
2. mcp__context7__query-docs  libraryId=<返回的ID>  query="ADC DMA 初始化配置"
```

支持查询的嵌入式库：`STM32 HAL`、`STM32 StdPeriph`、`ESP-IDF`、`Arduino`、`FreeRTOS`、`CMSIS`

---

## 浏览器自动化 — `agent-browser` 方法论优先，`/playwright-skill` 兜底

**适用场景**：任务真实发生在网页而不是代码仓库/终端里，例如：
- 打开芯片数据手册网页，截图提取时序参数
- 浏览器访问厂商网站，提取引脚复用表
- 自动填写在线工具（如 STM32CubeMX 在线版）
- 登录后台抓取 OTA/日志页面信息并留档

**前提边界**：
- `agent-browser` 当前不是本 skill 的内置依赖；只有环境已安装时才调用
- 若未安装或不可用，回退到 `/playwright-skill` 或手工浏览
- 不得在未读取当前版本帮助的情况下猜测 CLI 命令或参数

**推荐流程（借鉴 `agent-browser`）**：
1. 先读取当前 CLI 对应的技能说明/帮助，再开始操作  
   `agent-browser skills get agent-browser`
2. 打开页面后先抓可交互快照，再做动作  
   `open → snapshot -i/--json → 解析 refs`
3. 优先使用快照里的元素引用（refs）操作，不直接赌 CSS 选择器
4. 页面导航、刷新、弹窗或提交表单后，**必须重新抓取快照**，禁止复用旧 refs
5. 页面结构不清晰、包含图标按钮或画布元素时，先做 `screenshot --annotate` 再决策
6. 涉及登录态时，优先使用隔离会话/持久会话；一个站点一个 session，任务完成显式 `close`
7. 涉及后台、管理台或可能有破坏性的页面时，优先限制允许域名、动作策略和确认项

**嵌入式任务中的落地建议**：
- 在线 pinmux / datasheet 页面：`snapshot` + `get text` + `screenshot --annotate`
- OTA/监控后台：独立 `session-name`，保留截图或 trace 作为证据
- 需要多步页面操作且无中间解析时，可用批处理；需要读取快照决定下一步时，逐步执行

**兜底策略**：
- `agent-browser` 不可用：改用 `/playwright-skill`
- 页面高度动态、需要更复杂断言：优先 `/playwright-skill`
- 只需读取静态网页文本：优先 `web_fetch` / Document Skills，而不是启动浏览器

---

## GitHub 操作 — 已由 `gh` CLI 替代

> GitHub MCP 的功能已由 `gh` 命令行工具替代，直接在 Bash 中调用即可。
>
> 常用命令：
> - `gh search repos "STM32F103 SSD1306 driver"` — 搜索驱动仓库
> - `gh api repos/owner/repo/contents/path` — 读取仓库文件内容
> - `gh repo clone owner/repo` — 克隆仓库到本地评估

---

## Document Skills — 主要文档阅读工具（最高优先级）

**适用场景**：所有需要读取/处理文档的场景，包括芯片数据手册 PDF、引脚映射 Excel 表、技术规格 Word 文档等。

**核心优势**：
- 基于 `uv run` + PEP 723，无需手动安装 Python 依赖
- 支持 PDF/DOCX/XLSX/PPTX 四种格式，覆盖嵌入式开发全部文档需求
- PDF 支持文本提取、表格提取、OCR 扫描页、表单填写、合并拆分

**四个 Skill**：

| Skill | 触发方式 | 嵌入式开发典型用途 |
|-------|---------|-------------------|
| `/pdf` | 处理 PDF 文件 | 芯片数据手册提取（寄存器表、引脚图、电气参数、时序图） |
| `/xlsx` | 处理 Excel 文件 | 引脚映射表、BOM 清单、测试数据分析 |
| `/docx` | 处理 Word 文件 | 技术规格文档、设计文档读写 |
| `/pptx` | 处理 PPT 文件 | 竞赛答辩、技术方案演示文稿 |

**PDF 快速用法**（最常用）：

```python
# 使用 /pdf skill — 直接告诉 Claude "读取这个 PDF" 即可自动调用
# 或手动执行：
from pypdf import PdfReader
reader = PdfReader("datasheet.pdf")
text = reader.pages[0].extract_text()  # 提取指定页文本

# 表格提取（引脚复用表、寄存器位域表）
import pdfplumber
with pdfplumber.open("datasheet.pdf") as pdf:
    tables = pdf.pages[45].extract_tables()
```

**容灾备份**：
- Document Skills 不可用时，降级到 Claude 内置 Read 工具（支持 PDF，最多 20 页/次）

**系统依赖**（按需安装）：
- 必需：`poppler`（已安装）
- 可选：`pandoc`（docx 转换）、`tesseract`（OCR）、`libreoffice`（格式转换）、`qpdf`（PDF 修复）

---

## Sequential Thinking MCP — 结构化决策推理

适用：架构设计阶段的复杂决策，需要逐步推理、假设验证、分支比较

```
推荐使用场景：
- 引脚冲突分析：多个外设竞争同一 GPIO 时的重映射决策
- DMA 通道分配：通道冲突时评估中断轮询替代方案的性能影响
- 中断优先级排布：多个时间敏感任务的抢占关系推理
- 故障排查：HardFault / 外设不响应等问题的根因分析链
```

---

## 外部技能方法论借鉴（按需，不默认调用）

### `find-skills` — 先检索成熟能力，再决定是否造轮子

适用：需要引入浏览器自动化、测试、文档、研究、发布等**非固件核心**能力时。

**借鉴规则**：
- 先看 leaderboard / 安装量 / 来源信誉 / GitHub Stars，再决定是否引入
- 官方来源优先（如 `vercel-labs`、`anthropics`、`microsoft`）
- 低安装量、来源不明的 skill 只能作为试验选项，不能直接写入主协议
- 找不到合适 skill 时，先用现有通用能力完成任务，再评估是否沉淀新流程

### `summarize` — 先压缩长资料，再进入分析和决策

适用：超长网页、数据手册网页、视频讲解、播客转录、超长 issue/RFC。

**借鉴规则**：
- 先输出紧凑摘要，再按需展开局部，不把大段原文直接塞入主上下文
- 默认产出结构化摘要：`结论 / 证据 / 待确认 / 后续动作`
- 内容特别长时，先做短摘要，再决定是否需要二次提取表格、参数或时序

### `code-refactoring` — 小步重构，行为保持

适用：需要把 `main.c`、长函数、重复初始化序列、混杂职责代码拆分成模块 API。

**借鉴规则**：
- 一次只做一类重构，不把“重构 + 新功能”混在一起
- 先明确现有输入/输出/副作用，再拆函数和收敛参数
- 拆分后优先验证行为是否保持一致，再做下一步整理

---

## Embedded Debugger MCP — 实时硬件调试（需连接开发板）

适用：连接 J-Link / ST-Link / DAPLink 后进行实时调试

```
核心能力（22 个工具）：
- 固件烧录：flash_firmware → 直接烧录 .hex/.bin 到目标板
- 内存读写：read_memory / write_memory → 实时查看/修改寄存器值
- 断点调试：set_breakpoint / step → 单步跟踪执行流程
- RTT 双向通信：rtt_read / rtt_write → 替代 UART printf 调试
- 支持芯片：ARM Cortex-M (M0~M33)、RISC-V、STM32 全系列、nRF、ESP32-C
```

**注意**：此工具仅在物理连接开发板时可用，纯代码编写阶段无需调用。

---

## Serial MCP / mcp2serial — 串口通信

适用：通过 UART 与目标板交互，读取传感器数据、发送控制命令

```
Serial MCP Server（Rust 高性能版）：
- list_ports → 列出所有可用串口
- connect → 连接指定串口（波特率、数据位等）
- send / receive → 收发数据
- 适用场景：高频数据采集、实时串口监控

mcp2serial（Python 轻量版）：
- 同样功能，安装更简便（uvx 一键运行）
- 适用场景：简单串口调试、快速验证通信
```

---

## 工具降级与恢复策略

主协议只保留"优先级总表"。当需要具体降级方案、恢复条件和命令模板时，读取本节。

### 降级矩阵

| 主工具 | 不可用时的备用方案 | 降级影响 | 恢复条件 |
|--------|-------------|---------|---------|
| **Context7 MCP** | 本地 refs / 官方 PDF → grok-search 搜索官方文档 | 需人工验证搜索结果 | Context7 服务恢复 |
| **grok-search (CLI Skill)** | Claude WebSearch → 手动查询 | 搜索质量下降 | 网络恢复 + API 可用 |
| **gh CLI** | grok-search (`site:github.com`) → 手动访问 GitHub | GitHub 细节信息下降 | API 配额重置 |
| **Sequential Thinking MCP** | 人工推理 + WebSearch | 推理效率下降 | 服务恢复 |
| **Document Skills** | Claude 内置 Read 工具 → grok-search 搜索 | 文档处理能力降级 | Skill 恢复 |
| **Embedded Debugger MCP** | 串口日志 / 断言 / 寄存器转储 / 手工烧录 | 无法在线调试 | 硬件连接恢复 |

### 降级执行流程

1. 记录降级事件：工具名称、错误原因、时间、备份方案
2. 用备份方案完成原任务
3. 评估结果质量，不足时再追加搜索或请求用户资料
4. 在下一次同类任务前重新探测主工具是否恢复

### 常见场景模板

**1. Context7 不可用**

- STM32：先查 `refs/stm32-hal-api.md`、`refs/stm32-stdperiph-api.md`
- 其他平台：先用 grok-search 搜索官方 API 文档站
- 禁止凭记忆猜测函数签名

**2. grok-search 不可用**

- 第一降级：Claude WebSearch
- 第二降级：让用户手动搜索并给出链接/关键词
- Sequential Thinking 只能整理已拿到的证据，不能替代事实搜索

**3. gh CLI 不可用**

- 第一降级：grok-search + `site:github.com`
- 第二降级：用户手动访问 GitHub

**4. Document Skills 不可用**

- 第一降级：Claude 内置 Read 工具
- 第二降级：grok-search 搜索在线版数据手册或官方文档站

### 自动恢复原则

- grok-search、gh CLI：每次同类任务前重试一次
- Context7、Sequential Thinking：每 3-5 个相关任务后重试一次
- Document Skills：每次读文档前重试一次
