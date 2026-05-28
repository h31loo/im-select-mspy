# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# CMake (primary) — 输出在 build/Release/im-select-mspy.exe
md build && cd build
cmake ..
cmake --build . --config Release

# 指定目标平台 (x64 或 Win32)
cmake -A Win32 ..        # 32位
cmake -A x64 ..          # 64位

# xmake (alternative)
xmake
```

## 测试

没有单元测试框架。验证可执行文件是否正常工作的方式：

```bash
# 直接运行查看当前输入法状态
.\build\Release\im-select-mspy.exe

# 用 Node.js 脚本测试 UTF-8 输出（VS Code 扩展需要 UTF-8）
node check_launch.js
```

## Architecture

这是一个 Windows 下为微软拼音输入法设计的输入法切换 CLI 工具，配合 VsCodeVim 使用。单文件 C++17 项目（`main.cc`），无外部依赖，仅链接系统 `comsuppw.lib`。

核心流程：
1. 通过 **UIAutomation** 在任务栏找到"托盘输入指示器"按钮，用正则匹配获取当前输入法状态（中文/英文）
2. 如果任务栏隐藏找不到按钮，回退到 **输入法工具栏** (`get_ime_button_from_toolbar`) 查找
3. 切换输入法时通过 **SendInput** 模拟按键实现（默认 `Shift`）

CLI 参数：
- `-t` 任务栏名称，默认 `"任务栏"`
- `-i` 捕获输入法状态的正则，默认 `"输入指示\\S*\\s+(\\S+)"`（匹配"任务栏输入指示"和"托盘输入指示器"两种命名）
- `-k` 切换快捷键，默认 `shift`，支持 `ctrl+space`、`ctrl+0x7C`（16进制 Virtual Key）等组合
- `--toolbar` 输入法工具栏名称，默认 `"Windows 输入体验"`
- `--toolbar-i` 工具栏状态正则，默认 `"中/英文, (\\S+)"`
- `-v` 输出调试信息

无参数调用时输出当前输入法状态；传入模式名（如 `"中文模式"`）时切换到该模式。

输出通过 `w2utf8()` 将内部 `wstring` 转为 UTF-8 后写到 stdout，因为 VS Code 扩展需要 UTF-8 输出。内部所有字符串处理使用 `wstring`/`wchar_t*`。

### 关键函数

| 函数 | 作用 |
|------|------|
| `get_ime_button()` | 从任务栏遍历按钮，用正则匹配找到输入法指示器 |
| `get_ime_button_from_toolbar()` | 回退方案：从输入法工具栏 (`UIA_ListItemControlTypeId`) 查找 |
| `get_input_from_string()` | 解析快捷键字符串 (如 `"ctrl+shift"`) → `INPUT` 数组 (先按下再释放，释放顺序与按下相反) |
| `vk_from_text()` | 按键名/16进制 → Virtual Key 码 (`"shift"` → `VK_SHIFT`, `"0x7C"` → `VK_F13`) |
| `w2utf8()` | `wstring` → UTF-8 `string`，用于 stdout 输出 |
| `split_string()` | 按分隔符拆分字符串 (用于解析 `+` 连接的快捷键) |
| `parse_options()` | 命令行参数解析，带默认值回填 |
| `get_element_name()` | 从 UIAutomation Element 获取 `CurrentName` |

### COM 生命周期

`CoInitialize` / `CoUninitialize` 在 `wmain` 中配对调用。UIAutomation 的 `_com_ptr_t` 智能指针自动管理 COM 对象引用计数，无需手动 `Release`。

## CI

GitHub Actions 仅在推送 `v*.*.*` 标签时触发，构建 Windows x64/x86 Release 并发布到 GitHub Release。
