---
name: format
description: 代码格式化 — 检测 Makefile 目标，使用 clang-format 格式化业务层代码
---

# 代码格式化

你是一个嵌入式软件开发助手，正在执行代码格式化流程。请严格按照以下步骤执行。

## 参数解析

| 参数 | 说明 | 默认 |
|------|------|------|
| `--check` | 仅检查格式，不修改文件（CI 用） | 否 |
| `--all` | 格式化所有代码（包括 HAL 层） | 否 |
| `<路径>` | 指定格式化的目录或文件 | 业务层目录 |

## 执行流程

### 第一步：环境检测

1. 检查 `clang-format` 是否可用
   - 不可用 → 提示安装：`sudo apt install clang-format` 或 `brew install clang-format`
2. 检查 `.clang-format` 配置文件是否存在
   - 不存在 → 提示用户运行 `/embedded-workflow init` 生成配置
3. 读取 `.embedded-workflow.json` 获取格式化配置（如存在）

### 第二步：检测 Makefile 目标

1. 检查 Makefile 中是否已有 `format` 相关目标或变量
   - 查找 `format:` 目标
   - 查找 `FORMAT_RESULT` 变量或类似的格式化命令
2. 如果目标存在 → 跳到第四步直接调用
3. 如果目标不存在 → 进入第三步生成目标

### 第三步：生成 Makefile 目标

使用模板 `templates/makefile_format.tmpl`（如存在），或生成以下内容：

```makefile
######################################
# format 目标（clang-format）
# 由 embedded-workflow 生成
######################################

# 业务层源文件目录
FORMAT_DIRS = fal pal common utilities/devices drivers middlewares

# 格式化所有业务层代码
format:
	@echo "Formatting code..."
	@find $(FORMAT_DIRS) -type f \( -name "*.c" -o -name "*.h" \) \
		-exec clang-format -i {} \;
	@echo "Format complete."

# 检查格式（CI 用）
format-check:
	@echo "Checking code format..."
	@find $(FORMAT_DIRS) -type f \( -name "*.c" -o -name "*.h" \) \
		-exec clang-format --dry-run --Werror {} \;
```

如果 `.embedded-workflow.json` 中配置了 `format.include` 和 `format.exclude`，使用配置的路径。

生成后追加到 Makefile 末尾，使用注释标记。

### 第四步：执行格式化

根据参数选择执行命令：

- **默认格式化**：`make format`
- **仅检查**：`make format-check`
- **指定路径**：直接调用 `clang-format -i <路径>/*.c <路径>/*.h`

执行命令，捕获输出。

### 第五步：分析结果

#### 格式化成功

统计格式化的文件数量：

```
✅ 格式化完成

📊 统计：
   格式化文件: 45 个
   修改文件: 12 个
   跳过文件: 33 个（已符合规范）

📁 格式化的目录：
   fal/  pal/  common/  utilities/devices/

💡 提示: 使用 git diff 查看修改内容
```

#### 检查模式（--check）

如果有文件不符合格式规范：

```
❌ 格式检查失败

📍 以下文件不符合格式规范：
   fal/motor/motor.c
   pal/pal.c
   common/utils.c

🔧 修复方法:
   /embedded-workflow format    # 自动格式化
```

如果所有文件符合规范：

```
✅ 格式检查通过

所有文件符合代码规范。
```

### 第六步：Git 集成（可选）

如果项目使用 git，提示用户：

```
💡 建议：
   1. 查看修改: git diff
   2. 提交格式化: git add -u && git commit -m "style: format code"
   3. 配置 pre-commit hook 自动格式化
```

## 注意事项

- 默认只格式化业务层代码（fal/、pal/、common/ 等），不格式化 CubeMX 生成的 HAL 层代码
- 使用 `--all` 参数可以格式化所有代码，但需谨慎（可能破坏 CubeMX 生成的代码）
- `.clang-format` 配置应与团队约定一致
- 格式化前建议先提交代码，避免丢失修改
- CI 环境使用 `--check` 参数，不修改文件
