# 逐飞开源库管理模式

> 触发词：`逐飞` / `seekfree` / `英飞凌库`
>
> 用途：使用英飞凌（Infineon）芯片时，优先从逐飞科技开源库获取成熟的底层驱动和示例代码，并管理本地库缓存。
>
> **核心原则**：英飞凌芯片开发首选逐飞库，本地有缓存就用缓存，没有就下载后缓存，避免重复下载。

---

## 库索引文件

所有已下载的逐飞库信息统一记录在项目目录下的 `逐飞库索引.md` 中。

**文件位置**：`<项目根目录>/逐飞库索引.md`

**格式**：

```markdown
# 逐飞开源库本地索引

| 芯片/库名称 | Gitee 仓库地址 | 本地路径 | 下载日期 | 备注 |
|------------|---------------|---------|---------|------|
| TC264 开源库 | https://gitee.com/seekfree/seekfree_tc264_opensource_library | D:\seekfree\seekfree_tc264 | 2026-03-11 | 智能车竞赛常用 |
```

---

## 流程

### 1. 识别芯片型号

确认当前使用的英飞凌芯片型号（如 TC264、TC377、TC387 等）。

### 2. 检查本地索引

读取 `逐飞库索引.md`，查找是否已有对应芯片的本地库记录。

- **索引文件存在且有匹配记录** → 跳到步骤 5（直接使用本地库）
- **索引文件不存在或无匹配记录** → 继续步骤 3

### 3. 询问用户

向用户确认：

```
检测到你在使用英飞凌 <芯片型号>，我需要逐飞科技的开源库来开发。
请问：
1. 你本地是否已经下载了逐飞的 <芯片型号> 开源库？如果有，请告诉我路径。
2. 如果没有，我将从 Gitee (https://gitee.com/seekfree) 搜索并下载。
```

- **用户提供了本地路径** → 验证路径存在后，跳到步骤 4b（记录索引）
- **用户说没有** → 继续步骤 4a（搜索下载）

### 4a. 从 Gitee 搜索并下载

```bash
# 在 Gitee seekfree 组织下搜索对应芯片的仓库
# 常见仓库命名规律：seekfree_<芯片型号>_opensource_library

# 用 git clone 下载到统一目录
git clone https://gitee.com/seekfree/<仓库名>.git <本地保存路径>
```

**本地保存规范**：
- 默认保存到工程同级目录 `seekfree\<仓库名>`，如无法确定则询问用户指定保存目录
- 保持仓库原始目录结构，不要重命名

**逐飞常见仓库对照表**：

| 芯片系列 | 仓库名（参考） |
|---------|--------------|
| TC264 | `seekfree_tc264_opensource_library` |
| TC377 | `seekfree_tc377_opensource_library` |
| TC387 | 需在 Gitee 确认 |
| RT1064 | `seekfree_rt1064_opensource_library` |
| MM32F327X | `seekfree_mm32f327x_opensource_library` |
| CH32V307 | `seekfree_ch32v307_opensource_library` |

> 如果表中没有对应芯片，用 grok-search 搜索 `site:gitee.com/seekfree <芯片型号>` 获取实际仓库地址。

### 4b. 记录到索引文件

将库信息写入 `逐飞库索引.md`：

```markdown
| <芯片/库名称> | <Gitee 仓库地址> | <本地路径> | <当天日期> | <备注> |
```

如果索引文件不存在，先创建文件并写入表头。

### 5. 使用本地库

从索引中获取本地路径后：

1. **读取库目录结构**，了解可用的驱动模块
2. **定位目标驱动文件**（如 OLED 驱动、电机驱动、编码器驱动等）
3. **按主协议的驱动移植流程**，将需要的模块移植到当前项目中

> 移植时遵循主协议的"驱动库移植优先原则"（见 `refs/driver-porting.md`），适配层替换、命名规范化、去冗余、注明来源。

---

## 逐飞库典型目录结构

```
seekfree_tc264_opensource_library/
├── doc/                    # 文档和说明
├── libraries/
│   ├── infineon_libraries/ # 英飞凌官方底层库
│   ├── seekfree_libraries/ # 逐飞封装的驱动库
│   │   ├── zf_driver/      # 外设驱动（GPIO、SPI、I2C、UART 等）
│   │   ├── zf_device/      # 设备驱动（OLED、摄像头、编码器、电机等）
│   │   └── zf_common/      # 通用工具函数
│   └── sdk/                # SDK 配置文件
├── project/                # 示例工程
└── user/                   # 用户代码区
```

**移植时重点关注**：
- `seekfree_libraries/zf_device/` — 常用外设设备驱动
- `seekfree_libraries/zf_driver/` — 芯片外设底层驱动
- `project/` — 参考示例工程的初始化和调用方式
