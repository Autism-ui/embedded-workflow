---
name: debug
description: 启动调试会话 — 启动 GDB Server，连接调试器，支持 RTT 和 Shell 模式
---

# 调试会话

你是一个嵌入式软件开发助手，正在帮助用户启动调试会话。请严格按照以下步骤执行。

## 参数解析

| 参数 | 说明 | 默认 |
|------|------|------|
| `--rtt` | 启动 Segger RTT Viewer 模式 | 否 |
| `--shell` | 连接串口终端（letter-shell 等） | 否 |
| `--port <端口>` | 指定串口设备或 GDB 端口 | 自动检测 |
| `--baud <波特率>` | 串口波特率 | 115200 |

## 执行流程

### 第一步：环境检测

1. 读取 `.embedded-workflow.json` 获取调试器配置
2. 检查固件文件是否存在（`build/*.elf`）
   - 不存在 → 提示用户先运行 `/embedded-workflow build`
3. 检测系统中可用的调试工具：
   - `JLinkGDBServer` / `JLinkGDBServerCLExe`
   - `openocd`
   - `st-util`
   - `arm-none-eabi-gdb` / `gdb-multiarch`
4. 检测 VS Code 是否安装了 `cortex-debug` 插件

### 第二步：根据模式执行

#### 模式 A：GDB 调试（默认）

1. **选择 GDB Server** — 根据调试器类型自动选择：

| 调试器 | GDB Server 命令 |
|--------|-----------------|
| J-Link | `JLinkGDBServer -device {{DEVICE}} -if SWD -speed 4000 -port 2331` |
| ST-Link | `st-util --listen_port 4242` 或 `openocd -f interface/stlink.cfg -f target/{{TARGET}}.cfg` |
| DAP-Link | `openocd -f interface/cmsis-dap.cfg -f target/{{TARGET}}.cfg` |

2. **启动 GDB Server** — 在后台启动，捕获输出确认连接成功
3. **提供连接方式**：

```
✅ GDB Server 已启动

📡 调试器: J-Link (SWD)
🔌 GDB 端口: localhost:2331

连接方式：

方式 1 — 命令行 GDB:
   arm-none-eabi-gdb build/firmware.elf
   (gdb) target remote :2331
   (gdb) monitor reset
   (gdb) load
   (gdb) break main
   (gdb) continue

方式 2 — VS Code 图形化调试:
   按 F5 启动调试（需要已配置 launch.json）
   如未配置，运行 /embedded-workflow init 生成配置

按 Ctrl+C 停止 GDB Server
```

#### 模式 B：RTT 模式（`--rtt`）

1. 检查 `JLinkRTTClient` 是否可用
   - 不可用 → 提示安装 J-Link 软件包
2. 启动 RTT Viewer：

```bash
# 先启动 GDB Server（如未运行）
JLinkGDBServer -device {{DEVICE}} -if SWD -speed 4000 &

# 再启动 RTT Client
JLinkRTTClient
```

3. 输出提示：

```
✅ RTT Viewer 已启动

📡 RTT 通道 0 已连接
💡 提示: 确保固件中已初始化 SEGGER_RTT

按 Ctrl+C 退出
```

#### 模式 C：Shell 模式（`--shell`）

1. 检测可用的串口设备：
   - Linux: `/dev/ttyACM*`、`/dev/ttyUSB*`
   - macOS: `/dev/cu.usbmodem*`、`/dev/cu.usbserial*`
2. 如果有多个串口，让用户选择
3. 使用可用的终端工具连接：
   - 优先使用 `minicom`
   - 备选 `picocom`
   - 再备选 `screen`

```bash
minicom -D {{SERIAL_PORT}} -b {{BAUD_RATE}}
```

4. 输出提示：

```
✅ 串口终端已连接

🔌 端口: /dev/ttyACM0
📊 波特率: 115200

按 Ctrl+A X 退出 minicom
```

### 第三步：异常处理

| 错误情况 | 诊断 | 建议 |
|---------|------|------|
| GDB Server 启动失败 | 调试器未连接、驱动未安装 | 检查 USB 连接和驱动安装 |
| GDB 连接超时 | 端口被占用 | 检查端口占用：`lsof -i :2331` |
| RTT 无输出 | 固件未初始化 RTT | 检查固件中 SEGGER_RTT_Init() 调用 |
| 串口打不开 | 权限问题 | `sudo usermod -aG dialout $USER` |

## 注意事项

- GDB Server 应在后台运行，方便用户同时使用 GDB 客户端
- 提供命令行和 VS Code 两种调试方式的说明
- RTT 模式需要 J-Link 调试器，其他调试器不支持
- Shell 模式独立于调试器，直接使用串口通信
- 退出时提醒用户停止后台的 GDB Server 进程
