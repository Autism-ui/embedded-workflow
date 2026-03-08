---
name: lint
description: 静态代码检查 — 使用 cppcheck/clang-tidy 检查代码，AI 分析结果并过滤嵌入式误报
---

# 静态代码检查

你是一个嵌入式软件开发助手，正在执行静态代码检查流程。请严格按照以下步骤执行。

## 参数解析

| 参数 | 说明 | 默认 |
|------|------|------|
| `<路径>` | 指定检查的目录或文件 | 业务层目录 |
| `--tool <工具>` | 指定检查工具 | 自动检测 |
| `--severity <级别>` | 最低报告级别 | warning |

支持的检查工具：`cppcheck` / `clang-tidy`

## 执行流程

### 第一步：环境检测

1. 检查可用的静态检查工具：
   - `cppcheck` — 优先使用
   - `clang-tidy` — 备选
   - 都不存在 → 提示安装：`sudo apt install cppcheck` 或 `brew install cppcheck`
2. 读取 `.embedded-workflow.json` 获取 lint 配置（如存在）

### 第二步：检测 Makefile 目标

1. 检查 Makefile 中是否已有 `lint` 或 `check` 相关目标
2. 如果目标存在 → 跳到第四步直接调用
3. 如果目标不存在 → 进入第三步生成目标

### 第三步：生成 Makefile 目标

使用模板 `templates/makefile_lint.tmpl`（如存在），或生成以下内容：

#### cppcheck 版本

```makefile
######################################
# lint 目标（cppcheck）
# 由 embedded-workflow 生成
######################################

# 检查目录
LINT_DIRS ?= fal pal common

# 排除目录
LINT_EXCLUDE ?= hal/core

# 包含路径（用于分析）
LINT_INCLUDES = $(patsubst -I%,--include=%,$(filter -I%,$(C_INCLUDES)))

lint:
	cppcheck --enable=warning,style,performance,portability \
		--suppress=missingIncludeSystem \
		--suppress=unmatchedSuppression \
		$(LINT_INCLUDES) \
		$(patsubst %,-i %,$(LINT_EXCLUDE)) \
		--error-exitcode=1 \
		$(LINT_DIRS)
```

#### clang-tidy 版本

```makefile
######################################
# lint 目标（clang-tidy）
# 由 embedded-workflow 生成
######################################

LINT_SOURCES = $(shell find fal pal common -name "*.c" 2>/dev/null)

lint:
	clang-tidy $(LINT_SOURCES) -- $(C_INCLUDES) $(C_DEFS)
```

生成后追加到 Makefile 末尾。

### 第四步：执行检查

根据参数选择执行命令：

- **默认检查**：`make lint`
- **指定路径**：直接调用检查工具

```bash
# cppcheck 直接调用
cppcheck --enable=warning,style,performance,portability \
    --suppress=missingIncludeSystem \
    <路径>

# clang-tidy 直接调用
clang-tidy <路径>/*.c -- -I<include_paths>
```

捕获完整输出。

### 第五步：AI 分析结果

对检查结果进行智能分析，分类和过滤：

#### 分类

将结果按严重程度分类：

| 级别 | 说明 | 处理 |
|------|------|------|
| error | 明确的代码错误 | 必须修复 |
| warning | 潜在问题 | 建议修复 |
| style | 代码风格问题 | 可选修复 |
| performance | 性能优化建议 | 评估后修复 |

#### 过滤嵌入式误报

嵌入式开发中常见的误报，AI 应识别并标记：

| 误报类型 | 说明 | 处理 |
|---------|------|------|
| `unreadVariable` 用于寄存器写入 | 写入硬件寄存器后不需要读取 | 标记为误报 |
| `variableScope` 用于 volatile 变量 | ISR 共享变量需要文件作用域 | 标记为误报 |
| `unusedFunction` 用于回调函数 | HAL 回调函数由框架调用 | 标记为误报 |
| 类型转换警告用于寄存器地址 | 嵌入式中常见 `(uint32_t *)0x40000000` | 标记为误报 |
| `nullPointer` 用于外设基地址 | 如 `GPIOA` 等宏展开为常量地址 | 标记为误报 |

#### 输出格式

```
📊 静态检查报告

📍 发现 N 个问题 (X 个错误, Y 个警告, Z 个风格)

❌ 错误 (必须修复):

  1. fal/motor/motor.c:45
     [error] Array index out of bounds
     💡 数组 'buffer' 大小为 10，但访问了 index 12
     🔧 修改索引范围或增大数组大小

⚠️ 警告 (建议修复):

  2. fal/sensor/sensor.c:78
     [warning] Variable 'temp' is assigned but never used
     💡 变量已赋值但后续未使用
     🔧 移除未使用的变量或添加使用逻辑

📝 风格 (可选修复):

  3. common/utils.c:23
     [style] Variable 'i' can be declared in smaller scope
     💡 变量可以在更小的作用域中声明

🔇 已过滤的误报: M 个
   - 2 个 register write 相关误报
   - 1 个 HAL callback 相关误报

✅ 总结:
   需要修复: X 个错误 + Y 个警告
   建议修复: Z 个风格问题
   已过滤: M 个嵌入式误报
```

## 注意事项

- 默认只检查业务层代码（fal/、pal/、common/），不检查 CubeMX 生成的代码
- 嵌入式代码的特殊性：volatile 变量、寄存器操作、ISR 回调等需要特殊处理
- cppcheck 需要传入正确的 include 路径才能准确分析
- 检查结果应结合项目上下文给出修复建议，而不是机械地报告问题
- 如果用户同意，可以直接帮助修复简单的检查问题
