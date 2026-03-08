---
name: init
description: 嵌入式项目初始化 — 检测 CubeMX 项目环境，创建业务层目录结构，配置开发环境
---

# 嵌入式项目初始化

你是一个嵌入式软件开发助手，正在执行项目初始化流程。请严格按照以下步骤执行。

## 执行流程

### 第一步：检测项目环境

扫描当前目录，收集以下信息：

1. **Makefile 检测**：
   - 是否存在 `Makefile`
   - 提取 `TARGET` 变量（项目名称）
   - 提取 `MCU` 或 `CPU` 变量（芯片型号/架构）
   - 提取工具链前缀（如 `arm-none-eabi-`）
   - 列出已有的构建目标（`all`、`flash`、`download`、`clean` 等）
   - 检查是否已有业务层区域标记（`# 业务层`）

2. **CubeMX 检测**：
   - 是否存在 `.ioc` 文件
   - 从 `.ioc` 文件中提取：芯片型号、RTOS 类型、启用的外设列表
   - 是否存在 `Core/` 或 `hal/core/` 目录

3. **目录结构检测**：
   - 是否已有 `fal/`、`pal/`、`common/`、`utilities/` 等业务层目录
   - 是否存在 `.gitmodules`（子模块）
   - 是否存在 `.embedded-workflow.json`（已初始化标志）

4. **系统工具检测**：
   - 检查 `arm-none-eabi-gcc` 是否可用
   - 检查调试器工具：`JLinkExe`、`openocd`、`st-flash`
   - 检查 `clang-format` 是否可用

如果已存在 `.embedded-workflow.json`，提示用户项目已初始化，询问是否重新初始化。

### 第二步：交互式确认配置

基于检测结果，向用户确认以下配置：

1. **芯片型号**：如检测到 `STM32F429ZI` 或 `GD32F450ZK`，显示并确认
2. **RTOS 选择**：FreeRTOS / RT-Thread / Zephyr / 裸机
3. **调试器类型**：J-Link / ST-Link / DAP-Link
4. **调试接口**：SWD / JTAG
5. **架构模式**：PAL-FAL-HAL 三层架构 / 简单两层架构 / 自定义
6. **Flash 起始地址**：默认 `0x08000000`
7. **是否有 Bootloader**：是/否，如有则确认偏移地址

将所有确认信息以清晰的表格形式展示，等待用户确认后再继续。

### 第三步：创建业务层目录结构

根据确认的架构模式，创建以下目录和文件：

**PAL-FAL-HAL 三层架构（默认）**：

```
project/
├── hal/
│   ├── port/              # RTOS 移植适配层
│   ├── hal.c              # HAL 层初始化入口
│   └── hal.h
├── fal/
│   ├── common/            # FAL 层公共工具
│   ├── fal.c              # FAL 层初始化入口
│   └── fal.h
├── pal/
│   ├── pal.c              # PAL 层应用逻辑入口
│   └── pal.h
├── common/
│   ├── pubsub/            # 发布订阅模块
│   ├── common.c           # 公共工具函数
│   └── common.h
├── utilities/             # 第三方库/子模块
├── drivers/               # 设备驱动
├── middlewares/            # 中间件
└── docs/plans/            # 项目文档
```

对于每个 `.c` 文件，生成基本框架代码，包含：
- 文件头注释（文件名、描述、日期）
- 必要的 `#include`
- 模块初始化函数

对于每个 `.h` 文件，生成：
- 文件头注释
- `#ifndef` 头文件保护宏
- 函数声明

### 第四步：追加 Makefile 业务层配置

在 Makefile 中追加业务层源文件和头文件路径。使用注释标记分隔 CubeMX 区域和业务层区域：

```makefile
######################################
# CubeMX 生成部分（以上）
######################################

######################################
# 业务层（以下由 embedded-workflow 管理）
######################################

# 业务层源文件
C_SOURCES += \
hal/hal.c \
fal/fal.c \
pal/pal.c \
common/common.c

# 业务层头文件路径
C_INCLUDES += \
-Ihal \
-Ifal \
-Ipal \
-Icommon \
-Iutilities \
-Idrivers \
-Imiddlewares
```

**重要**：
- 使用 `+=` 追加，不覆盖 CubeMX 已有定义
- 在 `C_SOURCES` 列表最后一个条目后追加
- 保持与 CubeMX 生成部分相同的格式风格

### 第五步：生成 VS Code 配置

创建 `.vscode/` 目录和以下配置文件：

1. **c_cpp_properties.json** — IntelliSense 配置
   - 使用检测到的工具链路径
   - 包含所有头文件搜索路径
   - 设置正确的芯片 define（如 `STM32F429xx`、`USE_HAL_DRIVER`）
   - 设置 C 标准为 `c11`

2. **launch.json** — 调试配置
   - 配置 cortex-debug 插件
   - 根据调试器类型选择 servertype（jlink/openocd/stutil）
   - 设置正确的 device 和 svdFile

3. **tasks.json** — 构建任务
   - `build` 任务：`make -j$(nproc)`
   - `clean` 任务：`make clean`
   - `flash` 任务：`make flash`
   - `download` 任务：`make download`

使用模板文件（如可用）生成这些配置，替换其中的占位符变量。

### 第六步：生成辅助文件

1. **.gitignore** — 使用模板生成，包含：
   - 构建产物（`build/`、`*.o`、`*.elf`、`*.bin`、`*.hex`）
   - IDE 文件（`.vscode/` 可选）
   - 系统文件（`.DS_Store`、`Thumbs.db`）
   - CubeMX 备份

2. **.clang-format** — 使用模板生成，提供合理的嵌入式 C 代码格式化规则

3. **.embedded-workflow.json** — 项目配置文件，记录：
   - 芯片型号
   - RTOS 类型
   - 调试器配置
   - Flash 地址
   - 格式化和 lint 的包含/排除路径

### 第七步：输出初始化报告

初始化完成后，输出以下信息：

```
✅ 项目初始化完成

📋 项目信息：
   芯片：GD32F450ZK
   RTOS：FreeRTOS
   架构：PAL-FAL-HAL 三层架构
   调试器：J-Link (SWD)

📁 创建的目录：
   hal/port/  fal/  fal/common/  pal/  common/  common/pubsub/
   utilities/  drivers/  middlewares/  docs/plans/

📄 生成的文件：
   hal/hal.c  hal/hal.h  fal/fal.c  fal/fal.h  pal/pal.c  pal/pal.h
   common/common.c  common/common.h
   .vscode/c_cpp_properties.json
   .vscode/launch.json
   .vscode/tasks.json
   .gitignore  .clang-format  .embedded-workflow.json

🔧 Makefile 已追加业务层配置

🚀 下一步：
   1. /embedded-workflow build    — 编译项目
   2. /embedded-workflow flash    — 烧录固件
   3. /embedded-workflow module   — 生成业务模块
```

## 注意事项

- 永远不要覆盖 CubeMX 生成的文件（`Core/` 或 `hal/core/` 下的文件）
- 如果目录或文件已存在，跳过创建并提示用户
- 所有生成的代码使用中文注释
- 保持与项目已有代码风格一致
- 如果检测不到 Makefile，提示用户先用 CubeMX 生成项目
