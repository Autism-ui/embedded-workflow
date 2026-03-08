# Embedded Workflow Skill 使用指南

本文档详细说明每个子命令的使用方法、参数选项和常见场景。

## 目录

- [init - 项目初始化](#init---项目初始化)
- [build - 编译项目](#build---编译项目)
- [flash/download - 烧录固件](#flashdownload---烧录固件)
- [debug - 调试会话](#debug---调试会话)
- [module - 模块生成](#module---模块生成)
- [format - 代码格式化](#format---代码格式化)
- [lint - 静态检查](#lint---静态检查)

---

## init - 项目初始化

### 功能说明

在 STM32CubeMX 生成的项目基础上，创建业务层目录结构，配置开发环境。

### 前提条件

- 已使用 STM32CubeMX 生成项目代码和 Makefile
- 当前目录包含 Makefile 和 .ioc 文件

### 使用方式

```bash
/embedded-workflow init
```

### 执行流程

1. 检测 CubeMX 生成的内容（Makefile、.ioc、hal/core/）
2. 交互式确认配置（芯片型号、RTOS、调试器、架构模式）
3. 创建业务层目录结构（fal/、pal/、common/ 等）
4. 追加 Makefile 业务层配置
5. 生成 VS Code 配置（.vscode/）
6. 生成 .gitignore、.clang-format、.embedded-workflow.json

### 生成的目录结构

```
project/
├── hal/
│   ├── core/          # CubeMX 生成（不修改）
│   ├── port/          # RTOS 移植适配
│   ├── hal.c
│   └── hal.h
├── fal/               # FAL 业务层
│   ├── common/
│   ├── fal.c
│   └── fal.h
├── pal/               # PAL 应用层
│   ├── pal.c
│   └── pal.h
├── common/            # 公共工具
│   ├── pubsub/
│   ├── common.c
│   └── common.h
├── utilities/         # 第三方库
├── drivers/           # 设备驱动
├── middlewares/       # 中间件
├── docs/plans/        # 项目文档
└── .vscode/           # VS Code 配置
```

### 注意事项

- 不会覆盖 CubeMX 生成的文件
- 如果目录已存在，会跳过创建
- 可以重复运行以更新配置

---

## build - 编译项目

### 功能说明

执行 make 构建，显示固件大小，AI 诊断编译错误。

### 使用方式

```bash
/embedded-workflow build              # 默认编译
/embedded-workflow build --clean      # 清理后编译
/embedded-workflow build --iap        # 编译 IAP 版本
/embedded-workflow build --verbose    # 显示详细编译过程
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `--clean` | 先执行 make clean 再编译 |
| `--iap` | 编译 IAP/Bootloader 版本（需要 Makefile 中有 iap 目标） |
| `--verbose` | 显示详细的编译命令（make V=1） |

### 成功输出示例

```
✅ 编译成功

📊 固件大小：
   text    data     bss     dec     hex filename
  45678    1234    5678   52590    cd6e build/firmware.elf

💾 Flash 占用: 46912 / 524288 bytes (8.9%)
📦 RAM 占用:   6912 / 131072 bytes (5.3%)

📁 输出文件：
   build/firmware.elf
   build/firmware.bin
   build/firmware.hex
```

### 错误诊断

AI 会分析编译错误并给出修复建议：

```
❌ 编译失败 (2 个错误)

📍 错误 1: fal/motor/motor.c:45
   undefined reference to 'HAL_TIM_PWM_Start'

💡 分析: CubeMX 中可能没有启用 TIM PWM 功能
🔧 建议: 在 CubeMX 中启用对应的 TIM，或检查 Makefile 中是否包含 stm32f4xx_hal_tim.c
```

---

## flash/download - 烧录固件

### 功能说明

将编译好的固件烧录到芯片，支持 OpenOCD 和 J-Link 两种方式。

### 使用方式

```bash
/embedded-workflow flash              # OpenOCD 方式（ST-Link/DAP-Link）
/embedded-workflow download           # J-Link 方式
/embedded-workflow flash --bootloader # 烧录 Bootloader
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `--bootloader` | 烧录 Bootloader 而非应用固件 |
| `--address <addr>` | 指定烧录地址（默认从配置读取） |
| `--verify` | 烧录后验证（默认启用） |

### 自动生成 Makefile 目标

如果 Makefile 中没有 flash/download 目标，会自动生成并追加：

```makefile
######################################
# flash 目标（OpenOCD）
######################################
flash: all
	openocd -f interface/stlink.cfg -f target/stm32f4x.cfg \
		-c "program $(BUILD_DIR)/$(TARGET).elf verify reset exit"
```

### 成功输出示例

```
✅ 烧录成功

📡 调试器: J-Link (SWD)
📦 固件: build/firmware.bin (46912 bytes)
📍 地址: 0x08000000
⏱️  耗时: 2.3 秒
```

---

## debug - 调试会话

### 功能说明

启动调试会话，支持 GDB 调试、RTT Viewer 和串口 Shell 三种模式。

### 使用方式

```bash
/embedded-workflow debug              # GDB 调试
/embedded-workflow debug --rtt        # RTT Viewer
/embedded-workflow debug --shell      # 串口终端
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `--rtt` | 启动 Segger RTT Viewer（需要 J-Link） |
| `--shell` | 连接串口终端（letter-shell 等） |
| `--port <端口>` | 指定串口设备或 GDB 端口 |
| `--baud <波特率>` | 串口波特率（默认 115200） |

### GDB 调试模式

启动 GDB Server 并提供连接方式：

```
✅ GDB Server 已启动

📡 调试器: J-Link (SWD)
🔌 GDB 端口: localhost:2331

连接方式：

方式 1 — 命令行 GDB:
   arm-none-eabi-gdb build/firmware.elf
   (gdb) target remote :2331
   (gdb) load
   (gdb) break main
   (gdb) continue

方式 2 — VS Code 图形化调试:
   按 F5 启动调试
```

### RTT Viewer 模式

启动 Segger RTT Viewer 查看实时日志：

```
✅ RTT Viewer 已启动

📡 RTT 通道 0 已连接
💡 提示: 确保固件中已初始化 SEGGER_RTT
```

### Shell 模式

连接串口终端：

```
✅ 串口终端已连接

🔌 端口: /dev/ttyACM0
📊 波特率: 115200
```

---

## module - 模块生成

### 功能说明

生成 FAL 业务模块或设备驱动代码框架，自动匹配项目代码风格。

### 使用方式

```bash
# 生成 FAL 业务模块
/embedded-workflow module add fal motor

# 生成设备驱动
/embedded-workflow module add device uart sensor
/embedded-workflow module add device iic mpu6050
/embedded-workflow module add device spi w25q128
```

### 支持的总线类型

| 总线 | 说明 | 生成的接口 |
|------|------|-----------|
| uart | UART 串口 | init, send, recv |
| iic | I2C 总线 | init, read_reg, write_reg |
| spi | SPI 总线 | init, transfer |
| can | CAN 总线 | init, send |
| adc | ADC 采样 | init, read, to_voltage |
| gpio | GPIO 控制 | init, set, get, toggle |
| tim | 定时器/PWM | init, pwm_start, pwm_stop, set_duty |

### 代码风格学习

生成前会扫描项目已有模块，学习：
- 命名风格（驼峰式 / 下划线式）
- 头文件保护宏格式
- Include 列表顺序
- 注释风格
- FreeRTOS Task 创建方式

### 生成示例

FAL 模块生成到 `fal/<模块名>/`：

```c
// fal/motor/motor.c
void motor_init(void)
{
    xTaskCreate(motor_task, "motor", 512, NULL, osPriorityNormal, &motor_task_handle);
}

void motor_task(void *argument)
{
    for (;;)
    {
        // TODO: 添加任务逻辑
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

设备驱动生成到 `utilities/devices/<总线>/<设备名>/` 或 `drivers/<设备名>/`。

---

## format - 代码格式化

### 功能说明

使用 clang-format 格式化业务层代码，统一代码风格。

### 使用方式

```bash
/embedded-workflow format             # 格式化业务层代码
/embedded-workflow format --check     # 仅检查格式（CI 用）
/embedded-workflow format --all       # 格式化所有代码
/embedded-workflow format fal/motor   # 格式化指定目录
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `--check` | 仅检查格式，不修改文件 |
| `--all` | 格式化所有代码（包括 HAL 层） |
| `<路径>` | 指定格式化的目录或文件 |

### 格式化范围

默认只格式化业务层代码：
- fal/
- pal/
- common/
- utilities/devices/
- drivers/
- middlewares/

不格式化 CubeMX 生成的 HAL 层代码（hal/core/）。

### 输出示例

```
✅ 格式化完成

📊 统计：
   格式化文件: 45 个
   修改文件: 12 个
   跳过文件: 33 个（已符合规范）

💡 提示: 使用 git diff 查看修改内容
```

---

## lint - 静态检查

### 功能说明

使用 cppcheck 或 clang-tidy 进行静态代码检查，AI 分析结果并过滤嵌入式误报。

### 使用方式

```bash
/embedded-workflow lint               # 检查业务层代码
/embedded-workflow lint fal/motor     # 检查指定目录
/embedded-workflow lint --tool cppcheck  # 指定检查工具
```

### 参数说明

| 参数 | 说明 |
|------|------|
| `<路径>` | 指定检查的目录或文件 |
| `--tool <工具>` | 指定检查工具（cppcheck / clang-tidy） |
| `--severity <级别>` | 最低报告级别（error / warning / style） |

### AI 智能分析

AI 会对检查结果进行分类和过滤：

**分类**：
- ❌ 错误（必须修复）
- ⚠️ 警告（建议修复）
- 📝 风格（可选修复）

**过滤嵌入式误报**：
- 寄存器写入后不读取
- ISR 共享的 volatile 变量
- HAL 回调函数未使用
- 硬件地址的类型转换

### 输出示例

```
📊 静态检查报告

📍 发现 5 个问题 (1 个错误, 3 个警告, 1 个风格)

❌ 错误 (必须修复):
  1. fal/motor/motor.c:45
     [error] Array index out of bounds
     💡 数组 'buffer' 大小为 10，但访问了 index 12
     🔧 修改索引范围或增大数组大小

⚠️ 警告 (建议修复):
  2. fal/sensor/sensor.c:78
     [warning] Variable 'temp' is assigned but never used
     💡 变量已赋值但后续未使用
     🔧 移除未使用的变量

🔇 已过滤的误报: 3 个
```

---

## 配置文件

### .embedded-workflow.json

项目配置文件，记录项目环境和用户偏好：

```json
{
  "chip": "GD32F450ZK",
  "rtos": "FreeRTOS",
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
    "include": ["fal/", "pal/", "common/"],
    "exclude": ["hal/core/"]
  },
  "lint": {
    "tool": "cppcheck",
    "include": ["fal/", "pal/", "common/"],
    "exclude": ["hal/core/", "utilities/*/"]
  }
}
```

此文件由 `init` 子命令生成，可手动编辑。

---

## 常见问题

### Q: 如何在已有项目中使用？

A: 在项目根目录运行 `/embedded-workflow init`，会检测已有结构并补全缺失部分。

### Q: 是否会覆盖 CubeMX 生成的代码？

A: 不会。所有操作都在业务层目录进行，不修改 CubeMX 生成的 hal/core/ 内容。

### Q: 如何自定义代码模板？

A: 编辑 `templates/` 目录下的 `.tmpl` 文件，参考 [模板说明](TEMPLATES.md)。

### Q: 支持其他构建系统吗？

A: 当前主要支持 Makefile，CMake 和 Meson 支持正在开发中。

### Q: 如何在 CI 中使用？

A: 使用 `--check` 参数：
```bash
/embedded-workflow format --check
/embedded-workflow lint
```
