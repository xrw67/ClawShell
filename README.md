# ClawShell

ClawShell 是为 **AI Agent** 设计的**宿主机侧能力网关**。它使用虚拟化技术将 AI Agent 隔离在独立环境中运行，同时将其能力安全、可控地延伸到宿主机，使 Agent 能够在用户授权与审查下操作真实桌面。

**核心理念**：AI Agent 运行在隔离环境中，通过 ClawShell 这扇受控的窗户访问宿主机；所有操作经安全网关审查，用户始终掌握最终决策权。

---

## 项目愿景

系统由两个隔离域构成：

| 域 | 组成 | 职责 |
|----|------|------|
| **隔离环境** | AI Agent + MCP Client | Agent 在虚拟化环境中运行，无法直接访问宿主机 |
| **宿主机** | ClawShell daemon + 能力插件 | 接收 Agent 请求，经安全审查后执行 GUI 操作 |

Agent 想做任何事，必须经过 ClawShell 的审批。安全网关运行在宿主机上，独立于 Agent，即使 Agent 被劫持，攻击者能发出的也只是「请求」，能否执行由宿主机单独决定。

**能力延伸**：ClawShell 将宿主机的能力（窗口枚举、UI 树获取、点击、输入、滚动、按键等）以 MCP Tool 形式暴露给 Agent，使 Agent 在隔离环境中仍能操作用户真实桌面，但全程处于安全可控状态。

---

## 特性

- **虚拟化隔离** — AI Agent 运行在独立环境中，无法直接访问宿主机，所有操作经 ClawShell 网关审批
- **能力延伸** — 将宿主机 GUI 能力以 MCP Tool 形式安全暴露给 Agent，支持窗口枚举、UI 树、点击、输入等
- **结构化 GUI 描述** — 使用 UI Automation 输出 AXT 紧凑文本，无需截图，纯文本模型即可理解
- **安全网关** — 操作前/后双向审查，危险操作需用户确认，用户始终掌握最终决策权
- **C++ 实现** — 宿主机 daemon 单一二进制分发，无需 Python 运行时
- **MCP 协议** — 兼容 Claude Desktop、Cursor、OpenClaw 等 MCP 客户端

---

## 架构概览

```
╔══════════════════════════════════════════════════════════════╗
║  隔离环境（虚拟机 / 沙箱）                                    ║
║                                                              ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │  AI Agent (OpenClaw / Claude Desktop / Cursor 等)      │  ║
║  │  接收 GUI 描述 → 决策 → 发出操作意图                     │  ║
║  └────────────────────────────┬───────────────────────────┘  ║
║                               │ MCP (stdio)
║                               ▼                              ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │  mcp_server.py — MCP 协议桥接                           │  ║
║  │  将 Tool 调用转发至宿主机                               │  ║
║  └────────────────────────────┬───────────────────────────┘  ║
║                               │                              ║
╚═══════════════════════════════╪══════════════════════════════╝
                                │ 跨域通信 (Named Pipe / vsock 等)
                                │
╔═══════════════════════════════╪══════════════════════════════╗
║  宿主机                        ▼                              ║
║                                                              ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │  crew_shell_service (daemon) — ClawShell 核心           │  ║
║  │  ├── SecurityChain — 安全审查链（入站 + 出站）           │  ║
║  │  ├── CapabilityService — 能力路由                        │  ║
║  │  └── capability_ax.dll — GUI 识别 + 操作执行 (UIA)        │  ║
║  └────────────────────────────┬───────────────────────────┘  ║
║                               │                              ║
║                               ▼                              ║
║  ┌────────────────────────────────────────────────────────┐  ║
║  │  用户桌面应用窗口                                          ║
║  └────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════╝
```

> **Phase 1 兼容**：当前 Windows 版本中，MCP Server 可直接运行在宿主机，通过 Named Pipe 与 daemon 通信，用于开发调试与 Claude Desktop 等本地 MCP 客户端集成。完整虚拟化隔离架构将在后续版本中提供。

---

## 支持的操作

| 操作 | 说明 |
|------|------|
| `list_windows` | 枚举可访问的顶层窗口 |
| `get_ui_tree` | 获取窗口的 UI 元素树（AXT 格式） |
| `click` / `double_click` / `right_click` | 点击元素 |
| `set_value` | 向输入框写入文本 |
| `focus` | 将焦点移至元素 |
| `scroll` | 滚动可滚动区域 |
| `key_press` / `key_combo` | 按键与组合键 |
| `activate_window` | 激活窗口到前台 |

---

## 构建

### 环境要求

- Windows 10 及以上
- CMake 3.20+
- Visual Studio 2022（或 Ninja + MSVC）
- C++20 编译器

### 构建步骤

```powershell
# 配置
cmake -S . -B build -G "Visual Studio 17 2022" -A x64

# 编译
cmake --build build

# 生成 dist 发布目录（exe + dll + config）
cmake --build build --target dist
```

### 构建产物

| 路径 | 说明 |
|------|------|
| `build\src\daemon\Release\crew_shell_service.exe` | 主程序 |
| `build\src\capability\ax\capability_ax.dll` | AX 能力模块 |
| `build\src\security\filter\security_filter.dll` | 安全过滤模块 |
| `dist\` | 发布目录（`make dist` 后生成） |

---

## 运行

### 方式一：从 dist 目录运行（推荐）

```powershell
cd dist
.\bin\crew_shell_service.exe -f -c config\clawshell.toml
```

`-f` 表示前台运行，日志输出到控制台。

### 方式二：从项目根目录运行（开发模式）

```powershell
.\build\src\daemon\Release\crew_shell_service.exe -f -c clawshell.toml
```

顶层 `clawshell.toml` 的 `module_dir` 指向 `build\src`，适合开发调试。

### 命令行参数

| 参数 | 说明 |
|------|------|
| `-c, --config <path>` | 配置文件路径 |
| `-f, --foreground` | 前台运行 |
| `--log-level <level>` | 日志级别：trace / debug / info / warn / error |
| `--socket <path>` | 覆盖 IPC 管道路径 |
| `--module-dir <path>` | 覆盖模块 DLL 目录 |

---

## 测试工具

### 安装 Python 依赖

```powershell
pip install pywin32 mcp
```

### AX 测试客户端（直连 IPC）

```powershell
# 先启动 daemon，再在另一终端运行
python tools\ax_test_client.py

# 批量执行脚本
python tools\ax_test_client.py -f tools\scripts\01_normal_task.txt
```

### 用户确认客户端（需确认时弹出）

当安全策略要求用户确认时，需运行确认客户端：

```powershell
python tools\confirm_client.py
```

### MCP Server（供 AI Agent 使用）

```powershell
python tools\mcp_server.py
```

---

## 与 Claude Desktop 集成

在 `%APPDATA%\Claude\claude_desktop_config.json` 中添加：

```json
{
  "mcpServers": {
    "clawshell": {
      "command": "python",
      "args": ["C:\\path\\to\\ClawShell\\tools\\mcp_server.py"]
    }
  }
}
```

将 `C:\path\to\ClawShell` 替换为实际项目路径。

---

## 安全机制

### 安全审查链

每次能力调用依次经过安全模块的 `preHook`（入站审查）和 `postHook`（出站过滤）：

```
请求到达
  │
  ▼ preHook（执行前）
  │  Deny → 拒绝，不执行
  │  NeedConfirm → 阻塞等待用户确认
  │  Pass → 继续
  │
  ▼ 能力执行
  │
  ▼ postHook（执行后）
  │  可对返回内容脱敏
  │
  ▼ 返回结果
```

**裁决合并**：多个安全模块按优先级执行，最严格者胜出（Deny > NeedConfirm > Pass > Skip）。

### 规则配置

安全规则在 `config/security_filter_rules.toml` 中配置：

```toml
# 拒绝规则
[[deny]]
capability = "capability_ax"
operations = ["key_combo"]
params_field = "keys"
params_patterns = ["Win", "Cmd"]
reason = "Win/Cmd 组合键可触发系统级操作，已拦截"

# 确认规则（需用户确认）
[[confirm]]
capability = "capability_ax"
operations = ["click", "double_click", "right_click"]
reason = "GUI 点击操作需用户确认"
```

---

## 配置说明

### 主配置（clawshell.toml）

```toml
[daemon]
socket_path      = "\\\\.\\pipe\\crew-shell-service"  # IPC 管道
thread_pool_size = 4
log_level        = "debug"
module_dir       = "lib"                              # DLL 目录（相对 dist）
confirm_socket_path = "\\\\.\\pipe\\crew-shell-service-confirm"

[[modules]]
name = "capability_ax"

[[modules]]
name       = "security_filter"
priority   = 10
rules_file = "config\\security_filter_rules.toml"
```

### 路径说明

- **从 dist 运行**：`module_dir = "lib"`，`rules_file` 使用 `config\` 相对路径
- **从项目根运行**：使用顶层 `clawshell.toml`，`module_dir = "build\\src"`

---

## 项目结构

```
ClawShell/
├── include/           # 公开头文件
│   ├── common/       # 错误码、日志、类型
│   ├── core/base/    # 模块接口、安全链、能力服务
│   ├── capability/ax/
│   └── ipc/
├── src/
│   ├── core/         # 核心实现
│   ├── daemon/       # 主程序入口
│   ├── capability/ax/# AX 能力模块（Windows UIA）
│   ├── security/filter/  # 基于 TOML 规则的安全过滤
│   └── ipc/          # Named Pipe + JSON-RPC
├── config/           # 配置文件
├── tools/            # Python 测试与 MCP 工具
└── third_party/      # 第三方 header-only 库
```

---

## 技术栈

| 组件 | 选型 |
|------|------|
| 语言 | C++20 |
| 构建 | CMake |
| JSON | nlohmann/json |
| 错误处理 | tl::expected |
| 日志 | spdlog |
| 配置 | toml++ |
| JSON-RPC | json-rpc-cxx |
| 命令行 | cxxopts |
| GUI API | Windows UI Automation |

---

## 许可证

请参阅项目根目录的 LICENSE 文件。
