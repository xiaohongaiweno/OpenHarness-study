# OpenHarness 源码分析：核心引擎（engine）与 API 客户端（api）——深度版

> 分析对象：
> - `src/openharness/engine/` — `query.py`(1057 行)、`query_engine.py`(306 行)、`messages.py`(222 行)、`stream_events.py`(90 行)、`cost_tracker.py`(24 行)、`__init__.py`(80 行)
> - `src/openharness/api/` — `client.py`(271 行)、`registry.py`(437 行)、`openai_client.py`(484 行)、`codex_client.py`(407 行)、`copilot_client.py`(134 行)、`copilot_auth.py`(240 行)、`provider.py`(186 行)、`errors.py`(19 行)、`usage.py`(17 行)、`__init__.py`(21 行)
> - 交叉引用：`tools/base.py`(80 行)、`hooks/events.py`(20 行)、`permissions/checker.py`(200 行)、`services/tool_outputs.py`(55 行)
>
> 文中所引用的**类名、函数名、行号均经源码核对属实**，可直接用编辑器跳转验证。

---

## 目录

1. [模块职责总述](#一模块职责总述)
2. [全局常量速查表](#二全局常量速查表)
3. [run_query() 逐行走读——Agent 主循环 16 步拆解](#三run_query-逐行走读agent-主循环-16-步拆解)
4. [_execute_tool_call() 逐段拆解](#四_execute_tool_call-逐段拆解querypy887-1018)
5. [消息模型逐字段（messages.py）](#五消息模型逐字段messagespy)
6. [流式事件全部类型（stream_events.py）](#六流式事件全部类型stream_eventspy)
7. [API 客户端层逐方法拆解](#七api-客户端层逐方法拆解)
8. [Provider 注册表（registry.py）](#八provider-注册表registrypy)
9. [真实代码片段逐行注释](#九真实代码片段逐行注释)
10. [关键调用链](#十关键调用链)
11. [设计要点、扩展点与注意点](#十一设计要点扩展点与注意点)
12. [实际示例合集（结构示意与走查）](#十二实际示例合集结构示意与走查)

---

## 一、模块职责总述

### 1.1 engine（核心引擎）

`src/openharness/engine/` 实现了一个**带工具调用（tool-use）能力的多轮 Agent 会话循环**，是 OpenHarness 的"大脑"。它的核心矛盾是：模型是无状态的（一次调用只返回一个回合），而 Agent 是跨回合有状态的（要跑工具、回灌结果、循环直到模型不再请求工具）。所有状态都靠 `messages` 历史列表 + `tool_metadata` 字典显式携带。

| 文件 | 职责 |
| --- | --- |
| `query.py` | 真正的 Agent Loop：`run_query()` 把消息喂给模型、流式接收输出、解析 `tool_use`、执行工具、把结果回灌，循环直至模型不再请求工具；同时负责超限保护、自动压缩调度、图片预处理、权限检查、钩子（hook）与工具超长输出落盘。 |
| `query_engine.py` | 高层门面 `QueryEngine`：持有会话历史与全部依赖（API 客户端、工具注册表、权限检查器、hooks、settings），对外暴露 `submit_message()` / `continue_pending()` 两个异步流式接口，并负责记忆（session memory / durable memory / auto-dream）的生命周期。 |
| `messages.py` | 会话消息的 Pydantic 模型（TextBlock / ImageBlock / ToolUseBlock / ToolResultBlock / ConversationMessage）、Anthropic 协议序列化、断点会话消毒 `sanitize_conversation_messages`。 |
| `stream_events.py` | 引擎对外暴露的流式事件类型（文本增量、回合完成、工具开始/结束、错误、状态、压缩进度）。 |
| `cost_tracker.py` | 跨回合累加 token 用量（纯聚合，不换算金额）。 |
| `__init__.py` | 公共导出，使用**延迟导入（模块级 `__getattr__`）**避免循环依赖。 |

### 1.2 api（API 客户端层）

`src/openharness/api/` 实现"模型提供商抽象层"。所有客户端都遵守同一个 **`SupportsStreamingMessages` 协议**（`client.py:80`），因此引擎可以无差别地对接 Anthropic / OpenAI 兼容 / Codex / Copilot 四类后端。

| 文件 | 职责 |
| --- | --- |
| `client.py` | 定义请求/事件 DTO（`ApiMessageRequest`、`ApiTextDeltaEvent`、`ApiMessageCompleteEvent`、`ApiRetryEvent`）、`SupportsStreamingMessages` 协议，以及 **Anthropic SDK 客户端** `AnthropicApiClient`（含三类重试退避、Claude OAuth 支持、错误翻译 `_translate_api_error`）。 |
| `registry.py` | **Provider 注册表**：`ProviderSpec` 数据类 + 一个有序的 `PROVIDERS` 元组（22 条），是"单一事实来源"，提供按 API Key 前缀、base_url 关键字、模型名关键字的三级探测。 |
| `provider.py` | 基于 registry 的高层能力判断：`detect_provider()`、`auth_status()`、`is_model_multimodal()`（23 条视觉模型名正则启发式）。 |
| `openai_client.py` | **OpenAI 兼容客户端** `OpenAICompatibleClient`（DashScope、DeepSeek、Gemini、Groq 等），负责 Anthropic 消息/tool schema → OpenAI Chat Completions 格式的双向转换、thinking 模型参数适配与 ` thinking` 流式剥除。 |
| `codex_client.py` | **Codex 订阅客户端** `CodexApiClient`，不走 SDK，直接 httpx + SSE 打 `chatgpt.com/backend-api/codex/responses`。 |
| `copilot_client.py` | **GitHub Copilot 客户端** `CopilotClient`：组合 `OpenAICompatibleClient` + 注入 Copilot 专属 header 的轻量封装。 |
| `copilot_auth.py` | Copilot 的 OAuth device-flow 认证：请求设备码、轮询 token、持久化到 `~/.openharness/copilot_auth.json`。 |
| `errors.py` | 异常层级：`OpenHarnessApiError` → `AuthenticationFailure` / `RateLimitFailure` / `RequestFailure`。 |
| `usage.py` | `UsageSnapshot`：token 用量快照（input/output/total）。 |
| `__init__.py` | 公共导出（四个客户端类 + ProviderInfo + 工具函数）。 |

### 1.3 对外接口

`engine/__init__.py:23` 声明 `__all__`：`AssistantTextDelta`、`AssistantTurnComplete`、`ConversationMessage`、`ImageBlock`、`QueryEngine`、`TextBlock`、`ToolExecutionCompleted`、`ToolExecutionStarted`、`ToolResultBlock`、`ToolUseBlock`。

注意其实现手法（PEP 562 模块级 `__getattr__`，`engine/__init__.py:37-81`）：`__init__.py` 里并没有真实 import，而是在 `__getattr__` 中按名字延迟加载（`ConversationMessage` 等 5 个来自 `messages`，`QueryEngine` 来自 `query_engine`，4 个事件来自 `stream_events`）。`TYPE_CHECKING` 分支（`engine/__init__.py:7-21`）仅供类型检查器使用，运行期零导入开销，同时规避了 messages ↔ stream_events 与 api ↔ engine 之间的循环依赖。

`api/__init__.py:3-9, 11-21` 导出：`AnthropicApiClient`、`CodexApiClient`、`CopilotClient`、`OpenAICompatibleClient`、`OpenHarnessApiError`、`ProviderInfo`、`UsageSnapshot`、`auth_status`、`detect_provider`。注意它**没有**导出 `registry.py`（`PROVIDERS`/`ProviderSpec` 属于内部实现）。

---

## 二、全局常量速查表

### 2.1 engine/query.py（56-63, 46-48）

| 常量 | 行号 | 值 | 用途 |
| --- | --- | --- | --- |
| `AUTO_COMPACT_STATUS_MESSAGE` | 46 | `"Auto-compacting conversation memory to keep things fast and focused."` | auto 触发的压缩提示文案 |
| `REACTIVE_COMPACT_STATUS_MESSAGE` | 47 | `"Prompt too long; compacting conversation memory and retrying."` | 提示词过长时强制压缩的文案 |
| `MAX_SAFE_COMPLETION_TOKENS` | 48 | `128_000` | 单请求输出 token 安全上限 |
| `MAX_TRACKED_READ_FILES` | 56 | `6` | `read_file` 状态记忆条数上限 |
| `MAX_TRACKED_SKILLS` | 57 | `8` | `invoked_skills` 记忆上限 |
| `MAX_TRACKED_ASYNC_AGENT_EVENTS` | 58 | `8` | `async_agent_state` 活动摘要上限 |
| `MAX_TRACKED_ASYNC_AGENT_TASKS` | 59 | `12` | `async_agent_tasks` 上限 |
| `MAX_TRACKED_WORK_LOG` | 60 | `10` | `recent_work_log` 上限 |
| `MAX_TRACKED_USER_GOALS` | 61 | `5` | `recent_goals` 上限 |
| `MAX_TRACKED_ACTIVE_ARTIFACTS` | 62 | `8` | `active_artifacts` 上限 |
| `MAX_TRACKED_VERIFIED_WORK` | 63 | `10` | `recent_verified_work` 上限 |

### 2.2 api/client.py（31-36）

| 常量 | 行号 | 值 | 用途 |
| --- | --- | --- | --- |
| `MAX_RETRIES` | 32 | `3` | 最大重试次数（总尝试 = 4） |
| `BASE_DELAY` | 33 | `1.0` 秒 | 指数退避基数 |
| `MAX_DELAY` | 34 | `30.0` 秒 | 退避上限 |
| `RETRYABLE_STATUS_CODES` | 35 | `{429,500,502,503,529}` | Anthropic 可重试 HTTP 状态码 |
| `OAUTH_BETA_HEADER` | 36 | `"oauth-2025-04-20"` | Claude OAuth beta 头 |

### 2.3 api/openai_client.py（40-43）

`MAX_RETRIES=3`(40)、`BASE_DELAY=1.0`(41)、`MAX_DELAY=30.0`(42)、`_MAX_COMPLETION_TOKEN_MODEL_PREFIXES=("gpt-5","o1","o3","o4")`(43)。

### 2.4 api/codex_client.py（23-27）

`DEFAULT_CODEX_BASE_URL="https://chatgpt.com/backend-api"`(23)、`JWT_CLAIM_PATH="https://api.openai.com/auth"`(24)、`MAX_RETRIES=3`(25)、`BASE_DELAY_SECONDS=1.0`(26)、`MAX_DELAY_SECONDS=30.0`(27)。

### 2.5 services/tool_outputs.py（10-12, 26-39）

`DEFAULT_TOOL_OUTPUT_INLINE_CHARS=16_000`、`DEFAULT_TOOL_OUTPUT_PREVIEW_CHARS=3_000`、`DEFAULT_MICROCOMPACT_TOOL_RESULT_CHARS=4_000`。运行时读取环境变量：`OPENHARNESS_TOOL_OUTPUT_INLINE_CHARS`（下限 256）、`OPENHARNESS_TOOL_OUTPUT_PREVIEW_CHARS`（下限 128）、`OPENHARNESS_MICROCOMPACT_TOOL_RESULT_CHARS`（下限 256）。

---

## 三、run_query() 逐行走读——Agent 主循环 16 步拆解

### 3.0 函数形态（633-636）

```python
async def run_query(
    context: QueryContext,
    messages: list[ConversationMessage],
) -> AsyncIterator[tuple[StreamEvent, UsageSnapshot | None]]:
```

- 它是一个**异步生成器**：调用方 `async for (event, usage) in run_query(context, messages)` 逐事件消费；
- 每次 `yield` 出 `(StreamEvent, UsageSnapshot | None)` 二元组。`usage` 绝大多数时候是 `None`，只有 `AssistantTurnComplete`（803）和空消息异常（799）时附带本次 token 用量；
- `QueryContext`（137-155）是一次查询运行共享的不可变上下文（dataclass）；`messages` 是**可变列表**，`run_query` 会在压缩、图片预处理、回灌工具结果时就地改写它——这是"状态"在多个 turn 之间延续的关键。

`QueryContext` 字段（137-155）：`api_client`(141)、`tool_registry`(142)、`permission_checker`(143)、`cwd: Path`(144)、`model`(145)、`system_prompt`(146)、`max_tokens`(147)、`effort: str|None=None`(148)、`context_window_tokens`(149)、`auto_compact_threshold_tokens`(150)、`permission_prompt`(151)、`ask_user_prompt`(152)、`max_turns: int|None=200`(153)、`hook_executor`(154)、`tool_metadata: dict|None`(155)。

### 3.1 Step 0 —— 惰性导入与状态初始化（645-657）

```
645-648: from openharness.services.compact import AutoCompactState, auto_compact_if_needed
```

**为什么惰性导入**：`query.py` 处于导入关系的最底层，而 `services/compact` 会反过来依赖 engine 的类型，直接 import 会形成环。函数体内导入把依赖推迟到真正需要压缩时。

```
650: compact_state = AutoCompactState()
651: reactive_compact_attempted = False
652: last_compaction_result: tuple[list[ConversationMessage], bool] = (messages, False)
653-656: effective_max_tokens = _bounded_completion_tokens(context.max_tokens, context.context_window_tokens)
657: reported_token_clamp = False
```

- `compact_state` 跨压缩调用持久保存压缩进度状态（防止同一轮重复压缩）；
- `reactive_compact_attempted` 保证"提示词过长时的强制压缩"每轮探测只尝试一次（768）；
- `last_compaction_result` 是 `nonlocal` 桥接变量（664），由内部生成器 `_stream_compaction` 写入"`(压缩后的消息列表, 是否真的压缩了)`"，主循环读取（714）；
- `effective_max_tokens = _bounded_completion_tokens(...)`（90-100）：把用户配置的 `max_tokens` 压到 `MAX_SAFE_COMPLETION_TOKENS`(128K) 以内，且取 `min` 不超过 `context_window_tokens`（若配置），返回 `max(1, ...)`。**为什么**：某些 OpenAI 兼容服务在请求到达模型侧上下文管理之前就拒绝超大 `max_tokens`，这一步防止"用户配置过大导致每一轮都失败"；
- `reported_token_clamp` 保证截断提示（702）只发一次而不是每轮刷屏。

### 3.2 Step 1 —— 内部生成器 `_stream_compaction`（659-697）

这是把「异步压缩任务」桥接成「流式进度事件」的适配器：

```
664: nonlocal last_compaction_result          —— 声明要写外层变量
665: progress_queue: asyncio.Queue[CompactProgressEvent] = asyncio.Queue()
667-668: async def _progress(event) -> None: await progress_queue.put(event)
             —— 压缩服务的进度回调：把 CompactProgressEvent 塞进队列
670-685: task = asyncio.create_task(auto_compact_if_needed(
             messages, api_client=..., model=..., system_prompt=...,
             state=compact_state, progress_callback=_progress,
             force=force, trigger=trigger, hook_executor=...,
             carryover_metadata=context.tool_metadata,
             context_window_tokens=..., auto_compact_threshold_tokens=...))
             —— 压缩在后台 asyncio task 里跑，不阻塞主循环
686-692: while True:
             event = await asyncio.wait_for(progress_queue.get(), timeout=0.05)
             yield event, None                            # 转发进度事件
          except asyncio.TimeoutError:
             if task.done(): break                        # 压缩结束退出轮询
             continue                                     # 50ms 轮询一次
694-695: while not progress_queue.empty(): yield progress_queue.get_nowait(), None   # 排空尾巴
696: last_compaction_result = await task                    # 拿到最终结果写给外层
```

**为什么用 `asyncio.wait_for(..., 0.05)` 而非直接 `await queue.get()`**：压缩可能跑几十秒，若直接阻塞等待队列，UI 会彻底卡死；用 50ms 超时轮询 + `task.done()` 检测，保证压缩结束时能立刻退出循环并 `await task` 拿到结果。

### 3.3 Step 2 —— 进入主循环（699-701）

```
699: turn_count = 0
700: while context.max_turns is None or turn_count < context.max_turns:
701:     turn_count += 1
```

循环条件：`max_turns` 为 `None`（不限）不设上限；否则每一个"模型回合"计数一次。注意：`max_turns` 语义是**每个用户 prompt 的 agentic 回合数**（含工具回灌后的续跑），不是消息条数。

### 3.4 Step 3 —— max_tokens 被截断时的一次性提示（702-710）

```
702: if effective_max_tokens != context.max_tokens and not reported_token_clamp:
703:     reported_token_clamp = True
704-710: yield StatusEvent(message=f"Requested max_tokens={context.max_tokens} exceeds the safe per-request output cap; using {effective_max_tokens}."), None
```

仅当 Step 0 的 `_bounded_completion_tokens` 真的压缩了上限时才提示一次；`reported_token_clamp` 保证整个 `run_query` 生命周期只发一次，避免每个 turn 都打扰 UI。

### 3.5 Step 4 —— 每轮开头的 auto-compact 检查（711-717）

```
712-713: async for event, usage in _stream_compaction(trigger="auto"): yield event, usage
714: compacted_messages, was_compacted = last_compaction_result
715-716: if compacted_messages is not messages: messages[:] = compacted_messages
```

- `trigger="auto"` 表示"预估 token 超阈值"的主动压缩（force 默认 False，意味着只在超阈值时才真正压）；
- 压缩服务内部先试便宜的 microcompact（清掉旧的 tool_result 内容），不够再做 LLM 摘要；
- `compacted_messages is not messages` 用**对象同一性**判断压缩是否替换了列表——若压缩器返回了全新列表，就把 `messages[:]` 就地换成新列表（保引用）。

### 3.6 Step 5 —— 图片预处理（719-722）

```
720-721: async for event in _preprocess_images_in_messages(messages, context): yield event, None
```

`_preprocess_images_in_messages`（563-630）：

- 572: `is_model_multimodal(context.model)` 为 True → 直接 return（模型天生支持视觉）；
- 575-578: 从 `context.tool_metadata["vision_model_config"]` 找视觉模型配置，没有则 return；
- 581-587: 扫描所有 user 消息里的 `ImageBlock`，收集 `(消息索引, 块索引, block)`；
- 592: `yield StatusEvent(_IMAGE_PREPROCESS_STATUS)`（"Converting image to text description via vision model…"，560）；
- 595-623: 内部 `_describe()`：查 `image_to_text` 工具 → `input_model.model_validate` 校验 → `ToolExecutionContext(cwd, metadata={vision_model_config, **tool_metadata})` → `await tool.execute(...)`；工具缺失/解析失败/执行报错分别返回带说明的占位文本；
- 625: `asyncio.gather(*[ _describe(...) for pending ])` **并行**描述所有图片；
- 628-630: 逐个把 `msg.content[blk_idx]` 就地替换为 `TextBlock(text=description)`。

### 3.7 Step 6 —— 组装 `ApiMessageRequest` 并发起流式调用（724-737）

```
724-725: final_message: ConversationMessage | None = None
         usage = UsageSnapshot()
727: try:
728-737: async for event in context.api_client.stream_message(ApiMessageRequest(
             model=context.model,
             messages=messages,                 # 完整历史，文件在被预处理/回灌后仍指向同一对象
             system_prompt=context.system_prompt,
             max_tokens=effective_max_tokens,   # ← 用压缩后的上限，不是原始 context.max_tokens
             tools=context.tool_registry.to_api_schema(),
             effort=context.effort,
         )):
```

- `tools=context.tool_registry.to_api_schema()`（tools/base.py:78）等价于 `[tool.to_api_schema() for tool in registry._tools.values()]`，而单工具 schema 是 `{name, description, input_schema: model_json_schema()}`（tools/base.py:51-57）；
- `context.api_client` 是协议对象，具体是哪个实现完全透明。

### 3.8 Step 7 —— API 事件到引擎事件的三路分发（738-752）

专用场景：`stream_message` 返回 **API 层事件**（client.py 定义的三个 dataclass），这里用 `isinstance` 逐一转译：

```
738-740: if isinstance(event, ApiTextDeltaEvent):
             yield AssistantTextDelta(text=event.text), None; continue
741-747: if isinstance(event, ApiRetryEvent):
             yield StatusEvent(message=f"Request failed; retrying in {event.delay_seconds:.1f}s (attempt {event.attempt+1} of {event.max_attempts}): {event.message}"), None; continue
750-752: if isinstance(event, ApiMessageCompleteEvent):
             final_message = event.message; usage = event.usage
```

- `ApiTextDeltaEvent`（client.py:51-55）→ `AssistantTextDelta`：纯转发文本；
- `ApiRetryEvent`（client.py:67-74）→ `StatusEvent`：**不**打断流，重试是客户端内部行为，这里只是给用户一个可感知的进度提示（attempt 从 0 计，故显示 `+1`）；
- `ApiMessageCompleteEvent`（client.py:58-64）→ 只**存** `final_message` 与 `usage`（不在此 yield），等处理完工具判断后再统一 `yield AssistantTurnComplete`（803）。

### 3.9 Step 8 —— 异常处理四选一（753-782）

```
753: except Exception as exc:
754:     error_msg = str(exc)
```

**路径 A：max_tokens 过大错误 → 提取上限重试（755-767）**

```
755: if _is_completion_token_limit_error(exc):
756:     supported_limit = _extract_completion_token_limit(exc)
757:     if supported_limit is not None and effective_max_tokens > supported_limit:
758-759:     previous_max_tokens = effective_max_tokens; effective_max_tokens = supported_limit
760-765:     yield StatusEvent(message=f"Model rejected max_tokens={previous_max_tokens}; retrying with provider limit {effective_max_tokens}."), None
766:     turn_count = max(0, turn_count - 1)     # 抵消本轮自增，不占用预算
767:     continue                                 # 不 return，直接带着新上限重跑整个 turn
```

- `_is_completion_token_limit_error`（121-126）：文本里同时出现 `max_tokens`/`max_completion_tokens` 与 `too large`/`at most`/`completion tokens` 之一；
- `_extract_completion_token_limit`（103-118）：用三条正则从报错里抠数字，例如 provider 报 `"supports at most 128000 completion tokens"` → 返回 128000；
- `turn_count = max(0, turn_count - 1)` 是精细的**预算补偿**：本轮失败没产出任何模型输出，不应计入 turn 预算。

**路径 B：prompt 过长且未尝试过 reactive 压缩 → 强制压缩后重试（768-777）**

```
768: if not reactive_compact_attempted and _is_prompt_too_long_error(exc):
769:     reactive_compact_attempted = True
770:     yield StatusEvent(message=REACTIVE_COMPACT_STATUS_MESSAGE), None
771-772: async for event, usage in _stream_compaction(trigger="reactive", force=True): yield event, usage
773-775: compacted_messages, was_compacted = last_compaction_result; messages[:] = ...
776-777: if was_compacted: continue
```

- `_is_prompt_too_long_error`（66-87）匹配 15 个关键字（`prompt too long`、`context_length_exceeded`、`maximum context`、`too many tokens`……）；
- `trigger="reactive"` + `force=True`：**无视阈值强制**执行压缩（auto 模式只在超阈值时压缩）；
- 注意 771 的整个 try/except 之外的机制：压缩成功 → `continue` 重跑，失败 → 落到下一段发错误事件。

**路径 C：网络类错误（778-779）**

```
778-779: if "connect" in error_msg.lower() or "timeout" in ... or "network" in ...:
             yield ErrorEvent(message=f"Network error: {error_msg}. Check your internet connection and try again."), None
```

**路径 D：其它 API 错误（780-781）→ `yield ErrorEvent("API error: ...")`**

**782: return** —— 所有错误路径最终都 return（不 raise），把错误转成事件交给 UI。这是刻意的设计：UI 期望"永远有结果可显示"，而不是异常冒泡。

### 3.10 Step 9 —— `final_message` 空检查（784-785）

```
784-785: if final_message is None: raise RuntimeError("Model stream finished without a final message")
```

唯一一个会裸抛的场景：流里既没增量也没 complete 事件（例如某个兼容客户端实现坏了）。这属于"编程错误"而非"运行时错误"，所以用 `raise`。

### 3.11 Step 10 —— coordinator 上下文弹出/结尾回塞（787-806）

```
787: coordinator_context_message: ConversationMessage | None = None
788: if context.system_prompt.startswith("You are a **coordinator**."):
789:     if messages and messages[-1].role == "user" and messages[-1].text.startswith("# Coordinator User Context"):
790:         coordinator_context_message = messages.pop()
```

- coordinator（`coordinator_mode.get_coordinator_user_context()`）模式会把"Worker 工具环境上下文"作为一条合成 user 消息追加到历史末尾（query_engine.py:126-135, 264-267）；
- 这条消息**必须在模型调用前处于最新位置**（让模型第一时间看到），但它本身不是用户真正的新输入，所以：
  - **模型调用前**：保证它是 `messages[-1]`（若确实在末尾则 pop 出来）；
  - **模型回应后**：先 append assistant 消息，再在 805-806 把 coordinator 上下文重新塞回末尾，这样下一轮模型又能在最新位置看到它。

### 3.12 Step 11 —— 空 assistant 消息防护（792-800）

```
792: if final_message.role == "assistant" and final_message.is_effectively_empty():
793:     log.warning("dropping empty assistant message from provider response")
794-800: yield ErrorEvent(message="Model returned an empty assistant message. The turn was ignored to keep the session healthy."), usage
800: return
```

- `is_effectively_empty`（messages.py:108-116）：content 为空，或只有空白文本，且没有 image/tool_use/tool_result；
- 某些 provider 会偶发返回空 assistant 消息，直接 append 会导致下一轮请求畸形，所以丢弃（不打进历史）并发可恢复错误事件，并返回。注意这里的 `usage` 被透传（799），不会丢失 token 计数。

### 3.13 Step 12 —— 记录消息、产出回合完成事件（802-803）

```
802: messages.append(final_message)
803: yield AssistantTurnComplete(message=final_message, usage=usage), usage
```

- `AssistantTurnComplete`（stream_events.py:19-24）是 UI 判断"一个回合结束"的主信号；`QueryEngine.submit_message` 监听它把本地历史同步为 `query_messages`（query_engine.py:270-271）；
- `usage` 在这里作为元组第二项下发，QueryEngine 把它喂给 `CostTracker`（query_engine.py:272-273）。

### 3.14 Step 13 —— 无工具则 STOP 收尾（808-817）

```
808: if not final_message.tool_uses:
809-816: if context.hook_executor is not None:
             await context.hook_executor.execute(HookEvent.STOP, {"event": HookEvent.STOP.value, "stop_reason": "tool_uses_empty"})
817:     return
```

这是主循环**唯一正常的退出条件**：模型这一轮没有请求任何工具 → Agent 任务完成 → 触发 `STOP` hook（hooks/events.py:19）→ `return`。

### 3.15 Step 14 —— 工具执行：单工具串行 vs 多工具并发（819-880）

```
819: tool_calls = final_message.tool_uses
```

**单工具分支（821-840）**：

```
821: if len(tool_calls) == 1:
822:     tc = tool_calls[0]
824:     yield ToolExecutionStarted(tool_name=tc.name, tool_input=tc.input), None
825-833: try:
             result = await _execute_tool_call(context, tc.name, tc.id, tc.input)
         except Exception as exc:                    # 兜底：任何异常都变成 is_error 的 ToolResultBlock
             log.exception(...)
             result = ToolResultBlock(tool_use_id=tc.id, content=f"Tool {tc.name} failed: ...", is_error=True)
834-839: yield ToolExecutionCompleted(tool_name=tc.name, output=result.content, is_error=result.is_error, metadata=result.result_metadata), None
840:     tool_results = [result]
```

- 单工具时**立刻**发 Started 事件、等待执行完毕再发 Completed（事件有序、UI 友好）；
- 外层另有 try/except **兜底**：`_execute_tool_call` 内部已经拦了大量错误，这里再包一层，保证连"程序性 bug"都不会让循环悬挂在未回填 tool_use 上。

**多工具分支（841-878）**：

```
841: else:
843-844: for tc in tool_calls: yield ToolExecutionStarted(...)      # 先全部广播"开始"
846-847: async def _run(tc): return await _execute_tool_call(...)
849-855: raw_results = await asyncio.gather(*[_run(tc) for tc in tool_calls], return_exceptions=True)
             # 注释（849-852）：
             # return_exceptions=True 保证单个工具失败不会把兄弟协程 cancel 掉，
             # 从而不会留下「没有对应 tool_result 的 tool_use」——
             # 而 Anthropic 的 API 只要发现 session 里有一个 tool_use 没有配对的
             # tool_result，就会拒绝下一次请求。
856-870: for tc, result in zip(tool_calls, raw_results):
             if isinstance(result, BaseException):      # gather 把异常作为返回对象带回来
                 log.exception(..., exc_info=result)
                 result = ToolResultBlock(id=tc.id, content=f"Tool {tc.name} failed: ...", is_error=True)
             tool_results.append(result)
872-878: for tc, result in zip(tool_calls, tool_results):
             yield ToolExecutionCompleted(...)
```

**为什么并发**：同一模型回合里多个工具往往是独立的（并行读文件、并行搜索），串行执行会成倍拖长时延；`asyncio.gather` 让它们同时跑。

### 3.16 Step 15 —— 结果回灌与循环退出（880-884）

```
880: messages.append(ConversationMessage(role="user", content=tool_results))
```

- 所有 `ToolResultBlock` 被打包成一条 `role="user"` 消息追加到历史——**这就是下一轮模型看到的"工具结果"**；
- 然后 while 回到 Step 3（下一轮 auto-compact + 模型调用），直到模型不再请求工具走 Step 13 的 return。

**882-884（循环外）**：

```
882-883: if context.max_turns is not None: raise MaxTurnsExceeded(context.max_turns)
884: raise RuntimeError("Query loop exited without a max_turns limit or final response")
```

- 若 while 条件因为 `turn_count >= max_turns` 退出（即模型一直要工具、永远不 stop），抛出 `MaxTurnsExceeded`（129-134，携带 `.max_turns` 属性）；
- 884 是防御性的"不可达"分支。

### 3.17 run_query 状态机小结

| turn 内顺序 | 动作 | 行号 |
| --- | --- | --- |
| 1 | 回合计数 | 701 |
| 2 | max_tokens 截断提示（一次性） | 702-710 |
| 3 | auto 压缩检查（超阈值才压） | 711-717 |
| 4 | 图片 → 文字预处理（非多模态模型） | 719-722 |
| 5 | 流式调用模型 | 727-737 |
| 6 | delta/retry/complete 三路转译 | 738-752 |
| 7 | 异常四路径（降 token / 强制压缩 / 网络错 / 其它错） | 753-782 |
| 8 | final_message 判空 | 784-785 |
| 9 | coordinator 上下文弹出 | 787-790 |
| 10 | 空 assistant 防护 | 792-800 |
| 11 | 记录历史 + 回合完成事件 | 802-803 |
| 12 | coordinator 上下文口回塞 | 805-806 |
| 13 | 无工具 → STOP hook → return | 808-817 |
| 14 | 工具执行（单串行 / 多并发） | 819-878 |
| 15 | 工具结果回灌 → 回到 1 | 880 |
| 循环外 | 超限 raise / 不可达 raise | 882-884 |

---

## 四、_execute_tool_call() 逐段拆解（query.py:887-1018）

### 4.0 签名（887-892）

```python
async def _execute_tool_call(
    context: QueryContext,
    tool_name: str,
    tool_use_id: str,
    tool_input: dict[str, object],
) -> ToolResultBlock:
```

一次工具调用从触发到产出 `ToolResultBlock` 的完整流水线，共 11 个环节。

### 4.1 环节 1 —— PRE_TOOL_USE hook（893-903）

```
893: if context.hook_executor is not None:
894-897: pre_hooks = await context.hook_executor.execute(HookEvent.PRE_TOOL_USE,
            {"tool_name": tool_name, "tool_input": tool_input, "event": HookEvent.PRE_TOOL_USE.value})
898-903: if pre_hooks.blocked:
            return ToolResultBlock(tool_use_id=tool_use_id, content=pre_hooks.reason or f"pre_tool_use hook blocked {tool_name}", is_error=True)
```

- 钩子（hooks/events.py:15 `PRE_TOOL_USE="pre_tool_use"`）可以抢先拦截工具调用；
- 拦下后返回 **is_error=True** 的 ToolResultBlock 并携带被挡原因——这个"错误结果"照样会回灌给模型，模型可据此调整策略。

### 4.2 环节 2 —— 注册表查找（905-914）

```
907: tool = context.tool_registry.get(tool_name)      # ToolRegistry.get（tools/base.py:70）纯查 dict
908-914: if tool is None:
             return ToolResultBlock(..., content=f"Unknown tool: {tool_name}", is_error=True)
```

模型可能请求一个引擎不认识（或未注册）的工具——此时返回 `Unknown tool` 错误块，模型会自己意识到"这个工具不存在"并换一种做法。

### 4.3 环节 3 —— pydantic 输入校验（916-924）

```
916-924: try:
             parsed_input = tool.input_model.model_validate(tool_input)
         except Exception as exc:
             return ToolResultBlock(..., content=f"Invalid input for {tool_name}: {exc}", is_error=True)
```

- 每个工具的 `input_model`（tools/base.py:40）都是 Pydantic `BaseModel`，`model_validate` 负责类型/必填/约束校验；
- 模型给的 JSON 参数可能缺字段或类型错，校验失败同样转成错误块回灌。

### 4.4 环节 4 —— 权限路径/命令归一化（926-931）

```
929: _file_path = _resolve_permission_file_path(context.cwd, tool_input, parsed_input)
930: _command = _extract_permission_command(tool_input, parsed_input)
```

`_resolve_permission_file_path`（1021-1042）统一 home 目录下的工具参数命名：
- 先在**原始 dict** 里找 `file_path` / `path` / `root` 任一非空字符串键；
- 再在**parsed model** 上反射 `getattr` 找同名属性；
- 相对路径用 `cwd` 拼接、`~` 展开、`resolve()` 成绝对路径。

`_extract_permission_command`（1045-1057）相应地找 `command` 字段（dict 或 model 属性）。

**为什么归一并让路径绝对化**：`PermissionChecker.evaluate` 里的路径规则是 fnmatch 模式匹配，且 `_policy_match_paths`（checker.py:159-169）会对路径做 `路径` 与 `路径/` 两种形态匹配——如果路径不 resolve，`*/.ssh/*` 这类规则会失效。

### 4.5 环节 5 —— 权限评估（933-938）

```
931-932: log.debug("permission check: %s read_only=%s path=%s cmd=%s", ...)
933-938: decision = context.permission_checker.evaluate(
             tool_name,
             is_read_only=tool.is_read_only(parsed_input),   # BaseTool.is_read_only（tools/base.py:46）默认 False
             file_path=_file_path,
             command=_command,
         )
```

`PermissionChecker.evaluate`（checker.py:75-156）的判定顺序：
1. **内置敏感路径保护**（88-98）——`SENSITIVE_PATH_PATTERNS`（checker.py:18-37）覆盖 `~/.ssh/*`、`~/.aws/credentials`、`~/.kube/config`、`~/.openharness/copilot_auth.json` 等，**任何模式/用户配置都无法覆盖**（防注入）；
2. 显式 deny 列表（101-102）→ 显式 allow 列表（105-106）→ 路径规则（109-117）→ command 拒绝模式（120-126）；
3. `FULL_AUTO` 全放行（129-130）；只读工具放行（132-134）；`PLAN` 模式拦截写操作（137-141）；
4. 默认模式：写工具 → `allowed=False, requires_confirmation=True`（143-156），并视命令是否为 `npm install` 等安装类命令追加提示（`_bash_permission_hint`）。

### 4.6 环节 6 —— 需确认 → 询问用户；否则硬阻断（939-966）

**可确认场景**（`decision.requires_confirmation=True` 且配了 `permission_prompt`）：

```
940-951: if decision.requires_confirmation and context.permission_prompt is not None:
             await context.hook_executor.execute(HookEvent.NOTIFICATION,
                 {"event": ..., "notification_type": "permission_prompt", "tool_name": ..., "reason": ...})
952:     confirmed = await context.permission_prompt(tool_name, decision.reason)   # 同步阻塞等用户
953-959: if not confirmed: return ToolResultBlock(..., content=decision.reason or f"Permission denied for {tool_name}", is_error=True)
```

先发 `NOTIFICATION` hook（hooks/events.py:18）让外部系统知道"正在请求用户授权"，再异步等待 `permission_prompt(tool_name, reason) -> bool` 的返回。

**硬阻断场景**（不该问用户/没有 prompt 可用）：

```
960-966: else: return ToolResultBlock(..., content=decision.reason or f"Permission denied for {tool_name}", is_error=True)
```

拒绝理由被塞进 ToolResultBlock.content 回灌模型——**模型能看到自己被拒绝的原因**并调整方案。

### 4.7 环节 7 —— 执行工具（968-981）

```
968: log.debug("executing %s ...", tool_name)
969: t0 = time.monotonic()
970-981: result = await tool.execute(
             parsed_input,
             ToolExecutionContext(
                 cwd=context.cwd,
                 metadata={
                     "tool_registry": context.tool_registry,        # 工具可以再调其它工具（子工具）
                     "ask_user_prompt": context.ask_user_prompt,     # 工具可反问用户（如 ask_user_question）
                     **(context.tool_metadata or {}),                # 展开全部 carryover 记忆
                 },
                 hook_executor=context.hook_executor,
             ),
         )
982-984: elapsed = time.monotonic() - t0; log.debug("executed %s in %.2fs ...", ...)
```

`ToolExecutionContext`（tools/base.py:17-23）：`cwd`、`metadata`、`hook_executor`。metadata 里注入了 `tool_registry`（允许工具内调用其它工具）、`ask_user_prompt`（允许工具弹出交互问题）以及**整个 `tool_metadata`**——这是 carryover 记忆在工具侧可见的通道。

### 4.8 环节 8 —— 超长输出截断 + 落盘（985-991）

```
985-989: inline_output, artifact_path = _offload_tool_output_if_needed(
             tool_name=tool_name, tool_use_id=tool_use_id, output=result.output,
         )
990-991: if artifact_path is not None: _remember_active_artifact(context.tool_metadata, str(artifact_path))
```

`_offload_tool_output_if_needed`（524-553）逻辑：

```
530: inline_limit = tool_output_inline_chars()      # 默认 16000，min 256
531-532: if len(output) <= inline_limit: return output, None     # 没超 → 原样返回
534-537: artifact_path = get_data_dir()/"tool_artifacts"/f"{时间戳}-{工具名清洗}-{uuid12hex}.txt"
538: artifact_path.write_text(output, ...)           # 完整输出落盘
539: preview = output[:tool_output_preview_chars()]  # 默认 3000，min 128
540: omitted = max(0, len(output) - len(preview))
541-553: inline = "[Tool output truncated]\nTool: ...\nTool use id: ...\nOriginal size: ...\nFull output saved to: ...\nInline preview: ..." + ("(N chars omitted)" 若 omitted) + 预览正文
```

`_safe_tool_artifact_name`（519-521）把工具名里非法字符替换为 `_` 并截到 80 字符，防止路径注入。

### 4.9 环节 9 —— 构造 ToolResultBlock（992-997）

```
992-997: tool_result = ToolResultBlock(
             tool_use_id=tool_use_id,          # ← 模型给出的 tool_use id，回灌时按此配对
             content=inline_output,
             is_error=result.is_error,
             result_metadata=dict(result.metadata or {}),
         )
```

`result_metadata`（`ToolResult`，tools/base.py:27-32）被拷贝成 dict——工具的附加元数据（如异步 agent 的 `agent_id`/`task_id`）原样透传。

### 4.10 环节 10 —— carryover 记忆更新（998-1006）

```
998-1006: _record_tool_carryover(context, tool_name=..., tool_input=..., tool_output=tool_result.content,
             tool_result_metadata=result.metadata, is_error=tool_result.is_error, resolved_file_path=_file_path)
```

`_record_tool_carryover`（395-510）是"按工具类型的记忆路由"，**is_error=True 时直接早退**（405-406）。每种工具写入对应的 `tool_metadata` 桶：

- **任意工具 + resolved_file_path** → `_remember_active_artifact`（408）；
- `read_file` → `read_file_state`（409-422：先按 path 去重再追加，`_remember_read_file` 260-284）+ `verified_state`「Inspected file …」；
- `skill` → `invoked_skills`（423-431）+ active_artifact `skill:<name>` + verified `Loaded skill ...`；
- `agent` / `send_message` → `async_agent_state` 摘要 + `async_agent_tasks`（靠 `_parse_spawned_agent_identity` 326-338 从输出文本或 metadata 提取 `agent_id`/`task_id`）+ verified（432-450）；
- `enter_plan_mode` / `exit_plan_mode` → 写 `tool_metadata["permission_mode"]`（451-454，`_update_plan_mode` 389-392）；
- `web_fetch` / `web_search` / `glob` / `grep` / `bash` → verified 摘要（455-478）；
- 第二段（479-510）再往 `recent_work_log`（`_remember_work_log` 375-386）追加 `Read file X` / `Ran bash: ...` / `Searched with grep ...` / `Loaded skill ...` / `Async agent action ...` / `Entered/Exited plan mode`。

这些记忆**不进对话、不烧 token**，专供压缩与后续工具做上下文延续。

### 4.11 环节 11 —— POST_TOOL_USE hook（1007-1017）

```
1007-1017: if context.hook_executor is not None:
              await context.hook_executor.execute(HookEvent.POST_TOOL_USE,
                  {"tool_name":..., "tool_input":..., "tool_output": tool_result.content, "tool_is_error": tool_result.is_error, "event": ...})
```

hook 无返回值，纯观察（如审计、通知、更新 UI 侧状态）。最后 1018 返回 `tool_result`。

### 4.12 执行流程图

```
_model 返回 tool_use_
      ▼
PRE_TOOL_USE hook ──blocked──> ToolResultBlock(is_error=True, reason)
      ▼
ToolRegistry.get(name) ── None ──> "Unknown tool"
      ▼
input_model.model_validate ── 失败 ──> "Invalid input"
      ▼
_resolve_permission_file_path / _extract_permission_command
      ▼
PermissionChecker.evaluate
  ├─ not allowed + requires_confirmation + permission_prompt → NOTIFICATION hook → ask user → 拒绝 → 错误块
  ├─ not allowed + 其它                         → 静态错误块
  └─ allowed →▼
await tool.execute(parsed, ToolExecutionContext)      （可 await 子工具/反问用户）
      ▼
_offload_tool_output_if_needed → inline（超长时落盘 tool_artifacts/）
      ▼
ToolResultBlock 构造（tool_use_id 配对）
      ▼
_record_tool_carryover → tool_metadata 记忆桶
      ▼
POST_TOOL_USE hook
      ▼
返回 ToolResultBlock → run_query 主循环拼接成 user(tool_results) 回灌
```

---

## 五、消息模型逐字段（messages.py）

### 5.1 TextBlock（14-18）

```python
class TextBlock(BaseModel):
    type: Literal["text"] = "text"   # 常量 discriminator，默认值 "text"
    text: str                        # 纯文本（无默认，必填）
```

### 5.2 ImageBlock（21-37）

```python
class ImageBlock(BaseModel):
    type: Literal["image"] = "image"   # 默认 "image"
    media_type: str                    # 如 image/png、image/jpeg（必填）
    data: str                          # base64 编码的图片二进制（"ascii" 字符串）
    source_path: str = ""              # 源文件路径，仅用于溯源，不上协议
```

`from_path(cls, path)`（29-37）：
1. `Path(path).expanduser().resolve()` 解析；
2. `mimetypes.guess_type` 猜 MIME，**不是 `image/*` 就抛 `ValueError("Unsupported image attachment: ...")`**；
3. `base64.b64encode(resolved.read_bytes()).decode("ascii")` 编码。

### 5.3 ToolUseBlock（40-46）

```python
class ToolUseBlock(BaseModel):
    type: Literal["tool_use"] = "tool_use"
    id: str = Field(default_factory=lambda: f"toolu_{uuid4().hex}")   # ★ id 生成规则
    name: str                       # 工具名（必填）
    input: dict[str, Any] = Field(default_factory=dict)   # 工具参数，默认 {}
```

**id 生成规则**：本地构造缺省为 `"toolu_" + uuid4().hex`（16 hex chars 后接 32 字符 hex）。但如果消息来自真实 API 回包（`assistant_message_from_api`，见 5.8），id 用的是 **provider 给的真实 id**——这至关重要，因为回灌 `tool_result.tool_use_id` 必须与模型请求的 id 精确配对。

### 5.4 ToolResultBlock（49-56）

```python
class ToolResultBlock(BaseModel):
    type: Literal["tool_result"] = "tool_result"
    tool_use_id: str              # 与 ToolUseBlock.id 配对（必填）
    content: str                  # 工具输出文本（截断后内联部分）
    is_error: bool = False        # 是否错误结果
    result_metadata: dict[str, Any] = Field(default_factory=dict)   # 工具元数据透传
```

### 5.5 ContentBlock（59-62）

```python
ContentBlock = Annotated[
    TextBlock | ImageBlock | ToolUseBlock | ToolResultBlock,
    Field(discriminator="type"),
]
```

Pydantic 判别联合：按 `type` 字段的值自动选择具体模型。

### 5.6 ConversationMessage（65-116）

```python
class ConversationMessage(BaseModel):
    role: Literal["user", "assistant"]                # 只有两种角色
    content: list[ContentBlock] = Field(default_factory=list)
```

| 成员 | 行号 | 说明 |
| --- | --- | --- |
| `_normalize_content` validator | 71-77 | `mode="before"`：历史遗留的 `content=None` 归一为 `[]`（否则校验会炸） |
| `from_user_text(text)` | 80-82 | `role="user", content=[TextBlock(text=text)]` |
| `from_user_content(content)` | 85-87 | 显式传入块列表 |
| `text` property | 89-94 | 拼全部 `TextBlock.text`（`"".join`） |
| `tool_uses` property | 96-99 | 筛出全部 `ToolUseBlock`（返回新列表） |
| `to_api_param()` | 101-106 | → **Anthropic wire format**（见 5.7） |
| `is_effectively_empty()` | 108-116 | 有文本块内容 或 有任何 image/tool_use/tool_result 块 → False；否则 True |

### 5.7 序列化 `serialize_content_block`（174-202）+ `to_api_param`

`to_api_param` 把整条消息变成 Anthropic SDK 参数：`{"role": ..., "content": [serialize_content_block(b) for b in content]}`。

`serialize_content_block` 四种块的输出（这就是模型实际看到的协议 JSON）：

```python
# TextBlock      → {"type": "text", "text": ...}
# ImageBlock     → {"type": "image", "source": {"type": "base64", "media_type": ..., "data": ...}}
# ToolUseBlock   → {"type": "tool_use", "id": ..., "name": ..., "input": ...}
# ToolResultBlock→ {"type": "tool_result", "tool_use_id": ..., "content": ..., "is_error": ...}
```

注意 `ToolResultBlock` 序列化时**丢弃** `result_metadata`（模型不需要，留在本地记忆里）。

### 5.8 反向转换 `assistant_message_from_api`（205-222）

把 Anthropic SDK 返回的 message 对象转回 `ConversationMessage(role="assistant")`：
- `content[].type == "text"` → `TextBlock(text=...)`；
- `content[].type == "tool_use"` → `ToolUseBlock(id=真实id, name, input=dict(...))`；若 SDK 里没有 id（理论上不会）兜底生成 `toolu_{uuid4().hex}`（216）。

### 5.9 会话消毒 `sanitize_conversation_messages`（119-171）

这是恢复会话（从磁盘加载历史）时的**安全网**，docstring 点明动机：中断的会话会残留"assistant tool_use 消息没有对应的 user tool_result 回复"，这类残缺尾巴会让 OpenAI 兼容 provider 拒绝恢复后的会话。算法是一个"待匹配 tool_use 集合"状态机：

```
sanitized, pending_tool_use_ids=∅, pending_tool_use_index=None  （128-130）
for message in messages:
  1) assistant 且 is_effectively_empty → 整条丢弃（133-134）
  2) 计算 tool_uses（assistant）与 tool_results（user）（136-139）
  3) 若存在 pending：
       - 下一条不是 user，或 user 的 result_ids 不覆盖 pending 全集
         → pop 掉那段 pending 的 assistant 消息，清空 pending（144-148）
       - 覆盖 → matched=True，清空 pending（149-152）
  4) user 消息带 tool_result 但未被匹配 → 剥掉 ToolResultBlock；
     剥完为空 → 整条丢弃；否则重建 user 消息（154-160）
  5) append；若本消息是 assistant 且有 tool_uses → 登记为新的 pending（162-166）
循环结束仍有 pending → pop 掉对应 assistant 消息（168-169）
```

**结果**：恢复后的历史保证每个 `tool_use` 都有配对的 `tool_result`，且不会残留空 assistant 尾巴。

---

## 六、流式事件全部类型（stream_events.py）

### 6.1 引擎对外事件（全部 `@dataclass(frozen=True)`，不可变可哈希）

| 事件 | 行号 | 字段（类型/默认） | 语义 | 被 emit 的位置（query.py） |
| --- | --- | --- | --- | --- |
| `AssistantTextDelta` | 12-16 | `text: str` | 模型输出文本增量 | 739（从 `ApiTextDeltaEvent` 转译） |
| `AssistantTurnComplete` | 19-24 | `message: ConversationMessage`, `usage: UsageSnapshot` | 一个 assistant 回合完成 | 803 |
| `ToolExecutionStarted` | 27-32 | `tool_name: str`, `tool_input: dict[str, Any]` | 即将执行工具 | 824（单工具）、844（多工具循环） |
| `ToolExecutionCompleted` | 35-42 | `tool_name: str`, `output: str`, `is_error: bool = False`, `metadata: dict\|None = None` | 工具执行完毕 | 834-839（单）、873（多循环） |
| `ErrorEvent` | 45-50 | `message: str`, `recoverable: bool = True` | 应向用户展示的错误 | 779（网络）、781（API）、794（空 assistant 消息） |
| `StatusEvent` | 53-57 | `message: str` | 一次性系统状态（自动压缩、重试、max_tokens 截断、图片预处理） | 704（截断）、742-747（重试）、760-765 / 770（降级/压缩）、592（图片预处理） |
| `CompactProgressEvent` | 60-79 | 见下 | 压缩的结构化进度 | 689、695（`_stream_compaction` 转发自压缩服务的 `progress_callback`）；`CompactProgressEvent` 原料由 `services/compact` 内部构造 |

`CompactProgressEvent` 字段（60-79）：

| 字段 | 类型 | 默认 | 含义 |
| --- | --- | --- | --- |
| `phase` | `Literal[...]`（64-74） | 无 | 压缩阶段，见下 |
| `trigger` | `Literal["auto","manual","reactive"]` | 无 | 触发方式 |
| `message` | `str \| None` | `None` | 阶段说明 |
| `attempt` | `int \| None` | `None` | 第几次尝试 |
| `checkpoint` | `str \| None` | `None` | 检查点标识 |
| `metadata` | `dict\|None` | `None` | 附加数据 |

`phase` 的 9 个取值（64-74）：`hooks_start`、`context_collapse_start`、`context_collapse_end`、`session_memory_start`、`session_memory_end`、`compact_start`、`compact_retry`、`compact_end`、`compact_failed`——覆盖一次压缩生命周期的每个阶段。

`StreamEvent`（82-90）是以上 7 个事件的联合类型，即 `QueryEngine.submit_message` / `continue_pending` 的 yield 类型。

### 6.2 API 层内部事件（client.py）

引擎层之下、client 之上还有一层 **API 事件**（`ApiStreamEvent = ApiTextDeltaEvent | ApiMessageCompleteEvent | ApiRetryEvent`，client.py:77）：

| 事件 | 行号 | 字段 | 语义 |
| --- | --- | --- | --- |
| `ApiTextDeltaEvent` | 51-55 | `text: str` | 原始文本增量（每个客户端各自产出） |
| `ApiMessageCompleteEvent` | 58-64 | `message`, `usage`, `stop_reason: str\|None=None` | 完整消息 + 用量 + 停止原因 |
| `ApiRetryEvent` | 67-74 | `message: str`, `attempt: int`, `max_attempts: int`, `delay_seconds: float` | 自动重试通知 |

### 6.3 双层事件转译关系

```
各 client 内部                    ApiStreamEvent（client.py）
  Anthropic  → ApiTextDeltaEvent / ApiMessageCompleteEvent / ApiRetryEvent
  OpenAI     → 同上（额外用 -> 内部把 reason 存 final_message._reasoning）
  Codex      → 同上
        │
        ▼  run_query 的 isinstance 转译（query.py:738-752 / 803）
  引擎事件 StreamEvent（stream_events.py）
  ApiTextDeltaEvent        → AssistantTextDelta（文本增量，即时转发）
  ApiRetryEvent            → StatusEvent（“重试中”提示）
  ApiMessageCompleteEvent  → 暂存 → 回合末 yield AssistantTurnComplete
```

API 事件是"传输层协议细节"（重试、delta），由每个 client 自由实现；引擎事件是"业务层语义"（工具开始/结束、压缩进度），对 UI 稳定。中间经 `run_query` 转译解耦，新增 provider 无需知道 UI 事件模型。

---

## 七、API 客户端层逐方法拆解

### 7.0 协议前提

`SupportsStreamingMessages`（client.py:80-85）只是结构型 `Protocol`：

```python
class SupportsStreamingMessages(Protocol):
    async def stream_message(self, request: ApiMessageRequest) -> AsyncIterator[ApiStreamEvent]: ...
```

引擎只依赖这一个方法。四个客户端（`AnthropicApiClient`、`OpenAICompatibleClient`、`CodexApiClient`、`CopilotClient`）都满足它。

`ApiMessageRequest`（39-48，frozen dataclass）：

| 字段 | 类型 | 默认 |
| --- | --- | --- |
| `model` | `str` | 必填 |
| `messages` | `list[ConversationMessage]` | 必填 |
| `system_prompt` | `str \| None` | `None` |
| `max_tokens` | `int` | `4096` |
| `tools` | `list[dict]` | `field(default_factory=list)`（空列表） |
| `effort` | `str \| None` | `None` |

---

### 7.1 AnthropicApiClient（client.py:118-262）

**构造 `__init__`（121-136）**：五种输入 `api_key` / `auth_token`（keyword） / `base_url` / `claude_oauth: bool=False` / `auth_token_resolver: Callable[[], str]|None`。`claude_oauth=True` 时读取 `get_claude_code_session_id()` 记录 `_session_id`（135）。立即构造底层 SDK 客户端 `_create_client()`（136）。

**`_create_client`（138-151）**：
- 有 `api_key` → `kwargs["api_key"]`；
- 有 `auth_token` → `kwargs["auth_token"]`，且 `default_headers` 二选一：OAuth 模式用 `claude_oauth_headers()`（来自 `openharness.auth.external`），否则 `{"anthropic-beta": "oauth-2025-04-20"}`；
- 有 `base_url` → `kwargs["base_url"]`；最后 `AsyncAnthropic(**kwargs)`。

**`close`（153-155）**：关底层 http client。

**`_refresh_client_auth`（157-163）**：仅 OAuth 模式下，用 `auth_token_resolver()` 取最新 token，与本地不一致则重建整个 `AsyncAnthropic` 客户端（token 轮换场景）。

**`stream_message`（165-201）重试外壳**：
- `for attempt in range(MAX_RETRIES + 1)`（总量 4 次）；
- 每次先 `_refresh_client_auth()`（171）；
- `except OpenHarnessApiError: raise`（175-176）——**认证类错误不重试**；
- 其它异常：`attempt >= MAX_RETRIES or not _is_retryable(exc)` → 包装抛出；否则算退避、打日志、`yield ApiRetryEvent(message, attempt+1, max_attempts=4, delay_seconds)`、`await asyncio.sleep(delay)`（177-196）；
- 循环外的收尾（198-201）同样把 `last_error` 翻译后抛出。

**`_is_retryable`（87-95）**：
- `APIStatusError` → `exc.status_code in {429,500,502,503,529}`；
- 其它 `APIError`（网络类）→ True；
- `(ConnectionError, TimeoutError, OSError)` → True；
- 否则 False。

**`_get_retry_delay`（98-115）**：
- 若为 `APIStatusError` 且响应头带 `Retry-After` → `min(float(retry_after), 30.0)`（**优先尊重服务端指示**）；
- 否则 `delay = min(1.0 * 2**attempt, 30.0)`，再加 `random.uniform(0, delay * 0.25)` 的 25% 抖动。

**`_stream_once`（203-262）单次流式**：
- 组装 `params`：`model`、`messages=[m.to_api_param() for m in request.messages]`、`max_tokens`、若 system_prompt 则 `params["system"]`（205-211）；
- OAuth 追加（212-233）：system 前拼接 `claude_attribution_header()` 前缀；`params["betas"]=claude_oauth_betas()`；`metadata={"user_id": json.dumps({device_id:"openharness", session_id, account_uuid:""}, separators=(",",":"))}`；`extra_headers={"x-client-request-id": str(uuid.uuid4())}`；
- 选 client API（236）：OAuth → `self._client.beta.messages.stream`，否则 `self._client.messages.stream`；
- 只转发 `type == "content_block_delta"` 且 `delta.type == "text_delta"` 的文本（238-246），其余事件（thinking delta、content_block_start 等）一律忽略；
- 结束 `stream.get_final_message()`（248）→ `assistant_message_from_api` 转回模型层 → `UsageSnapshot(input_tokens, output_tokens)`（254-261）→ `yield ApiMessageCompleteEvent`；
- `except APIError`（249-252）：可重试状态码原地 `raise` 交回外层，其余 `raise _translate_api_error(exc)`。

**`_translate_api_error`（265-271）错误分类**：

| SDK 异常类名 | 映射 |
| --- | --- |
| `AuthenticationError` / `PermissionDeniedError` | `AuthenticationFailure` |
| `RateLimitError` | `RateLimitFailure` |
| 其它 | `RequestFailure` |

---

### 7.2 OpenAICompatibleClient（openai_client.py:260-484）

**构造（267-277）**：强制 `default_headers={"Authorization": f"Bearer {api_key}"}`；`_normalize_openai_base_url`（244-257）规整 base_url——保留 path 段（不清空），无 path 时补 `/v1`（例如传 `https://dashscope.aliyuncs.com` → `.../compatible-mode/v1` 原样保留，空 path 补 `/v1`）；可选 `timeout`。

**字段与非公开方法**：

- `_token_limit_param_for_model(model, max_tokens)`（46-57）：模型名没有 `provider/` 前缀时按小写 prefix 判断；前缀命中 `("gpt-5","o1","o3","o4")` → `{"max_completion_tokens": n}`，否则 `{"max_tokens": n}`。**原因**：GPT-5 与当前 reasoning 家族拒绝 `max_tokens` 字段，必须用 `max_completion_tokens`。
- `_convert_tools_to_openai`（60-78）：**工具 schema 转换**（Anthropic → OpenAI）：

```
Anthropic: {"name","description","input_schema"}
OpenAI:    {"type":"function","function":{"name","description","parameters"}}
```

实现：`content` 外层包 `{"type":"function","function":{name, description, parameters}}`，`input_schema` 直接改名为 `parameters`（保持 JSON Schema 结构相同）。

- `_convert_messages_to_openai`（81-126）：**消息转换核心**（详见 7.2.1 映射规则大表与第九节逐行注释）。
- `_convert_user_content_to_openai`（129-146）：user 的 text/image 块 → OpenAI content。无图时拼成单个字符串；有图时变成数组，图是 `{"type":"image_url","image_url":{"url": f"data:{media_type};base64,{data}"}}`（data URI）。
- `_convert_assistant_message`（162-217）：assistant 消息转换，含 **reasoning_content 回放逻辑**（196-202）：流式解析存下来的 `msg._reasoning` 非空时总是回放；没有 reasoning 但有工具调用且环境变量 `OPENHARNESS_REQUIRE_EMPTY_REASONING_CONTENT` 开启（`_empty_reasoning_required`，152-159，接受 `1/true/yes/on`）时才写 `reasoning_content:""`，否则**省略该字段**。`tool_calls` 序列化为 `[{id, type:"function", function:{name, arguments: json.dumps(input)}}]`（204-215）。空文本时 `content=None`（193）。
- `_parse_assistant_response`（220-241）：非流式场景把 ChatCompletion 转回 `ConversationMessage`（`choices[0].message`；`tool_calls[].function.arguments` 用 `json.loads` 解析，坏 JSON 兜底 `{}`）。
- `_strip_think_blocks`（461-484）跨 chunk ` thinking… response` 剥除（见 7.2.3）。

**`stream_message`（283-313）重试外壳**：与 Anthropic 同构，`for range(4)`；`except OpenHarnessApiError: raise`（不重试）；否则 `attempt>=3 or not self._is_retryable` → `raise self._translate_error`；退避 `min(1.0*2**attempt, 30)` 无 Retry-After 支持；每次重试前 `yield ApiRetryEvent`。

**`_stream_once`（315-434）单次流式**：

```
317-318: openai_messages = _convert_messages_to_openai(...) ; openai_tools = _convert_tools_to_openai(...) or None
320-325: params = {model, messages, stream:True, stream_options:{include_usage:True}}
326: params.update(_token_limit_param_for_model(...))
327-333: if openai_tools: params["tools"]=openai_tools; params.pop("stream_options", None)
        # ★ 关键怪癖：有工具时剔除 stream_options——
        # Kimi 等 provider 若带 include_usage 会触发模型侧 thinking 模式，
        # 从而要求每条 assistant 消息都有 reasoning_content，代价太高
336-342: 初始化收集变量：collected_content / collected_reasoning / collected_tool_calls: dict[int,...] /
        finish_reason / usage_data / _think_buf
344: stream = await self._client.chat.completions.create(**params)   # 异步流
345-398: async for chunk in stream:
  346-353: 无 choices 的 usage-only chunk → 提取 usage → continue
  355-359: delta & finish_reason 累积
  362-364: reasoning_content 累计到 collected_reasoning
  367-372: delta.content 进 _think_buf → _strip_think_blocks → 可见文本立即 yield ApiTextDeltaEvent
  375-391: tool_calls 按 index 累积（id / name / arguments 分片拼装）
  394-398: chunk.usage → usage_data（部分 provider 只在末尾给）
400-420: 组装最终 ConversationMessage：文本块 + 按 index 排序、跳过无 name 的“幽灵”tool_call、
        arguments 用 json.loads（坏→{}）
424-425: collected_reasoning 非空 → final_message._reasoning = ...  # 供下一轮回放
427-434: yield ApiMessageCompleteEvent(message, UsageSnapshot(usage_data), stop_reason=finish_reason)
```

**`_is_retryable`（436-443）**：`status_code` 在 `{429,500,502,503}`，或 `(ConnectionError, TimeoutError, OSError)`。

**`_translate_error`（445-453）**：`status==401 or 403` → `AuthenticationFailure`；`status==429` → `RateLimitFailure`；否则 `RequestFailure`。

#### 7.2.1 Anthropic → OpenAI 消息映射规则总表

| Anthropic 概念 | OpenAI 表示 | 实现位置 |
| --- | --- | --- |
| 顶层 `system` 参数 | 首条 `{"role":"system","content":...}` | 95-96 |
| assistant 文本块 | `{"role":"assistant","content": "<拼接文本>"}`（空则 None） | 192-193 |
| assistant 的 `tool_use` 块 | `tool_calls: [{id, type:"function", function:{name, arguments(JSON字符串)}}]` | 204-215 |
| user 的 `tool_result` 块 | **独立** `{"role":"tool","tool_call_id":<tool_use_id>,"content":...}` 消息，每个一块 | 107-114 |
| user 文本/图片块 | 无图 → 单字符串；有图 → `[{type:text,...},{type:image_url, image_url:{url: data URI}}]` | 115-121, 129-146 |
| `tools` schema | `[{type:"function", function:{name, description, parameters}}]`（`input_schema`→`parameters`） | 60-78 |
| `max_tokens` | `max_tokens` 或 `max_completion_tokens`（gpt-5/o1/o3/o4 前缀） | 46-57 |
| usage | `chunk.usage.prompt_tokens / completion_tokens`（`include_usage`） | 348-352, 394-398 |
| stop_reason | `choices[0].finish_reason` 原地透传 | 356-359 |

**注意**：Anthropic 里 `tool_use` 与后续文本都在一条 assistant 消息内；OpenAI 里 tool 结果被拆成独立 `role="tool"` 消息（每条之间无位置自由度），且 `tool` 消息**必须紧跟**对应的 `tool_calls` assistant 消息——这也解释了为什么 `sanitize_conversation_messages` 要保证 tool_use 一定有配对 tool_result。

#### 7.2.2 工具 schema 转换示例

```python
# 入参（Anthropic schema）
{"name": "bash", "description": "Run a shell command",
 "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}}
# 出参（OpenAI schema）
{"type": "function",
 "function": {"name": "bash", "description": "Run a shell command",
              "parameters": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}}}
```

#### 7.2.3 `_strip_think_blocks`（461-484）流式剥除

- `_THINK_RE = re.compile(r"地...?response", re.DOTALL)`（457）——正则删掉完整 `<thinking>…</response>`；
- `open_idx = cleaned.find("<thinking>")`：**未闭合**的 `<thinking>` 前截断，剩余保留在 leftover 供下个 chunk；
- 466-482 处理"开标签本身被跨 chunk 切开"（如 `"<thi"` + `"nk>"`）：找最长能成为 `<thinking>` 前缀的后缀，hold 住；
- 返回 `(可见文本, 暂存缓冲)`。

---

### 7.3 CodexApiClient（codex_client.py:221-407）

不走 OpenAI SDK，直接 **httpx + SSE** 打 ChatGPT Codex Responses 接口。

**构造（224-227）**：`auth_token` + 可选 `base_url` → `_resolve_codex_url(base_url)`。

**模块级辅助**：

- `_extract_account_id(token)`（30-46）：解码 JWT payload（`base64.urlsafe_b64decode` + padding 补齐），取 `payload["https://api.openai.com/auth"]["chatgpt_account_id"]`；JWT 非法/缺字段一律 `AuthenticationFailure`。**服务端用 account_id 关联订阅身份**。
- `_resolve_codex_url`（49-58）：规整到 `.../codex/responses`——已带 `/codex/responses` 原样、带 `/codex` 补 `/responses`、否则拼 `codex/responses`；只要 base_url 不含 `chatgpt.com/backend-api` 就退回默认值。
- `_build_codex_headers`（61-74）：`Authorization: Bearer`、`chatgpt-account-id`、`originator: openharness`、`User-Agent: openharness (<os> <machine>)`、`OpenAI-Beta: responses=experimental`、`accept: text/event-stream`、`content-type: application/json`、可选 `session_id`。
- `_normalize_reasoning_effort`（136-142）：`max` → `"xhigh"`（Codex 没有 max 档）。
- `_usage_from_response`（145-152）：响应体 `usage.input_tokens/output_tokens`。
- `_stop_reason_from_response`（155-165）：`completed`+有工具 → `tool_use`；`completed` → `stop`；`incomplete` → `length`；`failed/cancelled` → `error`。
- `_format_error_message`（168-185）：从错误 body 或 detail 提取人类可读信息。
- `_format_codex_stream_error`（188-210）：从 SSE error 事件里取 `message`/`code`/`request_id` 拼成错误串。
- `_translate_status_error`（213-218）：`401/403` → AuthenticationFailure；`429` → RateLimitFailure；否则 RequestFailure。

**`stream_message`（229-251）重试外壳**：`for range(4)`；`except Exception as exc` 都不直接重抛，`if attempt>=3 or not self._is_retryable(exc)` → `raise self._translate_error(exc)`；退避 `min(1.0*2**n, 30)`；重试前 `yield ApiRetryEvent`；循环外翻译 `last_error`。

**`_stream_once`（253-356）**：
- **请求体**（254-264）：`model`、`store:False`、`stream:True`、`instructions`（system，缺省 `"You are OpenHarness."`）、`input=_convert_messages_to_codex(messages)`、`text:{"verbosity":"medium"}`、`include:["reasoning.encrypted_content"]`、`tool_choice:"auto"`、`parallel_tool_calls:True`；
- `tools` 存在 → `_convert_tools_to_codex`（直通 `{name, description, parameters}`，124-133）；`effort` 归一化后进 `body["reasoning"]["effort"]`（267-269）；
- 建 httpx 异步客户端：`timeout=60`、`follow_redirects=True`（276）；
- 状态码 ≥400 → 读 body 格式化后 `raise httpx.HTTPStatusError`（278-281）；
- SSE 循环（283-341）按事件类型分发：
  - `response.output_text.delta` → `current_text_parts.append(delta)` + `yield ApiTextDeltaEvent`；
  - `response.output_item.done` + `item.type=="message"` → 攒 `output_text`/`refusal` 拼 `TextBlock`；
  - `+ item.type=="function_call"` → `json.loads(arguments)`（坏 JSON→`{}`）→ `ToolUseBlock(id=call_id, name, input)`，**id 由服务端给的真实 call_id**；
  - `response.completed` → 存完整 response（供 usage/stop_reason）；
  - `response.failed` / `error` → `raise RequestFailure(...)`；
- 文本块兜底（343-344）：只攒了 delta 无 output_text.done 时手动 `insert(0, TextBlock(...))`；
- 收尾（346-356）：`AssistantTurnComplete` 前先 `yield ApiMessageCompleteEvent`（含 usage 与 stop_reason 归一化）。

**`_iter_sse_events`（358-383）**：按空行切分 SSE 帧，收集以 `data:` 开头的行（去前缀），跳过 `[DONE]`，`json.loads` 成 dict 后 yield；结尾残留帧同样处理。

**`_is_retryable`（385-396）**：
- `httpx.HTTPStatusError` → 状态在 `{429,500,502,503,504}`（多了 504）；
- `RateLimitFailure` → True（防御性）；
- `RequestFailure` → 消息文本含 `timeout/connect/network/rate/overloaded` 之一；
- `httpx` 的 `TimeoutException`/`NetworkError` → True。

**`_translate_error`（398-406）**：`OpenHarnessApiError` 原样透传；`HTTPStatusError` → `_translate_status_error`；其它 `httpx.HTTPError` → `RequestFailure`；兜底 `RequestFailure`。

---

### 7.4 CopilotClient（copilot_client.py:48-134）

`CopilotClient` 是**组合而非继承**的轻量封装：

**构造（67-110）**：
- `load_copilot_auth()`（copilot_auth.py:111-127）读持久化 auth；`token = github_token or auth.github_token`；两者都没有 → `raise AuthenticationFailure("No GitHub Copilot token found. Run 'oh auth copilot-login' first.")`（77-79）；
- `ent_url = enterprise_url or auth.enterprise_url`（82）；
- `base_url = copilot_api_base(ent_url)`（89）——公开 GitHub 是 `https://api.githubcopilot.com`（copilot_auth.py:48-57）；
- `raw_openai = AsyncOpenAI(api_key=token, base_url=base_url, default_headers={"User-Agent":"openharness/0.1.0", "Openai-Intent":"conversation-edits"})`（94-98）；
- `self._inner = OpenAICompatibleClient(api_key=token, base_url=base_url)`（99-102）；
- **关键 hack（103-104）**：`self._inner._client = raw_openai` —— 侵入私有属性 `_client`，把构造出来的带 Copilot header 的 `AsyncOpenAI` 塞进内部客户端。**为什么能这么干**：`OpenAICompatibleClient` 的消息/工具转换逻辑完全在 `_stream_once` 的前半段，底层 `_client` 只负责 `chat.completions.create`，两者正交，所以只换 `_client` 就能复用全部转换逻辑。

**`stream_message`（112-130）**：构造期指定的 `model` 覆盖请求里的 model（`effective_model = self._model or request.model`），其余字段原样透传构造新的 `ApiMessageRequest`，委托 `self._inner.stream_message(patched)`（注意**丢掉 effort**——`ApiMessageRequest` 重新构造时没有传 effort 字段）。

**`close`（132-134）**：关内部 client。

**`copilot_auth`（copilot_auth.py，OAuth device flow）**：

- `copilot_api_base(enterprise_url=None)`（48-57）：
  - 有 enterprise → 剥离 `https://`/`http://`，`https://copilot-api.<domain>`；
  - 否则 `COPILOT_DEFAULT_API_BASE = "https://api.githubcopilot.com"`（40）。
- `DeviceCodeResponse`（65-73 frozen dataclass）：`device_code` / `user_code` / `verification_uri` / `interval` / `expires_in`。
- `CopilotAuthInfo`（76-85）：`github_token` + `enterprise_url=None`，`api_base` property 委托 `copilot_api_base`。
- `save_copilot_auth`（97-108）：`atomic_write_text`（原子写）+ `mode=0o600`，落到 `get_config_dir()/copilot_auth.json`（`_AUTH_FILE_NAME="copilot_auth.json"`，45；`_auth_file_path` 93-94）。
- `load_copilot_auth`（111-127）：读 key `github_token`，缺文件/坏 JSON/缺 key → 返回 `None`（降级不抛）。
- 兼容别名（131-145）：`save_github_token = save_copilot_auth`；`load_github_token()`；`clear_github_token()`（unlink）。
- `request_device_code`（153-177）：POST `https://{github_domain}/login/device/code`，body `{"client_id","scope":"read:user"}`，timeout 30s；返回 `DeviceCodeResponse`（interval 默认 5s，expires_in 默认 900s）。
- `poll_for_access_token`（180-240）：
  - 每次轮询前 `time.sleep(poll_interval + 3.0)`——`_POLL_SAFETY_MARGIN=3.0`（43）防止触发服务端限速；
  - `progress_callback(poll_number, elapsed)` 可选回调（205-206）；
  - 响应带 `access_token` → 立即返回（223-224）；
  - `error=="authorization_pending"` → 继续轮询（227-228）；
  - `error=="slow_down"` → 用服务端新 interval，否则 `+=5.0`（229-235）；
  - 其它错误 → `raise RuntimeError`（237-238）；
  - 超时（默认 900s）→ `RuntimeError`（240）。

---

### 7.5 重试与退避策略汇总

| Client | 最大尝试 | 可重试状态码 | 退避 | Retry-After | 认证错误 |
| --- | --- | --- | --- | --- | --- |
| AnthropicApiClient（client.py:31-35, 87-115） | 4 | `{429,500,502,503,529}` + `APIError` 网络类 + `ConnectionError/TimeoutError/OSError` | `min(1.0×2^n, 30)` + 25% 抖动 | **支持**（≤30s） | `OpenHarnessApiError` 不重试直接抛 |
| OpenAICompatibleClient（openai_client.py:40-42, 436-443） | 4 | `{429,500,502,503}` + `ConnectionError/TimeoutError/OSError` | `min(1.0×2^n, 30)` | 无 | `OpenHarnessApiError` 不重试直接抛 |
| CodexApiClient（codex_client.py:25-27, 385-396） | 4 | `{429,500,502,503,504}` + RateLimitFailure + RequestFailure 关键字 + httpx 网络类 | `min(1.0×2^n, 30)` | 无 | 经 `_translate_error` 原样透传（不重试） |
| CopilotClient | 继承 OpenAICompatibleClient | 同上 | 同上 | 无 | 同上 |

### 7.6 错误分类矩阵（`errors.py`：`AuthenticationFailure`/`RateLimitFailure`/`RequestFailure` 如何产生）

| 上游信号 | Anthropic | OpenAI | Codex |
| --- | --- | --- | --- |
| `401/403` | `AuthenticationError`/`PermissionDeniedError` → Auth | `_translate_error` 401/403 → Auth | `_translate_status_error` 401/403 → Auth |
| `429` | 先重试；最终 `RateLimitError` → RateLimit | 先重试；最终 429 → RateLimit | 先重试；最终 429 → RateLimit |
| `500/502/503` | 重试；最终 Request（`APIStatusError` 翻译走 `_translate_api_error`） | 重试；最终 Request | 重试；最终 Request |
| `504` | 不在集合（不重试） | 不在集合（不重试） | **可重试** |
| 网络类（连接/超时/OSError） | 可重试；最终 `RequestFailure` | 可重试；最终 `RequestFailure` | 可重试；最终 `RequestFailure` |
| 其它 | `RequestFailure` | `RequestFailure` | `RequestFailure`（`_translate_error` 兜底） |

---

## 八、Provider 注册表（registry.py）

### 8.1 ProviderSpec 逐字段（17-48）

```python
@dataclass(frozen=True)
class ProviderSpec:
    # Identity
    name: str                # 规范名，如 "dashscope"（探测/配置的唯一 key）
    keywords: tuple[str, ...]  # 模型名关键字（小写），用于第 3 级探测
    env_key: str             # 主 API key 环境变量名（如 "DASHSCOPE_API_KEY"）
    display_name: str = ""   # 展示名（status/诊断用），空则用 name.title()
    # Routing
    backend_type: str = "openai_compat"   # "anthropic" | "openai_compat" | "copilot"
    default_base_url: str = ""            # 该 provider 的默认 base_url
    # Auto-detection signals
    detect_by_key_prefix: str = ""        # 第 1 级：api_key 前缀，如 "sk-or-" → OpenRouter
    detect_by_base_keyword: str = ""      # 第 2 级：base_url 子串，如 "aihubmix"
    # Classification flags
    is_gateway: bool = False              # 网关：可路由任意模型（OpenRouter/AiHubMix/…）
    is_local: bool = False                # 本地部署（vLLM/Ollama）
    is_oauth: bool = False                # OAuth 认证（Copilot）
```

`label` property（47-48）：`display_name or name.title()`。

文件头注释（1-10）明确扩展方式：**新增 provider 只需在 `PROVIDERS` 加一个 `ProviderSpec`，探测、展示、配置全部自动派生**。顺序即优先权——网关/云优先 → 标准云 → 云平台 → 本地。

### 8.2 `PROVIDERS` 元组清单（55-368，共 **22 条**）

| # | spec | 行号 | backend | key 前缀 | base 关键字 | 标志 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | github_copilot | 57-69 | copilot | — | — | oauth |
| 2 | openrouter | 72-84 | openai_compat | `sk-or-` | `openrouter` | gateway，默认 URL `openrouter.ai/api/v1` |
| 3 | aihubmix | 86-98 | openai_compat | — | `aihubmix` | gateway，`aihubmix.com/v1` |
| 4 | siliconflow | 100-112 | openai_compat | — | `siliconflow` | gateway，`api.siliconflow.cn/v1` |
| 5 | volcengine | 114-126 | openai_compat | — | `volces` | gateway，keywords=(volcengine,volces,ark) |
| 6 | modelscope | 128-140 | openai_compat | — | `modelscope` | `api-inference.modelscope.cn/v1` |
| 7 | anthropic | 143-155 | anthropic | — | — | keywords=(anthropic,claude)，`ANTHROPIC_API_KEY` |
| 8 | openai | 157-169 | openai_compat | — | — | keywords=(openai,gpt,o1,o3,o4) |
| 9 | deepseek | 171-183 | openai_compat | — | `deepseek` | `api.deepseek.com/v1` |
| 10 | gemini | 185-197 | openai_compat | — | `googleapis` | `generativelanguage.googleapis.com/v1beta/openai` |
| 11 | dashscope | 199-211 | openai_compat | — | `dashscope` | keywords=(qwen,dashscope) |
| 12 | moonshot | 213-225 | openai_compat | — | `moonshot` | keywords=(moonshot,kimi) |
| 13 | minimax | 227-239 | openai_compat | — | `minimax` | `api.minimax.io/v1` |
| 14 | zhipu | 241-253 | openai_compat | — | `bigmodel` | keywords=(zhipu,glm,chatglm) |
| 15 | groq | 255-267 | openai_compat | `gsk_` | `groq` | `api.groq.com/openai/v1` |
| 16 | mistral | 269-281 | openai_compat | — | `mistral` | keywords=(mistral,mixtral,codestral) |
| 17 | stepfun | 283-295 | openai_compat | — | `stepfun` | keywords=(step-,stepfun) |
| 18 | baidu | 297-309 | openai_compat | — | `baidubce` | keywords=(ernie,baidu)，env `QIANFAN_ACCESS_KEY` |
| 19 | bedrock | 312-324 | openai_compat | — | `bedrock` | env `AWS_ACCESS_KEY_ID` |
| 20 | vertex | 326-338 | openai_compat | — | `aiplatform` | env `GOOGLE_APPLICATION_CREDENTIALS` |
| 21 | ollama | 341-353 | openai_compat | — | `localhost:11434` | local，`http://localhost:11434/v1` |
| 22 | vllm | 355-367 | openai_compat | — | — | local，keywords=(vllm) |

### 8.3 多级探测 `detect_provider_from_registry`（408-437）——含逐行注释

```python
def detect_provider_from_registry(
    model: str,
    api_key: str | None = None,
    base_url: str | None = None,
) -> ProviderSpec | None:
    """Detection priority:
       1. api_key prefix  (e.g. "sk-or-" → OpenRouter)
       2. base_url keyword (e.g. "aihubmix" in URL → AiHubMix)
       3. model name keyword (e.g. "qwen" → DashScope)
    """
    # 1. api_key 前缀探测 ---------------------------------------------
    if api_key:
        for spec in PROVIDERS:                                    # 按 PROVIDERS 顺序
            if spec.detect_by_key_prefix and api_key.startswith(spec.detect_by_key_prefix):
                return spec                                       # 命中即返回（前缀最具区分度）
    # 2. base_url 关键字探测 -----------------------------------------
    if base_url:
        base_lower = base_url.lower()
        for spec in PROVIDERS:
            if spec.detect_by_base_keyword and spec.detect_by_base_keyword in base_lower:
                return spec                                       # 子串匹配（不区分大小写）
    # 3. 模型名关键字探测 --------------------------------------------
    if model:
        return _match_by_model(model)
    return None
```

**`_match_by_model`（384-405，第三级内部）**：

```python
model_lower = model.lower()
model_normalized = model_lower.replace("-", "_")               # "-" → "_" 归一化
model_prefix = model_lower.split("/", 1)[0] if "/" in model_lower else ""  # 显式 provider 前缀
normalized_prefix = model_prefix.replace("-", "_")
std_specs = [s for s in PROVIDERS if not s.is_local and not s.is_oauth]

# a) 优先显式 provider 前缀："deepseek/xxx" → deepseek spec
for spec in std_specs:
    if model_prefix and normalized_prefix == spec.name:
        return spec
# b) 关键字扫描：kw 命中两种形态（原始小写 / "-"↔"_" 归一后）
for spec in std_specs:
    if any(kw in model_lower or kw.replace("-", "_") in model_normalized for kw in spec.keywords):
        return spec
return None
```

### 8.4 `find_by_name`（376-381）

线性扫 `PROVIDERS`，按 `spec.name` 精确匹配规范名（如 `"dashscope"`）。

### 8.5 provider.py 能力判断

- `_AUTH_KIND`（13-19）：`anthropic→api_key`、`openai_compat→api_key`、`copilot→oauth_device`、`openai_codex→external_oauth`、`anthropic_claude→external_oauth`。
- `ProviderInfo`（32-39 frozen dataclass）：`name` / `auth_kind` / `voice_supported` / `voice_reason`。
- `detect_provider(settings)`（42-94）优先级：
  1. `settings.provider == "openai_codex"` → name `openai-codex`、external_oauth（44-50）；
  2. `== "anthropic_claude"` → `claude-subscription`（51-57）；
  3. `settings.api_format == "copilot"` → `github_copilot`（58-64）；
  4. 否则 `detect_provider_from_registry(model, api_key, base_url)`（66-70）；命中则取 `spec.backend_type` 映射 `_AUTH_KIND`（72-79）；
  5. 兜底：`api_format=="openai"` → `openai-compatible`，否则 `anthropic`（82-94）。
- `auth_status(settings)`（97-126）：生成 CLI/状态面板用的紧凑认证状态串。`copilot` 读 `load_copilot_auth()`，缺 → `missing (run 'oh auth copilot-login')`；有 enterprise 显示 `configured (enterprise: …)`；非 copilot 走 `settings.resolve_auth()`，捕获 `ValueError` 分支：codex → `missing (run 'oh auth codex-login')`；claude → 查外部绑定（含 `invalid base_url` 特判）；`resolved.source.startswith("external:")` → `configured (external:)...`。
- `is_model_multimodal(model)`（175-186）：大小写归一、去掉 `provider/` 前缀后，用 **23 条正则**（135-172）判断。覆盖 Claude 3+/Claude NX、GPT-4o、`o[1349]-`、Gemini、Qwen-VL/QvQ、DeepSeek-VL、LLaVA/CogVLM/InternVL/GLM-4V、Kimi-k2.5、Step-2/1v、MiniMax-VL、Pixtral、`vision`、通用 `vl` 词边界。docstring 明说"宁可误报不支持"（这样会走 image_to_text 回退而不是静默失败）。

---

## 九、真实代码片段逐行注释

### 片段一：run_query 的工具执行 if 分支（query.py:819-855）

```python
tool_calls = final_message.tool_uses                                   # 819  本回合模型请求的所有工具
                                                                       #      （ConversationMessage.tool_uses，messages.py:96-99）

if len(tool_calls) == 1:                                               # 821  单工具：走串行、即时上报路径
    tc = tool_calls[0]                                                 # 822  唯一的 ToolUseBlock
    yield ToolExecutionStarted(tool_name=tc.name, tool_input=tc.input), None   # 824  先广播“开始”
    try:                                                               # 825  _execute_tool_call 内部已拦大部分错误，
        result = await _execute_tool_call(context, tc.name, tc.id, tc.input)   # 826  这里再包一层兜底
    except Exception as exc:                                           # 827  连“程序性异常”（bug）也不放过
        log.exception("tool execution raised: name=%s id=%s",
                      tc.name, tc.id)                                  # 828  记录完整堆栈便于排障
        result = ToolResultBlock(                                      # 829  构造错误结果对象
            tool_use_id=tc.id,                                        # 830  ★ 必须用模型给的 tool_use id 配对，
            content=f"Tool {tc.name} failed: "
                    f"{type(exc).__name__}: {exc}",                   # 831  否则模型侧无法把 result 跟那次
            is_error=True,                                            # 832  tool_use 关联起来（Anthropic 直接拒收）
        )                                                             # 833
    yield ToolExecutionCompleted(                                     # 834  广播“完成”
        tool_name=tc.name,                                            # 835
        output=result.content,                                        # 836  回灌给 UI 的就是截断后的内联文本
        is_error=result.is_error,                                     # 837
        metadata=result.result_metadata,                              # 838  工具元数据（如 agent_id/task_id）
    ), None                                                           # 839
    tool_results = [result]                                           # 840  单元素列表，随后整体回灌
else:                                                                 # 841  多工具：并发执行、事后统一上报
    for tc in tool_calls:                                             # 843
        yield ToolExecutionStarted(tool_name=tc.name,
                                   tool_input=tc.input), None          # 844  先把所有“开始”都广播出去
                                                                       #      （单工具顺序上报；多工具先全 Start 再全 Complete）
    async def _run(tc):                                               # 846  包装器：只返回结果不抛异常
        return await _execute_tool_call(context, tc.name, tc.id, tc.input)  # 847
                                                                       # 848
    # 849-852 行是源码注释：说明 return_exceptions=True 的动机——
    # 单个工具失败不应把兄弟协程 cancel，否则会话里会留下没有对应
    # tool_result 的 tool_use，Anthropic 后续请求会被整段拒绝。
    raw_results = await asyncio.gather(                               # 853
        *[_run(tc) for tc in tool_calls], return_exceptions=True      # 854  并发执行全部工具
    )                                                                 # 855  异常会以返回值（BaseException）形式出现
```

### 片段二：`_convert_messages_to_openai`（openai_client.py:81-126）

```python
def _convert_messages_to_openai(
    messages: list[ConversationMessage],
    system_prompt: str | None,
) -> list[dict[str, Any]]:
    """Convert Anthropic-style messages to OpenAI chat format.
    Key differences:
    - Anthropic: system prompt is a separate parameter
    - OpenAI: system prompt is a message with role="system"
    - Anthropic: tool_use / tool_result are content blocks
    - OpenAI: tool_calls on assistant message, tool results are separate messages
    """
    openai_messages: list[dict[str, Any]] = []                        # 93  输出容器

    if system_prompt:                                                 # 95  Anthropic 顶层 system 参数
        openai_messages.append({"role": "system", "content": system_prompt})  # 96  → OpenAI 的首条 system 消息

    for msg in messages:                                              # 98  逐条会话消息
        if msg.role == "assistant":                                   # 99
            openai_msg = _convert_assistant_message(msg)              # 100 assistant 整体转换（含 tool_calls / reasoning）
            openai_messages.append(openai_msg)                        # 101
        elif msg.role == "user":                                      # 102
            # 用户消息可能同时含 文本/图片 与 tool_result 块
            tool_results = [b for b in msg.content
                            if isinstance(b, ToolResultBlock)]        # 104  把 tool_result 块单独捞出
            user_blocks = [b for b in msg.content
                           if isinstance(b, (TextBlock, ImageBlock))] # 105  其余文本/图块

            if tool_results:                                          # 107
                # 每个工具结果都是一条独立 role="tool" 消息
                for tr in tool_results:                               # 109
                    openai_messages.append({                          # 110
                        "role": "tool",                               # 111  固定角色 tool
                        "tool_call_id": tr.tool_use_id,              # 112  ★ 靠 tool_use_id 与 assistant 消息里的
                        "content": tr.content,                        # 113    tool_calls[].id 一一对应
                    })                                                # 114
            if user_blocks:                                           # 115
                content = _convert_user_content_to_openai(user_blocks)  # 116  文本/图 → str 或数组
                if isinstance(content, str):                          # 117
                    if content.strip():                               # 118  空白文本不产生空消息
                        openai_messages.append({"role": "user", "content": content})  # 119
                elif content:                                         # 120
                    openai_messages.append({"role": "user", "content": content})  # 121  多模态数组
            if not tool_results and not user_blocks:                   # 122
                # 空 user 消息兜底（正常不会发生，但保证结构合法）
                openai_messages.append({"role": "user", "content": ""})  # 124

    return openai_messages                                            # 126
```

### 片段三：`_offload_tool_output_if_needed`（query.py:524-553）

```python
def _offload_tool_output_if_needed(
    *,
    tool_name: str,
    tool_use_id: str,
    output: str,
) -> tuple[str, Path | None]:                                        # 529  返回 (内联文本, 落盘路径|None)
    inline_limit = tool_output_inline_chars()                          # 530  默认 16000 字符，环境可调（下限 256）
    if len(output) <= inline_limit:                                   # 531  没超上限
        return output, None                                           # 532  原样内联，不落盘

    artifact_path = (                                                 # 534
        _tool_artifact_dir()                                          # 535  get_data_dir()/tool_artifacts/（自动 mkdir）
        / f"{time.strftime('%Y%m%d-%H%M%S')}"                        # 536  文件名 = 时间戳 + 工具名 + uuid 前 12 hex
          f"-{_safe_tool_artifact_name(tool_name)}"
          f"-{uuid4().hex[:12]}.txt"
    )
    artifact_path.write_text(output, encoding="utf-8", errors="replace")  # 538  完整输出落盘（errors 防解码崩）
    preview = output[:tool_output_preview_chars()]                     # 539  截预览（默认 3000，下限 128）
    omitted = max(0, len(output) - len(preview))                       # 540  被省略的字符数

    inline = (                                                         # 541  构造占位文本
        "[Tool output truncated]\n"
        f"Tool: {tool_name}\n"                                        # 543  让模型知道来源工具
        f"Tool use id: {tool_use_id}\n"                               # 544  便于模型按 id 引用
        f"Original size: {len(output)} chars\n"                       # 545  原始体量
        f"Full output saved to: {artifact_path}\n"                    # 546  落盘路径（模型可读文件）
        f"Inline preview: first {len(preview)} chars"                 # 547
    )
    if omitted:                                                       # 549  有省略则注明
        inline += f" ({omitted} chars omitted)"                       # 550
    if preview:                                                       # 551
        inline += f"\n\nPreview:\n{preview}"                          # 552  预览正文拼在末尾
    return inline, artifact_path                                      # 553
```

---

## 十、关键调用链

### 10.1 正常对话（纯文本，无工具）

```
QueryEngine.submit_message("你好")                       query_engine.py:227
  ├─ remember_user_goal(tool_metadata, text)             query_engine.py:234-235  / query.py:204-215
  ├─ _prepare_session_memory()                           query_engine.py:236      / :155-168
  ├─ sanitize_conversation_messages(self._messages)      query_engine.py:237      / messages.py:119
  └─ run_query(QueryContext, query_messages)             query.py:633
       └─ context.api_client.stream_message(ApiMessageRequest)   query.py:728
            └─ AnthropicApiClient.stream_message          client.py:165
                 └─ _stream_once                          client.py:203
                      └─ AsyncAnthropic.messages.stream  （SDK → anthropic API）
            # 或 OpenAICompatibleClient._stream_once      openai_client.py:315 → AsyncOpenAI.chat.completions.create
            # 或 CodexApiClient._stream_once              codex_client.py:253 → httpx.stream(POST .../codex/responses)
       API 事件 → run_query isinstance 转译（738-752）→ AssistantTextDelta / AssistantTurnComplete + usage
  QueryEngine: AssistantTurnComplete → 同步历史；usage → CostTracker.add；yield 给 UI   query_engine.py:270-274
  finally: _update_session_memory → _extract_durable_memories → _schedule_auto_dream   query_engine.py:275-278
```

### 10.2 工具调用回合

```
模型返回带 tool_use 的 final_message
  run_query: len(tool_calls)==1 → 串行；>1 → asyncio.gather 并发     query.py:821-855
  _execute_tool_call(context, name, id, input)                        query.py:887
    ├─ PRE_TOOL_USE hook              → blocked? → 错误块             query.py:893
    ├─ ToolRegistry.get(name)         → None?   → "Unknown tool"      query.py:907  /tools/base.py:70
    ├─ input_model.model_validate     → 失败?   → "Invalid input"     query.py:917
    ├─ _resolve_permission_file_path / _extract_permission_command     query.py:929-930 / :1021 / :1045
    ├─ PermissionChecker.evaluate     → not allowed? → 确认/阻断       query.py:933   /checker.py:75
    ├─ tool.execute(parsed, ToolExecutionContext)                     query.py:970   /tools/base.py:43
    ├─ _offload_tool_output_if_needed → tool_artifacts/               query.py:985   /query.py:524
    ├─ _record_tool_carryover → tool_metadata 记忆桶                   query.py:998   /query.py:395
    └─ POST_TOOL_USE hook                                              query.py:1007
  结果 → ToolResultBlock
  messages.append(ConversationMessage(role="user", content=tool_results))   query.py:880
  下一轮 while → stream_message（含 tool_result）回灌                  query.py:728
  直到模型不再请求工具 → HookEvent.STOP → return                      query.py:808-817
```

### 10.3 权限交互路径

```
_execute_tool_call → PermissionChecker.evaluate（query.py:933 / checker.py:75）
  decision.allowed == False
   ├─ decision.requires_confirmation && permission_prompt
   │    → NOTIFICATION hook（query.py:942-951）→ await permission_prompt(tool_name, reason)（952）
   │    → 拒绝 → ToolResultBlock(is_error=True, reason)（953-959）
   └─ 否则硬阻断 → ToolResultBlock(is_error=True)（960-966）
被阻断/拒绝 → 该 ToolResultBlock 照样回灌模型，模型可读原因并调整策略重试
```

### 10.4 并发工具防御策略

`asyncio.gather(..., return_exceptions=True)`（query.py:853-855）+ 事后 `isinstance(result, BaseException)` 兜底（858-869）——失败工具的兄弟任务不受影响，且每个 tool_use 都能补上配对 tool_result。单工具路径用 try/except 实现相同保证（825-833）。

### 10.5 会话恢复消毒

```
QueryEngine.submit_message / continue_pending → sanitize_conversation_messages   query_engine.py:237/283 / messages.py:119
  → 丢弃空 assistant；为待匹配 tool_use 与 user tool_result 配对；孤置 tool_result 剥除
  → 保证喂给任何 provider 的历史都“每个 tool_use 有配套 tool_result”
```

### 10.6 客户端选择（在哪一层决定用哪个 client？）

`QueryEngine.__init__` 接收一个具体的 `api_client: SupportsStreamingMessages`（query_engine.py:27）。client 的装配逻辑**不在 engine 层**，而在上层（CLI/服务装配点）：依据 `settings` 的 `api_format`/`provider`/`api_key`/`base_url` 与 `api/provider.py:detect_provider` 的推断构造对应客户端后注入。引擎只认协议接口——典型的**依赖倒置**。

### 10.7 视觉回退链路

```
run_query → _preprocess_images_in_messages（query.py:563）
  非多模态模型（is_model_multimodal 为 False）+ 配置了 vision_model_config → 收集所有 ImageBlock
  → asyncio.gather 并行调用 image_to_text 工具（query.py:595-625）
  → 就地把 ImageBlock 替换为 TextBlock（628-630）
```

---

## 十一、设计要点、扩展点与注意点

### 11.1 设计要点

1. **协议统一 + 多后端适配**：引擎只依赖 `SupportsStreamingMessages`（client.py:80）这个三事件（delta/complete/retry）协议；四个客户端各自做格式转换，差异被完全隔离。
2. **双层事件模型**：API 事件（传输语义：重试、delta）→ 引擎事件（业务语义：工具开始/结束、压缩进度）在 `run_query`（738-752）转译；UI 从不直接见 provider 细节。
3. **turn 级防护机制**：`_bounded_completion_tokens` 上限保护（90-100）、max_tokens 超限动态降级重试（755-767）、prompt 过长 reactive 强制压缩（768-777）、空 assistant 消息丢弃（792-800）、并发工具 `return_exceptions=True` 防悬挂 tool_use（853-855）——层层避免"一次坏请求毁掉整个会话"。
4. **无状态工具结果 + 外置状态记忆**：工具结果以 `ToolResultBlock` 回灌（对话内状态）；跨回合"任务聚焦"信息（`task_focus_state`：goal/recent_goals/active_artifacts/verified_state/next_step，query.py:166-194；`read_file_state` 260-284；`invoked_skills`；`async_agent_state`/`async_agent_tasks`；`recent_work_log`）全部通过 `tool_metadata` 携带（不入对话、不烧 token），经 `carryover_metadata` 传给压缩与工具（query.py:681, 977）。
5. **超长输出的三级处理**：截断（inline 上限）→ 落盘 `tool_artifacts/`（文件名 = 时间戳 + 工具名清洗 + uuid，query.py:534-537）+ 内联占位与 preview（query.py:541-553）。
6. **延迟导出避免循环依赖**：`engine/__init__.py` 模块级 `__getattr__`（37-81）+ 函数内惰性 import（query.py:645-648）。
7. **provider 数据驱动**：registry 是单一事实来源，22 条 `ProviderSpec` 驱动探测/展示/配置。
8. **历史安全**：`sanitize_conversation_messages`（messages.py:119）处理恢复会话的畸形尾巴，保证跨 provider 可复放。
9. **thinking 模型兼容矩阵**：OpenAI 端 `max_completion_tokens`（46-57）、`reasoning_content` 回放/opt-in（149-159, 196-202）、` thinking` 跨 chunk 剥除（457-484）、有工具时剔除 `stream_options`（327-333）、Codex 的 effort 归一化（136-142）——针对不同厂商协议怪癖打的补丁，大多可用环境变量切换。
10. **分级错误语义**：客户端重试层只重试"瞬时"错误（429/5xx/网络），认证错误（401/403）永不重试、立刻以 `OpenHarnessApiError` 家族抛出，供 CLI 提示用户重新登录。

### 11.2 扩展点

| 扩展场景 | 需要改动的文件 |
| --- | --- |
| 接入新 OpenAI 兼容 provider | `registry.py` 加一条 `ProviderSpec`（探测自动生效，registry.py:55）；若协议特需（如 Kimi），参考 `openai_client.py` 的 env 开关模式。 |
| 接入全新协议的后端 | 新建 client 实现 `SupportsStreamingMessages`（client.py:80），在 `api/__init__.py` 导出，装配层按 settings 选择。 |
| 新增工具 | 继承 `BaseTool`（tools/base.py:35）并 `ToolRegistry.register`；其 `to_api_schema()`（tools/base.py:51）自动进入 `ApiMessageRequest.tools`。 |
| 新增引擎对外事件 | `stream_events.py` 定义 dataclass 并入 `StreamEvent` 联合（82-90）。 |
| 调整重试/退避 | 各 client 顶部的 `MAX_RETRIES`/`BASE_DELAY`/`MAX_DELAY`/`RETRYABLE_STATUS_CODES` 常量；Anthropic 额外支持 `Retry-After`（client.py:103-111）。 |
| 新增压缩策略 | 在 `services/compact` 内扩展 `auto_compact_if_needed` 的 `trigger`/phase，事件模型已预留 `CompactProgressEvent`。 |
| 视觉回退 | 在 `tool_registry` 注册 `image_to_text` 工具并在 `tool_metadata["vision_model_config"]` 配置，即可在非多模态模型上启用图片转文字（query.py:563-630）。 |
| 调整工具输出预算 | `OPENHARNESS_TOOL_OUTPUT_INLINE_CHARS` / `OPENHARNESS_TOOL_OUTPUT_PREVIEW_CHARS` / `OPENHARNESS_MICROCOMPACT_TOOL_RESULT_CHARS`（services/tool_outputs.py:26-47）。 |

### 11.3 潜在注意点（源码观察）

- `OpenAICompatibleClient` 把每个 `ToolResultBlock` 拆成独立 `role:"tool"` 消息（openai_client.py:107-114），这会**打乱 tool 与 text 的顺序**；若 UI 期望严格顺序，需要注意。Codex 端则特意把 `function_call_output` 排在最前（codex_client.py:80-91）。
- `tool_metadata` 是一个可变 dict 贯穿引擎与工具上下文（query.py:977 把 `**tool_metadata` 展开进工具 metadata），存在被工具意外覆盖同键的隐性耦合。
- `CostTracker`（cost_tracker.py:8-24）只做 token 聚合不做金额换算，"成本"语义需要上层另算。
- `CopilotClient` 通过 `self._inner._client = raw_openai`（copilot_client.py:104）侵入私有属性，是一种"组合+私有注入"的轻量 hack，升级 openai SDK 时需留意 `_client` 属性是否仍存在；同时 `stream_message` 会**丢弃 effort**（copilot_client.py:122-128 未透传）。
- `AnthropicApiClient._refresh_client_auth`（client.py:157-163）在 OAuth token 变化时整体重建 SDK client，会短暂中断连接池。
- `run_query` 的 `yield ... , usage` 二元组里，usage 绝大多数为 `None`；只有 `AssistantTurnComplete`（803）与空消息错误（799）携带真实用量——**任何想拿"全程用量"的逻辑必须监听 `AssistantTurnComplete`**，而不是依赖每个事件。
- `detect_provider_from_registry` 第 3 级 `_match_by_model` 过滤掉了 `is_local` 与 `is_oauth` 的 spec（registry.py:391），因此本地模型（如 llamafile）或 Copilot 只能靠前缀/关键字命中，模型名探测对它们天然失效——这是有意的（避免把任意本地模型名误判成某云厂商）。
- registry 中 `PROVIDERS` 数量为 **22**（不是 30），`is_model_multimodal` 的正则位 **23** 条——本文档已按实测校正。

---

## 十二、实际示例合集（结构示意与走查）

> 本章所有示例均为"结构示意"：字段名、嵌套层级、事件流严格对照源码（`messages.py`、`query.py`、`client.py`、`openai_client.py` 等处），ID、文本、数字等取值可虚构，仅用于辅助理解。每条示例前的 `> 示例` 块会给出对应的权威出处，可跳转源码逐一核对。

### 12.1 一段典型多轮 Agent 会话的完整消息序列

> 示例：以下为结构示意，数值为示例，值可虚构；字段结构来自 `messages.py:14-116`，序列化形态见 `messages.py:174-202`。

下面 4 条消息呈现一次"用户提问 → 模型 tool_use → 工具结果 → 最终回答"的完整沟通。它既是引擎内存中 `list[ConversationMessage]` 的 JSON 化表示，也正好是 `to_api_param()`（`messages.py:101-106`）落到协议层的形态。

```json
[
  {
    "role": "user",
    "content": [
      { "type": "text", "text": "帮我查一下 run_query 函数在哪个文件、第几行定义？" }
    ]
  },
  {
    "role": "assistant",
    "content": [
      {
        "type": "tool_use",
        "id": "toolu_01Ax9mBeng3nQhsHpZqf6qJz",
        "name": "grep",
        "input": { "pattern": "def run_query", "path": "src/openharness/engine" }
      }
    ]
  },
  {
    "role": "user",
    "content": [
      {
        "type": "tool_result",
        "tool_use_id": "toolu_01Ax9mBeng3nQhsHpZqf6qJz",
        "content": "src/openharness/engine/query.py\n633: async def run_query(context, messages)",
        "is_error": false
      }
    ]
  },
  {
    "role": "assistant",
    "content": [
      { "type": "text", "text": "run_query 定义在 src/openharness/engine/query.py 的 633 行，它是一个异步生成器。" }
    ]
  }
]
```

对照注释：

- **第 1 条**：用户真实输入。由 `ConversationMessage.from_user_text()` 构造（`messages.py:80-82`），content 内是单个 `TextBlock`（`messages.py:14-18`），序列化为 `{"type":"text","text":...}`（`messages.py:176-177`）。
- **第 2 条**：模型第 1 轮回包。content 内是单个 `ToolUseBlock`（`messages.py:40-46`），字段 `id / name / input` 与序列化输出逐一对齐（`messages.py:189-195`）。真实场景下这个 `id` 来自 provider 回包（`assistant_message_from_api`，`messages.py:213-219`），本地默认工厂 `toolu_{uuid4().hex}`（`messages.py:44`）只是"构造时没给 id"的兜底。
- **第 3 条**：引擎执行完 grep 后由 `query.py:880` 打包的"结果回灌"消息——`role="user"`，content 内含 `ToolResultBlock`（`messages.py:49-56`），`tool_use_id` 精确回填第 2 条的 id（配对命门见 12.4）。序列化时 `result_metadata` 被丢弃（`messages.py:197-202`），模型只看到 `tool_use_id / content / is_error`。
- **第 4 条**：模型第 2 轮回包，不再请求工具，只有一个 `TextBlock`。主循环随即走"无工具 → STOP"分支退出（`query.py:808-817`）。

### 12.2 一次实际 HTTP 请求体的结构示意

> 示例：以下为结构示意，数值为示例；键名/嵌套层级来自 `client.py:203-233` 的 `_stream_once` 组装逻辑；`tools` 结构来自 `tools/base.py:51-57`。

**Anthropic 形态**：`AnthropicApiClient._stream_once` 把请求参数拼进 `params` dict，再整体交给 `AsyncAnthropic.messages.stream(**params)`（`client.py:236-237`）。假设模型上一轮请求了 `bash` 工具、引擎已回灌结果，下一轮的历史请求体长这样：

```json
{
  "model": "claude-sonnet-4-5",
  "system": "<QueryContext.system_prompt>",
  "messages": [
    { "role": "user", "content": [ { "type": "text", "text": "帮我在工作区里 ls 一下有什么文件？" } ] },
    {
      "role": "assistant",
      "content": [
        {
          "type": "tool_use",
          "id": "toolu_7fE3c2aB4d9178f0aAbBcCdDeEfF",
          "name": "bash",
          "input": { "command": "ls -la" }
        }
      ]
    },
    {
      "role": "user",
      "content": [
        {
          "type": "tool_result",
          "tool_use_id": "toolu_7fE3c2aB4d9178f0aAbBcCdDeEfF",
          "content": "total 32\ndrwxr-xr-x  src docs pyproject.toml",
          "is_error": false
        }
      ]
    }
  ],
  "max_tokens": 4096,
  "tools": [
    {
      "name": "bash",
      "description": "Run a shell command in the workspace.",
      "input_schema": {
        "type": "object",
        "properties": { "command": { "type": "string" } },
        "required": ["command"]
      }
    }
  ]
}
```

- `system` 仅在 `system_prompt` 非空时出现（`client.py:210-211`）；
- `tools` 数组来自 `ToolRegistry.to_api_schema()`（`tools/base.py:78-80`）→ 每项是 `to_api_schema()` 返回的 `{"name","description","input_schema"}`（`tools/base.py:51-57`），`input_schema` 由 `input_model.model_json_schema()` 生成；
- 若启用 Claude OAuth，`params` 还会追加 `betas`、`metadata.user_id`、`extra_headers` 等（`client.py:221-233`）。

**OpenAI 兼容形态**：同一批历史经 `_convert_messages_to_openai`（`openai_client.py:81-126`）与 `_convert_tools_to_openai`（`openai_client.py:60-78`）转换后，请求体变成：

```json
{
  "model": "deepseek-chat",
  "messages": [
    { "role": "system", "content": "<system_prompt>" },
    { "role": "user", "content": "帮我在工作区里 ls 一下有什么文件？" },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_9aB7cDe3fGhI",
          "type": "function",
          "function": { "name": "bash", "arguments": "{\"command\": \"ls -la\"}" }
        }
      ]
    },
    { "role": "tool", "tool_call_id": "call_9aB7cDe3fGhI", "content": "total 32\ndrwxr-xr-x  src docs pyproject.toml" }
  ],
  "max_tokens": 4096,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "bash",
        "description": "Run a shell command in the workspace.",
        "parameters": {
          "type": "object",
          "properties": { "command": { "type": "string" } },
          "required": ["command"]
        }
      }
    }
  ],
  "stream": true
}
```

- `system` 变成首条 `role="system"` 消息（`openai_client.py:95-96`）；
- assistant 无文本时 `content` 为 `null`（`openai_client.py:192-193`）；`arguments` 是 `json.dumps(tool_input)`（`openai_client.py:204-215`）；
- 每个 `ToolResultBlock` 被拆成独立 `role="tool"` 消息，`tool_call_id` 与上方 `tool_calls[].id` 配对（`openai_client.py:107-114`）；
- `max_tokens`/`max_completion_tokens` 按模型名前缀切换（`openai_client.py:46-57`，如 gpt-5/o1/o3/o4 前缀 → `max_completion_tokens`）；
- 带 tools 时 `stream_options` 被剔除（`openai_client.py:327-333`）。

### 12.3 重试退避数值示例

> 示例：以下为结构示意，数值为示例（抖动为随机区间）；公式来源 `client.py:32-34, 98-115`：`delay = min(BASE_DELAY × 2^attempt, MAX_DELAY)`，再加 `random.uniform(0, delay × 0.25)` 的 25% 抖动；总尝试次数由 `for attempt in range(MAX_RETRIES + 1)` 限定（`client.py:169`）。

假设某次请求连续收到 429（限流）失败，Anthropic 端的退避过程如下（`attempt` 从 0 起，`MAX_RETRIES=3` 意味着最多 3 次等待 + 4 次尝试）：

| 调用次序 | 失败时 `attempt` | 基础退避 `1.0×2^attempt` | 抖动区间 | 实际 `sleep` 区间 | 对外 `ApiRetryEvent` |
| --- | --- | --- | --- | --- | --- |
| 第 1 次调用 | — | — | — | 立即失败，无等待 | — |
| 第 1 次重试 | 0 | 1.0s | 0 ~ 0.25s | ≈ 1.00 – 1.25s | attempt=1 / max_attempts=4 |
| 第 2 次重试 | 1 | 2.0s | 0 ~ 0.50s | ≈ 2.00 – 2.50s | attempt=2 / max_attempts=4 |
| 第 3 次重试 | 2 | 4.0s | 0 ~ 1.00s | ≈ 4.00 – 5.00s | attempt=3 / max_attempts=4 |
| 第 4 次调用 | 3 | — | — | 再失败 → `attempt >= MAX_RETRIES` 直接 raise，**无第 4 次重试** | — |

- 指数基数实际上只在 `attempt = 0, 1, 2` 生效；理论上 `1.0 × 2^5 = 32` 会触顶 `MAX_DELAY = 30`，但 `MAX_RETRIES = 3` 下永远走不到那一步；
- 若响应头带 `Retry-After`，则直接 `min(retry_after, 30)` 顶替整段退避（`client.py:103-111`）；
- OpenAI 兼容端公式相同，但**不支持** `Retry-After`（`openai_client.py:40-42, 299`）；
- 每次等待前都 `yield ApiRetryEvent`，引擎把它转译成 `StatusEvent` 提示（`query.py:741-747`），不打断流。

### 12.4 tool_use → tool_result 配对示例

> 示例：以下为结构示意，id 值可虚构；id 默认工厂 `messages.py:44`，API 回包取真实 id `messages.py:213-219`；配对回填 `query.py:992-997`，打包 `query.py:880`。

**① 模型发出的 tool_use**（assistant 消息的 content[0]）：

```json
{
  "type": "tool_use",
  "id": "toolu_8f7e6d5c4b3a29f1aabccddeeff0011",
  "name": "read_file",
  "input": { "file_path": "src/openharness/engine/query.py", "offset": 0, "limit": 40 }
}
```

`id` 的形态：本地构造缺省为 `f"toolu_{uuid4().hex}"`（`messages.py:44`），即 `toolu_` + 32 位小写 hex；但真实链路里这条 `id` 来自 provider 的流式回包——`assistant_message_from_api` 把 SDK 块翻译回 `ToolUseBlock(id=真实id, ...)`（`messages.py:213-219`）。

**② 引擎执行后回填的 ToolResultBlock**（`_execute_tool_call` 的返回值，`query.py:992-997`）：

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_8f7e6d5c4b3a29f1aabccddeeff0011",
  "content": "1: \"\"\"Core tool-aware query loop.\"\"\"\n2: from __future__ import annotations",
  "is_error": false
}
```

注意 `tool_use_id` 必须**逐字符等于**①的 `id`——引擎所有错误路径（hook 拦截、工具未注册、参数校验失败、权限被拒、执行异常）返回的 `ToolResultBlock` 都用同一个 `tool_use_id` 回填（`query.py:899-903, 910-914, 920-924, 955-959, 962-966, 829-833`），绝不另造新 id。

**③ 打包进 user 消息**（`query.py:880`）：

```python
messages.append(ConversationMessage(role="user", content=tool_results))
```

序列化（`messages.py:197-202`）后，下一轮 Anthropic 请求体里就是 `{"role":"user","content":[②]}`；OpenAI 端则转成 `{"role":"tool","tool_call_id": <②的tool_use_id>, "content": ...}`（`openai_client.py:110-114`）。

**为何配对不可破**：源码注释明说——如果会话里存在一个 `tool_use` 没有配对的 `tool_result`，Anthropic 会**拒绝下一次请求**（`query.py:849-852`）。这正是并发工具强制 `return_exceptions=True`（`query.py:853-855`，失败工具不许悬挂）、恢复会话时用 `sanitize_conversation_messages`（`messages.py:119-171`）剪掉"没配对 tool_result 的 tool_use 尾巴"的原因。

### 12.5 `run_query` 逐步骤运行注释走查

> 示例：以下为结构示意，会话内容为虚构"走查剧本"；坐标对应第三节 run_query 主循环的 16 步编号与 `query.py` 行号。

**极小假想会话**（3 句对白）：

```
用户：帮我读一下 src/openharness/tools/base.py 开头 3 行。
模型第 1 轮：发出 read_file 工具请求。
工具层：读文件成功，把结果回灌。
模型第 2 轮：给出一句纯文本回答，不再请求工具。
```

映射到主循环 16 步：

```
请求进入 run_query(context, messages)
│
├─ Step 0  状态初始化（650-657）：compact_state / reactive_compact_attempted /
│          last_compaction_result / effective_max_tokens（假设 4096 ≤ 128000，未截断）/
│          reported_token_clamp=False
│
├─ turn_count=1，进入 while（699-701）────────（第 1 轮）
│  ├─ Step 3  effective_max_tokens == context.max_tokens，跳过截断提示（702-710）
│  ├─ Step 4  auto-compact：仅 1 条 user 消息，预估不超阈值，
│  │          _stream_compaction 不产出任何 CompactProgressEvent（711-717）
│  ├─ Step 5  图片预处理：user 消息里没有 ImageBlock，_preprocess_images_in_messages
│  │          直接 return（719-722）
│  ├─ Step 6  组装 ApiMessageRequest 并发起流式调用（727-737）
│  ├─ Step 7  三路分发（738-752）：
│  │          ├─ 收到若干 ApiTextDeltaEvent → 逐条转发 AssistantTextDelta：
│  │          │   "我先看一下这个文件的头部……"（738-740）
│  │          ├─ 假设无 ApiRetryEvent（742-747 不触发）
│  │          └─ 收 ApiMessageCompleteEvent → 暂存 final_message + usage（750-752）
│  ├─ Step 8  正常路径，无异常（753-782 不进入任一分支）
│  ├─ Step 9  final_message 非空（784-785）
│  ├─ Step 10 system_prompt 不以 "You are a **coordinator**." 开头，跳过弹/回塞（787-790）
│  ├─ Step 11 final_message 含 tool_use，非空消息，防护不触发（792-800）
│  ├─ Step 12 messages.append(final_message) → yield AssistantTurnComplete（802-803）
│  ├─ Step 13 final_message.tool_uses 非空 → 不 return，继续（808-817 跳过）
│  ├─ Step 14 单工具（len==1）：yield ToolExecutionStarted("read_file"...)，（824）
│  │          → _execute_tool_call（887-1018）：PRE_TOOL_USE 放行 → 注册表命中 →
│  │            input_model 校验通过 → 权限评估（read_only 放行）→ tool.execute 实际
│  │            读文件 → 输出未超 16000 字符原样内联（985-989）→
│  │            ToolResultBlock(tool_use_id=回填, ...)（992-997）→
│  │            yield ToolExecutionCompleted（834-839）
│  └─ Step 15 messages.append(ConversationMessage(role="user", content=tool_results))
│           回灌（880）→ while 循环回到下一轮
│
├─ turn_count=2，再次进入 while（701）────────（第 2 轮）
│  ├─ Step 3~5 与第 1 轮相同，均跳过（702-722）
│  ├─ Step 6  再次流式调用，此刻历史已含
│  │          "assistant(tool_use) + user(tool_result)" 两个相邻回合
│  ├─ Step 7  收文本增量"已读完成"→ AssistantTextDelta（738-740）
│  ├─ Step 9~12 同第 1 轮：final_message 入历史、yield AssistantTurnComplete（802-803）
│  └─ Step 13 final_message.tool_uses 为空（这是最终回答文本）→ 触发 HookEvent.STOP
│            （stop_reason="tool_uses_empty"）→ return ✔ 正常退出（808-817）
│
└─ 循环外的 882-884 两个 raise 永不触达（正常退出只走 Step 13 的 return）
```

走查要点：

- **一个 turn ≠ 用户一句话**：用户只发了一个 prompt，但主循环跑了 2 个 turn（工具回合 + 最终回答回合），`turn_count` 都在默认 `max_turns=200` 内；
- **usage 的两次出现**：每次 `AssistantTurnComplete`（803）都携带真实 `UsageSnapshot`；其它事件（StatusEvent 等）usage 恒为 `None`——想统计全程 token 只能监听该事件；
- **Step 5 的"隐身"**：模型是多模态（`is_model_multimodal`）或未配置 vision 模型时，图片预处理整段静默返回，这也是绝大多数会话里它"不存在"的原因。