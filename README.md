# mcp-sync

**MCP 配置同步**：用**一份** `mcp-servers.json`（本仓库中的 MCP 服务清单）作为数据源，**同时**同步到 **Cursor**、**Codex**、**Claude Code**，以及 **Xcode 内置** Coding Assistant（Intelligence 里的 Codex / Claude Agent），避免在多个工具里重复维护 MCP 列表。

| 项目 | 说明 |
|------|------|
| 工程名 | **mcp-sync** |
| 默认远程 | `https://github.com/i-stack/mcp-sync.git`（见下方「克隆」） |

## 功能概览

| 目标 | 同步方式 | 说明 |
|------|----------|------|
| **Cursor** | 符号链接 | `~/.cursor/mcp.json` → 本仓库的 `mcp-servers.json`，改源文件即生效 |
| **Codex（终端 / CLI）** | 生成 TOML + 合并 | 写入 `~/.codex/mcp.generated.toml`，并把 `[mcp_servers.*]` 合并进 `~/.codex/config.toml` 中带标记的区块，**不覆盖**你在该文件中的其他设置 |
| **Codex（Xcode 内）** | 同上 | 与上相同的生成与合并逻辑，写入 **`~/Library/Developer/Xcode/CodingAssistant/codex/`**（`mcp.generated.toml` + `config.toml`）。仅影响在 Xcode 里启动的 Codex，与 `~/.codex` 独立（见 Apple [Setting up coding intelligence](https://developer.apple.com/documentation/Xcode/setting-up-coding-intelligence)） |
| **Claude Code（终端）** | JSON 合并 | 将 `mcpServers` 合并进 `~/.claude.json`，**仅更新** MCP 相关键，其它配置保留 |
| **Claude Agent（Xcode 内）** | JSON 合并 | 将 `mcpServers` 合并进 **`~/Library/Developer/Xcode/CodingAssistant/ClaudeAgentConfig/.claude.json`**。Xcode 侧 MCP 挂在 **`projects` → 各工程路径 → `mcpServers`**：脚本会对**已有**的每个工程条目合并（同名 key 以仓库为准覆盖），其它键保留；若无 `projects` 则写入根级 `mcpServers` |

一次执行 `sync_all.sh` 即可完成：Cursor 软链 → Codex（含 Xcode 目录）→ Claude（含 Xcode 配置）。

## 环境要求

- macOS / Linux（需 `bash`）
- Python 3（用于 Codex / Claude 同步脚本）
- 使用 **Xcode 内 Codex / Claude Agent** 时需在 **macOS**，且 Xcode 已按 [Apple 文档](https://developer.apple.com/documentation/Xcode/setting-up-coding-intelligence) 在 Intelligence 中启用相应能力（配置目录为 `~/Library/Developer/Xcode/CodingAssistant/`）

## 克隆仓库

```bash
git clone https://github.com/i-stack/mcp-sync.git
cd mcp-sync
```

## 快速开始

1. 进入本仓库目录（克隆见上；本地文件夹名可自定）。

2. 准备本地配置（勿提交密钥）：

   ```bash
   cp mcp-servers.json.example mcp-servers.json
   ```

   编辑 `mcp-servers.json`，填入你的 Token、项目 ID 等。

   `mcpServers` 里每一项的**键名**由你自定，会在 Cursor / 客户端里作为该 MCP 的显示名；**建议按功能命名**（见 `mcp-servers.json.example`，如 `browser-automation`、`api-documentation`），不必与底层包名一致。

3. 执行同步：

   ```bash
   chmod +x sync_all.sh   # 仅需首次
   ./sync_all.sh
   ```

4. 重启或重新加载 **Cursor / Codex / Claude Code**（若当前会话未自动读取新配置）。若在 **Xcode** 中使用 Coding Assistant，建议**重启 Xcode** 后再试。

## 脚本说明

| 文件 | 作用 |
|------|------|
| `sync_all.sh` | 一键：Cursor 软链 → `sync_mcp.py`（含 Xcode `codex` 目录）→ `sync_claude.py`（含 Xcode `ClaudeAgentConfig`） |
| `sync_mcp.py` | 生成 Codex 用 TOML，并合并进 `~/.codex/config.toml` 与 **`~/Library/Developer/Xcode/CodingAssistant/codex/config.toml`** |
| `sync_claude.py` | 将 `mcpServers` 合并进 `~/.claude.json` 与 **`~/Library/Developer/Xcode/CodingAssistant/ClaudeAgentConfig/.claude.json`**（Xcode 侧为按工程 `projects.*.mcpServers` 合并） |
| `mcp-servers.json` | **唯一数据源**（本地文件，已加入 `.gitignore`） |
| `mcp-servers.json.example` | 无密钥的模板，可安全提交到 Git |

也可单独运行：

```bash
python3 sync_mcp.py
python3 sync_claude.py
```

单独执行 `sync_mcp.py` 时仍会更新 **本机 Codex** 与 **Xcode Codex** 目录下的 TOML / `config.toml`；单独执行 `sync_claude.py` 会更新 **`~/.claude.json`** 与 **Xcode 内** `.claude.json`。Cursor 需自行保证 `~/.cursor/mcp.json` 指向本仓库的 `mcp-servers.json`（或直接运行 `sync_all.sh`）。

## 蓝湖 MCP（可选）

`lanhu-mcp/` 为蓝湖相关 MCP 服务实现，需单独按目录内说明启动服务；`mcp-servers.json` 中通过 `url` 指向本地 HTTP 端点即可，与上述同步逻辑独立。

## 安全提示

- 勿将含真实 Token 的 `mcp-servers.json` 提交到远程仓库。
- 若密钥曾泄露，请在对应平台**轮换** Token。
