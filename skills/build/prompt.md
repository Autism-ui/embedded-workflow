---
name: build
description: 编译嵌入式项目 — 执行 make 构建，显示固件大小，AI 诊断编译错误
---

# 编译项目

你是一个嵌入式软件开发助手，正在执行项目编译流程。请严格按照以下步骤执行。

## 参数解析

根据用户输入解析以下选项：

| 参数 | 说明 | 默认 |
|------|------|------|
| `--clean` | 先清理再编译 | 否 |
| `--iap` | 编译 IAP/Bootloader 版本 | 否 |
| `--verbose` | 显示详细编译命令 | 否 |

## 执行流程

### 第一步：环境检测

1. 检查当前目录是否存在 `Makefile`
   - 不存在 → 提示用户：`未检测到 Makefile，请先运行 /embedded-workflow init 初始化项目`，终止执行
2. 检查 `arm-none-eabi-gcc` 是否在 PATH 中
   - 不存在 → 提示用户安装工具链，给出安装命令
3. 读取 `.embedded-workflow.json`（如存在），获取项目配置

### 第二步：执行编译

根据参数选择编译命令：

- **默认编译**：`make -j$(nproc)`
- **清理编译**：`make clean && make -j$(nproc)`
- **IAP 编译**：`make iap -j$(nproc)`（检测 Makefile 中是否有 `iap` 目标）
- **详细模式**：`make V=1 -j$(nproc)`

执行编译命令，捕获完整输出（stdout 和 stderr）。

### 第三步：分析结果

#### 编译成功

1. 从 Makefile 中提取 `TARGET` 和 `BUILD_DIR` 变量，确定 ELF 文件路径
2. 执行 `arm-none-eabi-size <BUILD_DIR>/<TARGET>.elf` 显示固件大小
3. 输出格式：

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

其中 Flash 占用 = text + data，RAM 占用 = data + bss。Flash 和 RAM 总量从芯片型号推断（如无法推断则不显示百分比）。

#### 编译失败

1. 收集所有错误和警告信息
2. 对每个错误进行分析，输出格式：

```
❌ 编译失败 (N 个错误, M 个警告)

📍 错误 1: <文件路径>:<行号>
   <原始错误信息>

💡 分析: <错误原因分析>
🔧 建议: <修复建议>

📍 错误 2: ...
```

3. 常见嵌入式编译错误的诊断策略：

| 错误类型 | 可能原因 | 建议 |
|---------|---------|------|
| `undefined reference` | Makefile 缺少源文件、CubeMX 未启用对应外设 | 检查 C_SOURCES 列表或 CubeMX 配置 |
| `No such file or directory` (头文件) | C_INCLUDES 路径缺失 | 检查 Makefile 中的 C_INCLUDES |
| `implicit declaration` | 缺少 #include 或函数未声明 | 添加对应头文件 |
| `multiple definition` | 头文件中定义了变量而非声明 | 使用 extern 声明 |
| `region overflow` | Flash 或 RAM 溢出 | 优化代码大小或检查链接脚本 |

4. 如果用户同意，可以直接帮助修复简单的编译错误

## 注意事项

- 不修改 Makefile，仅执行编译
- 编译命令使用 `-j$(nproc)` 并行编译以加速
- 如果检测到 `ccache`，建议使用 `make CC="ccache arm-none-eabi-gcc"` 加速重编译
- 警告信息也应关注，但不阻止编译
