# Embedded Workflow Skill 实施计划

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 创建一个通用的嵌入式开发工作流 Claude Code Skill，支持项目初始化、编译构建、烧录调试、模块生成和代码规范检查。

**Architecture:** 采用子命令架构，每个子命令对应一个独立的 prompt.md 文件。核心逻辑是检测项目环境（Makefile、.ioc、目录结构），优先使用已有的 Makefile 目标，缺失时智能生成并追加。

**Tech Stack:** Claude Code Skill 框架、Markdown（Prompt 文件）、模板文件（代码生成）

---

## 实施策略

本计划分为三个阶段：
1. **第一阶段**：基础设施 + init 子命令（最复杂）
2. **第二阶段**：build / flash / download / debug 子命令
3. **第三阶段**：module / format / lint 子命令

每个任务遵循：创建目录 → 编写 prompt.md → 创建模板文件 → 测试验证

---

## 第一阶段：基础设施与 init 子命令

### Task 1: 创建 Skill 基础结构

**Files:**
- Create: `skill.json`
- Create: `README.md`

**Step 1: 创建项目根目录结构**

```bash
mkdir -p skills/{init,build,flash,debug,module,format,lint}
mkdir -p templates
```

**Step 2: 创建 skill.json**

在项目根目录创建 `skill.json`：

```json
{
  "name": "embedded-workflow",
  "version": "0.1.0",
  "description": "嵌入式软件开发全流程工作流辅助工具",
  "commands": {
    "init": "skills/init/prompt.md",
    "build": "skills/build/prompt.md",
    "flash": "skills/flash/prompt.md",
    "download": "skills/flash/prompt.md",
    "debug": "skills/debug/prompt.md",
    "module": "skills/module/prompt.md",
    "format": "skills/format/prompt.md",
    "lint": "skills/lint/prompt.md"
  }
}
```

**Step 3: 创建 README.md**

```markdown
# Embedded Workflow Skill

嵌入式软件开发全流程工作流辅助工具。

## 使用方式

\`\`\`bash
/embedded-workflow init              # 项目初始化
/embedded-workflow build             # 编译项目
/embedded-workflow flash             # 烧录固件
/embedded-workflow module add fal pump  # 生成模块
\`\`\`

## 支持的平台

- STM32 全系列 / GD32 全系列 / 其他 Cortex-M 芯片
- FreeRTOS / RT-Thread / Zephyr / 裸机
- Makefile / CMake / Meson
```

**Step 4: 提交基础结构**

```bash
git add skill.json README.md skills/ templates/
git commit -m "feat: create embedded-workflow skill structure"
```

---

### Task 2: 实现 init 子命令

**Files:**
- Create: `skills/init/prompt.md`
- Create: `templates/.gitignore.tmpl`
- Create: `templates/.clang-format.tmpl`
- Create: `templates/vscode_launch.json.tmpl`
- Create: `templates/vscode_tasks.json.tmpl`
- Create: `templates/vscode_c_cpp_properties.json.tmpl`

参考设计文档中的 init 子命令规格，创建完整的 prompt.md 文件，包含：
1. 检测 CubeMX 生成的内容
2. 交互式确认配置
3. 创建业务层目录结构
4. 追加 Makefile 业务层配置
5. 生成 VS Code 配置
6. 生成 .gitignore 和 .embedded-workflow.json

---

### Task 3: 创建 init 所需的模板文件

为 init 子命令创建所有必需的模板文件（.gitignore、.clang-format、VS Code 配置等）。

---

## 第二阶段：编译与烧录子命令

### Task 4: 实现 build 子命令

**Files:**
- Create: `skills/build/prompt.md`

创建 build 子命令的 prompt，实现：
1. 检测 Makefile 存在
2. 执行 make 编译
3. 成功时显示固件大小
4. 失败时 AI 诊断错误

---

### Task 5: 实现 flash/download 子命令

**Files:**
- Create: `skills/flash/prompt.md`
- Create: `templates/makefile_flash.tmpl`
- Create: `templates/makefile_download.tmpl`

创建烧录子命令的 prompt，实现：
1. 检测 Makefile 中的 flash/download 目标
2. 有则调用，无则生成
3. 失败时诊断调试器连接问题

---

### Task 6: 实现 debug 子命令

**Files:**
- Create: `skills/debug/prompt.md`

创建调试子命令的 prompt，实现：
1. 启动 GDB Server
2. 连接 GDB 或提示使用 VS Code
3. 支持 RTT 和 Shell 模式

---

## 第三阶段：模块生成与代码规范

### Task 7: 实现 module 子命令

**Files:**
- Create: `skills/module/prompt.md`
- Create: `templates/fal_module.c.tmpl`
- Create: `templates/fal_module.h.tmpl`
- Create: `templates/device_uart.c.tmpl`
- Create: `templates/device_iic.c.tmpl`
- Create: `templates/device_spi.c.tmpl`
- Create: `templates/device_can.c.tmpl`
- Create: `templates/device_adc.c.tmpl`
- Create: `templates/device_gpio.c.tmpl`
- Create: `templates/device_tim.c.tmpl`

创建模块生成子命令，实现：
1. 扫描项目已有代码风格
2. 根据模板生成 FAL 模块或设备驱动
3. 自动追加到 Makefile

---

### Task 8: 实现 format 子命令

**Files:**
- Create: `skills/format/prompt.md`
- Create: `templates/makefile_format.tmpl`

创建格式化子命令，实现：
1. 检测 Makefile 中的 format 目标
2. 有则调用，无则生成
3. 输出格式化结果

---

### Task 9: 实现 lint 子命令

**Files:**
- Create: `skills/lint/prompt.md`
- Create: `templates/makefile_lint.tmpl`

创建静态检查子命令，实现：
1. 检测 Makefile 中的 lint 目标
2. 有则调用，无则生成
3. AI 分析结果并过滤误报

---

## 测试与文档

### Task 10: 端到端测试

在真实的嵌入式项目中测试所有子命令：
1. 测试 init 在 CubeMX 项目上的初始化
2. 测试 build 的编译和错误诊断
3. 测试 flash/download 的烧录
4. 测试 module 的代码生成
5. 测试 format 和 lint

---

### Task 11: 完善文档

**Files:**
- Update: `README.md`
- Create: `docs/USAGE.md`
- Create: `docs/TEMPLATES.md`

补充完整的使用文档、模板说明和常见问题解答。

---

## 实施建议

1. **优先级**：按 Task 1 → Task 2 → Task 4 → Task 5 的顺序实施，这是最核心的功能
2. **测试驱动**：每完成一个 Task，立即在真实项目中测试
3. **迭代优化**：根据实际使用反馈，持续优化 prompt 和模板
4. **文档同步**：每个功能完成后立即更新文档

