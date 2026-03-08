---
name: flash
description: 烧录固件到芯片 — 检测 Makefile 目标，自动生成烧录配置，诊断调试器连接问题
---

# 烧录固件

你是一个嵌入式软件开发助手，正在执行固件烧录流程。请严格按照以下步骤执行。

## 子命令说明

- `flash` — 使用 OpenOCD 烧录（适用于 ST-Link、DAP-Link）
- `download` — 使用 J-Link 烧录

两个子命令共享此 prompt，根据调用的命令名称选择对应的烧录方式。

## 参数解析

| 参数 | 说明 | 默认 |
|------|------|------|
| `--bootloader` | 烧录 Bootloader 而非应用固件 | 否 |
| `--address <addr>` | 指定烧录地址 | 从配置读取 |
| `--verify` | 烧录后验证 | 是 |

## 执行流程

### 第一步：环境检测

1. 检查 Makefile 是否存在
2. 检查固件文件是否存在（`build/*.elf` 或 `build/*.bin`）
   - 不存在 → 提示用户先运行 `/embedded-workflow build`
3. 读取 `.embedded-workflow.json` 获取调试器配置
4. 检测系统中可用的烧录工具：
   - `flash` 命令 → 检查 `openocd` 或 `st-flash`
   - `download` 命令 → 检查 `JLinkExe`

### 第二步：检测 Makefile 目标

1. 检查 Makefile 中是否已有对应目标：
   - `flash` 命令 → 查找 `flash:` 目标
   - `download` 命令 → 查找 `download:` 目标
2. 如果目标存在 → 跳到第四步直接调用
3. 如果目标不存在 → 进入第三步生成目标

### 第三步：生成 Makefile 目标

根据检测到的调试器类型和系统工具，生成对应的 Makefile 目标。

#### 生成 flash 目标（OpenOCD）

使用模板 `templates/makefile_flash.tmpl`（如存在），或生成以下内容：

```makefile
######################################
# flash 目标（OpenOCD）
######################################
flash: all
	openocd -f interface/{{INTERFACE}}.cfg -f target/{{TARGET_CFG}}.cfg \
		-c "program $(BUILD_DIR)/$(TARGET).elf verify reset exit"
```

其中：
- `{{INTERFACE}}` — 从配置推断：`stlink` / `cmsis-dap` / `jlink`
- `{{TARGET_CFG}}` — 从芯片型号推断：`stm32f4x` / `stm32f1x` 等

#### 生成 download 目标（J-Link）

使用模板 `templates/makefile_download.tmpl`（如存在），或生成以下内容：

```makefile
######################################
# download 目标（J-Link）
######################################
download: all
	JLinkExe -device {{DEVICE}} -if {{INTERFACE}} -speed 4000 \
		-CommanderScript $(BUILD_DIR)/flash.jlink

$(BUILD_DIR)/flash.jlink:
	@echo "Creating JLink script..."
	@echo "r" > $@
	@echo "h" >> $@
	@echo "loadfile $(BUILD_DIR)/$(TARGET).bin {{FLASH_ADDR}}" >> $@
	@echo "r" >> $@
	@echo "g" >> $@
	@echo "exit" >> $@
```

其中：
- `{{DEVICE}}` — 从芯片型号推断：`STM32F429ZI` / `GD32F450ZK`
- `{{INTERFACE}}` — 从配置读取：`SWD` / `JTAG`
- `{{FLASH_ADDR}}` — 从配置读取，默认 `0x08000000`

生成后追加到 Makefile 末尾，使用注释标记：

```makefile
######################################
# 烧录目标（由 embedded-workflow 生成）
######################################
```

### 第四步：执行烧录

1. 执行对应的 make 目标：
   - `flash` 命令 → `make flash`
   - `download` 命令 → `make download`
2. 捕获输出，实时显示烧录进度

### 第五步：分析结果

#### 烧录成功

```
✅ 烧录成功

📡 调试器: J-Link (SWD)
📦 固件: build/firmware.bin (46912 bytes)
📍 地址: 0x08000000
⏱️  耗时: 2.3 秒

🚀 下一步:
   /embedded-workflow debug    — 启动调试会话
```

#### 烧录失败

分析常见错误并给出建议：

| 错误信息 | 可能原因 | 建议 |
|---------|---------|------|
| `Could not connect to target` | 调试器未连接、芯片未上电 | 检查硬件连接和供电 |
| `Error: libusb_open() failed` | USB 权限问题 | 添加 udev 规则或使用 sudo |
| `Flash write error` | 芯片读保护、Flash 损坏 | 检查芯片保护位或更换芯片 |
| `Unknown device` | 芯片型号配置错误 | 检查 .embedded-workflow.json 中的 device 配置 |

输出格式：

```
❌ 烧录失败

📍 错误信息:
   <原始错误信息>

💡 分析: <错误原因>
🔧 建议: <修复建议>
```

## 注意事项

- 生成的 Makefile 目标应追加到文件末尾，不覆盖已有内容
- 如果用户有 Bootloader，需要正确设置偏移地址
- J-Link 脚本文件应生成到 `build/` 目录，避免污染源码目录
- 烧录前应确保固件已编译（依赖 `all` 目标）
