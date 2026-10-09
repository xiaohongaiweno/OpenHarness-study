# OpenHarness 源码深度剖析：CLI 入口与 UI 前后端（cli / ui / frontend-terminal）

本文是对 OpenHarness 命令行入口（`cli.py`，2551 行）、UI 后端服务（`src/openharness/ui/`，含 OHJSON 协议、运行时装配、三种 UI 适配器）以及 React Ink 终端前端（`frontend/terminal/`）的一次完整源码深潜。文中所有类名、函数名、字段名、文件路径、行号均来自当前仓库源码，可直接对照查阅。

## 目录

- 一、总体架构与进程拓扑
- 二、CLI 入口 cli.py 全面拆解
- 三、OHJSON 协议（前后端 IPC）
- 四、runtime.py 运行时状态机
- 五、app.py 会话入口与 coordinator_drain
- 六、textual_app.py 备选 TUI
- 七、react_launcher.py 前端进程拉起
- 八、React TUI 前端（frontend/terminal）
- 九、端到端交互时序
- 十、关键符号总表
- 十一、设计观察与演进痕迹

---

## 一、总体架构与进程拓扑

### 1.1 双进程模型

OpenHarness 的交互式 UI 采用「Python 后端 + React Ink 终端前端」的双进程架构：

- **后端**：`python -m openharness --backend-only` 启动的 `ReactBackendHost`，通过 stdin/stdout 与前端交换「换行分隔的 OHJSON 消息」。
- **前端**：Node.js 用 `tsx` 直接运行 `src/index.tsx`（React Ink 渲染到终端）。前端进程在启动时读取环境变量 `OPENHARNESS_FRONTEND_CONFIG` 拿到 `backend_command`，再 `spawn` 出后端进程，因此**前端是父进程、后端是子进程**（POSIX 下以独立进程组存在，退出时整体回收）。

```
┌───────────────────────────── user terminal ─────────────────────────────┐
│  oh / openharness / openh（pyproject.toml:48-50）                         │
│         │                                                                │
│         ▼                                                                │
│  python -m openharness  ──► openharness/__main__.py（4 行，调 app()）      │
│         │  app（cli.py:751 的 typer 应用，invoke_without_command=True）    │
│         │                                                                │
│         ├─ 子命令（mcp/plugin/auth/provider/config/cron/autopilot）        │
│         │      ──► 直接打印输出后退出                                      │
│         ├─ --dry-run ──► _build_dry_run_preview() 打印预览后退出           │
│         ├─ -p/--print ──► run_print_mode()（非交互 stdout 流式）            │
│         ├─ --task-worker ─► run_task_worker()（headless，stdin 驱动）      │
│         └─ 默认 REPL ─► run_repl() ─► launch_react_tui()                  │
│              └─ tsx src/index.tsx（Node 进程，父进程）                     │
│                   └─ spawn: python -m openharness --backend-only          │
│                        （ReactBackendHost，子进程）                       │
│                   useBackendSession <──OHJSON:…\n──> ReactBackendHost     │
└───────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键结论

- `frontend/terminal` **不是 webview**，而是 React Ink（终端渲染的 React 框架）：`package.json` 依赖 `ink ^5.1.0`、`react ^18.3.1`、`ink-text-input`、`marked`（Markdown 解析）、`string-width`（CJK 列宽），用 `tsx` 直接运行 TypeScript，无需构建步骤。
- 前后端之间**没有端口、没有 socket**，只有子进程 stdin/stdout 管道 + 换行分隔的 JSON；后端→前端方向每行以 `OHJSON:` 前缀 framing。
- 同一个后端进程也可以被 `--task-worker`（后台 agent 任务）与 `-p/--print`（脚本/管道场景）复用；`--backend-only` 是公开可用的隐藏入口。

---

## 二、CLI 入口 `cli.py` 全面拆解（2551 行）

### 2.1 文件布局总览

`cli.py` 按职责可分为五个逻辑段：

| 段落 | 行号 | 内容 |
|---|---|---|
| 常量与 dry-run 辅助 | 20-742 | `_PREVIEW_STOPWORDS`（20）、`_dry_run_command_behavior`（124-201）、`_recommend_preview_candidates`（253-330）、`_evaluate_dry_run_readiness`（333-393）、`_build_dry_run_preview`（396-597）、`_format_dry_run_preview`（600-742） |
| typer 应用与子应用注册 | 751-781 | `app` 定义（751）、7 次 `add_typer`（767-781） |
| 子命令实现 | 786-2174 | mcp / plugin / cron / autopilot / setup / auth / config / provider 全部命令体 |
| 交互式引导辅助 | 1238-1730 | questionary 封装、工作流 / 模型 / 认证引导 |
| `main` 回调与会话分流 | 2181-2551 | 参数解析（2181-2367）、五条运行路径（2368-2551） |

（注：引导辅助函数在源码中的物理位置夹在子命令实现之间，上表按逻辑分组而非纯行序。）

### 2.2 ENTRYPOINT 链路

- `pyproject.toml:48-50` 声明 console scripts：`openharness` / `oh` / `openh` 均指向 `openharness.cli:app`；另有 `ohmo = ohmo.cli:app`（另一套 CLI）。
- `src/openharness/__main__.py`（全文 4 行）：
  ```python
  from openharness.cli import app
  if __name__ == "__main__":
      app()
  ```
  即 `python -m openharness` 最终调用 `cli.py` 中定义的 typer 应用对象 `app`。
- `app` 定义于 `cli.py:751`：
  ```python
  app = typer.Typer(
      name="openharness",
      help=("Oh my Harness! An AI-powered coding assistant.\n\n"
            "Starts an interactive session by default, use -p/--print for non-interactive output."),
      add_completion=False,
      rich_markup_mode="rich",
      invoke_without_command=True,
  )
  ```
  `invoke_without_command=True` 让不带子命令的调用也能触发 `main` 回调（`cli.py:2181` 的 `@app.callback`），这就是 `oh` 默认进入交互会话的机制。

### 2.3 七个 typer 子应用注册

`cli.py:767-781` 创建 7 个子 typer 并挂到 `app` 下：

| 子应用 | 变量 | 行号 | help 文本 |
|---|---|---|---|
| `mcp` | `mcp_app` | 767 | Manage MCP servers |
| `plugin` | `plugin_app` | 768 | Manage plugins |
| `auth` | `auth_app` | 769 | Manage authentication |
| `provider` | `provider_app` | 770 | Manage provider profiles |
| `config` | `config_app` | 771 | Show or update settings |
| `cron` | `cron_app` | 772 | Manage cron scheduler and jobs |
| `autopilot` | `autopilot_app` | 773 | Manage repo autopilot |

注册调用位于 `cli.py:775-781`（`app.add_typer(...)` × 7）。

### 2.4 交互式引导辅助函数（setup / auth / provider 共用）

全部优先使用 `questionary`；stdin/stdout 不是 tty 时退化到 `typer.prompt`：

- TTY 检测：`_can_use_questionary`（1269）要求 stdin/stdout 均为 tty 且未被替换。
- 输入封装：`_select_with_questionary`（1282）、`_text_prompt`（1304）、`_secret_prompt`（1316）、`_confirm_prompt`（1328）、`_select_from_menu`（1340）。
- 模型选择：`_prompt_model_for_profile`（1364）——Claude 家族用 `CLAUDE_MODEL_ALIAS_OPTIONS` 别名列表，其余为自由文本。
- 工作流选择：`_select_setup_workflow`（1404）提供带富文本 hint 的两路选择（claude-api / openai-compatible）；`_specialize_setup_target`（1549）把顶层选择展开为具体 profile（如 kimi-anthropic / openrouter）；`_configure_custom_profile_via_setup`（1478）引导自定义 profile（API 家族 → 名 → label → base_url → auth_source → model → API key）。
- 认证路径：`_login_provider`（1716）三分发——copilot 走设备码流程；`openai_codex` / `anthropic_claude` 走外部 CLI 绑定（`_bind_external_provider`，1680）；其余走 `ApiKeyFlow`。`_ensure_profile_auth`（1622）、`_maybe_update_profile_auth`（1644）供 `setup` 按需初始化 / 更新 API key。
- 中文标签映射：`_PROVIDER_LABELS`（1238）、`_AUTH_SOURCE_LABELS`（1253）。

### 2.5 全部命令 / 子命令清单

**mcp 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `mcp list` | `mcp_list` | 787 | 打印已配置 MCP 服务器（名称 + transport） |
| `mcp add <name> <config_json>` | `mcp_add` | 805 | 以 JSON 字符串加入一个 MCP 配置并 `save_settings` |
| `mcp remove <name>` | `mcp_remove` | 826 | 删除 MCP 配置，不存在则退出码 1 |

**plugin 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `plugin list` | `plugin_list` | 844 | 列出插件（enabled/disabled） |
| `plugin install <source>` | `plugin_install` | 860 | 从路径/URL 安装插件（`install_plugin_from_path`） |
| `plugin uninstall <name>` | `plugin_uninstall` | 871 | 卸载插件，非法包名抛 `typer.BadParameter` |

**cron 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `cron start` | `cron_start` | 887 | 启动调度器守护进程 |
| `cron stop` | `cron_stop` | 899 | 停止调度器 |
| `cron status` | `cron_status_cmd` | 910 | 调度器状态 + 任务摘要 |
| `cron list` | `cron_list_cmd` | 922 | 列出所有 cron job（调度、payload、notify、last/next run） |
| `cron toggle <name> <enabled>` | `cron_toggle_cmd` | 955 | 启用/禁用某个 job |
| `cron history [name] [-n]` | `cron_history_cmd` | 970 | 执行历史（可过滤 job 名、限条数） |
| `cron logs [-n]` | `cron_logs_cmd` | 993 | 尾部输出 `~/logs/cron_scheduler.log` |

**autopilot 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `autopilot status [--cwd]` | `autopilot_status_cmd` | 1012 | 队列各状态计数 + next card + store 路径 |
| `autopilot list [status] [--cwd]` | `autopilot_list_cmd` | 1046 | 列出卡片（前 20） |
| `autopilot add <source> <title> [--body]` | `autopilot_add_cmd` | 1066 | 入队一张卡片（source 归一化映射在 1075-1087） |
| `autopilot context [--cwd]` | `autopilot_context_cmd` | 1099 | 打印合成的 repo context |
| `autopilot journal [-n]` | `autopilot_journal_cmd` | 1110 | 打印最近 journal 条目 |
| `autopilot scan <target> [-n]` | `autopilot_scan_cmd` | 1127 | 扫描 issues/prs/claude-code/all |
| `autopilot run-next [--model/--max-turns/--permission-mode]` | `autopilot_run_next_cmd` | 1153 | 端到端执行最高优先级卡片（`asyncio.run(RunNext)`） |
| `autopilot tick [...]` | `autopilot_tick_cmd` | 1180 | 先扫描再运行下一张（作为定时任务入口） |
| `autopilot install-cron` | `autopilot_install_cron_cmd` | 1213 | 注册默认 cron（scan/tick） |
| `autopilot export-dashboard [--output]` | `autopilot_export_dashboard_cmd` | 1224 | 导出静态 kanban 站点 |

**setup / auth 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `setup [profile]` | `setup_cmd` | 1753 | 统一配置：选工作流 → 认证 → 选模型 → 激活 profile |
| `auth login [provider]` | `auth_login` | 1811 | 交互式登录（可选 provider 菜单） |
| `auth status` | `auth_status_cmd` | 1840 | Auth source + 各 provider profile 状态表格 |
| `auth logout [provider]` | `auth_logout` | 1869 | 清除凭据（默认清活跃 profile） |
| `auth switch <provider>` | `auth_switch` | 1886 | 切换 auth source 或激活 profile |
| `auth copilot-login` | `auth_copilot_login` | 1946 | Copilot 设备码流程（别名 `_run_copilot_login`，1906） |
| `auth codex-login` | `auth_codex_login` | 1952 | 绑定本地 Codex CLI 订阅会话 |
| `auth claude-login` | `auth_claude_login` | 1958 | 绑定本地 Claude CLI 订阅会话 |
| `auth copilot-logout` | `auth_copilot_logout` | 1964 | 清除 Copilot token |

**config 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `config show` | `config_show` | 2006 | 打印解析后的设置 JSON |
| `config set <key> <value>` | `config_set` | 2015 | 持久化一个设置（支持点号嵌套 key，如 `permission.mode`） |

**provider 组**

| 命令 | 函数 | 行号 | 说明 |
|---|---|---|---|
| `provider list` | `provider_list` | 2042 | 列出 profile（`*` 标记活跃） |
| `provider use <name>` | `provider_use` | 2056 | 激活 profile（`manager.use_profile`） |
| `provider add <name> --label ...` | `provider_add` | 2072 | 创建 profile（可带 `--api-key`/`--allowed-model`/`--context-window-tokens` 等） |
| `provider edit <name> [...]` | `provider_edit` | 2116 | 编辑 profile |
| `provider remove <name>` | `provider_remove` | 2162 | 删除 profile |

### 2.6 `main` 回调参数表（cli.py:2181-2367）

`main` 是 `@app.callback(invoke_without_command=True)`，按 `rich_help_panel` 分组。完整参数表：

| 参数 | 选项 | 面板 | 说明 |
|---|---|---|---|
| version | `--version/-v` | — | 显示版本后退出（`_version_callback`，cli.py:745） |
| continue | `--continue/-c` | Session | 恢复当前目录最近会话 |
| resume | `--resume/-r` | Session | 按 session id 恢复，空值打开列表选择器 |
| name | `--name/-n` | Session | 会话显示名 |
| model | `--model/-m` | Model & Effort | 模型别名或完整 id |
| effort | `--effort` | Model & Effort | low/medium/high/xhigh |
| verbose | `--verbose` | Model & Effort | 覆盖配置中的 verbose |
| max_turns | `--max-turns` | Model & Effort | 最大 agentic turns（print 模式强制生效，交互模式为可选上限） |
| print | `--print/-p` | Output | 非交互：打印回复后退出（值即 prompt） |
| output_format | `--output-format` | Output | text / json / stream-json |
| dry_run | `--dry-run` | Output | 只预览解析后的运行配置，不执行模型/工具 |
| permission_mode | `--permission-mode` | Permissions | default / plan / full_auto |
| dangerously_skip_permissions | `--dangerously-skip-permissions` | Permissions | 跳过所有权限检查（强制 `permission_mode="full_auto"`，见 2387） |
| allowed_tools | `--allowed-tools` | Permissions | 允许列表（逗号或空格分隔） |
| disallowed_tools | `--disallowed-tools` | Permissions | 拒绝列表 |
| system_prompt | `--system-prompt/-s` | System & Context | 覆盖默认 system prompt |
| append_system_prompt | `--append-system-prompt` | System & Context | 追加到默认 system prompt 之后 |
| settings_file | `--settings` | System & Context | JSON settings 文件或内联 JSON |
| base_url | `--base-url` | System & Context | Anthropic 兼容 API base URL |
| api_key | `--api-key/-k` | System & Context | 覆盖配置与环境变量 |
| bare | `--bare` | System & Context | 最小模式：跳过 hooks/plugins/MCP/自动发现 |
| api_format | `--api-format` | System & Context | anthropic / openai / copilot |
| theme | `--theme` | System & Context | default/dark/minimal/cyberpunk/solarized/custom |
| debug | `--debug/-d` | Advanced | DEBUG 日志（直接写入 stderr） |
| mcp_config | `--mcp-config` | Advanced | 从 JSON 文件/字符串加载 MCP 服务器 |
| cwd | `--cwd`（hidden） | — | 会话工作目录（默认 `Path.cwd()`） |
| backend_only | `--backend-only`（hidden） | — | 运行结构化后端主机（供 React TUI 使用） |
| task_worker | `--task-worker`（hidden） | — | 运行 stdin 驱动的 headless worker（供后台 agent 任务） |

`--theme` 会被立即持久化（`cli.py:2390-2395`，写入 `settings.theme`）。

### 2.7 main 的五条运行路径（cli.py:2368-2551）

`cli.py:2368` 之后是真正的分流逻辑，顺序如下：

1. **`ctx.invoked_subcommand is not None` → 直接 `return`**（2370）：调用子命令时不走会话逻辑。
2. **debug / OPENHARNESS_LOG_LEVEL 配置日志**（2375-2384）。
3. **`--dangerously-skip-permissions` → `permission_mode="full_auto"`**（2386-2387）。
4. **`--theme` 写入配置**（2389-2395）。
5. 惰性导入 `from openharness.ui.app import run_print_mode, run_repl, run_task_worker`（2397）——UI 层只在需要时才加载。
6. **dry-run 分支**（2399-2434）：不允许与 `-c/-r` 并用；`-p` 必须带值；调用 `_build_dry_run_preview` 后按 output_format 输出 text/json/stream-json。
7. **continue/resume 分支**（2437-2496）：加载会话快照（见 2.9），把 `restore_messages`、`restore_tool_metadata` 传给 `run_repl`。
8. **print 分支**（2498-2519）：`run_print_mode(prompt=..., output_format=...)`。
9. **task_worker 分支**（2521-2535）：`run_task_worker(...)`。
10. **默认 REPL 分支**（2537-2551）：`run_repl(prompt=None, cwd, model, max_turns, backend_only, ...)`。

### 2.8 dry-run 实现细节

dry-run 全部是**纯静态解析**，不加载模型、不执行工具：

- `_build_dry_run_preview`（cli.py:396-597）：惰性导入各注册表 → `settings = load_settings().merge_cli_overrides(...)` → `detect_provider` + `auth_status` → `load_plugins` → `create_default_command_registry` → `load_skill_registry` → `load_mcp_server_configs` → `create_default_tool_registry` → 逐一构造 tool schema 预览 → 用 `redirect_stderr(StringIO())` 包裹 `_resolve_api_client_from_settings` 探测客户端（460-466，不真实联网）。
- **`preview` 产物结构**（537-589）：`mode`、`cwd`、`config_path`、`prompt`、`settings`（活跃 profile/provider/api_format/model/base_url/permission_mode/max_turns/effort/passes）、`validation`（auth_status、api_client、system_prompt_chars、mcp_validation）、`entrypoint`（kind 为 `slash_command` / `unknown_slash_command` / `model_prompt` / `interactive_session`，见 501-535）、`commands`、`skills`、`tools`、`mcp_servers`、`system_prompt_preview`、`recommendations`。
- `_evaluate_dry_run_readiness`（333-393）：汇总 `level`（ready / warning / blocked）+ `reasons` + `next_actions`；MCP 数量错误会升级为 warning；auth 缺失对 `interactive_session`/`model_prompt` 给出 warning。
- `_recommend_preview_candidates`（253-330）：基于 prompt 分词（`_tokenize_preview_text`，支持 ASCII + CJK，过滤 `_PREVIEW_STOPWORDS`）对 skill/tool/command 打分排序，返回 top-5/8/5 建议。
- `_dry_run_command_behavior`（124-201）：把斜杠命令分类为 `read_only` / `stateful` / `unknown`（两个集合见 125-187）。
- `_format_dry_run_preview`（600-742）：拼成人类可读的纯文本预览（Readiness / Execution / Resolved Settings / Validation / Discovery / Configured MCP / Available Tools / …）。

### 2.9 continue / resume 会话恢复

`cli.py:2437-2496` 负责会话恢复，依赖 `openharness.services.session_storage`：

- `load_session_snapshot`（session_storage.py:123）——按 cwd 找最近会话；
- `list_session_snapshots`（session_storage.py:131）——列出可恢复会话（供 `--resume` 空值时的选择器）；
- `load_session_by_id`（session_storage.py:194）——按 session id 精确恢复。

恢复出的 `restore_messages` 与 `restore_tool_metadata` 一路传给 `run_repl`，最终进入 `build_runtime` 的恢复元数据骨架（runtime.py:379-398）。

---

## 三、OHJSON 协议（前后端 IPC）

协议定义在 `ui/protocol.py`（pydantic 模型）+ `ui/backend_host.py`（传输实现）+ `frontend/terminal/src/hooks/useBackendSession.ts`（客户端）。协议名称取 Open**H**arness **JSON** 之意。

### 3.1 传输 framing

- **前端 → 后端**：每个请求独立一行纯 JSON，无前缀。`useBackendSession.sendRequest`（useBackendSession.ts:107-113）执行：
  ```ts
  const sendRequest = useCallback((payload: Record<string, unknown>) => {
    if (child.stdin && !child.stdin.destroyed) {
      child.stdin.write(JSON.stringify(payload) + '\n');
    }
  }, [child]);
  ```
- **后端 → 前端**：每行 `OHJSON:` 前缀 + JSON + `\n`。`_PROTOCOL_PREFIX = "OHJSON:"` 定义于 backend_host.py:45；写出由 `_emit`（844-854）在 `_write_lock` 保护下完成：
  ```python
  def _emit(self, event: BackendEvent) -> None:
      async with self._write_lock:
          sys.stdout.buffer.write(
              _PROTOCOL_PREFIX.encode() + event.model_dump_json().encode() + b"\n"
          )
          sys.stdout.buffer.flush()
  ```
- 前端解析：`child.stdout` 挂 `readline.createInterface`，`reader.on('line')`（129-136）先 `line.startsWith(PROTOCOL_PREFIX)`，命中则 `JSON.parse(line.slice(PROTOCOL_PREFIX.length))`，否则按后端普通日志塞进 transcript。
- **注意**：代码中并不存在字面量命名的 `write_json_frame` / `read_json_frame` 之类的函数；「帧」的写入方是 `backend_host._emit`，读取方是 `backend_host._read_requests`（Python 侧）与 `useBackendSession` 的 readline 回调（TS 侧）。文档 / 文章里若见到这两个名字，均指这两处实现。
- 后端 stdout 上除此协议行外不应输出其他内容（调试日志走 stderr），否则前端会把它渲染成一条 `log` transcript。

### 3.2 `FrontendRequest` —— 前端→后端请求（8 种）

`protocol.py:37-57`，`type` 为 `Literal` 八值；其余字段 `Optional`：

```python
type: Literal["submit_line","permission_response","question_response",
              "list_sessions","select_command","apply_select_command",
              "interrupt","shutdown"]
line/command/value/request_id/allowed/permission_reply/answer: 可选
images: list[FrontendImageAttachment]
```

| type | 携带字段 | 发起场景 |
|---|---|---|
| `submit_line` | `line`（文本）、`images`（base64 图） | 普通输入提交（`App.onSubmit`） |
| `permission_response` | `request_id`、`allowed`（bool）、`permission_reply` | 权限 / 编辑 diff 弹窗应答（y / n / a） |
| `question_response` | `request_id`、`answer` | 提问弹窗应答 |
| `list_sessions` | — | 请求可恢复会话列表 |
| `select_command` | `command` | 前端拦下的可选项斜杠命令（/provider 等） |
| `apply_select_command` | `command`、`value` | 选择框确认某一项 |
| `interrupt` | — | ctrl+c / esc（busy 时打断当前轮） |
| `shutdown` | — | 正常退出 |

`FrontendImageAttachment`（protocol.py:15-34）：`media_type`（必须 `image/` 前缀）、`data`（base64、非空）、`source_path` 可选，带 `field_validator` 校验。

### 3.3 `BackendEvent` —— 后端→前端事件（18 种）

`protocol.py:90-199`。`type` 的 **18** 个取值（旧版文档误记为 19 种）：

```
ready / state_snapshot / tasks_snapshot / transcript_item / compact_progress /
assistant_delta / assistant_complete / line_complete / tool_started /
tool_completed / clear_transcript / modal_request / select_request /
todo_update / plan_mode_change / swarm_status / error / shutdown
```

各事件用途：

| type | 载荷要点 | 触发点 |
|---|---|---|
| `ready` | `state`、`tasks`、`commands`（斜杠命令列表） | `ReactBackendHost.run` 启动后（118-124） |
| `state_snapshot` | `state`（`_state_payload`，202-223，约 20 键） | 状态变化后 |
| `tasks_snapshot` | `tasks`（TaskSnapshot 列表） | 任务列表变化 |
| `transcript_item` | `item`（TranscriptItem） | user/assistant/tool/system 各行 |
| `compact_progress` | `phase/trigger/attempt/checkpoint/metadata` | 上下文压缩进度 |
| `assistant_delta` | `message` | 流式文本增量 |
| `assistant_complete` | —（+transcript_item +tasks_snapshot） | 一轮回复结束 |
| `line_complete` | — | 整行处理完成（前端据此复位 busy） |
| `tool_started` | `tool_name`、`tool_input` | 工具开始 |
| `tool_completed` | `tool_name`、`output`、`is_error` | 工具结束 |
| `clear_transcript` | — | 清空转录 |
| `modal_request` | `modal`（permission/edit_diff/question/mcp_auth/null） | 模态请求 / 关闭 |
| `select_request` | `command`、`options`、`current` | 前端选择框 |
| `todo_update` | `todo_markdown` | TodoWrite 工具后 |
| `plan_mode_change` | `plan_mode` | 计划模式切换 |
| `swarm_status` | `swarm_teammates`、`swarm_notifications` | 协调者子代理状态 |
| `error` | `message`、`is_error` | 错误 |
| `shutdown` | — | 后端退出前 |

工厂方法（137-199）：`ready(state, tasks, commands)`（137-151）、`state_snapshot(state)`（153-155）、`tasks_snapshot(tasks)`（157-162）、`status_snapshot(state, mcp_servers, bridge_sessions)`（164-199，把 `McpConnectionStatus` / `BridgeSessionRecord` 压成扁平 dict）。辅助：`_state_payload`（202-223）、`_MODE_LABELS`（226-233）、`_format_permission_mode`（236-238，权限枚举 → 展示名）。

### 3.4 其它模型

- `TranscriptItem`（protocol.py:60-67）：`role`：`Literal["system","user","assistant","tool","tool_result","log"]`，`text`、`tool_name`、`tool_input`、`is_error`。
- `TaskSnapshot`（protocol.py:70-87）：UI 安全的任务表示 + `from_record(record: TaskRecord)` 工厂，避免把内部 `TaskRecord` 直接序列化给前端。

### 3.5 请求 → Future 配对机制

`ReactBackendHost` 构造时（backend_host.py:76-90）初始化 `_request_queue: asyncio.Queue[FrontendRequest]` 与三张 `request_id → asyncio.Future` 表：`_permission_requests` / `_edit_approval_requests` / `_question_requests`，外加 `_permission_lock`、`_busy`、`_running`、`_active_request_task`、`_last_tool_inputs`、`_edit_always_approved`。

`_read_requests`（190-222）在独立线程阻塞读 `sys.stdin.buffer.readline`，`FrontendRequest.model_validate_json` 校验后分派：

- **三种响应型请求**（`permission_response` / `question_response` / `edit_diff` 的应答）在 reader 线程内直接完成 future 投递（204-218），不进队列；
- `interrupt` 就地处理；
- 其余才 `_request_queue.put(...)` 交给主循环。

三个等待点（协程挂起等前端应答）：

- `_ask_permission`（762-784）：生成 `uuid4().hex` request_id + future → emit `modal_request{kind:"permission", tool_name, reason}` → `asyncio.wait_for(future, timeout=300)`；**300 秒超时**后视为拒绝并警告。
- `_ask_edit_approval`（791-823）：`full_auto` 或已 `always` → 直接返回 `"always"`；否则发 `modal_request{kind:"edit_diff", path, diff, added, removed}`；收到 `always` 时置 `_edit_always_approved=True`，模态结束后 emit `modal_request{modal:null}` 让前端关闭弹窗。
- `_ask_question`（825-842）：`modal_request{kind:"question"}`，**无超时**等待 future（用户可慢慢思考）。

主循环 `run`（92-188）按请求类型分发：`submit_line`（校验 `_busy` 后 `_run_active_request(_process_line(...))`）、`select_command`（`_handle_select_command` 发 `select_request`）、`apply_select_command`（`_run_active_request(_apply_select_command(...))`）、`list_sessions`（`_handle_list_sessions`）、`interrupt`（`_interrupt_active_request`，244，cancel 当前 task）、`shutdown`（emit shutdown 后 break）。

### 3.6 模态交互时序（permission 示例）

```
引擎要执行工具 → RuntimeBundle 注入的 _ask_permission(tool_name, reason)
  → emit modal_request{kind:"permission", request_id}
  → 前端 ModalHost 弹出 [y] Allow / [n] Deny
  → 用户按键 → sendRequest{permission_response, request_id, allowed}
  → backend_host._read_requests 发现 request_id 在 _permission_requests
        → future.set_result(bool)
  → _ask_permission 返回 → 引擎继续执行该工具
```

编辑类工具（Edit/FileEdit）走 `edit_diff` 弹窗（y=once / a=always / n=reject）；`always` 会在后端记录 `_edit_always_approved`（backend_host.py:821-822），本会话内不再询问。三种模态共用「request_id → Future」映射，天然支持并发模态（编辑 diff 与问题弹窗可叠加出现）。

---

## 四、runtime.py 运行时状态机（799 行）

`runtime.py` 是 headless / Textual / React 后端**共享**的「状态机 + 装配器」，把配置解析成可执行的一个 `RuntimeBundle`。

### 4.1 类型别名（runtime.py:46-117）

- 协程回调类型（46-51）：`PermissionPrompt`、`AskUserPrompt`、`EditApprovalPrompt`、`SystemPrinter`、`StreamRenderer`、`ClearHandler`。
- 配置解析辅助：`_resolve_vision_config`（93，视觉模型配置）、`_resolve_image_generation_config`（54，图像生成配置）分别把 AppState / settings 解析成引擎需要的配置对象。

### 4.2 `RuntimeBundle`（120-190，dataclass）

一次会话所有共享对象的容器：

| 字段 | 类型 | 说明 |
|---|---|---|
| `api_client` | `SupportsStreamingMessages` | 已解析的 API 客户端 |
| `cwd` | `str` | 工作目录 |
| `mcp_manager` | `McpClientManager` | 已连接的 MCP 管理器 |
| `tool_registry` | `ToolRegistry` | 内置 + 插件工具注册表 |
| `app_state` | `AppStateStore` | 可变的 UI 状态存储 |
| `hook_executor` | `HookExecutor` | 会话级 hook 执行器 |
| `engine` | `QueryEngine` | agent 循环引擎 |
| `commands` | `object`（命令注册表） | 斜杠命令注册表 |
| `external_api_client` | `bool` | 是否外部注入 client |
| `session_id` | `str` | `uuid4().hex[:12]` |
| `settings_overrides` | `dict` | CLI 覆盖项，防止 `/fast` 等命令刷新时将模型回退（见 `current_settings` 注释，143-151） |
| `session_backend` | `SessionBackend` | 快照持久化 |
| `extra_skill_dirs` / `extra_plugin_roots` | `tuple[str,...]` | 额外目录 |
| `memory_backend` / `include_project_memory` | — | 长期记忆 |
| `autodream_context` | `dict \| None` | autodream 上下文 |

实用方法：`current_settings`（143，磁盘设置 + CLI overrides 合并）、`current_plugins`（155）、`hook_summary`（163）、`plugin_summary`（167，多行文本）、`mcp_summary`（178，含 tools/resources 列表，供 `/mcp` 命令复用）。

### 4.3 `build_runtime` —— 15 步依赖装配（274-471）

1. `settings_overrides = {model, max_turns, effort, base_url, system_prompt, api_key, api_format, active_profile, permission_mode}`（302-312）→ `load_settings().merge_cli_overrides(...)`。
2. `load_plugins(settings, cwd, ...)`（317）。
3. 解析 API client：外部注入优先，否则 `_resolve_api_client_from_settings`（318-321；该函数见 193-238，按 `api_format=="copilot"` / `provider=="openai_codex"` / `provider=="anthropic_claude"` / `openai` / 默认 Anthropic 分流）。
4. `McpClientManager(...).connect_all()`（322-323）。
5. `tool_registry = create_default_tool_registry(mcp_manager)`，再注册插件工具（324-329）。
6. `AppStateStore(AppState(...))`（332-356）——materialize 后的 `permission_mode` / `theme` / `cwd` / `provider` / `auth_status` / `base_url` / `vim_enabled` / `voice_enabled` / `voice_available` / `voice_reason` / `fast_mode` / `effort` / `passes` / `mcp_connected` / `mcp_failed` / `bridge_sessions` / `output_style` / `keybindings`。
7. `HookReloader(get_config_file_path())` + `HookExecutor`（357-365；外部 client 时改用配置热加载注册表）。
8. `engine_max_turns = settings.max_turns if (enforce_max_turns or max_turns is not None) else None`（366）。
9. `build_runtime_system_prompt(...)`（367-374）。
10. 构造恢复元数据骨架（379-398）：`permission_mode` / `read_file_state` / `invoked_skills` / `async_agent_state` / `async_agent_tasks` / `recent_work_log` / `task_focus_state` / `compact_checkpoints`，并合并 `restore_tool_metadata`。
11. `QueryEngine(...)`（400-429）：传入 permission_checker、tool_metadata（含 `mcp_manager`、`bridge_manager`、`extra_skill_dirs`、`session_id`、`edit_approval_prompt`、`vision_model_config`、`image_generation_config` 与恢复元数据）。
12. `restore_messages` → `sanitize_conversation_messages` + `engine.load_messages`（433-437）。
13. docker sandbox 启动（440-443）。
14. 返回 `RuntimeBundle`。

### 4.4 `start_runtime` / `close_runtime` 生命周期（474-501）

- `start_runtime`：执行 `SESSION_START` 钩子。
- `close_runtime`：先停 docker sandbox → 抽离本地规则（`update_rules_from_session` 最佳努力）→ 关 MCP → 执行 `SESSION_END` 钩子 → 关 API client。

### 4.5 `handle_line` —— 一行输入的统一处理（621-774）

headless / Textual / React 三条路径共用的「调度中枢」：

1. 非外部 client 时热更新 hook 注册表（631-634）。
2. 构造 `CommandContext`（636-650，携带 engine / 各 summary / cwd / tool_registry / app_state / session_backend / session_id / extra 目录 / memory_backend）。
3. **斜杠命令匹配**：`bundle.commands.lookup(line) or lookup_skill_slash_command(line, command_context)`（651-653）。
4. 命令分支（654-730）：执行 `command.handler(args, context)`；若 `result.refresh_runtime` 则 `refresh_runtime_client`（596-618，重新解析 client / 模型 / 权限校验器 / system prompt）；`_render_command_result`（777-799，处理 `clear_screen`、`replay_messages` 回放为 `AssistantTextDelta` + `AssistantTurnComplete`、以及结果文本打印）；若 `result.submit_prompt` 则临时切换模型后 `engine.submit_message`；若 `result.continue_pending` 则 `engine.continue_pending(max_turns=...)`。`MaxTurnsExceeded` 会被捕获并打印 `_format_pending_tool_results`（517-564，「还有工具结果待续」摘要 + `/continue [COUNT]` 提示）。最后 `save_snapshot` + `sync_app_state`。
5. 普通 prompt 分支（732-774）：重建 system prompt → `engine.submit_message` 流式输出 → `save_snapshot` → `sync_app_state` → 返回 True。

### 4.6 其它状态刷新

- `sync_app_state`（567-593）：用当前 settings 刷新 `AppStateStore`。
- `refresh_runtime_client`（596-618）：provider/auth 变更后重建 API client、权限校验器与 system prompt。

---

## 五、app.py 会话入口与 coordinator_drain

### 5.1 `run_repl`（app.py:40-89，默认交互会话）

- 若 `backend_only=True` → 直接 `await run_backend_host(...)`（58-74），本进程即协议后端。
- 否则 `await launch_react_tui(...)`（76-87）：拉起 React Ink 前端进程并等待其退出；非零退出码 `raise SystemExit(exit_code)`。

### 5.2 `run_task_worker`（app.py:92-174，headless worker）

- 为 task-manager 的子代理进程（无 TTY）而设，刻意避开 Ink 渲染路径（docstring 105-110）。
- 内置 `_noop_permission` / `_noop_ask` / `_print_system` / `_render_event`（112-133），后者把 `AssistantTextDelta` 直接写 stdout、`ErrorEvent` / `StatusEvent` 打到 stdout。
- 主循环（154-172）：`asyncio.to_thread(sys.stdin.readline)` 读一行 → `_decode_task_worker_line`（19-37，兼容纯文本或 `{"text": ...}` JSON）→ `handle_line(...)` 执行一轮，然后 break（一次性 worker）。
- `_decode_task_worker_line` 支持 `send_message` / teammate 后端下发的 JSON 载荷。

### 5.3 `run_print_mode`（app.py:177-320，非交互一次性）

- `output_format` 三态：`text`（增量写 stdout，system 消息写 stderr）、`stream-json`（每事件一行 JSON：`system`/`assistant_delta`/`assistant_complete`/`tool_started`/`tool_completed`/`error`/`compact_progress`/`status`，见 236-295）、`json`（最后输出 `{"type":"result","text":...}`，见 316-318）。
- 单轮后若处于 coordinator 模式则调用 `drain_coordinator_async_agents`（307-314）等到后台 agent 完成。

### 5.4 `coordinator_drain.py` —— 协调者模式的后台 agent 排空（198 行）

当协调者用 `agent` 工具派发子代理时，子代理结果以 `user` 角色的 `<task-notification>` 消息在轮间到达。该模块实现轮询 + 格式化 + 提交回协调者：

- `_TERMINAL_TASK_STATUSES`（25）：`completed/failed/killed`。
- `pending_async_agent_entries`（37-46）：从 `tool_metadata["async_agent_tasks"]` 挑出未发送通知的条目。
- `wait_for_completed_async_agent_entries`（62-86）：每 0.1s 轮询 `get_task_manager().get_task` 直到出现终态。
- `format_completed_task_notifications`（89-115）：读取任务输出（`read_task_output(max_bytes=8000)`）并组装 `<task-notification>` 信封。
- `submit_follow_up`（118-156）：重建 system prompt 后 `engine.submit_message(notification_payload)` 再存快照。
- `drain_coordinator_async_agents`（159-198）：循环排空直到没有挂起的后台条目。

### 5.5 其它 UI 适配文件（保留的旧路径）

- `output.py`（265 行）：`OutputRenderer`（20-217）基于 rich，用于无 React 的纯终端渲染路径——`show_thinking`（34）spinner、`start_assistant_turn`（45）`⏺` 标记、`render_event`（56-139）按工具类型分派（Bash → Panel、read → `Syntax(...)`、edit → 绿色 Panel、grep → cyan Panel）；`print_status_line`（151）输出 `model/tokens↓↑/mode` 一行。
- `input.py:8-32`：`InputSession`（prompt_toolkit `PromptSession`），`set_modes(vim/voice)` 改提示前缀。
- `permission_dialog.py:8-14`：`ask_permission(tool_name, reason)` 打印 `Allow tool '...'? [reason] [y/N]`。
- `__init__.py`：只导出 `run_repl, run_print_mode`。

---

## 六、textual_app.py 备选 TUI（495 行）

`OpenHarnessTerminalApp(App[None])`（138）是 Textual 终端应用，当前默认 UI 已被 React Ink 取代但仍保留，作为无 Node 环境的退化路径：

- `PermissionScreen`（47-85）/ `QuestionScreen`（88-135）：基于 `ModalScreen` 的权限/提问弹窗，绑定 `action_allow/deny/submit/cancel`。
- CSS 在 141-197；`BINDINGS`（199-205）：`ctrl+l` 清屏、`ctrl+r` 刷新侧栏、`ctrl+k` 切 vim、`ctrl+v` 切 voice、`ctrl+d` 退出。
- `compose`（235-246）：左列 `RichLog`(#transcript) + `Static`(#current-response) + `Input`(#composer)，右侧 `status-bar` / `tasks-panel` / `mcp-panel` 三面板。
- `on_mount`（248-264）：`build_runtime`（注入 `_ask_permission` / `_ask_question`）；`_process_line`（292-321）调 `handle_line` + `drain_coordinator_async_agents`。
- `_render_event`（327-386）与 React 后端 `_render_event` 逻辑同构：delta 缓冲、compact 文案（334-359 的 phase 文案映射）、tool 行、error/status 行。
- 侧栏刷新 `_refresh_sidebars`（436-495）：基于快照去重（`_last_status_snapshot` / `_last_tasks_snapshot` / `_last_mcp_summary`）。

---

## 七、react_launcher.py 前端进程拉起（179 行）

- `_resolve_theme`（13-19）：默认主题解析。
- `_resolve_npm`（22-24）：定位 npm。
- `_resolve_tsx`（27-56）：尽量直接调 `node_modules/.bin/tsx`（避免 Windows/WSL 下 `npm exec` 产生中间 shell 破坏 TTY raw 模式），否则全局 tsx，最后 `npm exec -- tsx` 兜底。
- `get_frontend_dir`（59-78）：优先安装在包内 `openharness/_frontend`（pip install 后由 pyproject.toml 的 force-include 打包 `frontend/terminal/{package.json,tsconfig.json,src}`），其次按仓库布局 `parents[3]` 找源码 `frontend/terminal`，最后兜底路径。
- `build_backend_command`（81-113）：拼出 `[sys.executable, "-m", "openharness", "--backend-only", ...]`，随配置追加 `--cwd/--model/--max-turns/--effort/--base-url/--system-prompt/--api-key/--api-format/--permission-mode`。
- `launch_react_tui`（116-176）：若 `node_modules` 不存在先 `npm install --no-fund --no-audit`；构造 `OPENHARNESS_FRONTEND_CONFIG` 环境变量（JSON：`backend_command` + `initial_prompt` + `theme`，149-165）；`create_subprocess_exec(tsx_cmd, "src/index.tsx", stdin=None, stdout=None, stderr=None)`（继承终端）；`await process.wait()` 返回退出码。

---

## 八、React TUI 前端（frontend/terminal/）

### 8.1 技术栈（package.json）

```json
{
  "name": "@openharness/terminal",
  "type": "module",
  "scripts": { "start": "tsx src/index.tsx" },
  "dependencies": {
    "ink": "^5.1.0",
    "ink-text-input": "^6.0.0",
    "marked": "^18.0.0",
    "react": "^18.3.1",
    "string-width": "^7.2.0"
  },
  "devDependencies": {
    "@types/node": "^22.13.10", "@types/react": "^18.3.12",
    "tsx": "^4.19.2", "typescript": "^5.7.3"
  }
}
```

Markdown 渲染用 `marked` 词法分析器 + 自研 token→Ink 组件；CJK 列宽用 `string-width`；`tsx` 直接执行 TSX，无需先编译。

### 8.2 `index.tsx` —— 前端进程入口（81 行）

- EIO/EAGAIN 防御（9-38）：`setRawMode`、stdin 错误、uncaughtException 全部在 EIO/EAGAIN 时 `process.exit(1)`（应对 SSH/tmux/Docker 终端丢 TTY）。
- 读取 `OPENHARNESS_FRONTEND_CONFIG` JSON → `FrontendConfig`（40）。
- 退出时恢复光标并补换行（45-56，`\x1B[?25h\n`，并处理 SIGINT→130 / SIGTERM→143）。
- WSL/Windows TTY 兜底（61-79）：stdin 不是 TTY 时打开 `/dev/tty`，构造 `tty.ReadStream` 传给 Ink。
- `render(<App config={config} />, {stdin: stdinStream})`（81）。

### 8.3 `types.ts` —— 协议类型镜像（98 行）

与后端 `protocol.py` 一一对应：`FrontendConfig`（backend_command、initial_prompt）、`TranscriptItem`（前端版 role 多一个 `'status'`）、`ImageAttachmentPayload`、`TaskSnapshot`、`McpServerSnapshot`、`BridgeSessionSnapshot`、`SelectOptionPayload`、`TodoItemSnapshot`、`SwarmTeammateSnapshot`、`SwarmNotificationSnapshot`、`BackendEvent`（字段镜像，含扩展的 todo/plan/swarm 载荷）。

### 8.4 `hooks/useBackendSession.ts` —— 连接后端的唯一 Hook（462 行）

- 常量：`PROTOCOL_PREFIX = 'OHJSON:'`（17）、`ASSISTANT_DELTA_FLUSH_MS = 50`、`ASSISTANT_DELTA_FLUSH_CHARS = 384`、`TRANSCRIPT_EVENT_FLUSH_MS = 50`（18-20）。
- **进程 spawn**（115-177）：`spawn(backend_command[0], args, {stdio: ['pipe','pipe','inherit'], detached: POSIX, env: process.env})`；`child.stdout` 挂 `readline.createInterface` 按行读；命中 `OHJSON:` 前缀解析事件，否则作为 `log` transcript（131）。
- **发送**：`sendRequest(payload)` → `child.stdin.write(JSON.stringify(payload) + "\n")`（107-113）。
- **退出管理**：子进程 `exit` 事件 → 提示 "backend exited with code …" 并 `onExit(code)`（138-143）；父进程 exit/SIGINT/SIGTERM 时杀掉整个**进程组**（`process.kill(-child.pid, 'SIGTERM')`，150-168），避免僵尸进程。
- **降噪声显示**：`assistant_delta` 累积到 `pendingAssistantDeltaRef`，每 50ms 或 384 字符 flush 一次（56-97）；transcript item 同理按 50ms 批量入 state；`startTransition` 降低渲染优先级。
- **`handleEvent` 分发**（179-435）——前端状态机的主要驱动：

| 事件 | 处理 |
|---|---|
| `ready` | 置 `ready=true`，写 status/tasks/commands/mcpServers/bridgeSessions；配置了 `initial_prompt` 立即发 `submit_line`（205-209） |
| `state_snapshot` / `tasks_snapshot` | `stableStringify` 去重后才 setState |
| `transcript_item` | 追加 transcript |
| `status` | 更新 status |
| `compact_progress` | 映射 busyLabel 文案 |
| `assistant_delta` | 缓冲 flush |
| `assistant_complete` | 追加 assistant 块；**busy 不在此复位**（注释 342-343，等 `line_complete`） |
| `line_complete` | 复位 busy（342-343 注释之后的真正复位点） |
| `tool_started` / `tool_completed` | 补充 tool_name/input/is_error 入 transcript |
| `clear_transcript` | 清空转录 |
| `select_request` | `setSelectRequest` |
| `modal_request` | `setModal` |
| `error` | 错误展示 |
| `todo_update` | 更新 todoMarkdown |
| `swarm_status` | teammates + notifications（notifications 只留最近 20 条，415） |
| `plan_mode_change` | 就地改 status.permission_mode |
| `shutdown` | `onExit(0)` |

- 返回 memo 化的 session 对象（437-461），含全部状态 + `setModal/setSelectRequest/setBusy/setBusyLabel/sendRequest`。

### 8.5 `App.tsx` —— 前端主状态机（589 行）

结构：`App`（52-59，包一层 `ThemeProvider`）→ `AppInner`（61-588）。

顶层 state（64-83）：`input`、`modalInput`、`history`/`historyIndex`（↑↓ 历史）、`imageAttachments`、`clipboardStatus`、`lastEscapeAt`（esc 双击清空）、`scriptIndex`（自动化剧本）、`pickerIndex`、`selectModal`/`selectIndex`。`session = useBackendSession(config, () => exit())`。deferred 系列（76-82）：transcript / assistantBuffer / status / tasks / todoMarkdown / swarmTeammates / swarmNotifications 全部经 `useDeferredValue`，缓解高频更新。

**前端拦截命令 `handleCommand`（188-221）：**

- `SELECTABLE_COMMANDS`（31-44）= /provider /model /theme /output-style /permissions /resume /effort /passes /turns /fast /vim /voice —— 直接发 `select_command`，由后端返回选项列表。
- `/plan`：按当前模式发 `/plan on|off`。
- `/resume`：发 `select_command{command:"resume"}`。

**全局按键 `useInput`（223-449）分支顺序：**

- `ctrl+c`：busy 时发 `interrupt`，空闲时发 `shutdown` + exit（228-237）。
- `ctrl+v`：贴剪贴板图片（239-242，`attachClipboardImage` 111-124）。
- selectModal 导航（250-280）。
- `rawReturnSubmit` 脚本回车（282-298）。
- permission 弹窗 `y/n/esc`（301-321）。
- edit_diff 弹窗 `y/a/n`（324-356）。
- question 弹窗放行给输入框（359-361）。
- busy 时 `esc` → `interrupt`（363-367）。
- `tab` 空输入快速打开权限选择器（376-379）。
- 命令补全 picker 导航（382-416）。
- 双击 esc 清空输入与附件（418-429）。
- ↑↓ 历史（431-445）。
- 普通 Enter 提交留给 `PromptInput` 处理（注释 447-449）。

**提交路径 `onSubmit`（451-485）：** question 弹窗应答 → `/stop` 打断 → `handleCommand` 拦截 → 否则 `sendRequest({type:'submit_line', line, images})` 并 `setBusy(true)`。

**scripted 自动化（488-501）：** `OPENHARNESS_FRONTEND_SCRIPT` 环境变量读取步骤数组，每步 200ms 延迟、busy/modal 时等待。

**布局渲染（503-588）：** `ConversationView`（对话区）→ `ModalHost`（后端模态）→ `SelectModal`（前端选择框）→ `CommandPicker` → `TodoPanel` → `SwarmPanel` → `StatusBar` → `PromptInput`（或 "Connecting to backend..." 占位）→ 底部按键提示。

### 8.6 组件清单

| 组件 | 文件 | 职责 |
|---|---|---|
| `ConversationView` | components/ConversationView.tsx:30 | 把 transcript 渲染成消息流；`groupToolPairs`（13-28）把 `tool+tool_result` 合并；`MessageRow` 按 role+codex 风格渲染；最多显示最近 40 条（`items.slice(-40)`） |
| `MarkdownText` | components/MarkdownText.tsx:307 | `marked` lexer 解析（lexer 在 309），`renderInline`（58-127），`MarkdownBlock`（139-305）绘制 heading/list/table/blockquote/code；表格用 `string-width` 对齐 CJK（249-296）；`React.memo`（307） |
| `ToolCallDisplay` | components/ToolCallDisplay.tsx:7 | 工具调用行 + 结果状态行（`→ 12L` / error 行数截断，error 行数上限 codex 8 / 默认 5）；`summarizeInput`（116-148）对 bash/read/write/edit/grep/glob/agent 提取单行摘要 |
| `ModalHost` | components/ModalHost.tsx:187 | 四类模态：`QuestionModal`（26-87，shift+enter 多行）、`EditDiffModal`（116-185，自带 diff 渲染器 `parseDiffLines` 96-114，`MAX_DIFF_LINES` 40）、permission（198-221）、mcp_auth（235-249） |
| `CommandPicker` | components/CommandPicker.tsx:4 | `/` 开头命令补全浮层 |
| `SelectModal` | components/SelectModal.tsx:11 | 后端 `select_request` 触发的单选列表（显示 `(current)`） |
| `StatusBar` | components/StatusBar.tsx:57 | 单行信息条（model/tokens/mode/tasks/mcp）+ `PlanModeIndicator`（14-55，离开 plan 模式闪烁 **PLAN MODE OFF**，`WRITE_TOOLS`（9-12）被阻时标红）；`formatNum`（114-118） |
| `PromptInput` | components/PromptInput.tsx:199 | 输入区：`MultilineTextInput`（19-197）自绘光标（chalk.inverse 高亮当前字符 + 尾部空格）、shift+enter 换行、backspace/delete 区分（`getBackspaceDeleteCount` 11-17 处理 DEL 序列） |
| `TodoPanel` | components/TodoPanel.tsx:21 | 解析 `- [x]` checklist（`parseTodoItems` 9-19），`ctrl+t` 折叠（31-35） |
| `SwarmPanel` | components/SwarmPanel.tsx:39 | 协调者模式下子代理状态 + 最近通知（`ctrl+w` 折叠 50-54；notifications 只留最近 3 条，115-121） |
| `Spinner` | components/Spinner.tsx:19 | 100ms 转圈 + 轮换动词；Windows 用 `-\\|/`（`WINDOWS_SAFE_FRAMES`） |
| `WelcomeBanner` | components/WelcomeBanner.tsx:18 | ASCII LOGO（版本号 0.1.0）+ 快捷键提示 |
| `Footer` / `SidePanel` / `TranscriptPane` / `Composer` | 四个文件 | **遗留组件**：`App.tsx` 及 `index.tsx` 均未引用，分别为旧布局的底部状态条、右侧信息面板、纯文本转录窗与输入框 |

### 8.7 主题（theme/）

- `ThemeContext.tsx:17-36`：`ThemeProvider` 缓存 `ThemeConfig`，`setThemeName` 按名取内置主题否则退回 default。
- `builtinThemes.ts:26-157`：五种内置主题 `default` / `dark`（Tokyo Night 配色）/ `minimal` / `cyberpunk` / `solarized`，每主题含 11 色 + icons（spinner 帧、tool/assistant/user/system/success/error 图标）。
- 主题名来源：`FrontendConfig.theme`（启动时）与后端 `state_snapshot.theme`（App.tsx:85-90 同步）。

### 8.8 `clipboardImage.ts` —— 剪贴板截图（173 行）

`readClipboardImage`（24-37）→ 平台分发（39-47）：

- macOS：依次尝试 `pngpaste`、`osascript`（`<<class PNGf>>` / TIFF）+ `sips` 转 PNG（49-88）。
- Windows：PowerShell `System.Windows.Forms.Clipboard`（90-116）。
- Linux：尝试 `wl-paste`（png/jpeg）→ `xclip` → `xsel`（118-134）。
- 统一限制 15MB（`MAX_CLIPBOARD_IMAGE_BYTES`，15）、超时 2.5s；产物是 base64 + media_type。

---

## 九、端到端交互时序（一次完整的「帮我修 bug」）

场景：用户在 `/Users/me/proj` 下执行 `oh`，输入 `帮我修 bug`，随后模型读文件、跑命令、编辑文件、完成回答。

**启动阶段（1-6）**

1. `oh` → `cli.app` → `main` 回调（cli.py:2181），未带子命令与 `-p/-r/-c`，走默认 REPL 分支（2537-2551）→ `run_repl(...)`（app.py:40）→ `launch_react_tui`（app.py:76）。
2. `react_launcher` 解析 tsx（`_resolve_tsx`）、前端目录（`get_frontend_dir`）、拼后端命令（`build_backend_command`），`npm install`（如缺依赖）后 `create_subprocess_exec(tsx, "src/index.tsx")`（116-176）。
3. `index.tsx` 读取 `OPENHARNESS_FRONTEND_CONFIG`（含 `backend_command`），设置 raw 模式，`render(<App/>)`（81）。
4. `App` 挂载 → `useBackendSession` 用 `backend_command` spawn **后端子进程** `python -m openharness --backend-only`（useBackendSession.ts:115-177，detached 进程组）。
5. 后端 `run_backend_host`（backend_host.py:883）`os.chdir(cwd)` → `ReactBackendHost.run`（92）→ `build_runtime`（runtime.py:274）→ `start_runtime`（474）执行 `SESSION_START` → emit `ready`（含 app_state / tasks / 斜杠命令列表）。
6. 前端 `ready` handler 置状态、渲染 `WelcomeBanner` + `StatusBar` + `PromptInput`；若配置了 `initial_prompt` 此刻已自动发 `submit_line`。

**提问阶段（7-11）**

7. 用户输入 `帮我修 bug` + Enter → `PromptInput` 提交 → `App.onSubmit`（451-485）→ `handleCommand` 无拦截 → `sendRequest({type:'submit_line', line:'帮我修 bug'})` + `setBusy(true)`。
8. 后端 `_read_requests`（190）读到该行，`model_validate_json` 校验后入 `_request_queue`。
9. 主循环取出 `submit_line`，`_busy` 校验通过 → `_run_active_request(_process_line(...))`（250）。
10. `_process_line` 组装 user message（`_build_user_message_with_images`）并 emit `transcript_item`（role=user）→ 前端对话区出现 `> 帮我修 bug`。
11. `_process_line` 调 `handle_line`（runtime.py:621）→ 命令匹配失败 → 普通 prompt 分支（732-774）→ `engine.submit_message` 启动一轮 agentic 循环。

**流式与工具阶段（12-21）**

12. 引擎流式返回 `AssistantTextDelta` → 后端 `_render_event` 映射为 `assistant_delta` 事件（274-366）→ emit。
13. 前端 `assistant_delta` 累积到 ref，每 50ms 或 384 字符 flush 一次（56-97），`useDeferredValue` + `startTransition` 渲染 → 用户看到逐字输出。
14. 引擎决定先读代码，`ToolExecutionStarted` → emit `tool_started` + `transcript_item`(tool)（记录 `_last_tool_inputs`）→ 前端 `ToolCallDisplay` 渲染工具行。
15. 工具需要批准（default 权限模式）→ `_ask_permission`（762-784）emit `modal_request{kind:"permission", request_id, tool_name, reason}`。
16. 前端 `ModalHost` 弹出 `[y] Allow / [n] Deny`；用户按 `y` → `sendRequest({type:'permission_response', request_id, allowed:true})`。
17. reader 线程发现 request_id 在 `_permission_requests` → `future.set_result(True)`（204-218）→ 引擎继续。
18. `ToolExecutionCompleted` → emit `tool_completed` + `transcript_item`(tool_result) + `tasks_snapshot`/`status_snapshot`；若工具是 `TodoWrite` 则额外 emit `todo_update`（334-349），`set_permission_mode`/`plan_mode` 则 emit `plan_mode_change`（351-354）。
19. 模型多次「读文件 → 跑测试 → 看报错」循环（重复 14-18）。
20. 模型要改文件（Edit）→ `_ask_edit_approval`（791-823）emit `modal_request{kind:"edit_diff", path, diff, added, removed}`。
21. 用户按 `a`（always）→ `permission_response` → `_edit_always_approved=True`；此后本会话编辑不再询问；模态结束 emit `modal_request{modal:null}`。

**收尾阶段（22-27）**

22. 引擎 `AssistantTurnComplete` → emit `assistant_complete` + `transcript_item`(assistant) + `tasks_snapshot`。
23. coordinator 模式则 `drain_coordinator_async_agents`（379-385）等待后台 agent 的 `<task-notification>`。
24. `handle_line` 完成 → `save_snapshot` + `sync_app_state`。
25. `_process_line` 末尾连发 `status_snapshot`、`tasks_snapshot`、`line_complete`（386-388）。
26. 前端 `line_complete` 复位 busy（342-343 注释说明 `assistant_complete` 不复位 busy）→ 输入框恢复可用；`StatusBar` / `TodoPanel` 依据快照刷新。
27. 用户空闲时 `ctrl+c` → App 发 `shutdown`（228-237）→ 后端 emit `shutdown` 后 break → 前端 `onExit(0)` → `restoreTerminal` 恢复光标退出；POSIX 下父进程杀整个进程组回收后端（150-168）。

**拦截路径补充说明：** 上例中 `/permissions`、`/model` 等命令不会走 27 步的模型轮。前端 `handleCommand` 直接把它们转成 `select_command{command}` 请求；后端 `_handle_select_command`（475-680）为 11 种命令构造选项（权限三态 507-535、theme 用 `list_themes()` 537-553、effort 四档 574-588、turns 集合 {32,64,128,200,256,512}+unlimited 605-622、model 走 `_model_select_options` 682-760）→ emit `select_request` → 前端 `SelectModal` 选择 → `apply_select_command{command, value}` → 后端 `_build_select_command_line`（401-426）拼回真正的斜杠命令文本执行。

---

## 十、关键符号总表

### 10.1 cli.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `app` | cli.py:751 | typer 根应用（invoke_without_command=True） |
| `main` | cli.py:2181 | 会话主回调，五条运行分流 |
| `_PREVIEW_STOPWORDS` | cli.py:20 | dry-run 推荐过滤词表 |
| `_dry_run_command_behavior` | cli.py:124 | 斜杠命令行为分类（read_only/stateful/unknown） |
| `_recommend_preview_candidates` | cli.py:253 | 基于 prompt 的推荐打分 |
| `_evaluate_dry_run_readiness` | cli.py:333 | ready/warning/blocked 评估 |
| `_build_dry_run_preview` | cli.py:396 | dry-run 静态预览构建 |
| `_format_dry_run_preview` | cli.py:600 | dry-run 文本渲染 |
| `_PROVIDER_LABELS` / `_AUTH_SOURCE_LABELS` | cli.py:1238 / 1253 | 中文标签映射 |
| `_can_use_questionary` | cli.py:1269 | TTY 检测 |
| `_login_provider` | cli.py:1716 | 三类登录分发 |
| `_bind_external_provider` | cli.py:1680 | Codex/Claude 订阅绑定 |
| `_run_copilot_login` | cli.py:1906 | Copilot 设备码流程 |
| `setup_cmd` | cli.py:1753 | 统一 setup 流程 |
| `mcp_list / mcp_add / mcp_remove` | cli.py:787 / 805 / 826 | MCP 配置管理 |
| `plugin_list / install / uninstall` | cli.py:844 / 860 / 871 | 插件管理 |
| `cron_start/stop/status/list/toggle/history/logs` | cli.py:887-1006 | 调度器管理 |
| `autopilot_*`（10 个） | cli.py:1012-1232 | 仓库自动代理队列 |
| `config_show / config_set` | cli.py:2006 / 2015 | 设置读写 |
| `provider_list/use/add/edit/remove` | cli.py:2042/2056/2072/2116/2162 | 供应商 profile 管理 |

### 10.2 ui/protocol.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `FrontendImageAttachment` | protocol.py:15 | 图片附件（base64） |
| `FrontendRequest` | protocol.py:37 | 前端→后端请求模型（8 种 type） |
| `TranscriptItem` | protocol.py:60 | 转录行模型 |
| `TaskSnapshot` | protocol.py:70 | 任务快照（from_record 工厂） |
| `BackendEvent` | protocol.py:90 | 后端→前端事件模型（18 种 type） |
| `ready / state_snapshot / tasks_snapshot / status_snapshot` | protocol.py:137 / 153 / 157 / 164 | 事件工厂 |
| `_state_payload` | protocol.py:202 | AppState → dict |
| `_MODE_LABELS` / `_format_permission_mode` | protocol.py:226 / 236 | 权限展示名 |

### 10.3 ui/backend_host.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `_PROTOCOL_PREFIX` | backend_host.py:45 | `"OHJSON:"` framing |
| `BackendHostConfig` | backend_host.py:48 | 后端会话全量配置 |
| `ReactBackendHost` | backend_host.py:73 | 协议后端主机 |
| `ReactBackendHost.run` | :92 | 主循环（构建 runtime + 请求分发） |
| `ReactBackendHost._read_requests` | :190 | stdin 行读取 + 响应投喂 |
| `ReactBackendHost._interrupt_active_request` | :244 | 打断当前轮 |
| `ReactBackendHost._process_line` | :250 | 处理 submit_line（事件映射 + handle_line） |
| `ReactBackendHost._render_event` | :274 | 引擎 StreamEvent → 协议事件 |
| `ReactBackendHost._apply_select_command` | :391 | 应用选择框结果 |
| `ReactBackendHost._build_select_command_line` | :401 | 选择值 → 斜杠命令文本 |
| `ReactBackendHost._status_snapshot` | :428 | status_snapshot 事件 |
| `ReactBackendHost._handle_list_sessions` | :454 | resume 会话选择 |
| `ReactBackendHost._handle_select_command` | :475 | 11 种命令的选择框 |
| `ReactBackendHost._model_select_options` | :682 | 模型候选列表 |
| `ReactBackendHost._ask_permission` | :762 | 权限模态（300s 超时） |
| `ReactBackendHost._ask_edit_approval` | :791 | 编辑 diff 模态（always 缓存） |
| `ReactBackendHost._ask_question` | :825 | 提问模态（无超时） |
| `ReactBackendHost._emit` | :844 | OHJSON 事件写出 |
| `ReactBackendHost._build_user_message_with_images` | :857 | 组装含图 user 消息 |
| `run_backend_host` | :883 | 模块级入口（chdir + 运行） |
| `_edit_approval_reply_from_request` | :937 | reply 归一为 once/always/reject |

### 10.4 ui/runtime.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `_resolve_image_generation_config` | runtime.py:54 | 图像生成配置 |
| `_resolve_vision_config` | runtime.py:93 | 视觉模型配置 |
| `RuntimeBundle` | runtime.py:120 | 一次会话共享对象容器 |
| `current_settings / current_plugins / hook_summary / plugin_summary / mcp_summary` | runtime.py:143/155/163/167/178 | bundle 摘要方法 |
| `_resolve_api_client_from_settings` | runtime.py:193 | 按设置选 API client |
| `build_runtime` | runtime.py:274 | 15 步装配运行时 |
| `start_runtime` | runtime.py:474 | 会话开始 hook |
| `close_runtime` | runtime.py:482 | 会话收尾（sandbox/MCP/hooks/client） |
| `_format_pending_tool_results` | runtime.py:517 | 工具结果待续摘要 |
| `sync_app_state` | runtime.py:567 | 用当前设置刷新 AppState |
| `refresh_runtime_client` | runtime.py:596 | provider/auth 变更后的运行时刷新 |
| `handle_line` | runtime.py:621 | 一行输入的统一处理（命令或模型轮） |
| `_render_command_result` | runtime.py:777 | 命令结果渲染（含消息回放） |

### 10.5 ui/app.py / coordinator_drain.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `_decode_task_worker_line` | app.py:19 | worker 行解码 |
| `run_repl` | app.py:40 | 默认交互会话（backend_only → host；否则 launch_react_tui） |
| `run_task_worker` | app.py:92 | headless stdin 驱动 worker |
| `run_print_mode` | app.py:177 | 非交互输出（text/stream-json/json） |
| `_TERMINAL_TASK_STATUSES` | coordinator_drain.py:25 | 终态集合 |
| `pending_async_agent_entries` | coordinator_drain.py:37 | 未通知的 agent 条目 |
| `wait_for_completed_async_agent_entries` | coordinator_drain.py:62 | 0.1s 轮询等终态 |
| `format_completed_task_notifications` | coordinator_drain.py:89 | 组装 `<task-notification>` |
| `submit_follow_up` | coordinator_drain.py:118 | 提交通知回协调者 |
| `drain_coordinator_async_agents` | coordinator_drain.py:159 | 循环排空后台 agent |

### 10.6 textual_app.py / react_launcher.py

| 名称 | 位置 | 职责 |
|---|---|---|
| `PermissionScreen` | textual_app.py:47 | 权限弹窗 |
| `QuestionScreen` | textual_app.py:88 | 提问弹窗 |
| `OpenHarnessTerminalApp` | textual_app.py:138 | Textual 备选 TUI |
| `_render_event` | textual_app.py:327 | 引擎事件 → Textual 组件 |
| `_refresh_sidebars` | textual_app.py:436 | 侧栏快照刷新 |
| `_resolve_theme` / `_resolve_npm` / `_resolve_tsx` | react_launcher.py:13 / 22 / 27 | 前端工具链解析 |
| `get_frontend_dir` | react_launcher.py:59 | 前端目录（包内/仓库/兜底） |
| `build_backend_command` | react_launcher.py:81 | 拼 --backend-only 命令 |
| `launch_react_tui` | react_launcher.py:116 | 拉起重端子进程 |

### 10.7 frontend/terminal

| 名称 | 位置 | 职责 |
|---|---|---|
| `index.tsx` 启动 | src/index.tsx:81 | 渲染 `<App/>`（EIO/TTY 防御） |
| `App` / `AppInner` | src/App.tsx:52 / 61 | 前端主状态机与按键分发 |
| `SELECTABLE_COMMANDS` | src/App.tsx:31 | 前端拦截的可选项命令 |
| `handleCommand` | src/App.tsx:188 | 命令拦截 |
| `useInput` | src/App.tsx:223 | 全局按键分发 |
| `onSubmit` | src/App.tsx:451 | 提交路径 |
| `useBackendSession` | src/hooks/useBackendSession.ts:24 | 子进程管理 + 事件分发 |
| `sendRequest` | useBackendSession.ts:107 | 写一行 JSON 到后端 stdin |
| `handleEvent` | useBackendSession.ts:179 | 协议事件 → React state |
| `MultilineTextInput` | components/PromptInput.tsx:19 | 自绘光标多行输入 |
| `groupToolPairs` | components/ConversationView.tsx:13 | tool/tool_result 配对 |
| `MarkdownText` | components/MarkdownText.tsx:307 | marked 驱动的富文本 |
| `summarizeInput` | components/ToolCallDisplay.tsx:116 | 工具入参摘要 |
| `parseDiffLines` / `EditDiffModal` | components/ModalHost.tsx:96 / 116 | diff 解析与渲染 |
| `parseTodoItems` | components/TodoPanel.tsx:9 | checklist 解析 |
| `PlanModeIndicator` | components/StatusBar.tsx:14 | 计划模式指示 |
| `readClipboardImage` | clipboardImage.ts:24 | 跨平台剪贴板截图 |
| `ThemeProvider` / `builtinThemes` | theme/ThemeContext.tsx:17 / builtinThemes.ts:26 | 主题上下文 / 五种内置主题 |

---

## 十一、设计观察与演进痕迹

1. **双进程 + 管道协议**：UI 性能关键路径上前后端通过 stdin/stdout JSON-lines 双向通信；后端是前端 spawn 的子进程，POSIX 下以独立进程组存在并整体回收，避免僵尸进程（useBackendSession.ts:150-168）。
2. **无端口、无 Socket**：协议极简（`OHJSON:` 前缀 + 单行 JSON），屏蔽了 TCP/WebSocket 的端口管理与鉴权复杂度，同时让后端可被任何脚本化前端复用（`--backend-only` / `--task-worker` / `-p` 三者共用同一套 `runtime.handle_line`）。
3. **状态机分层**：引擎层 `QueryEngine` 只吐 `StreamEvent`；`runtime.handle_line` 是渲染无关的「调度中枢」；`backend_host._render_event`（→OHJSON）、`textual_app._render_event`（→Textual）、`app._render_event`（→print 模式）分别把同一事件流映射到三种目标——事件映射保持同构，便于维护。
4. **前端节流渲染**：`useDeferredValue` + 50ms/384 字符批量 flush + `startTransition` + `stableStringify` 快照去重，四套独立机制协同避免每 token 一次全树重渲。
5. **React Ink 而非 webview**：整个 UI 位于终端内，配合 `PromptInput` 自绘光标、`marked` 精确排版（含 CJK 表格对齐）、`/dev/tty` 兜底，是「终端优先」产品定位的体现。
6. **权限模态的 request_id 配对**：三种模态共用「request_id → asyncio.Future」映射，reader 线程直接完成 future，主循环与引擎采用协程挂起等待，天然支持并发模态（编辑 diff 与问题弹窗可叠加）；`edit_diff` 的 `always` 会写回后端缓存（`_edit_always_approved`），同一会话内不再打扰。
7. **协议文档化痕迹**：`assistant_complete` 与 `line_complete` 的 busy 语义边界在源码注释里被刻意强调（useBackendSession.ts:342-343）——前端只在 `line_complete` 复位 busy，避免一轮内多次工具调用时输入框提前解锁。
8. **dry-run 是静态诊断**：不真实解析 API client（用 `redirect_stderr` 吞掉 SystemExit 并捕获异常），也不启动 MCP 外部进程，因此能在数毫秒内给出「输入会走到哪、哪一步可能失败」的预判（Readiness 的 ready/warning/blocked 三态 + recommendations）。
9. **遗留组件保留**：`Composer` / `TranscriptPane` / `SidePanel` / `Footer` 与 `textual_app.py`、`output.py`、`input.py`、`permission_dialog.py` 属于演进过程中的旧布局 / 旧渲染路径，未被当前默认路径引用；理解历史时可查阅，新增改动不应依赖它们。
10. **可观测性对齐**：后端 `AppStateStore`（runtime.py:332-356）、协议 `_state_payload`（protocol.py:202）、前端 `types.ts` 三方对「状态」的定义严格同步，新增状态字段需要三处一起改——这是本架构里最需要留意的一致面。

---

> 说明：本文所有行号取自当前仓库源码，与 `git log` 中的版本一一对应；若未来重构导致行号漂移，请以函数名 + 文件路径为准。前端遗留组件（Composer 等）与备选 TUI（textual_app.py）为演进产物，理解历史时参考，新增功能不应依赖。

---

## 十二、实际示例（基于真实代码的示意运行）

> 以下各示例的命令输出、协议帧、JSON 均为**按真实字段结构拼装的示意**，字段名与嵌套层级取自对应源码；「内容为示意」指模型回复文案、路径、diff、权限原因等随运行环境变化的取值。每个示例都给出字段/帧结构来源（`<文件>:<行号>`），可直接对照源码验证。

### 示例 1：OHJSON 线帧——`submit_line` 请求 + 后端事件帧

> 示例：完整的 OHJSON 线帧样例（帧结构来源 `protocol.py:37-57` / `protocol.py:90-135`、`backend_host.py:844-854`、`useBackendSession.ts:107-113`；内容为示意）

场景：用户在 React TUI 输入 `列出 src 目录的 Python 文件` 并回车。前端 → 后端方向**每行是一段纯 JSON（无前缀）**，后端 → 前端方向**每行以 `OHJSON:` 前缀开头的 JSON**；两端都是「一行一帧」（`sendRequest` 写 `JSON.stringify(payload) + '\n'`，`_emit` 写 `_PROTOCOL_PREFIX + model_dump_json() + '\n'`）。

**帧 ① 客户端 → 后端：`submit_line` 请求**

```json
{"type":"submit_line","line":"列出 src 目录的 Python 文件","images":[]}
```

- 载荷由 `App.onSubmit` 构造（App.tsx:477：`sendRequest({type:'submit_line', line, images})`），`sendRequest` 在 useBackendSession.ts:107-113 完成 framing；
- 字段来自 `FrontendRequest`（protocol.py:37-57）：`type` 必填，`line` / `images` 可选，`images` 为 `FrontendImageAttachment` 列表（base64 图片，protocol.py:15-34，此处为空列表）。

**帧 ② 后端 → 客户端：`transcript_item`（回显用户输入）**

```json
OHJSON:{"type":"transcript_item","item":{"role":"user","text":"列出 src 目录的 Python 文件"}}
```

- 来自 `_process_line` 的 user 回显（backend_host.py:259-267），`TranscriptItem` 结构见 protocol.py:60-67（`role` / `text` / `tool_name` / `tool_input` / `is_error`）。

**帧 ③ 后端 → 客户端：`assistant_delta`（流式文本增量）**

```json
OHJSON:{"type":"assistant_delta","message":"我来看看这个目录的结构。\n"}
OHJSON:{"type":"assistant_delta","message":"先列出匹配的 Python 文件。"}
```

- 引擎 `AssistantTextDelta` 被 `_render_event` 映射为 `BackendEvent(type="assistant_delta", message=event.text)`（backend_host.py:275-277）；`message` 字段即 protocol.py:114 的可选 `message` 载荷。
- 前端不做逐帧渲染：累积到 `pendingAssistantDeltaRef`，满 384 字符或 50ms 定时器到期时批量 flush（useBackendSession.ts:18-20 / 56-66）。

**帧 ④ 后端 → 客户端：`tool_started` / `tool_completed`（工具执行）**

```json
OHJSON:{"type":"tool_started","tool_name":"glob","tool_input":{"pattern":"src/**/*.py"},"item":{"role":"tool","text":"glob {\"pattern\": \"src/**/*.py\"}","tool_name":"glob","tool_input":{"pattern":"src/**/*.py"}}}
OHJSON:{"type":"tool_completed","tool_name":"glob","output":"src/openharness/cli.py\nsrc/openharness/main.py\n…","is_error":false,"item":{"role":"tool_result","text":"src/openharness/cli.py\nsrc/openharness/main.py\n…","tool_name":"glob","is_error":false}}
```

- `tool_started` 由 `ToolExecutionStarted` 映射而来，payload 含 `tool_name` / `tool_input` 以及带 `role:"tool"` 的 `item`（backend_host.py:301-315）；
- `tool_completed` 由 `ToolExecutionCompleted` 映射而来，含 `tool_name` / `output` / `is_error` 与 `role:"tool_result"` 的 `item`（backend_host.py:317-330）；
- 工具结束后后端还会补发 `tasks_snapshot` 与 `status_snapshot`（backend_host.py:332-333）；`TodoWrite` 额外发 `todo_update`，`set_permission_mode`/`plan_mode` 额外发 `plan_mode_change`（backend_host.py:334-354）。

**帧 ⑤ 后端 → 客户端：`line_complete`（整行结束）**

```json
OHJSON:{"type":"line_complete"}
```

- `_process_line` 收敛时连发 `status_snapshot`、`tasks_snapshot`、`line_complete`（backend_host.py:386-388）；
- 前端见到 `line_complete` 才复位 busy（useBackendSession.ts:347-353）——`assistant_complete` 不复位 busy，因为一轮内可能还有后续工具调用（注释 342-343）。

### 示例 2：非交互输出——`oh -p "列出文件"`

> 示例：`oh -p "列出文件"` 的 stdout 形态（来源 `cli.py:2498-2519` 与 `app.py:177-320`；内容为示意）

`-p/--print` 分支把参数直接交给 `run_print_mode`（app.py:177），headless 跑一轮 `handle_line` 后退出。`--output-format` 三态：

**（a）text（默认）——流式写 stdout**

```text
$ oh -p "列出文件"
（stdout 逐 token 流出模型的回答）
src/
├── openharness/
│   ├── cli.py
│   ├── main.py
│   ├── ui/
│   │   ├── app.py
│   │   ├── backend_host.py
│   │   └── protocol.py
└── …
```

- `AssistantTextDelta` → `sys.stdout.write(event.text)`（app.py:244-246）；
- `system` 类消息（如引擎状态提示）走 **stderr**：`print(message, file=sys.stderr)`（app.py:233-234），不污染 stdout 管道；
- `AssistantTurnComplete` → 补一个换行（app.py:251-254）。

**（b）`--output-format json`——只在全部结束后输出一个 result 对象**

```text
$ oh -p "列出文件" --output-format json
{"type":"result","text":"src/\n├── openharness/\n│   ├── cli.py\n│   ├── main.py\n│   └── ui/\n…"}
```

- json 模式全程把 `AssistantTextDelta` 累积进 `collected_text`（app.py:242-243），最后 `{"type":"result","text": collected_text.strip()}` 一次性打印（app.py:316-318）；
- 工具事件（tool_started / tool_completed）、error、compact_progress 等**不出现在 result 里**，只有 `stream-json` 才逐事件输出（见下文附注）。

**（c）`--output-format stream-json`（附注）——每事件一行 JSON**

```text
{"type":"assistant_delta","text":"src/"}
{"type":"tool_started","tool_name":"glob","tool_input":{"pattern":"src/**/*.py"}}
{"type":"tool_completed","tool_name":"glob","output":"src/openharness/cli.py\n…","is_error":false}
{"type":"assistant_complete","text":"src/\n├── …"}
```

- 各 `obj` 由 `_render_event` 按事件类型拼装后 `print(json.dumps(obj), flush=True)`（app.py:235-295）；`system` 回显为 `{"type":"system","message":…}`（app.py:236-238）。

### 示例 3：`oh --dry-run` 输出示例

> 示例：`oh --dry-run -p "列出文件" --output-format json` 的输出 JSON（preview 结构来源 `cli.py:396-597`，产物字段 537-589；内容为示意）

dry-run 分支（cli.py:2403-2434）调用 `_build_dry_run_preview` 之后，`--output-format json` 走 `json.dumps(preview, ensure_ascii=False, indent=2)`（cli.py:2425）。以下为**节选示意**，字段名、嵌套层级严格对应源码 `preview` dict：

```json
{
  "mode": "dry-run",
  "cwd": "/Users/me/proj",
  "config_path": "/Users/me/.config/openharness/settings.json",
  "prompt": "列出文件",
  "prompt_preview": "列出文件",
  "settings": {
    "active_profile": "my-claude",
    "profile_label": "Claude API",
    "provider": "anthropic",
    "api_format": "anthropic",
    "model": "claude-sonnet-4-5",
    "base_url": "",
    "permission_mode": "default",
    "max_turns": 16,
    "effort": "medium",
    "passes": 1
  },
  "validation": {
    "auth_status": "anthropic_api_key configured",
    "api_client": {"status": "ok", "detail": ""},
    "system_prompt_chars": 4291,
    "mcp_validation": "skipped in dry-run (configured only; external servers are not started)",
    "mcp_errors": 0
  },
  "entrypoint": {
    "kind": "model_prompt",
    "detail": "The first live step would be a model request. Exact tool calls and parameters are decided by the model at runtime."
  },
  "readiness": {
    "level": "ready",
    "reasons": ["Resolved configuration, prompt assembly, and static discovery checks all look usable."],
    "next_actions": ["You can run this prompt directly with `oh -p '...'` or open the interactive UI with `oh`."]
  },
  "commands": [
    {"name": "help", "description": "Show help …", "remote_invocable": true, "remote_admin_opt_in": false, "behavior": {"kind": "read_only", "detail": "…"}}
  ],
  "tools": [
    {"name": "Bash", "description": "Run a shell command…", "required_args": ["command"], "optional_args": []},
    {"name": "glob", "description": "Find files by pattern…", "required_args": ["pattern"], "optional_args": ["path"]}
  ],
  "skills": [{"name": "agentlens", "description": "…", "source": "…"}],
  "mcp_servers": [],
  "plugins": [],
  "recommendations": {"skills": [], "tools": [], "commands": []},
  "system_prompt_preview": "You are OpenHarness…（截断至 600 字符）"
}
```

- `entrypoint.kind` 三态（`slash_command` / `unknown_slash_command` / `model_prompt`，无 prompt 时为 `interactive_session`）见 cli.py:501-535；
- `readiness` 由 `_evaluate_dry_run_readiness`（cli.py:333-393）汇总 `level`（ready / warning / blocked）+ `reasons` + `next_actions`；
- `tools` 元素由 `_schema_argument_preview` 生成 `name` / `description` / `required_args` / `optional_args`（cli.py:52-66）；
- `mcp_servers` 元素由 `_validate_mcp_server` 生成 `name` / `transport` / `target` / `status` / `issues`（cli.py:89-121）。整个预览是**纯静态解析**：真实生成但不真实执行。

### 示例 4：React 前端「一次按键到渲染」

> 示例：用户在输入框回车 → `submit_line` 帧 → `assistant_delta` 一路渲染进对话区（来源 `App.tsx:451-485`、`useBackendSession.ts:107-136` / `297-321`、`types.ts:6-12`；`Composer.tsx` 为遗留组件，当前生效路径是 `PromptInput` + `ConversationView`）

> 补充说明：题目中的 `Composer.tsx`（components/Composer.tsx:5-32，基于 `ink-text-input` 的旧输入框）与 `TranscriptPane.tsx`（components/TranscriptPane.tsx:6-27，旧转录窗，接收 `items: TranscriptItem[]` 与 `assistantBuffer: string`）**当前均未被 `App.tsx` / `index.tsx` 引用**（见第 8.6 节组件清单）。今日的写入组件是 `PromptInput.tsx`（自绘光标多行输入），渲染组件是 `ConversationView.tsx`。以下示例按当前真实路径展开，并指出与旧组件的对应关系。

**步骤 1：用户输入并回车。** `PromptInput` 的 `TextInput.onSubmit` 触发 → `App.onSubmit(value)`（App.tsx:451）。`onSubmit` 先做三层拦截（question 弹窗应答 → `/stop` 打断 → `handleCommand` 拦截 `/provider`、`/permissions` 等可选项命令），均未命中则进入提交：

**步骤 2：编码并发送 `submit_line` 帧。** App.tsx:477：

```ts
session.sendRequest({type: 'submit_line', line: value, images: imagePayloads()});
session.setBusy(true);
```

`sendRequest`（useBackendSession.ts:107-113）把载荷 `JSON.stringify` 后写一行到后端 `child.stdin`：

```json
{"type":"submit_line","line":"列出 src 目录的 Python 文件","images":[]}
```

**步骤 3：后端消费、回显、流式返回。** `_read_requests` 读到该行并用 `FrontendRequest.model_validate_json` 校验（backend_host.py:190-222）→ 主循环 `submit_line` 分支（backend_host.py:166-181）→ `_process_line` → `handle_line` → 引擎流式产出。后端回推 `transcript_item`(user) / `assistant_delta` / `tool_started` / `tool_completed` / `line_complete`，见示例 1。

**步骤 4：前端按行解析。** `child.stdout` 挂 `readline.createInterface`，`reader.on('line')` 判断 `line.startsWith('OHJSON:')`，命中则 `JSON.parse(line.slice(PROTOCOL_PREFIX.length))`（即去掉 7 字符前缀 `OHJSON:`）得到 `BackendEvent`（useBackendSession.ts:129-136）；未命中则作为 `{role:'log', text:line}` 入 transcript。

**步骤 5：`assistant_delta` 进入状态。** `handleEvent` 的 `assistant_delta` 分支（useBackendSession.ts:297-321）：追加到 `pendingAssistantDeltaRef`，每 50ms 或 384 字符调用一次 `flushAssistantDelta`（56-66），把累计文本追加进 `assistantBufferRef` 并用 `startTransition(() => setAssistantBuffer(...))` 更新 **`assistantBuffer`** state。

**步骤 6：App 层降频接手。** App.tsx:77 `const deferredAssistantBuffer = useDeferredValue(session.assistantBuffer)`，连同 `session.transcript`（`useDeferredValue` 后的 `deferredTranscript`）一并传给 `ConversationView`（App.tsx:507-512）。旧 `TranscriptPane` 顶替了这条渲染路径中的 `items` / `assistantBuffer` 两个 props（TranscriptPane.tsx:6-12）。

**步骤 7：`assistant_complete` 落一条 `role:"assistant"` 的 transcript。** message 或累积的 buffer 追加进 transcript 数组并 `clearAssistantDelta`（useBackendSession.ts:322-346）——**busy 仍为 true**（注释 342-343）。

**步骤 8：`line_complete` 复位，输入框恢复。** `clearAssistantDelta(); setBusy(false); setBusyLabel(undefined)`（useBackendSession.ts:347-353）。此时 `transcript` 里最终含 user / assistant / tool / tool_result 等条目，`types.ts:6-12` 的 `TranscriptItem` 结构（role / text / tool_name? / tool_input? / is_error?）即每条的形态。

### 示例 5：权限弹窗——`tool_execution` 携带权限请求

> 示例：后端要执行写类工具 → `modal_request` 权限弹窗帧 → 用户在 ModalHost 选择 → `permission_response` 帧回后端（来源 `backend_host.py:762-784` / `209-212`、`App.tsx:301-321`、`ModalHost.tsx:187-221`；内容为示意）

**场景：** default 权限模式下，模型想运行 `Edit` 工具改 `src/app.py`。引擎调用注入的 `_ask_permission(tool_name, reason)`（backend_host.py:762）：

**帧 ① 后端 → 前端：`modal_request`（kind=permission）**

```json
OHJSON:{"type":"modal_request","modal":{"kind":"permission","request_id":"9f2c1b7a4e6d","tool_name":"Edit","reason":"Writing to src/app.py"}}
```

- 后端先生成 `request_id = uuid4().hex` 并注册 `self._permission_requests[request_id] = future`（764-766），再 emit 上述弹窗（767-777）；**300 秒内无人应答则超时拒绝**（`asyncio.wait_for(future, timeout=300)`，779-782）。

**用户侧渲染：** 前端 `modal_request` 分支 `setModal(event.modal)`（useBackendSession.ts:387-390）→ `App` 渲染 `ModalHost`（App.tsx:516-523）。`ModalHostInner` 对 `kind==='permission'` 画出 `Allow <tool_name>?` + reason + `[y] Allow / [n] Deny`（ModalHost.tsx:198-221）。

**帧 ② 前端 → 后端：`permission_response`（用户按 y）**

```json
{"type":"permission_response","request_id":"9f2c1b7a4e6d","allowed":true}
```

- 来自 App.tsx:302-308 的按键处理（`y` → allowed:true；`n`/Esc → allowed:false，超过则忽略）。

**后端恢复执行：** reader 线程发现 `request_id` 命中 `_permission_requests`，直接 `future.set_result(bool(request.allowed))`（backend_host.py:209-212）→ `_ask_permission` 返回 → 引擎继续执行该工具。

**编辑类工具的变体（edit_diff）：** 后端 `_ask_edit_approval`（backend_host.py:791-823）改发：

```json
OHJSON:{"type":"modal_request","modal":{"kind":"edit_diff","request_id":"e3a8…","path":"src/app.py","diff":"@@ -1,3 +1,3 @@\n-print('old')\n+print('new')","added":1,"removed":1}}
```

前端 `EditDiffModal` 渲染 diff 与 `[y] Once / [a] Always / [n] Deny`（ModalHost.tsx:116-184），按键走 App.tsx:324-356，回帧带 `permission_reply`：

```json
{"type":"permission_response","request_id":"e3a8…","allowed":true,"permission_reply":"always"}
```

后端 `_edit_approval_reply_from_request` 把 reply 归一为 once/always/reject（backend_host.py:937-941）；收到 `always` 时置 `_edit_always_approved=True`，模态结束 emit `modal_request{modal:null}` 让前端关闭弹窗（819 / 821-822）。若 `full_auto` 或已 always，直接返回 `"always"` 不弹窗（792-793）。

### 示例 6：`--continue` / `--resume` 恢复会话

> 示例：`oh --continue` 与 `oh -r <session_id>`（来源 `cli.py:2437-2496`、`session_storage.py:123-207`、`runtime.py:379-398` / `433-437`；内容为示意）

**（a）`oh --continue`——按目录恢复最近会话**

```text
$ oh --continue
Continuing session: 修复登录流程的 bug
```

- cli.py:2445-2450 调 `load_session_snapshot(cwd)`（session_storage.py:123-128），它读项目目录对应的 `latest.json`（目录由 cwd 的 sha1[:12] 派生，session_storage.py:54-60）；
- 找不到时 stderr 报 `No previous session found in this directory.` 并以退出码 1 退出（2447-2449）。

**（b）`oh -r <session_id>`——精确恢复**

```text
$ oh -r 7f3a1c9d2b0e
```

- 走 `load_session_by_id(cwd, resume)`（session_storage.py:194-207）：先找 `session-<id>.json`，回落后查 `latest.json` 的 `session_id`。

**（c）`oh -r`（不带值）——列表选择器**

```text
$ oh -r
Saved sessions:
  1. [7f3a1c9d2b0e] 修复登录流程的 bug (42 msgs)
  2. [9c2e11f8d3a4] 重构 MCP 连接 (18 msgs)
Enter session number or ID:
```

- 由 `list_session_snapshots(cwd, limit=10)`（session_storage.py:131-191）按 `created_at` 降序给出，输入序号或 ID 后 `load_session_by_id` 加载（cli.py:2451-2472）。

**快照文件形态：** 每个会话同时写 `latest.json` 与 `session-<id>.json`（session_storage.py:99-105），结构字段来自 `save_session_snapshot`（session_storage.py:85-96）：

```json
{
  "session_id": "7f3a1c9d2b0e",
  "cwd": "/Users/me/proj",
  "model": "claude-sonnet-4-5",
  "system_prompt": "…",
  "messages": [{"role":"user","content":[{"type":"text","text":"帮我看看登录流程"}], …}],
  "usage": {"input_tokens": 1203, "output_tokens": 891, …},
  "tool_metadata": {"permission_mode":"default","read_file_state":[],"invoked_skills":[],"async_agent_state":[],"async_agent_tasks":[],"recent_work_log":[],"task_focus_state":{"goal":"","recent_goals":[],"active_artifacts":[],"verified_state":[],"next_step":""},"compact_checkpoints":[]},
  "created_at": 1750000000.0,
  "summary": "修复登录流程的 bug",
  "message_count": 42
}
```

- `tool_metadata` 只持久化 `_PERSISTED_TOOL_METADATA_KEYS` 白名单内的键（session_storage.py:18-29），`_sanitize_metadata` 保证可 JSON 序列化（32-41）。

**重建 runtime 继续：** cli.py:2480-2495 把恢复出的东西交给 `run_repl`：

```python
asyncio.run(run_repl(
    prompt=None,
    cwd=cwd,
    model=session_data.get("model") or model,
    restore_messages=session_data.get("messages"),
    restore_tool_metadata=session_data.get("tool_metadata"),
    …))
```

两条恢复线在 `build_runtime` 汇合（runtime.py:379-398 与 433-437）：

1. `restore_tool_metadata` 逐键覆盖到 `restored_metadata` 骨架（权限模式 / read_file_state / invoked_skills / async_agent 状态 / recent_work_log / task_focus_state / compact_checkpoints），并作为 `engine.tool_metadata` 的一部分注入（runtime.py:396-398、418-428）；
2. `restore_messages` 经 `sanitize_conversation_messages` + `ConversationMessage.model_validate` 反序列化后 `engine.load_messages(restored)`（runtime.py:433-437），引擎因此带着完整历史继续推进——后续每次 `handle_line` 结束仍会 `save_snapshot` 覆盖回 `latest.json`（见 4.5 节）。

---

> 本章示例字段均取自当前仓库源码（见各示例索引）；实际运行时的具体文案、路径与 diff 内容随环境变化，不作为协议约定的一部分。
