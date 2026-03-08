# Embedded Workflow Skill

嵌入式软件开发全流程工作流辅助工具，为 Claude Code 提供从项目初始化、编译构建、烧录调试到模块生成、代码规范的完整支持。

## 功能特性

- 🚀 **项目初始化** — 检测 CubeMX 项目，创建业务层架构，生成开发环境配置
- 🔨 **智能编译** — 执行 make 构建，显示固件大小，AI 诊断编译错误
- 📡 **固件烧录** — 支持 OpenOCD/J-Link，自动生成 Makefile 目标
- 🐛 **调试会话** — 启动 GDB Server，支持 RTT Viewer 和串口 Shell
- 📦 **模块生成** — 扫描代码风格，生成 FAL 模块和设备驱动
- ✨ **代码格式化** — 使用 clang-format 统一代码风格
- 🔍 **静态检查** — cppcheck 检查 + AI 过滤嵌入式误报

## 快速开始

### 安装

将此 skill 放置到 Claude Code 的 skills 目录：

```bash
# 复制到 Claude Code skills 目录
cp -r embedded-workflow ~/.claude/skills/
```

### 使用方式

```bash
# 1. 项目初始化（在 CubeMX 生成的项目目录中执行）
/embedded-workflow init

# 2. 编译项目
/embedded-workflow build
/embedded-workflow build --clean      # 清理后编译

# 3. 烧录固件
/embedded-workflow flash              # OpenOCD 方式
/embedded-workflow download           # J-Link 方式

# 4. 启动调试
/embedded-workflow debug              # GDB 调试
/embedded-workflow debug --rtt        # RTT Viewer
/embedded-workflow debug --shell      # 串口终端

# 5. 生成模块
/embedded-workflow module add fal motor           # 生成 FAL 业务模块
/embedded-workflow module add device uart sensor  # 生成 UART 设备驱动

# 6. 代码格式化
/embedded-workflow format             # 格式化业务层代码
/embedded-workflow format --check     # 仅检查格式

# 7. 静态检查
/embedded-workflow lint               # 检查业务层代码
/embedded-workflow lint fal/motor     # 检查指定目录
```

## 支持的平台

### 芯片系列
- STM32 全系列（F0/F1/F2/F3/F4/F7/H7/L0/L1/L4/G0/G4/WB/WL）
- GD32 全系列（F1/F3/F4/E2/W5）
- 其他 Cortex-M 芯片

### RTOS
- FreeRTOS
- RT-Thread
- Zephyr
- 裸机（无 RTOS）

### 构建系统
- Makefile（主要支持）
- CMake
- Meson

### 调试器
- J-Link
- ST-Link
- DAP-Link

## 设计原则

1. **检测优先** — 自动检测项目环境，无需手动配置
2. **尊重已有** — Makefile 有目标就用，没有才生成补全
3. **AI 增值** — 核心价值在于智能诊断和代码生成
4. **风格一致** — 生成的代码匹配项目已有风格

## 项目结构

```
embedded-workflow/
├── skill.json              # Skill 元数据
├── README.md               # 本文档
├── skills/                 # 子命令 prompt
│   ├── init/
│   ├── build/
│   ├── flash/
│   ├── debug/
│   ├── module/
│   ├── format/
│   └── lint/
├── templates/              # 代码和配置模板
│   ├── fal_module.*.tmpl
│   ├── device_*.c.tmpl
│   ├── makefile_*.tmpl
│   └── vscode_*.tmpl
└── docs/                   # 文档
    ├── plans/              # 设计和实施计划
    ├── USAGE.md            # 详细使用指南
    └── TEMPLATES.md        # 模板说明
```

## 文档

- [详细使用指南](docs/USAGE.md) — 每个子命令的详细说明和示例
- [模板说明](docs/TEMPLATES.md) — 模板文件的使用和自定义
- [设计文档](docs/plans/2026-03-07-embedded-workflow-skill-design.md) — 完整的设计思路

## 版本

当前版本：0.1.0

## 许可

MIT License
