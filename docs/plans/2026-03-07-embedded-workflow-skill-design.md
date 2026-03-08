# Embedded Workflow Skill 设计文档

## 概述

为嵌入式软件开发团队设计一个 Claude Code Skill，名为 `embedded-workflow`，提供从项目初始化、编译构建、烧录调试到模块生成、代码规范的全流程辅助。

## 背景

### 项目特征
- 智能清洁机器人 MCU 固件，芯片为 GD32F450ZK（兼容 STM32F429ZI）
- 基于 FreeRTOS，采用 PAL → FAL → HAL 三层架构
- 使用 STM32CubeMX 生成初始化代码和 Makefile
- 通过 git submodule 在 utilities 目录下管理公共代码（24 个子模块）
- 约 80+ 自有源文件，19 个 FAL 业务模块，9 大类设备驱动

### 开发环境
- 平台：Ubuntu 20.04
- 工具链：arm-none-eabi-gcc
- 构建系统：Makefile（CubeMX 生成）
- 编辑器：VS Code
- 调试器：J-Link
- RTOS：FreeRTOS
- 团队规模：5 人以上，已有 CI/CD

### 设计目标
- 全流程改善（项目初始化、代码规范、构建烧录、模块开发效率）
- 以 Claude Code Skill 形式交付
- 核心流程优先，后续迭代扩展
- 通用化设计，适配不同芯片、外设、RTOS

---

## 核心设计原则

1. **检测优先** — 自动检测项目环境，不要求用户手动配置
2. **尊重已有** — Makefile 有目标就用，没有才生成补全
3. **AI 增值** — 核心价值在于智能诊断和代码生成，不是替代 make
4. **风格一致** — 生成的代码匹配项目已有风格
5. **渐进式** — 第一阶段实现 init/build/flash，后续迭代扩展

---

## 配置策略

### 配置优先级（从高到低）

1. **命令行参数**（临时覆盖）
2. **项目配置文件** `.embedded-workflow.json`（可选）
3. **自动检测**（智能推断）
4. **交互式询问**（首次使用或检测失败时）

### 自动检测机制

| 检测来源 | 提取信息 |
|---------|---------|
| Makefile | 工具链前缀、芯片型号（MCU 变量）、CPU 架构、构建目标列表 |
| .ioc 文件 | 芯片型号、RTOS 类型、启用的外设 |
| 目录结构 | 架构模式（PAL/FAL/HAL）、是否有 bootloader、子模块 |
| .gitmodules | 子模块列表 |
| 系统工具 | 已安装的调试器工具（JLinkExe / openocd / st-flash） |

### 配置文件（可选）

`.embedded-workflow.json` 只配置无法自动检测的部分：

```json
{
  "debugger": {
    "type": "jlink",
    "interface": "swd",
    "speed": "4000"
  },
  "flash": {
    "address": "0x08000000",
    "bootloader_offset": "0x10000"
  },
  "format": {
    "include": ["fal/", "pal/", "common/", "hal/", "utilities/devices/"],
    "exclude": ["hal/core/"]
  },
  "lint": {
    "tool": "cppcheck",
    "include": ["fal/", "pal/", "common/"],
    "exclude": ["hal/core/", "utilities/*/"]
  },
  "custom_commands": {
    "pre_build": "python scripts/version_gen.py",
    "post_build": "arm-none-eabi-size build/firmware.elf"
  }
}
```

---

## 子命令设计

### 所有子命令统一执行模式

```
检测 Makefile 目标
├─ 有 → 调用它，AI 分析输出
└─ 没有 → 根据项目环境生成目标，追加到 Makefile
```

### 实施阶段规划

- **第一阶段（核心流程）**：init / build / flash / download / debug
- **第二阶段（效率提升）**：module / format / clean
- **第三阶段（质量保障）**：lint

---

### 1. `init` — 项目初始化

**前提**：用户已用 STM32CubeMX 生成基础代码和 Makefile。

**使用方式**：
```bash
/embedded-workflow init
```

**执行流程**：

1. 检测 CubeMX 已生成的内容（Makefile、hal/core/ 源文件、.ioc 文件）
2. 交互式确认配置（芯片、RTOS、调试器、架构模式）
3. 创建业务层目录结构（fal/ pal/ common/ 等）
4. 在 Makefile 中追加业务层源文件路径（用注释分隔 CubeMX 区域和业务层区域）
5. 生成 .vscode/ 配置（c_cpp_properties.json、launch.json、tasks.json）
6. 生成 .gitignore、.embedded-workflow.json
7. 初始化 git 子模块（如果需要）
8. 输出初始化报告和下一步提示

**Makefile 分隔约定**：
```makefile
######################################
# CubeMX 生成部分（以上）
######################################

######################################
# 业务层（以下由 embedded-workflow 管理）
######################################
C_SOURCES += \
fal/fal.c \
pal/pal.c \
common/common.c
```

**生成的目录结构**：
```
project/
├── hal/core/          # CubeMX 已生成
├── hal/port/          # FreeRTOS 移植适配
├── hal/hal.c
├── hal/hal.h
├── fal/
│   ├── common/
│   ├── fal.c
│   └── fal.h
├── pal/
│   └── pal.c
├── common/
│   ├── pubsub/
│   ├── common.c
│   └── common.h
├── utilities/
├── drivers/
├── middlewares/
├── docs/plans/
├── .vscode/
├── .gitignore
├── .clang-format
└── .embedded-workflow.json
```

---

### 2. `build` — 编译项目

**使用方式**：
```bash
/embedded-workflow build              # make -j$(nproc)
/embedded-workflow build --clean      # make clean && make -j$(nproc)
/embedded-workflow build --iap        # make iap
/embedded-workflow build --verbose    # 显示详细编译过程
```

**执行流程**：

1. 检测 Makefile 存在
2. 执行 `make -j$(nproc)`
3. 成功 → 显示 `arm-none-eabi-size` 输出（Flash/RAM 占用）
4. 失败 → AI 读取错误信息，分析原因，给出修复建议

**AI 诊断示例**：
```
❌ 编译失败 (3 个错误)

📍 错误 1: fal/motor/motor.c:45
   undefined reference to 'HAL_TIM_PWM_Start'

💡 分析: CubeMX 中可能没有启用 TIM PWM 功能，
   或者 Makefile 中缺少 stm32f4xx_hal_tim.c。
```

---

### 3. `flash` / `download` — 烧录固件

**使用方式**：
```bash
/embedded-workflow flash              # make flash（OpenOCD 方式）
/embedded-workflow download           # make download（JLink 方式）
/embedded-workflow flash --bootloader # 烧录 Bootloader
```

**执行流程**：

1. 检测 Makefile 中是否有对应目标（flash / download）
2. 有 → 直接调用
3. 没有 → 根据项目环境生成目标追加到 Makefile
4. 失败时诊断（调试器连接、权限、芯片型号等）

**缺失目标时的自动生成依据**：
- .embedded-workflow.json 中的 debugger 配置
- Makefile 中已有的变量（如 OPENOCD 定义）
- 系统中已安装的工具（which JLinkExe / which openocd）
- 交互式询问用户

---

### 4. `debug` — 调试会话

**使用方式**：
```bash
/embedded-workflow debug              # 启动 GDB 调试
/embedded-workflow debug --rtt        # 启动 Segger RTT Viewer
/embedded-workflow debug --shell      # 连接 letter-shell 终端
```

**执行流程**：

1. 启动 GDB Server（根据调试器类型选择 JLinkGDBServer / openocd / st-util）
2. 启动 arm-none-eabi-gdb 连接，或提示使用 VS Code 图形化调试
3. RTT 模式启动 JLinkRTTClient
4. Shell 模式连接串口终端

---

### 5. `module` — 模块生成

**使用方式**：
```bash
/embedded-workflow module add fal <name>              # 新增 FAL 业务模块
/embedded-workflow module add device <bus_type> <name> # 新增设备驱动
```

**支持的总线类型**：uart / iic / spi / can / adc / gpio / tim

**执行流程**：

1. 扫描项目中已有模块，学习代码风格（命名、头文件保护宏、include 列表、Task 创建方式、注释风格）
2. 根据模板和已有风格生成代码框架
3. 自动追加新文件到 Makefile 的业务层区域

**FAL 模块生成示例**：
```
fal/<name>/
├── <name>.c       # 含 FreeRTOS Task 框架
└── <name>.h       # 模块接口
```

**设备驱动生成示例**：
```
utilities/devices/<bus_type>/<name>/
├── <name>.c       # 根据总线类型生成不同的初始化框架
└── <name>.h
```

---

### 6. `format` — 代码格式化

**使用方式**：
```bash
/embedded-workflow format              # 格式化业务层代码
/embedded-workflow format --check      # 仅检查（CI 用）
```

**执行流程**：

1. 检测 Makefile 中是否有 format 相关目标或变量（如已有的 `FORMAT_RESULT` 变量调用 `clang-format-all`）
2. 有 → 直接调用
3. 没有 → 检测 .clang-format 是否存在，生成 format 目标追加到 Makefile
4. 输出格式化结果

---

### 7. `lint` — 静态检查

**使用方式**：
```bash
/embedded-workflow lint                # 检查业务层代码
/embedded-workflow lint fal/motor      # 检查指定目录
```

**执行流程**：

1. 检测 Makefile 中是否有 lint/check 相关目标
2. 有 → 直接调用
3. 没有 → 检测系统中的 cppcheck/clang-tidy，生成 lint 目标
4. AI 对结果分类、过滤嵌入式场景误报、给出修复建议

---

## Skill 文件结构

```
embedded-workflow/
├── skill.json                    # Skill 元数据和入口定义
├── README.md                     # 使用说明
├── skills/
│   ├── init/
│   │   └── prompt.md             # init 子命令的 Prompt
│   ├── build/
│   │   └── prompt.md             # build 子命令的 Prompt
│   ├── flash/
│   │   └── prompt.md             # flash/download 子命令的 Prompt
│   ├── debug/
│   │   └── prompt.md             # debug 子命令的 Prompt
│   ├── module/
│   │   └── prompt.md             # module 子命令的 Prompt
│   ├── format/
│   │   └── prompt.md             # format 子命令的 Prompt
│   └── lint/
│       └── prompt.md             # lint 子命令的 Prompt
└── templates/                    # 代码模板
    ├── fal_module.c.tmpl
    ├── fal_module.h.tmpl
    ├── device_uart.c.tmpl
    ├── device_iic.c.tmpl
    ├── device_spi.c.tmpl
    ├── device_can.c.tmpl
    ├── device_adc.c.tmpl
    ├── device_gpio.c.tmpl
    ├── device_tim.c.tmpl
    ├── makefile_flash.tmpl
    ├── makefile_download.tmpl
    ├── makefile_format.tmpl
    ├── makefile_lint.tmpl
    ├── vscode_launch.json.tmpl
    ├── vscode_tasks.json.tmpl
    └── clang-format.tmpl
```

---

## 通用性保证

### 支持的芯片系列（通过自动检测）
- STM32 全系列（F0/F1/F2/F3/F4/F7/H7/L0/L1/L4/G0/G4/WB/WL）
- GD32 全系列（F1/F3/F4/E2/W5）
- 其他 Cortex-M 芯片（只要 Makefile 配置正确）

### 支持的 RTOS
- FreeRTOS
- RT-Thread
- Zephyr
- 裸机（无 RTOS）

### 支持的构建系统
- Makefile
- CMake
- Meson

### 支持的调试器
- J-Link
- ST-Link
- DAP-Link

---

## 实施路径

- **第一阶段**：init / build / flash / download / debug
- **第二阶段**：module / format / clean
- **第三阶段**：lint
