# 09 - MCP 桥接与渠道系统

> 本文档精读自 `src/openharness/mcp/`、`src/openharness/bridge/`、`src/openharness/channels/` 三个子系统的全部源码，
> 并覆盖其上游/下游调用点（`tools/mcp_tool.py`、`tools/mcp_auth_tool.py`、`tools/read_mcp_resource_tool.py`、
> `tools/list_mcp_resources_tool.py`、`tools/__init__.py`、`ui/runtime.py`、`commands/registry.py`、`config/schema.py`、
> `config/paths.py`、`utils/shell.py`、`engine/stream_events.py` 等）。
> 所有类名、函数名、字段名、文件路径与行号均取自当前仓库，并已用 Grep/Read 逐一核对，未作虚构。

---

## 一、各子系统职责总述

OpenHarness 的三条"对外/外接"通路构成了一个层次分明的架构：

| 子系统 | 目录 | 职责 | 一句话概括 |
| --- | --- | --- | --- |
| **MCP** | `src/openharness/mcp/` | Model Context Protocol 客户端 | 启动外部 MCP server（stdio/HTTP），发现其工具与资源，并把它们"翻译"成 OpenHarness 自己的工具/资源供引擎调用 |
| **Bridge** | `src/openharness/bridge/` | 桥接会话管理 | 把子进程/CLI 会话当作可观测的"桥接会话"来孵化、跟踪、截取输出，并提供 work secret（会话凭证）编解码与 SDK 接入 URL 构造 |
| **Channels** | `src/openharness/channels/` | 多渠道消息接入 | 用"消息总线 + 渠道适配器"架构把 Telegram/飞书/Matrix 等即时通讯渠道接入 QueryEngine，实现"聊天即对话" |

三个子系统各司其职：

- **MCP** 面向"**工具侧**"的横向扩展（把能力装进引擎），通过 `McpToolAdapter` 把 MCP server 的工具注册成 `mcp__<server>__<tool>` 形式的常规工具。
- **Bridge** 面向"**多会话**"的纵向编排（把外部 CLI 会话拉进本进程管理），服务于 UI 的 `bridge_sessions` 状态与 `/bridge` 命令族。
- **Channels** 面向"**人机交互**"的入口/出口（让真人通过 IM 与引擎对话），基于 `MessageBus` 解耦渠道与引擎，由 `ChannelBridge` 把 inbound 消息喂给 `QueryEngine` 并把回复送回总线。

> **注意**：`channels/UPSTREAM` 文件表明 channels 的 `bus/` 与 `impl/` 目录是从 [nanobot-ai/nanobot](https://github.com/nanobot-ai/nanobot)
> 仓库同步（sync commit `473ae5e`，2026-04-05）而来的。同步后 `impl/` 内部是"双份"渠道管理：
> `manager.py`（`ChannelManager`）是 OpenHarness 侧保留的原生调度器，而各渠道适配器类（`*Channel`）才是真正对接 IM 协议的主体。
> 各渠道实现还大量读取 `config.<渠道>.` 上的扩展字段（如 `react_emoji`、`group_policy`、`e2ee_enabled`、`reply_delay_mode` 等），
> 这些字段不在 `config/schema.py` 的公开模型里显式声明，而是由 ohmo/gateway 的扩展配置模型通过 `_CompatModel` 的宽容行为提供，阅读配置时需要注意这一点。

---

## 二、MCP 客户端（`src/openharness/mcp/`）

### 2.1 文件布局与导入策略

```
mcp/
├── __init__.py    # 79 行：惰性导出（__getattr__），避免导入时拉起 mcp 官方 SDK
├── client.py      # 298 行：McpClientManager —— 连接管理核心
├── config.py      # 16 行：load_mcp_server_configs —— 配置合并
└── types.py       # 76 行：配置/状态/元数据模型（pydantic + dataclass）
```

`mcp/__init__.py`（79 行）采用 **PEP 562 模块级 `__getattr__`** 做惰性导出：

- `TYPE_CHECKING` 块（`__init__.py:7-18`）仅用于类型检查期导入，避免循环依赖；
- `__all__`（`__init__.py:20-32`）声明 10 个公开符号；
- `__getattr__`（`__init__.py:35-78`）在真正访问时才 `from ... import ...`：
  - `McpClientManager` / `McpServerNotConnectedError` 来自 `client`（`:37-43`）；
  - `load_mcp_server_configs` 来自 `config`（`:44-47`）；
  - 其余 8 个类型一次性从 `types` 导入并按键返回（`:48-77`）。

这样设计是为了**延迟 import `mcp` 官方 SDK（以及其依赖的 `httpx`、`pydantic`）**：只要不实际使用 MCP 功能，
就不会因为装环境、导入失败等问题拖累整个应用启动。

### 2.2 类型模型（`types.py`）

`types.py` 定义了三种 server 配置（pydantic `BaseModel`）、一个联合类型、一个配置文件形状，以及三类运行期元数据（frozen dataclass）。

#### 2.2.1 配置模型（pydantic）

| 类 | 行号 | 字段 | 默认值 | 说明 |
| --- | --- | --- | --- | --- |
| `McpStdioServerConfig` | `types.py:11` | `type: Literal["stdio"]` | `"stdio"` | 本地子进程型 MCP server，通过标准输入/输出与 server 通信 |
| | | `command: str` | 必填 | 要启动的可执行文件（如 `npx`、`python`） |
| | | `args: list[str]` | `[]` | 命令行参数 |
| | | `env: dict[str,str] \| None` | `None` | 追加到子进程环境变量的映射（鉴权 token 通常放这里） |
| | | `cwd: str \| None` | `None` | 子进程工作目录 |
| `McpHttpServerConfig` | `types.py:21` | `type: Literal["http"]` | `"http"` | 远程 HTTP（Streamable HTTP）server |
| | | `url: str` | 必填 | server 的端点 URL |
| | | `headers: dict[str,str]` | `{}` | 请求头（Bearer 鉴权通常放这里） |
| `McpWebSocketServerConfig` | `types.py:29` | `type: Literal["ws"]` | `"ws"` | WebSocket server（**当前构建未支持**，连接时直接判 failed） |
| | | `url: str` | 必填 | WebSocket URL |
| | | `headers: dict[str,str]` | `{}` | 握手请求头 |
| `McpServerConfig` | `types.py:37` | — | — | 联合类型 `McpStdioServerConfig \| McpHttpServerConfig \| McpWebSocketServerConfig`，供 isinstance 分发 |
| `McpJsonConfig` | `types.py:40` | `mcpServers: dict[str, McpServerConfig]` | `{}` | 配置文件/插件约定的 JSON 形状（小驼峰 `mcpServers`，与官方约定一致） |

#### 2.2.2 运行期元数据（frozen dataclass）

- **`McpToolInfo`**（`types.py:46`）：一个 MCP 工具的元数据。字段：
  - `server_name: str` —— 所属 server 名；
  - `name: str` —— 工具名；
  - `description: str` —— 工具描述；
  - `input_schema: dict[str, object]` —— **JSON Schema**（工具入参结构），这是驱动 `McpToolAdapter` 生成 pydantic 输入模型的原料。
  - `@dataclass(frozen=True)` 保证不可变，作为快照/名片在引擎与 UI 间传递。

- **`McpResourceInfo`**（`types.py:56`）：一个 MCP 资源的元数据。字段：
  - `server_name`、`name`、`uri`（字符串化的资源 URI）、`description`（默认 `""`）。

- **`McpConnectionStatus`**（`types.py:66`，**非 frozen**）：单个 server 的运行状态快照。字段：
  - `name: str`；
  - `state: Literal["connected", "failed", "pending", "disabled"]`；
  - `detail: str = ""` —— 失败原因等补充信息；
  - `transport: str = "unknown"`；
  - `auth_configured: bool = False` —— 是否已配置鉴权（headers 或 env）；
  - `tools: list[McpToolInfo]`、`resources: list[McpResourceInfo]` —— 握手时发现并缓存的清单。

### 2.3 配置加载（`config.py`）

`load_mcp_server_configs(settings, plugins)`（`config.py:8`）负责把**全局设置**与**插件声明**的 MCP server 合并成一个字典，逻辑很短：

1. 以 `settings.mcp_servers`（`settings.py` 中的 `dict[str, McpServerConfig]`）为底（`config.py:10`）；
2. 遍历**已启用**的插件（`plugin.enabled` 才看，`config.py:12-13`）；
3. 把每个插件的 `plugin.mcp_servers` 里的 server，以 **`"{plugin.manifest.name}:{name}"`** 为键，通过 `setdefault` 加入（`config.py:14-15`）。

> **合并语义**：`setdefault` 保证**同名 server 以全局设置为准，插件永不覆盖**；
> 且插件 server 名带有 `插件名:` 前缀，天然避免与全局 key 撞名。

> 插件的 MCP server 声明来源：插件 frontmatter 的 `mcpServers` / `mcp_servers` 字段
> （`coordinator/agent_definitions.py:792` 附近），并可声明 `requiredMcpServers` 依赖，由
> `agent_definitions.py` 的 `has_required_mcp_servers` 做 agent 可用性过滤。

### 2.4 连接管理核心（`client.py`）

#### 2.4.1 异常类型与类结构

```python
class McpServerNotConnectedError(Exception):   # client.py:25
    """Raised when an MCP server is not connected or its session has been lost."""
```

`McpServerNotConnectedError` 是**所有 MCP 会话问题的统一异常**——包括：server 从未连接成功、会话已丢失、调用途中出错。
引擎侧只需捕获这一个异常即可把任何 MCP 失败转成 `ToolResult(is_error=True)`。

```python
class McpClientManager:                          # client.py:29
    def __init__(self, server_configs):          # client.py:32
        self._server_configs = server_configs
        self._statuses = {name: McpConnectionStatus(name, state="pending", transport=config.type) ...}  # :34-41
        self._sessions: dict[str, ClientSession] = {}   # :42  mcp 官方 ClientSession
        self._stacks:   dict[str, AsyncExitStack] = {}  # :43  资源栈
```

- **`_server_configs`**：合并后的配置字典（name → 配置对象）。
- **`_statuses`**：每个 server 的 `McpConnectionStatus`，初始一律 `pending`（`client.py:34-41`）。
- **`_sessions`**：`name → mcp 官方 `ClientSession``，只有握手成功才登记。
- **`_stacks`**：`name → AsyncExitStack`。`AsyncExitStack` 是资源生命周期核心——HTTP client、传输流（read/write stream）、
  ClientSession 都以 `enter_async_context` 挂进去，关闭时按**后进先出**逆序释放。

**状态机**：`pending → connected / failed`；`reconnect_all()` 先把所有状态重置回 `pending` 再重连。
`disabled` 状态在类型中保留（`types.py:71`），当前构建路径不产生。

#### 2.4.2 连接分发 `connect_all()`（`client.py:45`）

```python
async def connect_all(self) -> None:             # client.py:45
    for name, config in self._server_configs.items():
        if isinstance(config, McpStdioServerConfig):
            await self._connect_stdio(name, config)          # :48-49
        elif isinstance(config, McpHttpServerConfig):
            await self._connect_http(name, config)           # :50-51
        else:  # 例如 ws / 未知类型
            self._statuses[name] = McpConnectionStatus(... state="failed",
                detail=f"Unsupported MCP transport in current build: {config.type}", ...)  # :53-59
```

- 按配置对象的 **isinstance 类型** 分发：stdio → `_connect_stdio`，http → `_connect_http`；
- **不支持的类型（含 `ws`）直接标记 `failed`**，`detail` 写明 `Unsupported MCP transport in current build`，
  而不是抛异常中断整个 `connect_all()`（`client.py:53-59`）。

#### 2.4.3 stdio 传输：子进程启动与消息 framing（`_connect_stdio`，`client.py:180`）

```python
async def _connect_stdio(self, name, config):    # client.py:180
    stack = AsyncExitStack()                      # :181
    try:
        read_stream, write_stream = await stack.enter_async_context(
            stdio_client(StdioServerParameters(   # :184-191
                command=config.command,
                args=config.args,
                env=config.env,
                cwd=config.cwd,
            ))
        )
        await self._register_connected_session(name=name, config=config, stack=stack,
            read_stream=read_stream, write_stream=write_stream,
            auth_configured=bool(config.env))     # :193-200
    except asyncio.CancelledError as exc:         # :201
        await self._close_failed_stack(stack)     # :202
        self._mark_connection_failed(...)         # :203-208
    except Exception as exc:                      # :209
        ... 同样的失败处理（:210-216）
```

**子进程启动参数**：`StdioServerParameters` 直接把 `config.command / args / env / cwd` 透传给 `mcp.client.stdio.stdio_client()`。
mcp 官方 SDK 在 `stdio_client` 内部（**协议/SDK 层行为**，非本仓库代码）：

1. 用 `asyncio.create_subprocess_exec(command, *args, cwd=cwd, env=合并后的环境)` 拉起子进程；
   - 环境变量 = `os.environ` 拷贝 + `StdioServerParameters.env` 增量覆盖 + SDK 注入的 `MCP_*` 相关变量；
2. `stdout=PIPE, stdin=PIPE, stderr=PIPE`，为三路流分配独占的 asyncio reader/writer 任务；
3. **消息 framing**：stdio 传输把每条 JSON-RPC 消息写成**单行 JSON + `\n` 结尾**（newline-delimited JSON, NDJSON）写入子进程 stdin；
   从子进程 stdout 按行读取并解析回 JSON-RPC 响应；
4. 返回一对 `(read_stream, write_stream)`（`AnyIO` 风格的流对象），对外表现为一个 `AnyIOMemoryObjectReceiveStream` 与 `SendStream`。

**一次 stdio 会话的协议交互示例**（JSON-RPC 2.0，逐行 NDJSON，写在子进程 stdin / 读自子进程 stdout）：

```
→ {"jsonrpc":"2.0","id":1,"method":"initialize","params":{
     "protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"openharness","version":"x"}}}
← {"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05",
     "capabilities":{"tools":{"listChanged":false},"resources":{"subscribe":false}},
     "serverInfo":{"name":"filesystem","version":"0.1.0"}}}
→ {"jsonrpc":"2.0","method":"notifications/initialized"}        # 通知，无 id、无响应
→ {"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}     # 工具发现
← {"jsonrpc":"2.0","id":2,"result":{"tools":[{"name":"read_file","description":"...",
     "inputSchema":{"type":"object","properties":{...},"required":[...]}}]}}
→ {"jsonrpc":"2.0","id":3,"method":"resources/list","params":{}}  # 资源发现
← {"jsonrpc":"2.0","id":3,"result":{"resources":[]}}
→ {"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"read_file",
     "arguments":{"path":"/tmp/x"}}}                              # 工具调用
← {"jsonrpc":"2.0","id":4,"result":{"content":[{"type":"text","text":"..."}]}}
```

#### 2.4.4 HTTP 传输：Streamable HTTP（`_connect_http`，`client.py:218`）

```python
async def _connect_http(self, name, config):     # client.py:218
    stack = AsyncExitStack()
    try:
        http_client = await stack.enter_async_context(
            httpx.AsyncClient(headers=config.headers or None)   # :221-223
        )
        read_stream, write_stream, _get_session_id = await stack.enter_async_context(
            streamable_http_client(config.url, http_client=http_client)  # :224-226
        )
        await self._register_connected_session(name=name, config=config, stack=stack,
            read_stream=read_stream, write_stream=write_stream,
            auth_configured=bool(config.headers))               # :227-234
    except asyncio.CancelledError as exc:
        ...   # :235-242 同 stdio 的失败兜底
    except Exception as exc:
        ...   # :243-250
```

- 自建 `httpx.AsyncClient` 并注入 `config.headers`（**Bearer 鉴权头在这里生效**）；
- `streamable_http_client(config.url, http_client=...)` 是官方 Streamable HTTP 传输实现——它把 `httpx` 客户端包装成
  与 stdio 相同的 `(read_stream, write_stream)` 流接口，使上层 `_register_connected_session` **对两种传输完全透明**；
  第三元组元素 `_get_session_id` 用于拿会话 id（此处未使用）。
- `auth_configured=bool(config.headers)` 会在 UI 上提示"该 server 需要配置鉴权"。

#### 2.4.5 失败兜底与状态登记（`_close_failed_stack` / `_mark_connection_failed`）

- **`_close_failed_stack(stack)`**（`client.py:78`）：best-effort 关闭一个**从未完成**的连接尝试留下的半成品资源栈。
  `except BaseException` 捕获一切异常，但**只放行 `KeyboardInterrupt/SystemExit` 重新 raise**（`client.py:82-84`），
  其余异常全部吞掉——绝不因为清理动作打断应用启动。
- **`_mark_connection_failed(name, config, *, auth_configured, exc)`**（`client.py:86`）：把该 server 状态写成
  `state="failed"`、`detail=str(exc) or exc.__class__.__name__`（`client.py:95-101`）。
  **单个 server 连不上只影响它自己，不拖垮整个 `connect_all()` 启动流程。**

#### 2.4.6 握手与能力发现 `_register_connected_session`（`client.py:252`）

这是把"裸连接"升级为"可用会话"的关键步骤，两种传输共用：

```python
async def _register_connected_session(*, name, config, stack, read_stream, write_stream, auth_configured):  # :252
    session = await stack.enter_async_context(ClientSession(read_stream, write_stream))  # :262
    await session.initialize()                       # :263  协议握手（JSON-RPC initialize + notifications/initialized）
    tool_result = await session.list_tools()         # :264  工具发现
    resource_result = None
    try:
        resource_result = await session.list_resources()   # :267
    except Exception as exc:
        if "Method not found" not in str(exc):      # :269  资源发现失败时仅容忍"方法未实现"
            raise
    tools = [McpToolInfo(server_name=name, name=tool.name, description=tool.description or "",
        input_schema=dict(tool.inputSchema or {"type": "object", "properties": {}})) for tool in tool_result.tools]  # :271-279
    resources = [McpResourceInfo(server_name=name, name=resource.name or str(resource.uri),
        uri=str(resource.uri), description=resource.description or "")
        for resource in (resource_result.resources if resource_result is not None else [])]  # :280-288
    self._sessions[name] = session                   # :289
    self._stacks[name] = stack                       # :290
    self._statuses[name] = McpConnectionStatus(name=name, state="connected", ... tools=tools, resources=resources)  # :291-298
```

流程要点：

1. **建立会话**：`stack.enter_async_context(ClientSession(read_stream, write_stream))`（`client.py:262`）——进入上下文即初始化会话对象，
   随后 `await session.initialize()`（`client.py:263`）完成协议握手；
2. **工具发现**：`session.list_tools()`；`inputSchema` 缺失时兜底为 `{"type":"object","properties":{}}`（`client.py:276`）；
3. **资源发现**：`session.list_resources()`；若 server 不支持该方法（异常消息含 `Method not found`）则容忍为空，**不视为失败**（`client.py:266-270`）；
   其他异常才 re-raise；
4. **登记**：`_sessions`、`_stacks` 落键，状态更新为 `connected` 并附带工具/资源快照。

> **关键设计**：工具/资源发现是**一次性握手时完成**并缓存进 `McpConnectionStatus`。之后引擎注册工具、
> UI 展示状态、`list_tools()`/`list_resources()` 聚合，读的都是这份缓存快照，不再实时查询 server。

#### 2.4.7 工具/资源调用与结果字符串化

**`call_tool(server_name, tool_name, arguments)`**（`client.py:129`）：

1. `session = self._sessions.get(server_name)`，取不到 → 抛 `McpServerNotConnectedError`，detail 带上当前状态
   （`client.py:131-137`）；
2. `await session.call_tool(tool_name, arguments)`（`client.py:139`）；**任何异常**都被包装为
   `McpServerNotConnectedError("... call failed: ...")`（`client.py:140-143`）——因为"会话丢失"和"调用失败"在引擎看来都是"连不上"；
3. **结果字符串化**（`client.py:144-154`）：
   - 遍历 `result.content`，`type == "text"` 的元素取 `.text`；
   - 其余块（如 `image`）用 `item.model_dump_json()` 序列化；
   - 若没有文本块但有 `result.structuredContent`，用其字符串（`client.py:150-151`）；
   - 全空则 `"(no output)"`（`client.py:152-153`）；
   - 最终 `"\n".join(parts).strip()` 拼成**单条字符串**返回。

**`read_resource(server_name, uri)`**（`client.py:156`）：

1. 同样的会话校验与异常包装（`client.py:158-170`）；
2. 对 `result.contents` 逐条取 `text`（非 None 才追加）否则取 `blob` 的字符串（`client.py:171-177`）；
3. `"\n".join(parts).strip()`（`client.py:178`）。

> 两者都返回**纯字符串**，这是为了让引擎/模型拿到可直接嵌入上下文的扁平文本，而不是强类型对象。

#### 2.4.8 自动重连与关闭

- **`reconnect_all()`**（`client.py:61`）：`await self.close()` → 把所有状态重置回 `pending`（`client.py:64-67`）→ `await self.connect_all()`。
  这是"会话丢失后整体重连"的统一入口，`McpAuthTool` 在改完鉴权配置后也会调用它（见 §2.6）。
- **`close()`**（`client.py:103`）：遍历 `_stacks` 逐个 `stack.aclose()`（`AsyncExitStack` 按后进先出逆序关闭
  HTTP client → 传输流 → ClientSession），`RuntimeError` / `asyncio.CancelledError` 用 `contextlib.suppress` 吞掉（`client.py:106`）；
  最后清空两本字典（`client.py:108-109`）。

#### 2.4.9 查询与状态 API

| 方法 | 行号 | 说明 |
| --- | --- | --- |
| `update_server_config(name, config)` | `client.py:70` | 内存中替换单个 server 配置（供 `/mcp auth` 等命令在重连前更新鉴权） |
| `get_server_config(name)` | `client.py:74` | 读回单个配置（`None` 表示不存在） |
| `list_statuses()` | `client.py:111` | 按名字排序返回全部 `McpConnectionStatus`；UI 侧 `mcp_connected/mcp_failed` 统计（`ui/runtime.py:350-351`）与协议预览的数据源 |
| `list_tools()` | `client.py:115` | **聚合所有已连接 server 的 `McpToolInfo`**（遍历 status 快照），供工具注册用 |
| `list_resources()` | `client.py:122` | 聚合所有已连接 server 的 `McpResourceInfo`，供资源工具用 |

> 注意 `list_tools()`/`list_resources()` 返回的是**握手时缓存的快照**（来自 `_statuses`），不发起网络调用。

### 2.5 把 MCP 工具封装为引擎工具（`tools/mcp_tool.py`）

MCP 与引擎之间通过 `McpToolAdapter`（`mcp_tool.py:14`）桥接，它是 `BaseTool` 的子类（`tools/base.py`，`input_model` 声明于 `base.py:40`）。

#### 2.5.1 命名与清理

```python
def __init__(self, manager, tool_info):            # mcp_tool.py:17
    self._manager = manager
    self._tool_info = tool_info
    server_segment = _sanitize_tool_segment(tool_info.server_name)   # :20
    tool_segment   = _sanitize_tool_segment(tool_info.name)          # :21
    self.name = f"mcp__{server_segment}__{tool_segment}"             # :22
    self.description = tool_info.description or f"MCP tool {tool_info.name}"  # :23
    self.input_model = _input_model_from_schema(self.name, tool_info.input_schema)  # :24
```

- **工具名**：`mcp__<server>__<tool>` 三段式。引擎侧对 `mcp__` 前缀还有额外约定（如 `services/tool_outputs.py:53` 对输出做特殊处理）。
- **`_sanitize_tool_segment(value)`**（`mcp_tool.py:66`）：
  1. 用 `re.sub(r"[^A-Za-z0-9_-]", "_", value)` 把非法字符替换为下划线（`:67`）；
  2. 空结果 → 兜底 `"tool"`（`:68-69`）；
  3. 首字符非字母 → 加 `mcp_` 前缀（`:70-71`）。
  这样 server 名/工具名里带空格、连字符、特殊字符都能变成合法的工具名片段。

#### 2.5.2 动态 input_model：JSON Schema → pydantic

```python
_JSON_TYPE_MAP = {                                  # mcp_tool.py:39
    "string": str, "integer": int, "number": float,
    "boolean": bool, "array": list, "object": dict,
}

def _input_model_from_schema(tool_name, schema):    # mcp_tool.py:49
    properties = schema.get("properties", {})       # :50
    if not isinstance(properties, dict):            # :51
        return create_model(f"{tool_name.title()}Input")   # :52  无 properties → 空模型
    fields = {}
    required = set(schema.get("required", [])) if isinstance(schema.get("required", []), list) else set()  # :55
    for key in properties:
        prop = properties[key] if isinstance(properties[key], dict) else {}  # :57
        py_type = _JSON_TYPE_MAP.get(str(prop.get("type", "")), object)      # :58
        if key in required:
            fields[key] = (py_type, Field(default=...))        # :59-60  必填 → Field(default=...)
        else:
            fields[key] = (py_type | None, Field(default=None))  # :61-62  可选 → Optional + None 默认
    return create_model(f"{tool_name.title().replace('-', '_')}Input", **fields)  # :63
```

要点：

- **类型映射**：JSON Schema `type` → Python 类型，见下表；未知/缺失 → `object`（兼容最宽）；

  | JSON Schema 类型 | Python 类型 |
  | --- | --- |
  | `string` | `str` |
  | `integer` | `int` |
  | `number` | `float` |
  | `boolean` | `bool` |
  | `array` | `list` |
  | `object` | `dict` |
  | 其他/缺失 | `object` |

- **必填字段**（出现在 `schema["required"]`）用 `Field(default=...)`（Pydantic Required）强制存在；
- **可选字段**用 `py_type | None, Field(default=None)`；
- 最终 `create_model` 动态生成一个名为 `<工具名>Input` 的 pydantic 模型。引擎在渲染工具签名时调用
  `input_model.model_json_schema()`（`tools/base.py:56`）把它重新 JSON 化给模型看——**强类型在引擎内闭环**。

#### 2.5.3 执行

```python
async def execute(self, arguments, context):        # mcp_tool.py:26
    del context
    try:
        output = await self._manager.call_tool(
            self._tool_info.server_name,
            self._tool_info.name,
            arguments.model_dump(mode="json", exclude_none=True),   # :29-33
        )
    except McpServerNotConnectedError as exc:
        return ToolResult(output=str(exc), is_error=True)           # :34-35
    return ToolResult(output=output)                                # :36
```

- 参数用 `model_dump(mode="json", exclude_none=True)` 转成**纯 JSON 字典**（`exclude_none` 丢弃未填的可选字段）；
- `McpServerNotConnectedError` → `ToolResult(is_error=True)`，引擎会把这个错误反馈给模型，而不是抛异常中断回合。

### 2.6 MCP 工具/资源/提示三类工件（Tool / Resource / Prompt）在引擎中的封状

MCP 规范定义了三类服务器能力：**工具（tools）**、**资源（resources）**、**提示（prompts）**。本仓库的处理情况：

| MCP 工件 | 是否支持 | 封装方式 |
| --- | --- | --- |
| **工具** | ✅ 完整 | `McpToolAdapter`（`mcp_tool.py:14`）逐工具注册为 `mcp__server__tool` |
| **资源** | ✅ 完整 | 两个通用资源工具：`ReadMcpResourceTool`、`ListMcpResourcesTool` |
| **提示（prompts）** | ❌ **未实现** | 仓库中没有任何 `list_prompts` / `get_prompt` / `McpPrompt` 的引用（Grep 已核实），`_register_connected_session` 也只调用 `list_tools` + `list_resources`。若需要提示模板能力，只能由各 server 以工具形式暴露，或等待后续版本 |

#### 2.6.1 资源类工具

**`ReadMcpResourceTool`**（`tools/read_mcp_resource_tool.py:18`）：

- 输入模型 `ReadMcpResourceToolInput`（`:11`）：`server: str` + `uri: str`（各带 description）；
- `name = "read_mcp_resource"`、`description = "Read an MCP resource by server and URI."`（`:21-22`）；
- `is_read_only()` 恒返回 `True`（`:28-30`），让引擎可以安全预执行；
- `execute()`（`:32-38`）：`manager.read_resource(arguments.server, arguments.uri)`，同样把 `McpServerNotConnectedError` 转为 `is_error`。

**`ListMcpResourcesTool`**（`tools/list_mcp_resources_tool.py:15`）：

- 空输入模型 `ListMcpResourcesToolInput`（`:11`，`pass` 占位）；
- `name = "list_mcp_resources"`、`description = "List MCP resources available from connected servers."`（`:18-19`）；
- `is_read_only()` 恒 `True`（`:25-27`）；
- `execute()`（`:29-36`）：`manager.list_resources()`，无资源返回 `"(no MCP resources)"`，否则逐行输出
  `"{server_name}:{uri} {description}"`。

#### 2.6.2 统一注册入口 `create_default_tool_registry`（`tools/__init__.py:48`）

```python
if mcp_manager is not None:                          # tools/__init__.py:93
    registry.register(ListMcpResourcesTool(mcp_manager))   # :94
    registry.register(ReadMcpResourceTool(mcp_manager))    # :95
    for tool_info in mcp_manager.list_tools():             # :96
        registry.register(McpToolAdapter(mcp_manager, tool_info))  # :97
```

构建默认工具注册表时，除 40 余个内建工具（`BashTool`、`FileReadTool`、`AgentTool` 等，`:51-92`）外，
只要传入 `mcp_manager` 就追加：两个资源工具 + 每个已连接 MCP server 的每个工具一个 `McpToolAdapter`。
（注：`McpAuthTool` 作为内建工具在 `:59` 注册，见下。）

#### 2.6.3 MCP 鉴权工具 `McpAuthTool`（`tools/mcp_auth_tool.py`）

`McpAuthTool`（`mcp_auth_tool.py:21`，工具名 `mcp_auth`）允许模型在会话内修改 MCP 鉴权并热重连：

- 输入 `McpAuthToolInput`（`:12`）：`server_name`、`mode`（`bearer`/`header`/`env`）、`value`、可选 `key`；
- `execute()`（`:28`）流程：
  1. `load_settings()` 拿到配置，从 `settings.mcp_servers` 或 `mcp_manager.get_server_config` 取 server（`:29-37`）；
  2. **stdio**：`mode` ∈ `{env, bearer}`，默认键 `MCP_AUTH_TOKEN`；`bearer` 模式把值写成 `Bearer <value>`（`:39-45`）；
  3. **http/ws**：`mode` ∈ `{header, bearer}`，默认键 `Authorization`；仅当 `key=="Authorization"` 时才拼 `Bearer ` 前缀（`:46-54`）；
  4. `settings.mcp_servers[name] = updated; save_settings(settings)`（`:58-59`）；
  5. 若引擎注入了 `mcp_manager`（来自 `context.metadata["mcp_manager"]`，`:30`）：
     `update_server_config(...)` + `await mcp_manager.reconnect_all()` 热重连；失败则返回 `is_error`（`:61-69`）。

### 2.7 CLI 安装 / 连接流程

MCP 的"安装（配置）"与"连接"被拆成两段：

**配置阶段 —— `mcp` 子命令组（`cli.py:786-838`）：**

| 命令 | 行号 | 行为 |
| --- | --- | --- |
| `mcp list` | `cli.py:786` | 合并 settings + 插件配置后打印每个 server 的 `transport`/`command` |
| `mcp add <name> <config_json>` | `cli.py:804` | 把一段 JSON 写入 `settings.mcp_servers[name]` 并保存 |
| `mcp remove <name>` | `cli.py:825` | 从 `settings.mcp_servers` 删除 |

**鉴权阶段 —— `/mcp auth` 命令（`commands/registry.py:1397-1445`）：** 按 server 类型更新鉴权：

- HTTP/WS server：写 `headers`，`bearer` 模式把值格式化为 `Bearer <token>` 放进 `Authorization` 头（`registry.py:1423-1432`）；
- stdio server：写 `env`，默认键 `MCP_AUTH_TOKEN`（`registry.py:1433-1440`）；
- 保存后提示 "Restart session to reconnect"——因为配置只在连接时读取。

**连接阶段 —— 运行时引导（`ui/runtime.py:322-324`）：**

```python
mcp_manager = McpClientManager(load_mcp_server_configs(settings, plugins))
await mcp_manager.connect_all()
tool_registry = create_default_tool_registry(mcp_manager)
```

即：合并配置 → 连接所有 server → 用连接结果构建含 MCP 工具的默认工具注册表。
连接状态随后暴露给 UI（`backend_host.py:432`、`ui/protocol.py:169-185`），`mcp_connected`/`mcp_failed` 统计来自 `list_statuses()`。

---

## 三、Bridge（`src/openharness/bridge/`）

### 3.1 什么是 Bridge

Bridge 的职责集中在**会话孵化与凭证工具**上，用于把"另一个 CLI 会话"以受管子进程的形式纳入 OpenHarness 的观察范围，
并为"SDK 接入"提供握手凭证。它并不承载消息路由（那是 Channels 的 `ChannelBridge` 干的），而是面向 **UI 状态展示**
（`bridge_sessions`）与 **`/bridge` 命令族**的服务。

### 3.2 类型定义（`types.py`）

```python
DEFAULT_SESSION_TIMEOUT_MS = 24 * 60 * 60 * 1000     # types.py:9  默认会话超时 24h

@dataclass(frozen=True)
class WorkData:                                       # types.py:12
    type: Literal["session", "healthcheck"]           # 工作项类型
    id: str                                           # 工作项 ID

@dataclass(frozen=True)
class WorkSecret:                                     # types.py:20
    version: int                                      # 协议版本（当前恒为 1）
    session_ingress_token: str                        # 会话接入令牌
    api_base_url: str                                 # 服务地址（用于构造接入 URL）

@dataclass(frozen=True)
class BridgeConfig:                                   # types.py:29
    dir: str                                          # 工作目录
    machine_name: str                                 # 机器名
    max_sessions: int = 1                             # 最大并发会话数
    verbose: bool = False
    session_timeout_ms: int = DEFAULT_SESSION_TIMEOUT_MS
```

- `WorkData`：work item 元数据（`session`/`healthcheck` 二选一）。
- `WorkSecret`：解码后的会话接入凭证。
- `BridgeConfig`：最小桥接配置，默认 `max_sessions=1`、会话超时 24h。

### 3.3 work_secret（`work_secret.py`）—— 工作密钥编解码

`work_secret.py` 提供三个纯函数：

**`encode_work_secret(secret)`**（`work_secret.py:11`）：

```python
data = json.dumps(secret.__dict__, separators=(",", ":"))       # 压缩 JSON（去空白）
return base64.urlsafe_b64encode(data.encode("utf-8")).decode("utf-8").rstrip("=")
```

- 把 `WorkSecret` 的 `__dict__` 压成紧凑 JSON → **base64url** 编码 → **去掉 `=` 填充**，得到不透明短字符串。

**`decode_work_secret(secret)`**（`work_secret.py:17`）：

1. 按 `-len % 4` 补 `=` 填充（`:19`）→ base64url 解码 → JSON 解析（`:20-21`）；
2. **校验**：
   - `version` 必须为 `1`，否则 `ValueError("Unsupported work secret version")`（`:22-23`）；
   - 必须存在非空 `session_ingress_token`（`:24-25`）；
   - `api_base_url` 必须是 `str`（`:26-27`）；
3. 通过后构造 `WorkSecret`（`:28-32`）。

**`build_sdk_url(api_base_url, session_id)`**（`work_secret.py:35`）—— 按环境推断接入协议与路径版本：

```python
is_local = "localhost" in api_base_url or "127.0.0.1" in api_base_url
protocol = "ws" if is_local else "wss"
version  = "v2" if is_local else "v1"
host = api_base_url.replace("https://", "").replace("http://", "").rstrip("/")
return f"{protocol}://{host}/{version}/session_ingress/ws/{session_id}"
```

- **本地地址**（含 `localhost` / `127.0.0.1`）→ `ws://` + `v2` 路径；**否则** → `wss://` + `v1` 路径；
- 拼出形如 `wss://host/v2/session_ingress/ws/{session_id}` 的 WebSocket 接入 URL。

### 3.4 session_runner（`session_runner.py`）—— 会话孵化

**`SessionHandle`**（`session_runner.py:13`，普通 dataclass）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `session_id` | `str` | 会话 ID |
| `process` | `asyncio.subprocess.Process` | 子进程句柄 |
| `cwd` | `Path` | 工作目录（已 resolve） |
| `started_at` | `float` | 启动时刻（`field(default_factory=time.time)`，`:20`） |

**`kill()`**（`session_runner.py:22`）：先 `process.terminate()`，用 `asyncio.wait_for(self.process.wait(), timeout=3)` 等 3 秒；
超时（`TimeoutError`）则 `process.kill()` 强杀并 `await self.process.wait()`（`:24-29`）。

**`spawn_session(*, session_id, command, cwd)`**（`session_runner.py:32`）：

```python
resolved_cwd = Path(cwd).resolve()                  # :39
process = await create_shell_subprocess(            # :40-45
    command,
    cwd=resolved_cwd,
    stdout=asyncio.subprocess.PIPE,
    stderr=asyncio.subprocess.STDOUT,               # stderr 合并进 stdout，便于统一捕获
)
return SessionHandle(session_id=session_id, process=process, cwd=resolved_cwd)  # :46
```

- 通过 `openharness.utils.shell.create_shell_subprocess`（`utils/shell.py:51`）启动——它会按平台挑选 shell
  （Windows 下 `bash -lc` → `pwsh` → `cmd /c`，Unix 下 `bash -lc`，`utils/shell.py:17-48`），
  并在启用 docker sandbox 时改为走 `get_docker_sandbox().exec_command`（`utils/shell.py:66-83`）；
- `stdout=PIPE` + `stderr=STDOUT` 把**标准错误合并进标准输出**，为后续 `_copy_output` 的统一落盘做准备。

### 3.5 manager（`manager.py`）—— 会话跟踪

**`BridgeSessionRecord`**（`manager.py:13`，frozen dataclass）：UI 友好的会话快照：
`session_id`、`command`、`cwd`、`pid`、`status`、`started_at`、`output_path`。

**`BridgeSessionManager`**（`manager.py:26`）：

- 构造时初始化四本字典（`manager.py:29-33`）：
  - `_sessions: dict[str, SessionHandle]`；
  - `_commands: dict[str, str]`；
  - `_output_paths: dict[str, Path]`；
  - `_copy_tasks: dict[str, asyncio.Task[None]]`（后台输出搬运任务）。

**`spawn(*, session_id, command, cwd)`**（`manager.py:35`）：

```python
handle = await spawn_session(session_id=session_id, command=command, cwd=cwd)  # :36
self._sessions[session_id] = handle                # :37
self._commands[session_id] = command               # :38
output_dir = get_data_dir() / "bridge"             # :39
output_dir.mkdir(parents=True, exist_ok=True)      # :40
output_path = output_dir / f"{session_id}.log"     # :41
output_path.write_text("", encoding="utf-8")       # :42
self._output_paths[session_id] = output_path       # :43
self._copy_tasks[session_id] = asyncio.create_task(self._copy_output(session_id, handle))  # :44
return handle
```

- 孵化子会话后，在 `{data_dir}/bridge/{session_id}.log` 创建日志文件，并起后台任务把 stdout **实时追写**进去。
- `get_data_dir()` 来自 `config/paths.py:37`（`OPENHARNESS_DATA_DIR` 环境变量优先，否则 `~/.openharness/data`）。

**`_copy_output(session_id, handle)`**（`manager.py:85`）—— 输出搬运循环：

```python
if handle.process.stdout is not None:
    while True:
        chunk = await handle.process.stdout.read(4096)   # 按 4096 字节块读取
        if not chunk:
            break
        with path.open("ab") as stream:
            stream.write(chunk)
await handle.process.wait()                             # 循环结束后等待进程退出
```

**`list_sessions()`**（`manager.py:47`）：遍历 `_sessions`，按 `process.returncode` 推导状态：
`None → "running"`、`0 → "completed"`、其他 → `"failed"`（`manager.py:51-56`）；组装 `BridgeSessionRecord`
并**按 `started_at` 倒序**返回（`manager.py:68`）。

**`read_output(session_id, *, max_bytes=12000)`**（`manager.py:70`）：读日志尾部；超过 `max_bytes` 只返回末尾片段（`:75-76`）。

**`stop(session_id)`**（`manager.py:79`）：查不到会话抛 `ValueError`；否则 `await handle.kill()`。

**`get_bridge_manager()`**（`manager.py:100`）：模块级**惰性单例**——`_DEFAULT_MANAGER` 全局缓存，首次调用才构造（`:97-105`）。

### 3.6 下游使用

- **`/bridge` 命令族**（`commands/registry.py:1046-1098`）直接消费上述 API：`show/list/output/stop` 走 manager，`encode/decode/sdk` 走 work_secret。
- **UI 侧**（`ui/runtime.py:331`、`backend_host.py:433`）用 `get_bridge_manager().list_sessions()` 填充 `bridge_sessions` 状态。
- 顶层导出在 `bridge/__init__.py`（20 行）：把 4 个模块的 10 个符号直接 re-export。

---

## 四、Channels 渠道系统（`src/openharness/channels/`）

### 4.1 架构总览

```
IM 平台事件
   │  （各 Channel.start() 的 long-running 监听）
   ▼
*Channel（适配器）─────►  BaseChannel._handle_message()   ← 权限校验 is_allowed()
   │                           │
   │                           ▼
   │                   InboundMessage
   │                           │
   │                           ▼  bus.publish_inbound()
   │                   MessageBus.inbound 队列
   │                           │
   │                           ▼  bus.consume_inbound()
   │                   ChannelBridge._loop()
   │                           │  engine.submit_message(content)
   │                           ▼  （收集 AssistantTextDelta）
   │                   OutboundMessage
   │                           │
   │                           ▼  bus.publish_outbound()
   │                   MessageBus.outbound 队列
   │                           │
   │                           ▼  bus.consume_outbound()
   │                   ChannelManager._dispatch_outbound()  ──► channel.send(msg)
   └─────────────────────────────────────────────────────────────► IM 平台
```

四个核心构件：

| 构件 | 文件:行号 | 职责 |
| --- | --- | --- |
| 事件模型 | `bus/events.py:8` | `InboundMessage` / `OutboundMessage` 两个 dataclass |
| 消息总线 | `bus/queue.py:8` | 双向 `asyncio.Queue`，解耦渠道与引擎 |
| 适配基类 | `impl/base.py:34` | `BaseChannel`：抽象接口 + 权限 + 媒体目录 + 统一入站入口 |
| 引擎桥 | `adapter.py:29` | `ChannelBridge`：把 inbound 消息喂给 QueryEngine 并回收回复 |

调度层 `impl/manager.py:17` 的 `ChannelManager` 负责按配置初始化/启停渠道、跑 outbound 分发循环。
渠道名是注册表键（`"telegram"`、`"feishu"`、`"matrix"` …），与 `InboundMessage.channel` 对应。

### 4.2 统一消息模型（`bus/events.py`）

**`InboundMessage`**（`events.py:8`）—— 渠道收到的消息，字段：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `channel` | `str` | — | 渠道名（telegram/discord/slack/whatsapp/...） |
| `sender_id` | `str` | — | 发送者标识 |
| `chat_id` | `str` | — | 会话/聊天标识 |
| `content` | `str` | — | 消息文本 |
| `timestamp` | `datetime` | `field(default_factory=datetime.now)`（`:16`） | 接收时刻 |
| `media` | `list[str]` | `[]`（`:17`） | 媒体本地文件路径列表 |
| `metadata` | `dict[str, Any]` | `{}`（`:18`） | 渠道特有数据 |
| `session_key_override` | `str \| None` | `None`（`:19`） | **线程作用域会话**的可选覆盖（如飞书 thread、Matrix thread） |

关键属性 **`session_key`**（`events.py:21`）：

```python
@property
def session_key(self) -> str:
    return self.session_key_override or f"{self.channel}:{self.chat_id}"   # events.py:24
```

- 取 `session_key_override`，否则退化为 `"{channel}:{chat_id}"`——这是**会话识别的唯一键**，引擎据此决定沿用哪段对话上下文。
- `ChannelBridge` 会把 `msg.session_key` 放进出站消息的 `metadata["_session_key"]`，供上层路由（router）用。

**`OutboundMessage`**（`events.py:27`）—— 引擎要发出去的消息，字段：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `channel` | `str` | — | 目标渠道名 |
| `chat_id` | `str` | — | 目标会话 |
| `content` | `str` | — | 回复文本 |
| `reply_to` | `str \| None` | `None`（`:34`） | 要引用的消息 ID |
| `media` | `list[str]` | `[]`（`:35`） | 待发送媒体路径列表 |
| `metadata` | `dict[str, Any]` | `{}`（`:36`） | 渠道特有数据（如 `_session_key`、`_progress`、`message_id`） |

### 4.3 消息总线（`bus/queue.py`）

`MessageBus`（`queue.py:8`）内部是两个无界 `asyncio.Queue`：

```python
def __init__(self):                                   # queue.py:16
    self.inbound: asyncio.Queue[InboundMessage] = asyncio.Queue()   # :17 渠道→引擎
    self.outbound: asyncio.Queue[OutboundMessage] = asyncio.Queue() # :18 引擎→渠道
```

四个发布/消费方法与两个只读积压观测属性：

| 方法 | 行号 | 说明 |
| --- | --- | --- |
| `publish_inbound(msg)` | `queue.py:20` | `await self.inbound.put(msg)` |
| `consume_inbound()` | `queue.py:24` | `await self.inbound.get()`（阻塞直到有货） |
| `publish_outbound(msg)` | `queue.py:28` | `await self.outbound.put(msg)` |
| `consume_outbound()` | `queue.py:32` | `await self.outbound.get()` |
| `inbound_size`（属性） | `queue.py:36` | `self.inbound.qsize()` |
| `outbound_size`（属性） | `queue.py:41` | `self.outbound.qsize()` |

顶层导出在 `channels/__init__.py`（22 行）：`BaseChannel`、`ChannelManager`、`InboundMessage`、`MessageBus`、`OutboundMessage`。

### 4.4 适配基类（`impl/base.py`）

**`resolve_channel_media_dir(channel_name)`**（`base.py:16`）—— 解析渠道媒体下载目录：

```python
custom_root = os.environ.get("OPENHARNESS_CHANNEL_MEDIA_DIR")   # :18 优先级 1
if custom_root:
    root = Path(custom_root).expanduser().resolve()
else:
    ohmo_workspace = os.environ.get("OHMO_WORKSPACE")            # :22 优先级 2
    if ohmo_workspace:
        from ohmo.workspace import get_attachments_dir
        root = get_attachments_dir(ohmo_workspace)
    else:
        root = get_data_dir() / "media"                          # :28 优先级 3（默认）
media_dir = root / channel_name                                  # :29
media_dir.mkdir(parents=True, exist_ok=True)                     # :30
return media_dir
```

优先级：环境变量 `OPENHARNESS_CHANNEL_MEDIA_DIR` → `OHMO_WORKSPACE` 下的 attachments 目录 → `{data_dir}/media`。
最终目录是 `root / channel_name` 并按渠道名隔离，自动 `mkdir`。

**`BaseChannel`**（`base.py:34`，`ABC`）：

- 类属性 `name: str = "base"`（`:42`）；
- `__init__(self, config, bus)`（`:44`）：保存 `config`、`bus`，置 `_running=False`（`:52-54`）；
- 三个**抽象方法**：
  - `async start()`（`base.py:57`）：连接平台并进入长驻监听；
  - `async stop()`（`base.py:69`）：清理资源；
  - `async send(msg: OutboundMessage)`（`base.py:74`）：发消息。

**`is_allowed(sender_id)`**（`base.py:83`）—— 鉴权逻辑：

```python
allow_list = getattr(self.config, "allow_from", [])   # :85
if not allow_list:                                    # 空列表 → 全部拒绝 + 告警
    logger.warning("%s: allow_from is empty — all access denied", self.name)   # :87
    return False
if "*" in allow_list:                                 # "*" → 放行全部
    return True
sender_str = str(sender_id)
return sender_str in allow_list or any(p in allow_list for p in sender_str.split("|") if p)   # :92-93
```

- 空列表 → 默认拒绝一切远程访问（安全默认，配合 `manager._validate_allow_from` 的告警）；
- 含 `"*"` → 全部放行；
- 否则精确匹配，或把 `sender_id` 按 `|` 拆分逐段匹配（兼容 Telegram 的 `"<id>|<username>"` 复合格式）。

**`_handle_message(sender_id, chat_id, content, media=None, metadata=None, session_key=None)`**（`base.py:96`）—— **所有渠道入站的统一出口**：

```python
if not self.is_allowed(sender_id):                    # :118 权限前置
    logger.warning("Access denied for sender %s on channel %s. Add them to allowFrom list ...", ...)  # :119-124
    return
msg = InboundMessage(
    channel=self.name, sender_id=str(sender_id), chat_id=str(chat_id),
    content=content, media=media or [], metadata=metadata or {},
    session_key_override=session_key,                # :126-134
)
await self.bus.publish_inbound(msg)                  # :136
```

渠道实现只需收集 `sender_id / chat_id / content / media / metadata / session_key` 后调用它——鉴权、消息构造、总线发布都集中在这一处。

`is_running` 属性（`base.py:138`）返回 `_running`。

### 4.5 引擎桥（`adapter.py`）—— ChannelBridge

`ChannelBridge`（`adapter.py:29`）**每个 QueryEngine 一个实例**，把消息总线接到引擎。构造时接收 `engine` 与 `bus`（`:36-40`）。

**控制 API**：

| 方法 | 行号 | 说明 |
| --- | --- | --- |
| `start()` | `adapter.py:46` | 幂等；`asyncio.create_task(self._loop(), name="channel-bridge")` 起后台循环（`:51`） |
| `stop()` | `adapter.py:54` | 置 `_running=False` 并取消任务（`:57-63`） |
| `run()` | `adapter.py:66` | 内联运行（阻塞直到取消），finally 里复位 `_running` |

**主循环 `_loop()`**（`adapter.py:78`）：

```python
while self._running:
    try:
        msg = await asyncio.wait_for(self._bus.consume_inbound(), timeout=1.0)  # :82-85
        await self._handle(msg)                              # :86
    except asyncio.TimeoutError:
        continue                                             # :87-88  超时继续轮询
    except asyncio.CancelledError:
        break                                                # :89-90  取消退出
    except Exception:
        logger.exception("ChannelBridge: unhandled error processing message")   # :91-92
```

- `wait_for(..., timeout=1.0)` 让循环在无消息时也能**每秒醒来一次**检查 `_running`，实现优雅退出。

**单消息处理 `_handle(msg)`**（`adapter.py:94`）：

```python
reply_parts = []
try:
    async for event in self._engine.submit_message(msg.content):   # :100
        if isinstance(event, AssistantTextDelta):       # :101
            reply_parts.append(event.text)             # :102 累积文本增量
        elif isinstance(event, AssistantTurnComplete): # :103
            pass                                       # :105 回合结束，文本已由 delta 累积
except Exception:
    logger.exception(...)
    reply_parts = ["[Error: failed to process your message]"]   # :112
reply_text = "".join(reply_parts).strip()              # :114
if not reply_text:
    logger.debug("ChannelBridge: empty reply, skipping publish")   # :115-117
    return
outbound = OutboundMessage(
    channel=msg.channel, chat_id=msg.chat_id, content=reply_text,
    metadata={"_session_key": msg.session_key},         # :119-124
)
await self._bus.publish_outbound(outbound)              # :125
```

要点：

- 引擎事件类型：`AssistantTextDelta`（`engine/stream_events.py:13`）、`AssistantTurnComplete`（`:20`）；
- `metadata["_session_key"] = msg.session_key` 把会话键一路带给出站消息，保证回复路由回同一个会话；
- 引擎抛异常 → 占位文案 `[Error: failed to process your message]`，不让用户无反馈。

### 4.6 渠道注册与调度（`impl/manager.py`）—— ChannelManager

`ChannelManager`（`manager.py:17`）：

**`__init__(config, bus)`**（`manager.py:27`）：保存配置与总线，`self.channels: dict[str, BaseChannel] = {}`，置 `_dispatch_task=None`，调用 `_init_channels()`（`:30-33`）。

**`_init_channels()`**（`manager.py:35`）—— 按配置逐个实例化渠道：

| 渠道 | 配置开关（`config.py` 内联判断） | 类 | 实例化参数 |
| --- | --- | --- | --- |
| telegram | `config.channels.telegram.enabled`（`:39`） | `TelegramChannel`（`:42`） | `config, bus, groq_api_key=config.providers.groq.api_key`（`:44-45`） |
| whatsapp | `config.channels.whatsapp.enabled`（`:52`） | `WhatsAppChannel`（`:55`） | `config, bus` |
| discord | `config.channels.discord.enabled`（`:63`） | `DiscordChannel`（`:66`） | `config, bus` |
| feishu | `config.channels.feishu.enabled`（`:74`） | `FeishuChannel`（`:77`） | `config, bus` |
| mochat | `config.channels.mochat.enabled`（`:85`） | `MochatChannel`（`:89`） | `config, bus` |
| dingtalk | `config.channels.dingtalk.enabled`（`:97`） | `DingTalkChannel`（`:100`） | `config, bus` |
| email | `config.channels.email.enabled`（`:108`） | `EmailChannel`（`:111`） | `config, bus` |
| slack | `config.channels.slack.enabled`（`:119`） | `SlackChannel`（`:122`） | `config, bus` |
| qq | `config.channels.qq.enabled`（`:130`） | `QQChannel`（`:133`） | `config, bus` |
| matrix | `config.channels.matrix.enabled`（`:142`） | `MatrixChannel`（`:145`） | `config, bus` |

每个渠道的 import 都包在 `try/except ImportError` 里——**依赖 SDK 未装时只 `logger.warning` 不崩溃**（如 telegram `:48-49`）。
最后调用 `_validate_allow_from()`（`:153`）。

**`_validate_allow_from()`**（`manager.py:155`）：对 `allow_from == []` 的渠道告警——远程访问默认被拒，除非显式加名单或 `["*"]`（`:156-161`）。

**`_start_channel(name, channel)`**（`manager.py:163`）：`channel.start()` 异常时把 `str(e)` 写到 `channel.last_error` 并记日志（`:165-169`）。

**`start_all()`**（`manager.py:171`）：

```python
if not self.channels:
    logger.warning("No channels enabled"); return            # :173-175
self._dispatch_task = asyncio.create_task(self._dispatch_outbound())  # :178 先启动出站分发循环
tasks = [asyncio.create_task(self._start_channel(name, channel)) for name, channel in self.channels.items()]  # :181-184
await asyncio.gather(*tasks, return_exceptions=True)         # :187 等待所有渠道启动（理论上常驻）
```

**`stop_all()`**（`manager.py:189`）：取消分发任务（`:194-199`），逐个 `channel.stop()`（`try/except` 记错不中断，`:201-207`）。

**`_dispatch_outbound()`**（`manager.py:209`）—— 出站循环：

```python
while True:
    try:
        msg = await asyncio.wait_for(self.bus.consume_outbound(), timeout=1.0)   # :215-218
        if msg.metadata.get("_progress"):                       # :220 进度/工具提示过滤
            if msg.metadata.get("_tool_hint") and not self.config.channels.send_tool_hints:  # :221-222
                continue
            if not msg.metadata.get("_tool_hint") and not self.config.channels.send_progress:  # :223-224
                continue
        channel = self.channels.get(msg.channel)                # :226
        if channel:
            try:
                await channel.send(msg)                         # :229
            except Exception as e:
                logger.error("Error sending to %s: %s", msg.channel, e)  # :230-231
        else:
            logger.warning("Unknown channel: %s", msg.channel)  # :233
    except asyncio.TimeoutError:
        continue
    except asyncio.CancelledError:
        break
```

- **进度/工具提示过滤**：`metadata["_progress"]` 为真的消息，只有 `channels.send_progress` / `channels.send_tool_hints`
  打开时才发送（这两个开关在 `config/schema.py:102-104` 的 `ChannelConfigs` 上，默认都 `True`）；
- 按 `msg.channel` 查 `self.channels` 找到渠道就 `await channel.send(msg)`，找不到记 warning。

**查询接口**：`get_channel(name)`（`:240`）、`get_status()`（`:244`，返回 `{name: {"enabled": True, "running": is_running}}`）、
`enabled_channels` 属性（`:254`）。

### 4.7 渠道逐个精读

> Feishu / Telegram / Matrix 详讲；其余七渠道简讲但保留具体函数名与行号。
> 所有渠道最终都调用 `BaseChannel._handle_message()`（`base.py:96`）完成入站（含鉴权与总线发布），并在 `send()` 中消费 `OutboundMessage`。

#### 4.7.1 Feishu 飞书（`impl/feishu.py`，1342 行）—— 详讲

**SDK 与连接方式**：`lark-oapi`，**WebSocket 长连接**（无需公网 IP / 无需 webhook）。`FeishuChannel`（`feishu.py:411`，`name="feishu"`）。
`FEISHU_AVAILABLE = importlib.util.find_spec("lark_oapi") is not None`（`feishu.py:25`）做依赖探测。

**启动 `start()`**（`feishu.py:455`）：

1. 检查 SDK 与 `app_id/app_secret`（`:457-463`）；`self._running = True`、记录当前 loop（`:466-467`）；
2. `_ensure_rest_client()`（`feishu.py:435`）：构建 `lark.Client.builder().app_id(...).app_secret(...).domain(...).build()` REST 客户端
   （发消息/下载/上传全用它）（`:447-452`）；
3. 用 `lark.EventDispatcherHandler.builder(encrypt_key, verification_token)` 只注册 **`im.message.receive_v1`** 处理器 `_on_message_sync`
   （`:473-478`）——只收消息事件，其它事件忽略；
4. 创建 `lark.ws.Client(app_id, app_secret, event_handler=event_handler, domain=...)` WebSocket 客户端（`:481-487`）；
5. **关键技巧 `run_ws()`**（`:494-513`）：在**独立线程 + 独立事件循环**里跑 WS：
   - `ws_loop = asyncio.new_event_loop()` + `asyncio.set_event_loop(ws_loop)`（`:497-498`）；
   - 打补丁 `_lark_ws_client.loop = ws_loop`（`:500`）——规避 lark 模块级 `asyncio.get_event_loop()` 拿到正在运行的 main loop
     而报 "This event loop is already running"；
   - 外层 `while self._running:` + `try: self._ws_client.start()` + `except` 记 warning + `time.sleep(5)` 实现**自动重连**（`:502-508`）；
6. 主协程 `while self._running: await asyncio.sleep(1)` 保持运行（`:519-520`）。

**stop()**（`feishu.py:522`）：`lark.ws.Client` 不暴露 stop 方法，仅置 `_running=False`，靠主协程退出与进程收尾关闭连接（`:530-531`）。

**接收消息**：WS 线程回调 `_on_message_sync`（`feishu.py:1205`）用 `asyncio.run_coroutine_threadsafe(self._on_message(data), self._loop)`
把事件调度回主事件循环（`:1210-1211`）。

`_on_message`（`feishu.py:1213`）的完整流程：

1. **去重**：`_processed_message_ids`（`OrderedDict`，容量 1000，超限 `popitem(last=False)`）（`:1220-1228`）；
2. **跳过机器人消息**：`sender.sender_type == "bot"`（`:1231-1232`）；
3. `sender_id = sender.sender_id.open_id`，并用 `_resolve_sender_display_name_sync`（`feishu.py:1167`，REST 查询联系人
   `contact.v3.user.get`，`name → en_name → nickname → open_id` 兜底，带 512 条 LRU 缓存 `_sender_cache`）解析显示名（`:1234-1240`）；
4. **按 `msg_type` 解析 content**（`:1245-1287`）：
   - `text`：取 `content_json["text"]`；
   - `post`：`_extract_post_content`（`feishu.py:344`）支持 **Direct / Localized / Wrapped** 三种结构，并下载内嵌图片（`:1259-1270`）；
   - `image/audio/file/media`：`_download_and_save_media`（`feishu.py:901`）经线程池 `_download_image_sync`/`_download_file_sync`
     （`feishu.py:846/869`，走 `im.v1.message_resource.get`）下载到 `resolve_channel_media_dir`，文件名经 `safe_filename` 且
     `file_path.is_relative_to(media_root)` **防路径穿越**（`:937-951`）；
   - 分享卡片/互动消息：`_extract_share_card_content`（`feishu.py:210`）递归 `_extract_interactive_content`（`:230`）/
     `_extract_element_content`（`:271`）抽取文本与链接（`markdown/lark_md/div/a/button/img/note/column_set/plain_text` 等 tag）；
5. **群组策略**（`:1291-1312`）：
   - `_extract_feishu_mentions`（`feishu.py:74`）归一化 mentions（支持 dict/list/递归 walk `tag=="at"`，按 key/name/open_id/user_id/union_id 五元组去重）；
   - `_feishu_mentions_bot`（`feishu.py:126`）判断是否 @ 了机器人：按 `bot_open_id` 或 `bot_names`（`_configured_bot_names` `feishu.py:52`，默认含 `"ohmo"`）匹配，文本正则兜底（`:144-147`）；
   - `_should_process_feishu_group_message`（`feishu.py:185`）+ `_normalize_feishu_group_policy`（`:151`，支持 `all/always/managed_mention/at` 等别名）实现策略：
     `open`（全回）/ `mention`（仅 @ 机器人）/ `managed`（仅 ohmo 托管群，`_is_ohmo_managed_feishu_group` `feishu.py:168`，查 `ohmo.group_registry.load_managed_group_record`）/ `managed_or_mention`（默认）；
6. 命中策略后 `_add_reaction`（`feishu.py:554`，线程池里跑 `_add_reaction_sync`，`CreateMessageReactionRequest` + `react_emoji` 表情回应，默认 `THUMBSUP`）（`:1314-1315`）；
7. 最后 `_handle_message`（`feishu.py:1323`）：
   - `reply_to = chat_id(群) or sender_id(私聊)`（`:1318`）；
   - metadata 携带 `message_id/thread_id/root_id/chat_type/msg_type/mentions/mentions_bot/sender_display_name/sender_label`（`:1328-1338`）；
   - `thread_id = message.thread_id or message.root_id`（`:1321`）——**线程作用域会话路由**（`session_key_override`）的来源。

**发送 `send(msg)`**（`feishu.py:1064`）：

1. `receive_id_type = "chat_id" if msg.chat_id.startswith("oc_") else "open_id"`（`:1071`）；
   群/线程消息用原 `message_id` 走 `ReplyMessageRequest` 回线程（`_send_message_sync`，`feishu.py:955`，`reply_in_thread=True`）；
2. **媒体发送**（`:1080-1130`）：图片（`_IMAGE_EXTS`，`feishu.py:786`）上传 `CreateImageRequest` 得 `image_key` 发 `"image"`；
   音频/视频（`_AUDIO_EXTS`/`_VIDEO_EXTS`，`:787-788`）上传得 `file_key` 发 `"media"`，其余发 `"file"`（`:1102-1121`）；
   失败媒体汇总成一条中文失败提示文本 `文件发送失败：{names}`（`:1123-1130`）；
3. **文本走格式自动检测 `_detect_msg_format`**（`feishu.py:704`，classmethod）：
   - `"text"`：≤200 字符、无任何 markdown（`_TEXT_MAX_LEN=200` `:699`）→ 纯文本；
   - `"post"`：有链接（`_MD_LINK_RE` `:690`）、中等长度 ≤2000（`_POST_MAX_LEN` `:702`）→ `_markdown_to_post`（`feishu.py:742`）
     转富文本（`[text](url)` → `a` 标签，每行一段，输出 `{"zh_cn":{"content":[[...]]}}` JSON）；
   - `"interactive"`：代码块/表格/标题（`_COMPLEX_MD_RE` `:673`）/粗斜体（`_SIMPLE_MD_RE` `:681`）/列表（`_LIST_RE`/`_OLIST_RE` `:693/696`）/超长
     → `_build_card_elements`（`feishu.py:595`）把 markdown 拆成卡片元素：表格经 `_parse_md_table`（`:576`）转 Feishu `table` 元素，
     标题经 `_split_headings`（`:637`）转 `div`（`**标题**` 加粗）；
   - **每卡限一张表**（API 错误 11310）：`_split_elements_by_table_limit`（`feishu.py:609`）把多表内容拆成多张卡片分别发送（`:1156-1162`）。

**其他能力**：`create_managed_group`/`rename_group`（`feishu.py:1036/1059`）供 ohmo 托管群流程调用（线程池执行 `_create_managed_group_sync`
`feishu.py:1007`、`_rename_group_sync` `:1041`，走 `im.v1.chat.create/update`）。

#### 4.7.2 Telegram（`impl/telegram.py`，525 行）—— 详讲

**SDK 与连接方式**：`python-telegram-bot`，**长轮询（long polling）**（无需 webhook/公网 IP）。`TelegramChannel`（`telegram.py:95`，`name="telegram"`）。
构造时额外接收 `groq_api_key`（`telegram.py:112-120`）用于语音转写。
命令注册表 `BOT_COMMANDS`（`telegram.py:105`）：`/start`、`/new`、`/stop`、`/help`。

**启动 `start()`**（`telegram.py:129`）：

1. 安全检查 `token`（`:131-133`）；`silence_telegram_token_url_loggers()`（`telegram.py:25`，`:138`）把 `httpx/httpcore/telegram.ext`
   日志级别提到 WARNING，防 bot token 泄漏进 INFO 日志；
2. `HTTPXRequest(connection_pool_size=16, pool_timeout=5.0, connect_timeout=30.0, read_timeout=30.0)` 大连接池（`:141`）；
   `Application.builder().token(...).request(req).get_updates_request(req)`，支持代理 `config.proxy`（`:143-144`）；
3. 注册 `/start` `/new` `/help` 命令处理器（`_on_start`/`_forward_command`/`_on_help`，`:149-151`）与通用 `MessageHandler`
   （`TEXT | PHOTO | VOICE | AUDIO | Document.ALL` 且 **排除 `COMMAND`**，`:154-160`）；`add_error_handler(_on_error)`（`:146`）；
4. `initialize()` → `start()` → `get_me()` 记录 bot 用户名 → `set_my_commands(BOT_COMMANDS)`（`:165-176`）；
5. `updater.start_polling(allowed_updates=["message"], drop_pending_updates=False)`（`:179-182`）；
6. 主协程 `while self._running: await asyncio.sleep(1)`（`:187-188`）。

**接收消息 `_on_message(update, context)`**（`telegram.py:352`）：

1. `sender_id = _sender_id(user)`（`telegram.py:336`）拼成 `"<id>|<username>"` 复合格式——方便 `is_allowed` 的 `|` 拆分匹配；
   缓存 `self._chat_ids[sender_id] = chat_id`（`:363`）；
2. 收集 `text` + `caption`（`:370-373`）；媒体按 `photo(取最大 `photo[-1]`)/voice/audio/document` 优先（`:379-390`），
   经 `bot.get_file(file_id)` 下载到 `resolve_channel_media_dir(self.name)`，文件名 `file_id[:16] + 扩展名`（`:393-403`）；
   语音/音频用 `GroqTranscriptionProvider(api_key=self.groq_api_key).transcribe(file_path)` 转写并附 `[transcription: ...]`（`:408-416`）；
3. **媒体组（media group）聚合**（`:432-451`）：有 `media_group_id` 时先缓冲进 `_media_group_buffers`（以 `f"{chat_id}:{media_group_id}"` 为键），
   由 `_flush_media_group`（`telegram.py:471`，`await asyncio.sleep(0.6)` 延迟 0.6s）合并成**一次 turn** 发总线（`:474-482`）；
4. `_start_typing(chat_id)`（`telegram.py:486`）起 "typing..." 指示循环（`_typing_loop` `:498`，每 4s `send_chat_action(typing)`），
   然后 `_handle_message` 发布入站（`:457-469`），metadata 带 `message_id/user_id/username/first_name/is_group`。

**发送 `send(msg)`**（`telegram.py:223`）：

- 非进度消息先 `_stop_typing(msg.chat_id)`（`:230-231`）；
- `config.reply_to_message` 且 metadata 有 `message_id` 时构造 `ReplyParameters(message_id=..., allow_sending_without_reply=True)`（`:239-246`）；
- 媒体按扩展名 `_get_media_type`（`telegram.py:211`）选 `send_photo/send_voice/send_audio` 或 `send_document`，失败发 `[Failed to send: 文件名]`（`:249-271`）；
- 文本经 `split_message(msg.content, TELEGRAM_MAX_MESSAGE_LEN)`（`utils/helpers.py:28`，`TELEGRAM_MAX_MESSAGE_LEN=4000` `telegram.py:21`）分片；
  `_markdown_to_telegram_html`（`telegram.py:32`）把 markdown 转为 Telegram 安全的 HTML：
  1. 抽出代码块/行内代码占位（`:40-53`）→ 2. 标题去 `#`（`:56`）→ 3. 引用去 `>`（`:59`）→ 4. 转义 HTML 特殊字符（`:62`）→
  5. `[text](url)`→`<a href>`（`:65`）→ 6. `**`/`__`→`<b>`（`:68-69`）→ 7. `_text_`→`<i>`（避免匹配变量名，`:72`）→
  8. `~~text~~`→`<s>`（`:75`）→ 9. 列表项→`• `（`:78`）→ 10. 还原 `code`/`pre><code`（`:81-90`）；
- 进度消息（`_progress`）用 `send_message_draft`（`telegram.py:282`）流式更新草稿；HTML 解析失败回退纯文本（`:296-310`）。

#### 4.7.3 Matrix（`impl/matrix.py`，700 行）—— 详讲

**SDK 与连接方式**：`matrix-nio`（`nio`），**长轮询 sync（`sync_forever`）**。`MatrixChannel`（`matrix.py:146`，`name="matrix"`）。
依赖导入失败时抛 `ImportError("Matrix dependencies not installed. Run: pip install nanobot-ai[matrix]")`（`matrix.py:33-36`）。

**启动 `start()`**（`matrix.py:162`）：

1. `_configure_nio_logging_bridge()`（`matrix.py:138`）把 `nio` 的 stdlib 日志桥接进 Loguru（`_NioLoguruHandler`，`matrix.py:124`）；
2. `AsyncClient(homeserver, user_id, store_path={data_dir}/matrix-store, config=AsyncClientConfig(store_sync_tokens=True, encryption_enabled=config.e2ee_enabled))`
   并手动设置 `user_id/access_token/device_id`（`matrix.py:170-177`）；
3. `_register_event_callbacks()`（`matrix.py:383`）：`RoomMessageText` → `_on_message`、`RoomMessageMedia|RoomEncryptedMedia` → `_on_media_message`、`InviteEvent` → `_on_room_invite`；
4. `_register_response_callbacks()`（`matrix.py:388`）：`SyncError/JoinError/RoomSendError` → `_log_response_error`（`matrix.py:393`，
   认证类错误 `M_UNKNOWN_TOKEN/M_FORBIDDEN/M_UNAUTHORIZED` 或 `soft_logout` 记 ERROR，其余 WARNING）；
5. 有 `device_id` 时 `load_store()` 恢复同步令牌（`:185-189`）；`_sync_task = create_task(self._sync_loop())`（`:193`），
   `_sync_loop`（`matrix.py:448`）循环 `sync_forever(timeout=30000, full_state=True)`，异常 sleep 2s 重试（`:451-455`）。

**入站过滤 `_should_process_message(room, event)`**（`matrix.py:478`）：

- `is_allowed(event.sender)` 不过则返回 False（`:480`）；
- 直聊（`_is_direct_room` `matrix.py:461`：`member_count <= 2`）直接处理；
- 群组按 `group_policy`：`open` 全处理 / `allowlist`（`room.room_id in config.group_allow_from`）/ `mention`（`_is_bot_mentioned` `matrix.py:465`，
  检查 `content.m.mentions.user_ids` 含 bot 的 `user_id`，或 `allow_room_mentions` 且 `mentions.room is True`）；
- 受邀房间 `_on_room_invite`（`matrix.py:457`）：`is_allowed(event.sender)` 时自动 `client.join(room.room_id)`。

**文本消息 `_on_message`**（`matrix.py:663`）：

- `event.sender == config.user_id`（自己的消息）或不过滤则跳过（`:664`）；
- 起 typing keepalive（`_start_typing_keepalive` `matrix.py:421`：先清旧，`_set_typing(True)`，每 `TYPING_KEEPALIVE_INTERVAL_MS=20_000` 刷新一次，
  超时 `TYPING_NOTICE_TIMEOUT_MS=30_000` 常量 `matrix.py:45-47`）；
- metadata 经 `_base_metadata`（`matrix.py:654`）携带 `room`（display_name）+ `event_id` + 线程信息
  （`_thread_metadata` `matrix.py:513`，解析 `content.m.relates_to` 得 `thread_root_event_id`/`thread_reply_to_event_id`）；
- 最后 `_handle_message`（`:668-671`），异常时清 typing 并 re-raise。

**媒体消息 `_on_media_message`**（`matrix.py:676`）→ `_fetch_media_attachment(room, event)`（`matrix.py:609`）：

1. `_event_attachment_type`（`:534`）按 `_MSGTYPE_MAP`（`matrix.py:54`，`m.image/m.audio/m.video/m.file`）取类型；
   `_event_mime`（`:549`）读 `info.mimetype`；`_event_filename`（`:556`）取 body → `safe_filename`；
2. 大小检查：`_effective_media_limit_bytes()`（`matrix.py:292`）取 `min(本地 config.max_media_bytes, homeserver 上报的上传上限)`；
   homeserver 上限经 `_resolve_server_upload_limit_bytes`（`matrix.py:275`，查 `content_repository_config`，每生命周期只查一次）；
3. `_download_media_bytes`（`matrix.py:576`）：`client.download(mxc)`，`DownloadError` 记 warning；支持 bytes/bytearray/MemoryDownloadResponse/本地 Path；
4. E2EE 附件 `_decrypt_media_bytes`（`matrix.py:597`）用 `nio.crypto.attachments.decrypt_attachment` 解密（key/hashes/iv）；
5. 落盘路径 `{data_dir}/media/matrix/{event_id_prefix}_{stem}{suffix}`（`_build_attachment_path` `matrix.py:563`，event_id 截 24 字符、文件名 stem 截 72 字符防超长）；
6. 返回 `attachment` 字典（type/mime/filename/event_id/encrypted/size_bytes/path/mxc_url）+ `[attachment: path]` 标记文本，`media` 列表带本地路径（`:646-652`）。

**发送 `send(msg)`**（`matrix.py:352`）：

- `_collect_outbound_media_candidates`（`matrix.py:225`）去重解析媒体路径；
  `_upload_and_send_attachment`（`matrix.py:300`）逐个处理：大小检查（超限返回 `[attachment: x - too large]`）→
  `client.upload(f, content_type=mime, filename=..., encrypt=e2ee and encrypted_room, filesize=size)`（`:323-328`）→
  `_build_outbound_attachment_content`（`matrix.py:242`，`msgtype` 按 mime 前缀映射 m.image/m.audio/m.video/m.file，E2EE 时把加密信息写进 `content["file"]`，否则 `url`）；
- 文本 `_build_matrix_text_content`（`matrix.py:115`）构造 `m.text`：`_render_markdown_html`（`matrix.py:99`，mistune `create_markdown(escape=True, plugins=["table","strikethrough","url","superscript","subscript"])`）
  若渲染出"非纯文本"的 HTML（非单纯 `<p>纯文本</p>`）才附加 `format=org.matrix.custom.html` 与 `formatted_body`；
  HTML 经 `nh3.Cleaner` 白名单清洗（`MATRIX_ALLOWED_HTML_TAGS` `:64`、`MATRIX_ALLOWED_HTML_ATTRIBUTES` `:70`、
  `_filter_matrix_html_attribute` `:77` 只放行 https/http/matrix:/mailto:/mxc: scheme，`MATRIX_HTML_CLEANER` `:89`）；
- 线程回复：`_build_thread_relates_to`（`matrix.py:521`）从 metadata 构造 `m.relates_to`（`rel_type: m.thread` + `m.in_reply_to` + `is_falling_back`）（`:344-345, 376-377`）；
- 非进度消息在 finally 中 `_stop_typing_keepalive(msg.chat_id, clear_typing=True)`（`:379-381`）。

**stop()**（`matrix.py:195`）：清 typing → `client.stop_sync_forever()` → `wait_for(shield(_sync_task), timeout=sync_stop_grace_seconds)` 超时再取消 → `client.close()`（`:196-213`）。

#### 4.7.4 其余渠道简讲

| 渠道 | 类（文件:行号） | SDK / 连接方式 | 鉴权 | 接收 → 处理要点 | 发送要点 |
| --- | --- | --- | --- | --- | --- |
| **Slack** | `SlackChannel`（`slack.py:21`） | `slack_sdk.socket_mode` WebSocket（Socket Mode，需 `mode=="socket"`，`slack.py:38-40`） | `bot_token` + `app_token`；`auth_test()` 解析 `_bot_user_id`（`slack.py:44-58`） | `_on_socket_request`（`slack.py:108`）：只处理 `events_api` 的 `message/app_mention`；跳过 subtype/bot 消息避免双份（`:134-143`）；`_is_allowed`（`:206`，DM 走 `dm.enabled/policy`，群走 `group_policy`）与 `_should_respond_in_channel`（`:219`，`open`/`mention`/`allowlist`）过滤；`_strip_bot_mention`（`:230`）去 @；加 `react_emoji` 回应（`:172-180`）；保留 `thread_ts` 给回复路由 | `send()`（`slack.py:76`）：群/频道内回线程（`use_thread = thread_ts and channel_type != "im"`，`:83-87`），`_to_mrkdwn`（`:242`）用 `slackify_markdown` + 表格转列表（`_convert_table` `:270`）+ `_fixup_mrkdwn`（`:250`）修残余 markdown 记号；媒体走 `files_upload_v2`（`:98`） |
| **Discord** | `DiscordChannel`（`discord.py:24`） | 手写 Gateway WebSocket（`websockets`，`_gateway_loop` `discord.py:127`） | Bot token 放 IDENTIFY（op 2，intents）与 REST `Authorization: Bot <token>` 头 | `_gateway_loop`（`:127`）：op10 HELLO → 心跳（`_start_heartbeat` `:188`）+ `_identify`（`:169`）；op0 `MESSAGE_CREATE` → `_handle_message_create`（`:205`）：跳过 bot、`is_allowed`、群按 `_should_respond_in_group`（`:267`，open/mention 检查 `<@id>` 与 mentions 数组）；附件经 `DISCORD_API_BASE` 下载（≤20MB `MAX_ATTACHMENT_BYTES` `:20`，`:231-249`）；起 typing（`_start_typing` `:288`，每 8s）；`reply_to` 取 `referenced_message.id` | `send()`（`discord.py:78`）：REST POST `/channels/{id}/messages`，`split_message` 2000 字符分片（`MAX_MESSAGE_LEN` `:21`，首片带 `message_reference`，`:95-98`），429 限流重试 3 次（`_send_payload` `:105`） |
| **DingTalk** | `DingTalkChannel`（`dingtalk.py:93`）+ `NanobotDingTalkHandler`（`dingtalk.py:42`） | `dingtalk-stream` SDK WebSocket 长连接（Stream 模式，仅收），发送走 HTTP API | `client_id` + `client_secret`（`Credential`）；发送 token 经 `_get_access_token`（`dingtalk.py:176`）OAuth2 缓存（提前 60s 过期） | `start()`（`dingtalk.py:122`）：注册 `ChatbotMessage.TOPIC` 处理器；启动自带 5s 重连循环（`:152-159`）；SDK 回调 `process()`（`:52`）用 `ChatbotMessage.from_dict` 解析，`create_task(channel._on_message(...))` 非阻塞转发（`:79-83`）；`_on_message`（`:427`）里 `chat_id == sender_id`（仅支持私聊，群消息回复仍发回私聊，`:100-102`） | `send()`（`dingtalk.py:405`）：文本 `_send_markdown_text`（`:338`）走 `sampleMarkdown`；媒体先 `_guess_upload_type`（`:207`）/`_read_media_bytes`（`:221`）/`_upload_media`（`:263`）再按 image（`sampleImageMsg` + `photoURL`，支持 URL 直发）或 file（`sampleFile`）发，失败发可见占位（`:414-425`） |
| **MoChat** | `MochatChannel`（`mochat.py:217`） | `python-socketio` WebSocket（msgpack/JSON 序列化）+ HTTP 轮询降级 | `claw_token`（auth 传 socket `auth={"token": ...}` 或 HTTP 头 `X-Claw-Token`） | `start()`（`mochat.py:253`）：`_start_socket_client`（`:347`）连 WS；订阅 `claw.session.events`/`claw.panel.events`（`:387-393`）+ `notify:chat.*`（`:395-398`）；`_process_inbound_event`（`:665`）做 filter/去重（`_remember_message_id` `:713`，2000 上限）/mention 判定（`resolve_was_mentioned` `:158`）/群需 mention（`resolve_require_mention` `:175`）；面板非 mention 可延迟聚合（`reply_delay_mode`，`_enqueue_delayed_entry` `:724`）；session 游标持久化（`_save_session_cursors` `:849`，0.5s 防抖）；WS 断线自动切 `_ensure_fallback_workers`（`:569`）用 `/api/claw/sessions/watch` 长轮询 + panel 轮询兜底 | `send()`（`mochat.py:300`）：`resolve_mochat_target`（`:121`）解析 panel/session，走 `/api/claw/groups/panels/send` 或 `/api/claw/sessions/send`，`replyTo` 回引用（`_api_send` `:882`） |
| **Email** | `EmailChannel`（`email.py:27`） | IMAP 轮询 + SMTP 发送（`imaplib/smtplib`） | `imap_username/imap_password`（IMAP4_SSL 或 IMAP4）+ `smtp_username/smtp_password` | `start()`（`email.py:63`）：需 `consent_granted=true`；每 `poll_interval_seconds` 调 `_fetch_new_messages`（`:193`）搜索 `UNSEEN`；`_fetch_messages`（`:228`）解析 RFC822（`BytesParser(policy.default)`）、`_extract_text_body`（`:358`）多 part 取纯文本优先/HTML 转文本（`_html_to_text` `:398`）、`_decode_header_value`（`:349`）、UID 去重 + `mark_seen`；内容模板化（From/Subject/Date/body，`:284-290`）；`fetch_messages_between_dates`（`:202`）支持日期区间检索（历史摘要任务用） | `send()`（`email.py:107`）：`is_reply` 受 `auto_reply_enabled` 控制（`force_send` metadata 可绕过，`:123-129`）；`_reply_subject`（`:405`）加 `Re:` 前缀（可 `subject_prefix` 覆盖）；`In-Reply-To/References` 关联（`:144-147`）；SSL/TLS 由 `_smtp_send`（`:175`）区分 |
| **WhatsApp** | `WhatsAppChannel`（`whatsapp.py:17`） | 连接 **Node.js 桥接进程**（`@whiskeysockets/baileys`，WebSocket 通信） | `bridge_token`（连接后发 `{"type":"auth","token":...}`，`:50`）；二维码在桥端终端扫描 | `start()`（`whatsapp.py:34`）：`websockets.connect(bridge_url)`，断线 5s 重连（`:44-70`）；`_handle_bridge_message`（`:97`）处理 `message/status/qr/error` 类型；`message` 按 `message_id` 去重（`:116-121`），`chat_id` 用完整 LID（`:135`），`pn`/`sender` 提取发件人；语音暂不支持转写（`[Voice Message: Transcription not available for WhatsApp yet]`，`:128-131`） | `send()`（`whatsapp.py:81`）：发 `{"type":"send","to":chat_id,"text":content}` |
| **QQ** | `QQChannel`（`qq.py:51`） | `botpy` SDK WebSocket（`_make_bot_class` 动态子类化 `botpy.Client`，`qq.py:30`，`Intents(public_messages=True, direct_message=True)`） | `app_id` + `secret`（`client.start(appid, secret)`，`:84`） | 重连循环 5s（`_run_bot` `qq.py:80`）；`on_c2c_message_create`/`on_direct_message_create` 收 C2C 私聊/频道单聊（`qq.py:42-46`）；`_on_message`（`:119`）按 `data.id` 去重（deque 容量 1000，`:60`），`chat_id=user_id` | `send()`（`qq.py:101`）：`api.post_c2c_message(openid=chat_id, msg_type=0, content, msg_id, msg_seq)`，`_msg_seq` 自增避免 QQ API 按序列去重（`:61/108`） |

### 4.8 渠道配置模型（`config/schema.py`）

| 类 | 行号 | 特有字段（除公共 `enabled`/`allow_from` 外） |
| --- | --- | --- |
| `BaseChannelConfig` | `schema.py:26` | `enabled: bool = False`（`:27`）、`allow_from: list[str] = []`（`:31`） |
| `TelegramConfig` | `:34` | `token`、`chat_id`、`proxy`、`reply_to_message`、`bot_name` |
| `SlackConfig` | `:42` | `bot_token`、`app_token`、`signing_secret` |
| `DiscordConfig` | `:48` | `token` |
| `FeishuConfig` | `:52` | `app_id`、`app_secret`、`encrypt_key`、`verification_token`、`group_policy`（默认 `managed_or_mention`）、`bot_open_id`、`bot_names`（默认含 ohmo/openclaw/openharness）、`domain`（默认 open.feishu.cn） |
| `DingTalkConfig` | `:65` | `client_id`、`client_secret`、`robot_code` |
| `EmailConfig` | `:71` | `smtp_host`、`smtp_port`、`smtp_username`、`smtp_password`、`from_address` |
| `QQConfig` | `:79` | `token`、`app_id`、`app_secret` |
| `MatrixConfig` | `:85` | `homeserver`、`access_token`、`user_id` |
| `WhatsAppConfig` | `:91` | `access_token`、`phone_number_id`、`verify_token` |
| `MochatConfig` | `:97` | `endpoint`、`token` |
| `ChannelConfigs` | `:102` | `send_progress`（`:103`）、`send_tool_hints`（`:104`）+ 10 个渠道子配置（`:105-114`） |

---

## 五、消息流全景

以一条完整的 Feishu 私聊消息为例，串联三个子系统中的 channels 链路：

1. **入站**：lark WS 线程收到 `im.message.receive_v1` → `_on_message_sync`（`feishu.py:1205`）→ 调度进主 loop `_on_message`（`feishu.py:1213`）
   → 去重、解析文本/媒体、群策略校验 → `self._handle_message(...)`（`feishu.py:1323`）。
2. **鉴权 + 总线**：`BaseChannel._handle_message`（`base.py:96`）做 `is_allowed(sender_id)` 校验 → 构造 `InboundMessage(channel="feishu", sender_id=..., chat_id=..., session_key_override=...)`
   → `bus.publish_inbound(msg)`（`base.py:136`）。
3. **引擎桥**：`ChannelBridge._loop`（`adapter.py:78`）`consume_inbound()` → `_handle`（`adapter.py:94`）→ `engine.submit_message(content)`
   → 累积 `AssistantTextDelta`，`AssistantTurnComplete` 结束回合 → 拼 `OutboundMessage(channel, chat_id, content, metadata={"_session_key": ...})` → `bus.publish_outbound()`（`adapter.py:125`）。
4. **出站分发**：`ChannelManager._dispatch_outbound`（`manager.py:209`）`consume_outbound()` → 进度过滤（`manager.py:220-224`）→ `channel.send(msg)`。
5. **发出**：`FeishuChannel.send`（`feishu.py:1064`）格式检测 → text/post/interactive 卡片（多表拆多卡）→ `_send_message_sync` REST 发出。

```
IM ──► *Channel._on_*() ──► BaseChannel._handle_message() ──► bus.publish_inbound()
                                                               │
                                       MessageBus.inbound 队列 │
                                                               ▼
      ChannelBridge._handle() ◄── bus.consume_inbound()  ◄────┘
               │  engine.submit_message()（收集 AssistantTextDelta）
               ▼
      bus.publish_outbound() ──► MessageBus.outbound 队列 ──► ChannelManager._dispatch_outbound()
                                                                       │
                                                       channel.send(OutboundMessage) ──► IM 平台
```

---

## 六、关键类 / 函数清单

### 6.1 MCP

| 名字 | 文件:行号 | 职责 |
| --- | --- | --- |
| `McpServerNotConnectedError` | `mcp/client.py:25` | 会话缺失/丢失/调用失败的统一异常 |
| `McpClientManager` | `mcp/client.py:29` | MCP 连接管理核心 |
| `connect_all` | `mcp/client.py:45` | 连接全部 server（isinstance 分发） |
| `reconnect_all` | `mcp/client.py:61` | 全量关闭→重置 pending→重连 |
| `update_server_config` / `get_server_config` | `mcp/client.py:70` / `:74` | 配置内存读写（供鉴权热更新） |
| `_close_failed_stack` / `_mark_connection_failed` | `mcp/client.py:78` / `:86` | 失败清理与状态登记（单点失败不拖垮启动） |
| `close` | `mcp/client.py:103` | 逆序关闭全部资源栈 |
| `list_statuses` / `list_tools` / `list_resources` | `mcp/client.py:111` / `:115` / `:122` | 状态与工具/资源快照聚合 |
| `call_tool` | `mcp/client.py:129` | 调用 MCP 工具并字符串化结果 |
| `read_resource` | `mcp/client.py:156` | 读取 MCP 资源并字符串化 |
| `_connect_stdio` / `_connect_http` | `mcp/client.py:180` / `:218` | stdio 子进程 / Streamable HTTP 两种传输连接 |
| `_register_connected_session` | `mcp/client.py:252` | 握手 + 工具/资源发现 + 状态登记 |
| `McpStdioServerConfig` / `McpHttpServerConfig` / `McpWebSocketServerConfig` | `mcp/types.py:11` / `:21` / `:29` | 三种 server 配置模型 |
| `McpServerConfig` / `McpJsonConfig` | `mcp/types.py:37` / `:40` | 联合类型 / 配置文件形状 |
| `McpToolInfo` / `McpResourceInfo` | `mcp/types.py:46` / `:56` | 工具/资源元数据 |
| `McpConnectionStatus` | `mcp/types.py:66` | 运行状态快照 |
| `load_mcp_server_configs` | `mcp/config.py:8` | 合并 settings 与插件 MCP 配置（setdefault） |
| `McpToolAdapter` | `tools/mcp_tool.py:14` | 把 MCP 工具封装成引擎工具 |
| `_JSON_TYPE_MAP` / `_input_model_from_schema` | `tools/mcp_tool.py:39` / `:49` | JSON Schema → pydantic 输入模型 |
| `_sanitize_tool_segment` | `tools/mcp_tool.py:66` | 工具名片段清理（`mcp__server__tool`） |
| `ReadMcpResourceTool` / `ListMcpResourcesTool` | `tools/read_mcp_resource_tool.py:18` / `tools/list_mcp_resources_tool.py:15` | 资源读写工具 |
| `McpAuthTool` | `tools/mcp_auth_tool.py:21` | 会话内修改 MCP 鉴权并热重连 |
| `create_default_tool_registry` | `tools/__init__.py:48` | 注册内建 + MCP 工具（MCP 块 `:93-97`） |
| `mcp_list` / `mcp_add` / `mcp_remove` | `cli.py:786` / `:804` / `:825` | CLI 配置子命令 |
| `_mcp_handler` | `commands/registry.py:1397` | `/mcp auth` 鉴权命令 |

### 6.2 Bridge

| 名字 | 文件:行号 | 职责 |
| --- | --- | --- |
| `WorkSecret` / `WorkData` / `BridgeConfig` | `bridge/types.py:20` / `:12` / `:29` | 桥接配置与凭证模型 |
| `DEFAULT_SESSION_TIMEOUT_MS` | `bridge/types.py:9` | 默认会话超时 24h |
| `encode_work_secret` / `decode_work_secret` | `bridge/work_secret.py:11` / `:17` | work secret 编解码（base64url JSON，version 校验） |
| `build_sdk_url` | `bridge/work_secret.py:35` | 构造 session ingress WebSocket URL（本地 ws/v2，远端 wss/v1） |
| `SessionHandle` | `bridge/session_runner.py:13` | 子会话句柄（含 kill，3s 超时强杀） |
| `spawn_session` | `bridge/session_runner.py:32` | 孵化 shell 子会话（stderr 并入 stdout） |
| `BridgeSessionRecord` | `bridge/manager.py:13` | UI 会话快照 |
| `BridgeSessionManager` | `bridge/manager.py:26` | 会话跟踪 + 输出落盘 |
| `spawn` / `list_sessions` / `read_output` / `stop` / `_copy_output` | `bridge/manager.py:35` / `:47` / `:70` / `:79` / `:85` | 会话生命周期与日志读取 |
| `get_bridge_manager` | `bridge/manager.py:100` | 单例访问 |
| `_bridge_handler` | `commands/registry.py:1046` | `/bridge` 命令族 |

### 6.3 Channels

| 名字 | 文件:行号 | 职责 |
| --- | --- | --- |
| `InboundMessage` / `OutboundMessage` | `channels/bus/events.py:8` / `:27` | 总线消息模型（`session_key` 属性 `:21`） |
| `MessageBus` | `channels/bus/queue.py:8` | 双向异步队列 + 积压观测 |
| `resolve_channel_media_dir` | `channels/impl/base.py:16` | 渠道媒体目录解析 |
| `BaseChannel` | `channels/impl/base.py:34` | 渠道适配基类（start/stop/send/is_allowed/_handle_message） |
| `is_allowed` | `channels/impl/base.py:83` | 白名单鉴权（空拒、`*` 放行、`|` 拆分） |
| `_handle_message` | `channels/impl/base.py:96` | 所有渠道统一入站出口 |
| `ChannelBridge` | `channels/adapter.py:29` | 引擎桥（inbound→引擎→outbound） |
| `_loop` / `_handle` | `channels/adapter.py:78` / `:94` | 轮询消费 / 单消息处理 |
| `ChannelManager` | `channels/impl/manager.py:17` | 渠道初始化/启停/出站分发 |
| `_init_channels` | `channels/impl/manager.py:35` | 按配置注册 10 个渠道 |
| `_validate_allow_from` | `channels/impl/manager.py:155` | 空 allow_from 告警 |
| `start_all` / `stop_all` | `channels/impl/manager.py:171` / `:189` | 启停全渠道 |
| `_dispatch_outbound` | `channels/impl/manager.py:209` | 出站消息调度循环（进度过滤） |
| `FeishuChannel` | `channels/impl/feishu.py:411` | 飞书渠道（start `:455`，send `:1064`，on_message `:1213`） |
| `TelegramChannel` | `channels/impl/telegram.py:95` | Telegram 渠道（start `:129`，send `:223`，on_message `:352`） |
| `MatrixChannel` | `channels/impl/matrix.py:146` | Matrix 渠道（start `:162`，send `:352`，on_media `:676`） |
| `SlackChannel` / `DiscordChannel` / `DingTalkChannel` | `slack.py:21` / `discord.py:24` / `dingtalk.py:93` | Slack / Discord / DingTalk 渠道 |
| `MochatChannel` / `EmailChannel` / `WhatsAppChannel` / `QQChannel` | `mochat.py:217` / `email.py:27` / `whatsapp.py:17` / `qq.py:51` | 其余渠道 |
| `BaseChannelConfig` | `config/schema.py:26` | 渠道公共配置（`enabled`、`allow_from`） |
| `ChannelConfigs` | `config/schema.py:102` | 渠道配置容器（`send_progress`、`send_tool_hints`） |

---

## 七、设计要点小结

1. **解耦**：渠道与引擎之间零直接依赖，仅通过 `MessageBus` 的两个队列通信；`ChannelBridge` 是唯一的"翻译官"。
2. **统一入站出口**：所有渠道最终都走 `BaseChannel._handle_message()`（`base.py:96`），鉴权、消息构造、总线发布集中在此，避免各渠道各自为政。
3. **安全默认**：`allow_from` 空列表默认拒绝一切远程访问（`base.py:83-94`、`manager.py:155-161`），需要显式加名单或 `["*"]`。
4. **会话亲和**：`InboundMessage.session_key`（`events.py:21`）把"渠道:会话"映射为会话键，`thread_id`/`root_id`/`_session_key` 等 metadata 支撑线程作用域会话。
5. **失败隔离**：MCP 单 server 失败不阻断启动（`_mark_connection_failed`，`client.py:86`）；渠道 ImportError/start 失败只告警（`manager.py:48-151`、`:163-169`）。
6. **类型安全桥接**：MCP 的 JSON Schema 通过 `_JSON_TYPE_MAP` + `create_model`（`mcp_tool.py:39-63`）转成 pydantic 模型，让引擎拿到结构化的强类型工具签名。
7. **无公网优先**：Feishu（WS 长连接）、Telegram（长轮询）、Slack（Socket Mode）、DingTalk（Stream）、Matrix（长轮询）都不依赖公网 IP/webhook；Discord/QQ 用网关 WebSocket；WhatsApp 依赖本地 Node.js 桥。
8. **MCP 只做"工具+资源"**：协议层的 prompts（提示模板）在本构建中未实现，仅有 `list_tools` + `list_resources` 两类发现（`client.py:264-288`）。
9. **惰性加载**：`mcp/__init__.py` 用模块级 `__getattr__` 延迟导入官方 SDK；渠道 SDK 在 `_init_channels` 里按配置惰性 import，缺失只告警。
10. **桥梁会话可观测**：`BridgeSessionManager._copy_output`（`manager.py:85`）把子进程输出实时落盘到 `{data_dir}/bridge/{session_id}.log`，`read_output` 提供尾部截断读取，UI 与 `/bridge` 命令共享同一快照。

---

## 八、实际示例（Example Walkthroughs）

> 本章把前文机制串成**一条条可运行的完整链路**。每个示例均为真实代码行为的示意——
> 所有字段值、函数名、`:行号` 均与源码逐一核对；仅对话内容、URL、ID 等为演示用途。

### 8.1 MCP stdio 连接完整对话

> 示例：MCP stdio 连接完整对话（帧结构来源 `mcp/client.py:180` `_connect_stdio`、`client.py:252` `_register_connected_session`、`client.py:129` `call_tool`；framing 为 NDJSON 单行 JSON-RPC 2.0，逐行写子进程 stdin / 读自 stdout；内容为示意）

假设配置了一个 stdio server（`npx` 拉起的 `docs` 文档服务器），由 `connect_all()` 统一接管：

```json
// McpJsonConfig.mcpServers（mcp/types.py:43）
{
  "docs": {
    "type": "stdio",
    "command": "node",
    "args": ["docs-server.js", "--root", "/workspace"],
    "env": {"MCP_AUTH_TOKEN": "sk-xxx"}
  }
}
```

**全程时序**（左侧箭头表流向，右侧标注触发的代码）：

| 步骤 | 代码触发 | 状态机 |
| --- | --- | --- |
| 0 | `McpClientManager.__init__`（`:34-41`）按配置登记 | `docs: pending` |
| 1 | `connect_all`（`:45`）`isinstance` 命中 `McpStdioServerConfig` → `_connect_stdio`（`:49` / `:180`） | `pending` |
| 2 | `stdio_client(StdioServerParameters(...))`（`:183-191`）拉起子进程 | `pending` |
| 3 | `_register_connected_session`：握手 + 发现（`:252-288`） | `pending` |
| 4 | 登记成功（`:291-298`） | `docs: connected` + 工具/资源快照 |
| 5 | 引擎回合中 `call_tool("docs", "read_document", ...)`（`:129`） | `connected` |

子进程起来后，stdin/stdout 上流通的 JSON-RPC 帧（每帧**单行**、`\n` 结尾，这是 stdio framing 的协议层行为）：

```
# → 1  initialize（阶段 3，_register_connected_session:263 session.initialize()）
→ {"jsonrpc":"2.0","id":1,"method":"initialize","params":{
     "protocolVersion":"2024-11-05","capabilities":{},
     "clientInfo":{"name":"openharness","version":"x"}}}
← {"jsonrpc":"2.0","id":1,"result":{
     "protocolVersion":"2024-11-05",
     "capabilities":{"tools":{"listChanged":false},"resources":{"subscribe":false}},
     "serverInfo":{"name":"docs-server","version":"0.2.0"}}}

# → 2  notifications/initialized（无 id 的通知，不需要响应）
→ {"jsonrpc":"2.0","method":"notifications/initialized"}

# → 3  tools/list（:264 工具发现；inputSchema 缺失时兜底 object 空 properties）
→ {"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}}
← {"jsonrpc":"2.0","id":2,"result":{"tools":[{
     "name":"read_document","description":"Read a documentation file",
     "inputSchema":{"type":"object","properties":{...},"required":["path"]}}]}}

# → 4  resources/list（:267；服务器不支持时仅容忍 "Method not found"，不视为失败）
→ {"jsonrpc":"2.0","id":3,"method":"resources/list","params":{}}
← {"jsonrpc":"2.0","id":3,"result":{"resources":[]}}

# → 5  tools/call（阶段 5，_connect_stdio 之后由 call_tool 驱动，参数 dict 原样透传）
→ {"jsonrpc":"2.0","id":4,"method":"tools/call","params":{
     "name":"read_document","arguments":{"path":"/workspace/OpenHarness/README.md"}}}
← {"jsonrpc":"2.0","id":4,"result":{
     "content":[{"type":"text","text":"# OpenHarness\n..."}],"isError":false}}
```

第 5 步的结果进入 `call_tool` 的字符串化逻辑（`client.py:144-154`）：`text` 类型的 content 逐条取 `text` 字段拼接、`\n` 连接、去首尾空白，最终引擎拿到一条纯文本字符串；若 `structuredContent` 非空而 content 为空，则改取 `structuredContent`（`:150-151`）。

### 8.2 一个 MCP 工具如何在 OpenHarness 中呈现

> 示例：远端工具 `read_document` 的动态封装（来源 `tools/mcp_tool.py:17` 构造、`:39` `_JSON_TYPE_MAP`、`:49` `_input_model_from_schema`、`:26` `execute`；`McpToolInfo` 字段来自 `mcp/types.py:46`；内容为示意）

握手发现的 `McpToolInfo`（frozen dataclass，`mcp/types.py:46-53`）：

```python
McpToolInfo(
    server_name="docs",
    name="read_document",
    description="Read a documentation file from the knowledge base",
    input_schema={
        "type": "object",
        "properties": {
            "path":   {"type": "string",  "description": "Path within the docs tree"},
            "page":   {"type": "integer", "description": "Page number, 1-based"},
            "format": {"type": "string",  "enum": ["markdown", "html"]},
        },
        "required": ["path"],
    },
)
```

`create_default_tool_registry`（`tools/__init__.py:96-97`）把这条 `McpToolInfo` 交给 `McpToolAdapter`，动态构造过程：

```python
# mcp_tool.py:20-22  —— 命名
sanitize("docs")                    == "docs"      # 字母数字_- 保留
sanitize("read_document")           == "read_document"
tool.name = "mcp__docs__read_document"             # 三段落：mcp__<server>__<tool>

# mcp_tool.py:49-63  —— JSON Schema → pydantic 输入模型
# _JSON_TYPE_MAP（mcp_tool.py:39-46）：string→str, integer→int, number→float,
#                            boolean→bool, array→list, object→dict, 其它→object
# 必填(path) → (str, Field(default=...))
# 可选(page/format) → (int|None / str|None, Field(default=None))
input_model = create_model("Mcp__Docs__Read_DocumentInput", **fields)
```

引擎渲染给模型看的，是 `input_model.model_json_schema()`（`tools/base.py:56`）重新输出的 JSON Schema：

```json
{
  "title": "Mcp__Docs__Read_DocumentInput",
  "type": "object",
  "properties": {
    "path":   {"anyOf": [{"type": "string"}],  "title": "Path"},
    "page":   {"anyOf": [{"type": "integer"}, {"type": "null"}], "default": null, "title": "Page"},
    "format": {"anyOf": [{"type": "string"},  {"type": "null"}], "default": null, "title": "Format"}
  },
  "required": ["path"]
}
```

模型填参后执行（`mcp_tool.py:26-36`）：

```python
# 引擎填 path=...、page=3（format 未填）→ model_dump(mode="json", exclude_none=True) 丢掉 None
await self._manager.call_tool("docs", "read_document",
    {"path": "/workspace/OpenHarness", "page": 3})
```

`call_tool` 内部走 `self._sessions["docs"].call_tool(...)`（`client.py:139`），即 8.1 中那条 id=4 的 `tools/call` 帧；服务器断开时抛 `McpServerNotConnectedError`，`execute` 把它转成 `ToolResult(is_error=True)` 回喂给模型（`mcp_tool.py:34-35`），而不是中断整个回合。

### 8.3 Feishu 消息往返

> 示例：用户在飞书私聊 @ 机器人发"检查一下代码"的完整往返（来源 `channels/impl/feishu.py:1213` `_on_message`、`:1323` `_handle_message`、`:1064` `send`、`bus/events.py:8` 字段；内容为示意）

**① 入站**：lark WS 长连接收到事件 `im.message.receive_v1`（`feishu.py:476` 只注册此事件），回调 `_on_message_sync`（`:1205`）把事件调度进主 loop。`data.event.message/content` 示意：

```json
{
  "message_id":  "om_3f9a...",
  "chat_id":     "ou_7a1b...",      // p2p 私聊：chat_id 即用户 open_id，非 oc_ 开头
  "chat_type":   "p2p",
  "message_type":"text",
  "content":     "{\"text\":\"检查一下代码\"}"
}
```

`_on_message`（`feishu.py:1213`）处理链：去重（`_processed_message_ids`，`:1220-1228`）→ 跳过 bot（`:1231-1232`）→ 解析 `content_json["text"]`（`:1254-1257`）→ p2p 非群不触发群策略（`:1292-1312` 跳过）→ `_add_reaction(THUMBSUP)`（`:1314-1315`）。

**② 统一入站出口**：`_handle_message`（`feishu.py:1323`）最终调到 `BaseChannel._handle_message`（`base.py:96`）——先 `is_allowed("ou_7a1b...")` 白名单校验（`:118-124`），通过后构造 `InboundMessage`（`base.py:126-134`）：

```python
InboundMessage(
    channel="feishu",
    sender_id="ou_7a1b...",
    chat_id="ou_7a1b...",                          # 私聊 reply_to = sender_id（feishu.py:1318）
    content="检查一下代码",
    media=[],
    metadata={
        "message_id": "om_3f9a...", "chat_type": "p2p", "msg_type": "text",
        "mentions": [], "mentions_bot": False,
        # feishu.py:1328-1338 还带 thread_id/root_id(均为 None)、sender_display_name、sender_label
    },
    session_key_override=None,
)
# bus.publish_inbound(msg)（base.py:136)  →  session_key 属性算出 "feishu:ou_7a1b..."（events.py:24）
```

**③ 引擎桥**：`ChannelBridge._handle`（`adapter.py:94`）`engine.submit_message("检查一下代码")`，累积全部 `AssistantTextDelta`（`adapter.py:101-102`），回合结束时拼出回复并回总线：

```python
OutboundMessage(
    channel="feishu",
    chat_id="ou_7a1b...",
    content="已检查：代码中有 2 处潜在问题。\n```python\n...\n```",
    metadata={"_session_key": "feishu:ou_7a1b..."},   # adapter.py:119-124
)
```

**④ 出站 + 发卡**：`_dispatch_outbound`（`manager.py:209`）命中 `feishu` 渠道 → `FeishuChannel.send`（`feishu.py:1064`）：

- `receive_id_type = "open_id"`（chat_id `ou_7a1b...` 不以 `oc_` 开头，`:1071`）；
- `_detect_msg_format`（`:704`）：内容含 ``` 代码块 → `_COMPLEX_MD_RE` 命中 → `"interactive"`（`:716-717`）；
- `_build_card_elements`（`:595`）+ `_split_elements_by_table_limit`（`:609`）→ 拼卡片：

```json
{"config": {"wide_screen_mode": true}, "elements": [
  {"tag": "markdown", "content": "已检查：代码中有 2 处潜在问题。\n```python\n...\n```"}
]}
```

- `_send_message_sync("open_id", "ou_7a1b...", "interactive", json.dumps(card))`（`:1158-1162`）→ lark REST `im.v1.message.create` 发出，用户在飞书看到卡片。

### 8.4 Telegram 消息往返

> 示例：Telegram 私聊"查看今天的日志"的完整往返（来源 `channels/impl/telegram.py:352` `_on_message`、`:336` `_sender_id`、`:223` `send`、`:278` 分片；Update 结构为 Telegram Bot API 形状，内容为示意）

**① 入站**：长轮询 `updater.start_polling(allowed_updates=["message"])`（`telegram.py:179-182`）拉到一个 `Update` 交给 `_on_message`（`:352`）：

```json
{
  "update_id": 1201,
  "message": {
    "message_id": 8842,
    "from":    {"id": 123456789, "is_bot": false, "first_name": "John", "username": "johndoe"},
    "chat":    {"id": 123456789, "type": "private", "first_name": "John", "username": "johndoe"},
    "date":    1787450000,
    "text":    "查看今天的日志"
  }
}
```

`_on_message` 处理链（`:352-469`）：`sender_id = "123456789|johndoe"`（`_sender_id`，`:336-340`，供 `is_allowed` 的 `|` 拆分匹配）→ 缓存 `_chat_ids[sender_id]=chat_id`（`:363`）→ 无媒体、无 caption → `content="查看今天的日志"`（`:425`）→ 无 `media_group_id` 走普通路径 → `_start_typing`（`:454`）→ `_handle_message(...)`。

`BaseChannel._handle_message` 构造的入站（`base.py:126-134`）：

```python
InboundMessage(
    channel="telegram",
    sender_id="123456789|johndoe",
    chat_id="123456789",
    content="查看今天的日志",
    media=[],
    metadata={"message_id": 8842, "user_id": 123456789, "username": "johndoe",
              "first_name": "John", "is_group": False},        # telegram.py:462-468
    session_key_override=None,                                  # session_key = "telegram:123456789"
)
```

**② 引擎桥 + 出站**：`ChannelBridge` 消费→引擎→`OutboundMessage(channel="telegram", chat_id="123456789", content=..., metadata={"_session_key": ...})`（与 8.3 ③相同，只是 channel 不同）。

**③ 发送**：`TelegramChannel.send`（`telegram.py:223`）→ `int(chat_id)`（`:234`）→ 非进度消息先 `_stop_typing`（`:230-231`）→ `split_message(content, 4000)`（`:278`，`TELEGRAM_MAX_MESSAGE_LEN=4000`，单条超长才分片）→ `_markdown_to_telegram_html`（`:32`）转 HTML → `bot.send_message(chat_id=123456789, text=<HTML>, parse_mode="HTML")`：

```
md:   **今日日志**
      - `ls /var/log` 输出 12 行
      - 未发现错误
HTML: <b>今日日志</b>
      • <code>ls /var/log</code> 输出 12 行
      • 未发现错误
```

若 HTML 解析失败则捕获异常回落纯文本重发（`:296-310`）。`config.reply_to_message` 开着时，会带 `ReplyParameters(message_id=8842, allow_sending_without_reply=True)` 作为对该消息的回复（`:239-246`）。

### 8.5 ChannelManager dispatch 示例

> 示例：两条入站消息分派到不同 session_key + 出站进度/工具提示过滤（来源 `channels/impl/manager.py:209` `_dispatch_outbound`、`channels/bus/events.py:21` `session_key`、`channels/adapter.py:123`；工具提示 metadata 见 `ohmo/gateway/runtime.py:584-589`；内容为示意）

**入站分派（到不同会话）**：两条消息先后 `publish_inbound`（`base.py:136`）：

| 消息 | `channel` | `chat_id` | `session_key`（`events.py:24`） |
| --- | --- | --- | --- |
| A | `feishu` | `ou_7a1b...`（p2p） | `feishu:ou_7a1b...` |
| B | `telegram` | `123456789`（私聊） | `telegram:123456789` |

两消息先后被 `ChannelBridge._loop`/`_handle` 消费（`adapter.py:82` / `:94`），引擎各自开一个新会话回合；每条回复的 `OutboundMessage` 都从入站消息的 `msg.session_key` 拷入 `metadata["_session_key"]`（`adapter.py:123`）——这样回程路由天然带回到**发出它的那个渠道/会话**，两条线互不串扰。

**出站过滤（`manager.py:209-237`）**：出站队列同时来了 3 条消息：

```python
# 1. 最终回复（无 _progress）→ 永远放行
OutboundMessage(channel="feishu", chat_id="ou_7a1b...",
    content="检查完成。", metadata={"_session_key": "feishu:ou_7a1b..."})

# 2. 处理进度（_progress=True，无 _tool_hint）
OutboundMessage(channel="feishu", chat_id="ou_7a1b...",
    content="⏳ 正在读取仓库结构…", metadata={"_progress": True, "_session_key": "feishu:ou_7a1b..."})

# 3. 工具提示（_progress + _tool_hint，metadata 结构同 ohmo/gateway/runtime.py:584-589）
OutboundMessage(channel="telegram", chat_id="123456789",
    content="🛠️ 正在使用 web_fetch: https://example.com",
    metadata={"_progress": True, "_tool_hint": True, "_session_key": "telegram:123456789"})
```

`_dispatch_outbound` 的判定（`manager.py:220-224`，开关来自 `config/schema.py:103-104`，默认都 `True`）：

| 场景 | 判定 | 结果 |
| --- | --- | --- |
| `send_progress=True`（默认） | 消息 2 无 `_tool_hint` → 走 `:223-224` 检查 | 放行 |
| `send_progress=False` | 消息 2 不满足 `:223` → `continue` | **滤掉** |
| `send_tool_hints=True`（默认） | 消息 3 有 `_tool_hint` → 走 `:221-222` 检查 | 放行 |
| `send_tool_hints=False` | 消息 3 不满足 `:221` → `continue` | **滤掉** |
| 未知 channel | `self.channels.get("github") is None` | `logger.warning("Unknown channel")`（`:233`） |

命中渠道后 `await channel.send(msg)`（`:229`），异常只记日志不中断分发循环。

### 8.6 Bridge 会话示例

> 示例：BridgeSessionManager 注册一个外部 CLI 会话 + work_secret 编解码与接入 URL（字段来源 `bridge/types.py:12` `WorkData`、`:20` `WorkSecret`、`bridge/manager.py:13` `BridgeSessionRecord`、`bridge/manager.py:35` `spawn`、`bridge/work_secret.py:11`/`:35`；内容为示意）

**注册一个受管会话**：用户在 `/bridge` 会话里执行 `/bridge spawn ohmo --init workspace-x`（`commands/registry.py:1068-1077`）：

```python
# get_bridge_manager().spawn()（manager.py:35-45）
session_id = "bridge-1743451200"                                  # registry.py:1071
handle = await spawn_session(session_id=..., command="ohmo --init workspace-x", cwd="/home/user/proj")
# spawn_session（session_runner.py:32-46）→ create_shell_subprocess（bash -lc）
# → SessionHandle(session_id, process, cwd, started_at)（session_runner.py:13-20）
# manager 登记 _sessions/_commands，建输出文件 {data_dir}/bridge/bridge-1743451200.log（:39-43）
# 启动 _copy_output 任务把 stdout 实时落盘（:44, :85-94）
```

随后 `/bridge list`（`registry.py:1078-1087`）读 `list_sessions()`，得到 UI 快照 `BridgeSessionRecord`（`manager.py:13-23`）：

```json
{
  "session_id": "bridge-1743451200",
  "command": "ohmo --init workspace-x",
  "cwd": "/home/user/proj",
  "pid": 48231,
  "status": "running",                  // returncode None→running / 0→completed / 其它→failed（manager.py:51-56）
  "started_at": 1743451200.123,
  "output_path": "~/.openharness/bridge/bridge-1743451200.log"
}
```

**work_secret 编解码**：`/bridge encode https://api.example.com tok_xxx`（`registry.py:1058-1062`）构造并编码：

```python
WorkSecret(version=1, session_ingress_token="tok_xxx", api_base_url="https://api.example.com")
# encode_work_secret（work_secret.py:11-14）：
#   json.dumps(__dict__, separators=(",", ":"))        → 紧凑 JSON
#   base64.urlsafe_b64encode(...).decode().rstrip("=") → base64url 无填充
# 结果示意：eyJ2ZXJzaW9uIjoxLCJzZXNzaW9uX2luZ3Jlc3NfdG9rZW4iOiJ0b2tfeHh4IiwiYXBpX2Jhc2VfdXJsIjoi...
```

解码路径 `decode_work_secret`（`work_secret.py:17-32`）反向补 `=` → base64 解码 → JSON 解析 → **校验 `version==1`、`session_ingress_token` 非空、`api_base_url` 为 str**，任一不过就抛 `ValueError`。

**构建 SDK 接入 URL**：`build_sdk_url(api_base_url, session_id)`（`work_secret.py:35-41`）按环境推断协议与路径版本：

```python
build_sdk_url("http://localhost:8080",  "session-42")  # 本地 → ws://localhost:8080/v2/session_ingress/ws/session-42
build_sdk_url("https://api.example.com", "session-42")  # 远端 → wss://api.example.com/v1/session_ingress/ws/session-42
```

`/bridge sdk https://api.example.com session-42`（`registry.py:1066-1067`）即打印上述第二个 URL。整体上：**接入用 `WorkData(type="session", id=...)` 描述工作项（`types.py:12`），用 `WorkSecret` 携带凭证，用户侧拿到 base64url 串与 ws 接入地址就是一把"钥匙"**；会话生命周期管理（spawn/list/output/stop）全部落在 `BridgeSessionManager` 上。
