# Codex IDE Bridge 1.1.1

包含原来的 8 个 IDE 工具，以及 22 个项目、语义和记忆工具。独立代码操作无需运行 IDE 或 Serena。

## 安装或更新

从插件集目录运行 `install.ps1 -Plugin codex-ide-bridge`；安装到日常环境后重启 Codex，打开新聊天。命令行也可以运行 `codex plugin add codex-ide-bridge@codex-local`，前提是已经注册插件集的本地市场。

首次使用先调用 `initial_instructions`，然后以项目绝对路径调用 `activate_project`。`workspace_root` 可为单次调用指定项目，不改变已激活项目。必须显式激活或传入工作区，避免在插件缓存目录中误操作。

## 独立后端

| 语言 | 实现 | 前置条件 |
|---|---|---|
| JavaScript / TypeScript / JSX / TSX | 随包 TypeScript 编译器语言服务 | 无额外服务器；遵循 tsconfig/jsconfig 的项目范围 |
| Python | 随包 Pyright stdio LSP | 可在个人配置指定解释器和分析设置 |
| Go | 独立 gopls | gopls 与 Go 工具链可用 |
| Rust | 独立 rust-analyzer | 对应服务器与 Rust 工具链可用 |
| C / C++ | 独立 clangd | 对应服务器、编译数据库等项目配置可用 |
| 其他语言 | 可配置 stdio LSP | 需要该语言服务器，并提供 extensions 配置 |

配置 JSON 通过启动 Codex 前设置的 `CODEX_IDE_BRIDGE_CONFIG` 路径读取。不要把个人配置写入安装缓存。例如：

```json
{
  "languageServers": {
    "go": { "command": "gopls", "args": ["serve"] },
    "ruby": { "command": "ruby-lsp", "args": [], "extensions": [".rb"] },
    "python": {
      "command": "C:/Program Files/nodejs/node.exe",
      "args": ["C:/path/to/pyright-langserver.js", "--stdio"],
      "settings": { "python": { "pythonPath": "C:/path/to/python.exe" } }
    }
  }
}
```

不配置 python 时使用插件内置 Pyright。设置某个语言为 `false` 可停用该后端。外部命令直接以 argv 启动，不经过 shell；Windows 的 .cmd 启动器请改为 Node 可执行文件加入口脚本。项目文件不用于配置服务器可执行命令。服务器启动、联网安装和系统依赖安装不会由查询工具自动完成。

## 工具

- 项目：`activate_project`、`get_current_config`、`initial_instructions`、`check_onboarding_performed`、`onboarding`。
- 语义：`get_symbols_overview`、`find_symbol`、`find_referencing_symbols`、`find_declaration`、`find_implementations`、`get_diagnostics_for_file`。
- 编辑：`rename_symbol`、`replace_symbol_body`、`insert_before_symbol`、`insert_after_symbol`、`restart_language_server`。
- 记忆：`list_memories`、`read_memory`、`write_memory`、`edit_memory`、`delete_memory`、`import_serena_memories`。
- 现有 `ide_*` 工具保留。状态始终区分 IDE 是否连接；选区、打开编辑器、界面命令仍需 IDE。

新增编辑工具默认 `dry_run: true`；检查计划后，在用户已经授权的任务内用 `dry_run: false` 应用。`replace_symbol_body` 替换整个定义。源文本中的缩进和换行由调用者明确提供。`find_symbol(include_body: true)` 返回 `file_hash`，可作为编辑的 `expected_hash` 防止使用过期内容。

预览列出文件、哈希以及具体修改前后文本。单处文本和整体预览都有大小限制，`preview_truncated: true` 表示需要进一步检查源文件；写入仍应用全部已验证的修改。

路径和符号范围采用零起始 UTF-16 行/字符坐标，`ide_*` 的既有行列输入仍为一起始。查询返回 `backend_errors`、`truncated`、诊断 `pending` 时要保留这些状态，不能把不完整结果当成“没有引用/没有错误”。全项目扫描最多 5000 个源文件；跳过 .git、依赖、构建目录、符号链接及忽略文件。多根 tsconfig 项目引用仍可能需要分项目激活。

连接 IDE 时优先使用 IDE 工具，保证未保存缓冲区、编辑器撤销和诊断一致。只有在确认没有发现 IDE 端点时，`ide_symbols`、`ide_diagnostics`、`ide_edit`、`ide_rename_symbol` 才回退到独立后端。远程写入超时或出错不会触发第二次磁盘写入。独立操作不能修改未保存的 IDE 缓冲区。

## 项目记忆与 Serena 迁移

项目元数据、初始化摘要和原生记忆分别位于 `.codex-ide-bridge/project.json`、`.codex-ide-bridge/onboarding.md`、`.codex-ide-bridge/memories/`，跨插件升级和服务器重启保留。

现有 `.serena/memories/*.md` 可带 `source: "serena"` 只读读取。`read_memory` 默认优先原生同名记忆；列表返回来源。`import_serena_memories` 复制到原生目录，默认跳过已存在名称；不会编辑或删除原 Serena 文件。项目激活不会自动迁移真实项目的记忆。

`onboarding` 不带摘要时检查项目清单，返回建议命令但不执行；了解项目并验证相关事实后，再调用 `onboarding(summary: "...")` 保存摘要。记忆是项目数据，应核对其适用性；不要写入密钥。

## 替代范围

见 [能力对照](SERENA-PARITY.md)。实现了常用工作流的功能对齐，并非 Serena 全量 API 或所有语言后端的兼容实现。没有导入 Serena 运行时，也不需要启动它。

TypeScript、Pyright、ignore 等依赖的许可证随包保留。源代码位于 `source/ide/`，完整重建脚本位于插件集的 `development/`。

## 与 DSH 共用核心

本版与 DSH IDE Bridge 0.4.0 共用源码。DSH 项目的 `core/` 为维护入口；`development/sync-ide.mjs` 同步到 `upstream/DSH-IDE-Bridge/core/`，Codex 从快照构建，运行时不依赖 DSH。两端共用项目中的 `.codex-ide-bridge/` 元数据与记忆。
