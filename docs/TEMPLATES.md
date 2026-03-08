# 模板文件说明

本文档说明 `templates/` 目录下各模板文件的用途、占位符和自定义方法。

## 模板文件列表

### VS Code 配置模板

#### vscode_c_cpp_properties.json.tmpl

IntelliSense 配置，用于代码补全和语法检查。

**占位符**：
- `{{CHIP_DEFINE}}` — 芯片宏定义（如 `STM32F429xx`、`GD32F450`）
- `{{COMPILER_PATH}}` — 工具链路径（如 `/usr/bin/arm-none-eabi-gcc`）
- `{{CPU}}` — CPU 架构（如 `cortex-m4`、`cortex-m3`）

**使用场景**：`init` 子命令生成 `.vscode/c_cpp_properties.json`

#### vscode_launch.json.tmpl

调试配置，用于 VS Code 的 cortex-debug 插件。

**占位符**：
- `{{TARGET}}` — 项目名称（从 Makefile 的 TARGET 变量读取）
- `{{DEBUGGER_TYPE}}` — 调试器类型（`jlink` / `openocd` / `stutil`）
- `{{DEVICE}}` — 芯片型号（如 `STM32F429ZI`）
- `{{INTERFACE}}` — 调试接口（`swd` / `jtag`）
- `{{SVD_FILE}}` — SVD 文件路径（可选，用于查看外设寄存器）

**使用场景**：`init` 子命令生成 `.vscode/launch.json`

#### vscode_tasks.json.tmpl

构建任务配置，定义 build/clean/flash/download 任务。

**占位符**：无（固定内容）

**使用场景**：`init` 子命令生成 `.vscode/tasks.json`

---

### 项目配置模板

#### .gitignore.tmpl

Git 忽略文件配置，排除构建产物和临时文件。

**内容**：
- 构建产物（`build/`、`*.o`、`*.elf`、`*.bin`、`*.hex`）
- CubeMX 备份（`*.ioc.bak`）
- IDE 文件（`.vscode/`、`.idea/`）
- 系统文件（`.DS_Store`、`Thumbs.db`）

**使用场景**：`init` 子命令生成 `.gitignore`

#### .clang-format.tmpl

代码格式化规则配置。

**风格**：
- 基于 Google 风格
- 缩进 4 空格
- 大括号换行（Allman 风格）
- 行宽 120 字符

**使用场景**：`init` 子命令生成 `.clang-format`

**自定义**：可根据团队规范修改此模板，常见配置项：
```yaml
IndentWidth: 4              # 缩进宽度
ColumnLimit: 120            # 行宽限制
BreakBeforeBraces: Allman   # 大括号风格
PointerAlignment: Right     # 指针对齐方式
```

---

### Makefile 目标模板

#### makefile_flash.tmpl

OpenOCD 烧录目标，适用于 ST-Link 和 DAP-Link。

**占位符**：
- `{{INTERFACE}}` — 接口配置文件（`stlink` / `cmsis-dap`）
- `{{TARGET_CFG}}` — 目标配置文件（`stm32f4x` / `stm32f1x`）
- `{{FLASH_ADDR}}` — Flash 起始地址（默认 `0x08000000`）

**使用场景**：`flash` 子命令检测到 Makefile 缺少 flash 目标时生成

#### makefile_download.tmpl

J-Link 烧录目标。

**占位符**：
- `{{DEVICE}}` — 芯片型号（如 `STM32F429ZI`、`GD32F450ZK`）
- `{{INTERFACE}}` — 调试接口（`SWD` / `JTAG`）
- `{{FLASH_ADDR}}` — Flash 起始地址

**使用场景**：`download` 子命令检测到 Makefile 缺少 download 目标时生成

#### makefile_format.tmpl

代码格式化目标。

**占位符**：
- `FORMAT_DIRS` — 格式化目录列表（默认 `fal pal common utilities/devices drivers middlewares`）

**使用场景**：`format` 子命令检测到 Makefile 缺少 format 目标时生成

#### makefile_lint.tmpl

静态检查目标。

**占位符**：
- `LINT_DIRS` — 检查目录列表
- `LINT_EXCLUDE` — 排除目录列表

**使用场景**：`lint` 子命令检测到 Makefile 缺少 lint 目标时生成

---

### 代码生成模板

#### fal_module.h.tmpl / fal_module.c.tmpl

FAL 业务模块模板，包含 FreeRTOS Task 框架。

**占位符**：
- `{{MODULE_NAME}}` — 模块名（小写，如 `motor`）
- `{{MODULE_NAME_UPPER}}` — 模块名（大写，如 `MOTOR`）
- `{{DATE}}` — 当前日期

**生成内容**：
- 模块初始化函数
- FreeRTOS Task 创建
- Task 主循环框架

**使用场景**：`module add fal <name>` 生成 FAL 模块

**自定义**：可修改：
- Task 栈大小（默认 512）
- Task 优先级（默认 `osPriorityNormal`）
- 注释风格
- Include 列表

#### device_uart.c.tmpl

UART 设备驱动模板。

**占位符**：
- `{{MODULE_NAME}}` — 设备名（如 `sensor`）
- `{{DATE}}` — 当前日期

**生成接口**：
- `init(UART_HandleTypeDef *huart)` — 初始化
- `send(uint8_t *data, uint16_t len)` — 发送数据
- `recv(uint8_t *data, uint16_t len)` — 接收数据

**使用场景**：`module add device uart <name>`

#### device_iic.c.tmpl

I2C 设备驱动模板。

**占位符**：
- `{{MODULE_NAME}}` — 设备名
- `{{MODULE_NAME_UPPER}}` — 设备名（大写）
- `{{DATE}}` — 当前日期

**生成接口**：
- `init(I2C_HandleTypeDef *hi2c, uint8_t addr)` — 初始化
- `read_reg(uint8_t reg, uint8_t *data, uint16_t len)` — 读寄存器
- `write_reg(uint8_t reg, uint8_t *data, uint16_t len)` — 写寄存器

**使用场景**：`module add device iic <name>`

#### device_spi.c.tmpl

SPI 设备驱动模板。

**生成接口**：
- `init(SPI_HandleTypeDef *hspi, GPIO_TypeDef *cs_port, uint16_t cs_pin)` — 初始化
- `transfer(uint8_t *tx_data, uint8_t *rx_data, uint16_t len)` — SPI 传输

**使用场景**：`module add device spi <name>`

#### device_can.c.tmpl

CAN 设备驱动模板。

**生成接口**：
- `init(CAN_HandleTypeDef *hcan)` — 初始化
- `send(uint32_t id, uint8_t *data, uint8_t len)` — 发送 CAN 帧

**使用场景**：`module add device can <name>`

#### device_adc.c.tmpl

ADC 设备驱动模板。

**生成接口**：
- `init(ADC_HandleTypeDef *hadc)` — 初始化
- `read(uint32_t channel)` — 读取 ADC 值
- `to_voltage(uint32_t adc_value, uint32_t vref)` — 转换为电压

**使用场景**：`module add device adc <name>`

#### device_gpio.c.tmpl

GPIO 设备驱动模板。

**生成接口**：
- `init(GPIO_TypeDef *port, uint16_t pin)` — 初始化
- `set(uint8_t state)` — 设置输出
- `get(void)` — 读取输入
- `toggle(void)` — 翻转输出

**使用场景**：`module add device gpio <name>`

#### device_tim.c.tmpl

定时器/PWM 设备驱动模板。

**生成接口**：
- `init(TIM_HandleTypeDef *htim, uint32_t channel)` — 初始化
- `pwm_start(void)` — 启动 PWM
- `pwm_stop(void)` — 停止 PWM
- `set_duty(uint16_t duty)` — 设置占空比（0-1000）

**使用场景**：`module add device tim <name>`

---

## 自定义模板

### 修改现有模板

1. 编辑 `templates/` 目录下的 `.tmpl` 文件
2. 保持占位符格式 `{{VARIABLE_NAME}}`
3. 测试生成效果

### 添加新模板

1. 在 `templates/` 目录创建新的 `.tmpl` 文件
2. 在对应的 prompt.md 中引用新模板
3. 定义占位符并在 prompt 中说明替换规则

### 占位符命名规范

- 使用大写字母和下划线
- 语义清晰（如 `MODULE_NAME` 而非 `NAME`）
- 避免与代码中的宏定义冲突

### 示例：自定义 FAL 模块模板

如果团队使用不同的 Task 创建方式，可以修改 `fal_module.c.tmpl`：

```c
// 原模板
xTaskCreate({{MODULE_NAME}}_task, "{{MODULE_NAME}}", 512, NULL, osPriorityNormal, &{{MODULE_NAME}}_task_handle);

// 自定义为 osThreadNew（CMSIS-RTOS v2）
const osThreadAttr_t {{MODULE_NAME}}_attr = {
    .name = "{{MODULE_NAME}}",
    .stack_size = 512,
    .priority = osPriorityNormal,
};
{{MODULE_NAME}}_task_handle = osThreadNew({{MODULE_NAME}}_task, NULL, &{{MODULE_NAME}}_attr);
```

---

## 常见问题

### Q: 如何让生成的代码使用英文注释？

A: 修改模板文件中的注释内容，将中文替换为英文。

### Q: 如何修改默认的 Task 栈大小？

A: 编辑 `fal_module.c.tmpl`，将 `512` 改为所需大小。

### Q: 如何添加自定义的设备驱动模板？

A: 
1. 创建 `templates/device_<总线类型>.c.tmpl`
2. 在 `skills/module/prompt.md` 中添加对应的总线类型支持
3. 定义生成逻辑

### Q: 模板中的占位符会被自动替换吗？

A: 是的，AI 会根据 prompt 中的说明自动替换占位符。确保 prompt 中明确说明了每个占位符的含义和替换规则。
