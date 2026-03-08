---
name: module
description: 生成业务模块或设备驱动 — 扫描项目代码风格，根据模板生成代码框架，自动追加到 Makefile
---

# 模块生成

你是一个嵌入式软件开发助手，正在帮助用户生成业务模块或设备驱动代码。请严格按照以下步骤执行。

## 使用方式

```bash
/embedded-workflow module add fal <模块名>                    # 生成 FAL 业务模块
/embedded-workflow module add device <总线类型> <设备名>      # 生成设备驱动
```

支持的总线类型：`uart` / `iic` / `spi` / `can` / `adc` / `gpio` / `tim`

## 执行流程

### 第一步：参数解析

从用户输入中提取：
1. **模块类型**：`fal` 或 `device`
2. **模块名称**：如 `motor`、`sensor`、`pump`
3. **总线类型**（仅 device）：如 `uart`、`iic`、`spi`

验证参数合法性：
- 模块名称只能包含字母、数字、下划线
- 总线类型必须在支持列表中

### 第二步：扫描项目代码风格

在生成代码前，扫描项目中已有的模块，学习以下代码风格：

1. **命名风格**：
   - 函数命名：驼峰式 `motorInit()` 还是下划线式 `motor_init()`
   - 变量命名：驼峰式还是下划线式
   - 宏定义：全大写 `MOTOR_SPEED_MAX` 还是其他

2. **头文件保护宏**：
   - 格式：`__MOTOR_H__` / `_MOTOR_H_` / `MOTOR_H`
   - 是否包含路径：`__FAL_MOTOR_H__`

3. **Include 列表**：
   - 项目中常用的头文件顺序
   - 是否使用 `#include "main.h"`
   - 是否使用 `#include "FreeRTOS.h"`

4. **注释风格**：
   - 文件头注释格式（作者、日期、描述）
   - 函数注释格式（Doxygen 风格还是简单注释）
   - 是否使用中文注释

5. **FreeRTOS Task 创建方式**（仅 FAL 模块）：
   - 任务栈大小惯例
   - 任务优先级惯例
   - 任务句柄命名方式

扫描方法：
- 读取 `fal/` 目录下 2-3 个已有模块的 `.c` 和 `.h` 文件
- 读取 `utilities/devices/` 目录下已有驱动（如存在）
- 提取上述风格特征

### 第三步：生成代码

#### 生成 FAL 业务模块

目标目录：`fal/<模块名>/`

**生成 `<模块名>.h`**：

```c
/**
 * @file <模块名>.h
 * @brief <模块名>模块接口
 * @date <当前日期>
 */

#ifndef __FAL_<模块名大写>_H__
#define __FAL_<模块名大写>_H__

#ifdef __cplusplus
extern "C" {
#endif

#include "main.h"
#include "FreeRTOS.h"
#include "task.h"

/* 公共函数声明 */
void <模块名>_init(void);
void <模块名>_task(void *argument);

#ifdef __cplusplus
}
#endif

#endif /* __FAL_<模块名大写>_H__ */
```

**生成 `<模块名>.c`**：

```c
/**
 * @file <模块名>.c
 * @brief <模块名>模块实现
 * @date <当前日期>
 */

#include "<模块名>.h"
#include "fal.h"

/* 任务句柄 */
static TaskHandle_t <模块名>_task_handle = NULL;

/**
 * @brief <模块名>模块初始化
 */
void <模块名>_init(void)
{
    /* TODO: 添加初始化代码 */
    
    /* 创建任务 */
    xTaskCreate(<模块名>_task, "<模块名>", 512, NULL, osPriorityNormal, &<模块名>_task_handle);
}

/**
 * @brief <模块名>任务主循环
 * @param argument 任务参数
 */
void <模块名>_task(void *argument)
{
    for (;;)
    {
        /* TODO: 添加任务逻辑 */
        
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

#### 生成设备驱动

目标目录：`utilities/devices/<总线类型>/<设备名>/` 或 `drivers/<设备名>/`

根据总线类型使用不同的模板（`templates/device_<总线类型>.c.tmpl`），生成对应的初始化框架。

**UART 设备示例**：

```c
/**
 * @file <设备名>.c
 * @brief <设备名> UART 设备驱动
 * @date <当前日期>
 */

#include "<设备名>.h"
#include "usart.h"

/**
 * @brief <设备名>初始化
 * @param huart UART 句柄
 * @return 0: 成功, -1: 失败
 */
int <设备名>_init(UART_HandleTypeDef *huart)
{
    /* TODO: 添加初始化代码 */
    return 0;
}

/**
 * @brief <设备名>发送数据
 * @param data 数据指针
 * @param len 数据长度
 * @return 发送的字节数
 */
int <设备名>_send(uint8_t *data, uint16_t len)
{
    /* TODO: 实现发送逻辑 */
    return 0;
}

/**
 * @brief <设备名>接收数据
 * @param data 数据缓冲区
 * @param len 缓冲区大小
 * @return 接收的字节数
 */
int <设备名>_recv(uint8_t *data, uint16_t len)
{
    /* TODO: 实现接收逻辑 */
    return 0;
}
```

**IIC 设备示例**：

```c
int <设备名>_init(I2C_HandleTypeDef *hi2c, uint8_t addr);
int <设备名>_read_reg(uint8_t reg, uint8_t *data, uint16_t len);
int <设备名>_write_reg(uint8_t reg, uint8_t *data, uint16_t len);
```

**SPI 设备示例**：

```c
int <设备名>_init(SPI_HandleTypeDef *hspi, GPIO_TypeDef *cs_port, uint16_t cs_pin);
int <设备名>_transfer(uint8_t *tx_data, uint8_t *rx_data, uint16_t len);
```

### 第四步：追加到 Makefile

在 Makefile 的业务层区域（`# 业务层` 注释后）追加新文件：

```makefile
# <模块名>模块
C_SOURCES += \
fal/<模块名>/<模块名>.c
```

或

```makefile
# <设备名>设备驱动
C_SOURCES += \
utilities/devices/<总线类型>/<设备名>/<设备名>.c
```

**重要**：
- 使用 `\` 续行符保持格式一致
- 追加到 `C_SOURCES` 列表末尾
- 添加注释说明模块用途

### 第五步：输出生成报告

```
✅ 模块生成成功

📁 生成的文件：
   fal/motor/motor.h
   fal/motor/motor.c

📝 Makefile 已更新

🔧 下一步：
   1. 编辑 fal/motor/motor.c 实现业务逻辑
   2. 在 fal/fal.c 中调用 motor_init()
   3. /embedded-workflow build 编译测试
```

## 模板文件

如果存在以下模板文件，优先使用模板生成：
- `templates/fal_module.c.tmpl`
- `templates/fal_module.h.tmpl`
- `templates/device_uart.c.tmpl`
- `templates/device_iic.c.tmpl`
- `templates/device_spi.c.tmpl`
- `templates/device_can.c.tmpl`
- `templates/device_adc.c.tmpl`
- `templates/device_gpio.c.tmpl`
- `templates/device_tim.c.tmpl`

模板中的占位符：
- `{{MODULE_NAME}}` — 模块名（小写）
- `{{MODULE_NAME_UPPER}}` — 模块名（大写）
- `{{DATE}}` — 当前日期
- `{{BUS_TYPE}}` — 总线类型

## 注意事项

- 生成的代码必须与项目已有风格一致
- 如果目录或文件已存在，提示用户并询问是否覆盖
- 所有生成的代码使用中文注释
- FAL 模块必须包含 FreeRTOS Task 框架
- 设备驱动应提供标准的 init/read/write 接口
