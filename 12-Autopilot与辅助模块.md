# OpenHarness 源码精读：Autopilot 子系统、autopilot-dashboard 与辅助模块

> 分析对象（全部行号均已用 Read/Grep 实测核对，可直接跳转复验）：
>
> - `src/openharness/autopilot/` — `service.py`（2239 行，全文逐段精读）、`types.py`（91 行）、`__init__.py`（23 行）
> - `autopilot-dashboard/` — `package.json`、`vite.config.ts`、`tsconfig.app.json`、`index.html`、`src/main.tsx`、`src/App.tsx`（290 行）、`src/types.ts`（84 行）、`src/components/PipelineAnimation.tsx`（223 行）、`src/components/HeroBackground.tsx`（181 行）、`src/index.css`（546 行）、`public/snapshot.json`（59 行）
> - `src/openharness/utils/` — `fs.py`（98 行）、`shell.py`（147 行）、`file_lock.py`（86 行）、`helpers.py`（77 行）（`network_guard.py` 见其它文档，此处不展开）
> - `src/openharness/themes/` — `schema.py`（54 行）、`builtin.py`（89 行）、`loader.py`（55 行）、`__init__.py`（21 行）
> - `src/openharness/output_styles/` — `loader.py`（42 行）、`__init__.py`（5 行）
> - `src/openharness/keybindings/` — `default_bindings.py`（11 行）、`parser.py`（18 行）、`resolver.py`（13 行）、`loader.py`（22 行）、`__init__.py`（14 行）
> - `src/openharness/vim/transitions.py`（8 行）
> - `src/openharness/voice/` — `voice_mode.py`（44 行）、`keyterms.py`（10 行）、`stream_stt.py`（8 行）
> - `src/openharness/state/` — `app_state.py`（30 行）、`store.py`（40 行）
> - `src/openharness/bridge/` — `types.py`（37 行）、`session_runner.py`（46 行）、`manager.py`（106 行）、`work_secret.py`（41 行）、`__init__.py`（20 行）
> - `src/openharness/platforms.py`（86 行）、`src/openharness/__main__.py`（6 行）
> - `scripts/` — 共 15 个安装/测试/维护脚本（`.sh` ×3、`.ps1` ×1、`.py` ×11），重点精讲 `install.sh`（387 行）、`install.ps1`（329 行）、`test_cli_flags.py`（157 行）等
>
> 关键外部引用：`src/openharness/ui/runtime.py`（`build_runtime`:274 / `start_runtime`:474 / `close_runtime`:482）、`src/openharness/engine/query_engine.py`（`QueryEngine.submit_message`:227）、`src/openharness/swarm/worktree.py`（`WorktreeManager.create_worktree`:150 / `remove_worktree`:213）、`src/openharness/cli.py`（`autopilot_app`:773）、`src/openharness/commands/registry.py`（`_autopilot_handler`:2051）。

---

## 目录

1. [Autopilot 子系统：定位与架构](#一autopilot-子系统定位与架构)
2. [数据模型（types.py 全部字段）](#二数据模型typespy-全部字段)
3. [默认策略常量（YAML 种子数据）](#三默认策略常量yaml-种子数据)
4. [RepoAutopilotStore：持久化布局与摄入层](#四repoautopilotstore持久化布局与摄入层)
5. [执行状态机 run_card（核心精读）](#五执行状态机-run_card核心精读)
6. [GitHub 自动化、CI 轮询与双语评论](#六github-自动化ci-轮询与双语评论)
7. [验证门（verification gates）设计](#七验证门verification-gates设计)
8. [与主引擎的交互：_run_agent_prompt 逐行](#八与主引擎的交互_run_agent_prompt-逐行)
9. [报告、仪表盘导出与 tick/cron 调度](#九报告仪表盘导出与-tickcron-调度)
10. [CLI 入口与斜杠命令路由](#十cli-入口与斜杠命令路由)
11. [autopilot-dashboard 前端全解](#十一autopilot-dashboard-前端全解)
12. [辅助模块 utils（fs/shell/file_lock/helpers）](#十二辅助模块-utilsfsshellfile_lockhelpers)
13. [辅助模块（themes / output_styles / keybindings / vim / voice / state / bridge / platforms / __main__）](#十三辅助模块themes--output_styles--keybindings--vim--voice--state--bridge--platforms--__main__)
14. [scripts 脚本清单与重点脚本精讲](#十四scripts-脚本清单与重点脚本精讲)
15. [关键类 / 函数清单总表](#十五关键类--函数清单总表)
16. [设计要点小结](#附设计要点小结)
17. [实际示例：从卡到合并的全链路数据形态](#十六实际示例从卡到合并的全链路数据形态)

---

## 一、Autopilot 子系统：定位与架构

`src/openharness/autopilot/` 实现 OpenHarness 的**仓库级"自进化"自动飞行（repo autopilot）子系统**。它的输入是分散在各个渠道的工作项：

| 来源渠道 | `source_kind` | 语义 |
| --- | --- | --- |
| GitHub open Issue | `github_issue` | 项目管理的 bug/feature 请求 |
| GitHub open PR | `github_pr` | 外部提交的 PR（可能被回收监控，也可能被接管执行） |
| `ohmo`（OpenAI 协议镜像 CLI）请求 | `ohmo_request` | 手动提交的任务 |
| 手动 idea（`oh autopilot add ...`） | `manual_idea` | 直接入队 |
| `$HOME/claude-code/{commands,agents}/` | `claude_code_candidate` | "借钟候选"——把 claude-code 的 skill/agent 当待评估素材 |

**归一化**：一切来源最终都折叠成一张 `RepoTaskCard`（任务卡）。卡按**可解释分数**（`_score_card`）排序放进统一队列（unified queue）；消费者 `tick()` / `run_next()` 一个接一个地取出最高分卡执行。

**一次性执行业务闭环**（`run_card`）：
1. 建隔离 **git worktree**（`WorktreeManager`，分支名 `autopilot/ap-xxxxxxxx`）；
2. 用 `ui.runtime.build_runtime` 组装一个**全新会话**，注入执行 prompt，让主引擎（`QueryEngine`）全自动跑 Agent（工具调用 + 多次回合 + 写代码）；
3. 跑 **service 级验证门**（pytest / ruff / tsc），不通过就进"修复循环"（最多 `max_attempts` 轮）；
4. 提交、推送分支、幂等创建 **PR**；
5. 轮询远端 **CI**（`gh pr view statusCheckRollup`）直到成功/失败/超时；
6. 按 auto-merge 策略**自动合并**（squash）或停在**人工 gate**；
7. 全程写**追加式 journal**、**双份运行报告**，并在 Issue/PR 上发**中英双语评论**；
8. 每次状态变更后同步刷新**静态看板**（`docs/autopilot/snapshot.json` + React 页面）。

**关键结论：它不是一个新的 Agent 循环，而是引擎之上的"编排层（orchestration）"**。关系和分层如下：

```
┌────────────────────────────────────────────────────────────────┐
│ RepoAutopilotStore（service.py:226）—— 选任务·隔离·验证·上报·合并  │
│    intake/评分/队列  ──►  run_card 状态机  ──►  tick/cron 驱动     │
│    job 源: gh issue/pr · claude-code · 手动/ohmo ──► enqueue_card │
└──────────────┬─────────────────────────────────────────────────┘
               │ _run_agent_prompt (service.py:2041)
               ▼
┌────────────────────────────────────────────────────────────────┐
│ ui.runtime.build_runtime (runtime.py:274)                       │
│   设置合并/API client/MCP/默认工具表/插件工具/AppStateStore/Hooks │
│   bundle.engine = QueryEngine                                   │
│   engine.submit_message(prompt)  ──► 事件流（含工具执行）        │
│   （全自动权限：permission_prompt 恒 True、ask_user 恒 ""）      │
└────────────────────────────────────────────────────────────────┘
```

**复用的上层**：`build_runtime`（runtime.py:274）会完成 load_settings→merge CLI 覆盖→`load_plugins`→解析 API client→`McpClientManager.connect_all`→`create_default_tool_registry`→注册插件工具→`detect_provider`→建 `AppStateStore`→建 `HookExecutor`→`build_runtime_system_prompt`→装配 `QueryEngine` 等一整套会话初始化（runtime.py:302-427）；autopilot 只负责"提交 prompt + 订阅事件"。因此主引擎"想 + 做"，autopilot"选 + 批 + 合并"。`oh autopilot ...` 子命令与 `/autopilot` 斜杠命令都只是对这个 Store 的薄壳（见第十章）。

---

## 二、数据模型（types.py 全部字段）

全部为 Pydantic v2 模型（`pydantic.BaseModel`），含四个 `Literal` 型状态/来源联合（types.py:9-30）。

### 2.1 `RepoTaskStatus`（types.py:9-23）——状态机的 13 个合法状态

```
queued / accepted / preparing / running / verifying / pr_open /
waiting_ci / repairing / completed / merged / failed / rejected / superseded
```

逐一语义：

| 状态 | 含义 | 谁写入 |
| --- | --- | --- |
| `queued` | 排队待执行（新卡的默认值） | `enqueue_card` |
| `accepted` | 人工/策略接受，尚未开跑 | 预留（`update_status` 任意可用） |
| `preparing` | 正在建 worktree / 初始化元数据 | `run_card` 开跑前 |
| `running` | Agent 正在执行（第 1 轮） | `run_card` 每轮开始 |
| `verifying` | 正在跑 service 级验证门 | `run_card` 验证前 |
| `pr_open` | PR 已创建 | 预留（CI 等待用 `waiting_ci`） |
| `waiting_ci` | 等待远端 CI 结论 | `run_card` 建 PR 后 / `_process_existing_pr_card` |
| `repairing` | 修复循环（attempt>1） | `run_card` 各失败分支 |
| `completed` | 本地验证通过 + 停在人工 gate | `run_card` / `_process_existing_pr_card` |
| `merged` | PR 已自动合并，闭环完成 | `run_card` / `_process_existing_pr_card` |
| `failed` | 终局失败（自动修复耗尽） | `run_card` 各终局分支 |
| `rejected` | 被拒绝 | 预留（斜杠命令可置） |
| `superseded` | 被新卡取代 | 预留 |

### 2.2 `RepoTaskSource`（types.py:24-30）

`ohmo_request / manual_idea / github_issue / github_pr / claude_code_candidate`，五种。

### 2.3 `RepoTaskCard`（types.py:33-48）——归一化工作项

| 字段 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `id` | `str` | — | `"ap-" + uuid4().hex[:8]`（service.py:311） |
| `fingerprint` | `str` | — | 去重指纹 `"{source_kind}:{sha1[:16]}"`（见 4.3） |
| `title` | `str` | — | 归一化标题 |
| `body` | `str` | `""` | 正文 |
| `source_kind` | `RepoTaskSource` | — | 来源 |
| `source_ref` | `str` | `""` | 归一化引用，三种形态：`issue:42` / `pr:7` / `command:/abs/path`（或 `agent:/abs/path`）/ `manual` |
| `status` | `RepoTaskStatus` | `"queued"` | 当前状态 |
| `score` | `int` | `0` | 排序分（见 4.4） |
| `score_reasons` | `list[str]` | `[]` | **每一条加/扣分的可解释来源** |
| `labels` | `list[str]` | `[]` | 标签（已 `_normalize_labels` 排序去重） |
| `metadata` | `dict[str, Any]` | `{}` | 自由字段，执行期大量落地关键状态（详见 5.2 表格） |
| `created_at` / `updated_at` | `float` | — | unix 秒时间戳 |

### 2.4 `RepoJournalEntry`（types.py:51-58）——追加式日志事件

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `timestamp` | `float` | unix 秒 |
| `kind` | `str` | `intake_added / intake_refresh / scan_all / run_finished / verification_failed / merged / human_gate_pending / status_* / run_failed / tick_skip / tick_idle / ci_failed_retry / existing_progress_detected / github_warning / scan_warning` 等 |
| `summary` | `str` | 人类可读摘要 |
| `task_id` | `str \| None` | 关联卡号 |
| `metadata` | `dict[str, Any]` | 补充结构 |

### 2.5 `RepoAutopilotRegistry`（types.py:61-66）

`version:int=1` / `updated_at:float=0.0` / `cards:list[RepoTaskCard]=[]`。整个注册表一个 JSON 文件持久化。

### 2.6 `RepoVerificationStep`（types.py:69-76）——单条验证命令结果

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `command` | `str` | 原始命令文本 |
| `returncode` | `int` | 返回码（`-1` 表示 error/未执行） |
| `status` | `Literal[success,failed,skipped,error]` | 四态结论 |
| `stdout` / `stderr` | `str` | 截尾后的输出（≤4000 字符） |

### 2.7 `RepoRunResult`（types.py:79-91）——一次执行尝试的结果

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `card_id` | `str` | 关联卡 |
| `status` | `RepoTaskStatus` | 终态（failed/completed/merged/…） |
| `assistant_summary` | `str` | 模型自报摘要 |
| `run_report_path` | `str` | 本次 run 报告路径 |
| `verification_report_path` | `str` | 本次验证报告路径 |
| `verification_steps` | `list[RepoVerificationStep]` | 全部验证步骤 |
| `attempt_count` | `int` | 第几次尝试 |
| `worktree_path` | `str` | 执行用的 worktree 目录 |
| `pr_number` / `pr_url` | `int \| None / str` | 若走到 PR 阶段则回填 |

---

## 三、默认策略常量（YAML 种子数据）

三类策略均按项目存放在 `.openharness/autopilot/` 下（路径见 `config/paths.py:119-159`），首次由 `_ensure_layout()`（service.py:1888-1899）落盘；之后 `load_policies()`（service.py:492-500）通过 `_read_yaml`（2002-2011）读取——**文件缺失或损坏时静默回退到这里的默认 dict**。

### 3.1 `_DEFAULT_AUTOPILOT_POLICY`（service.py:53-89）

| 段 | 键 | 默认 | 作用 |
| --- | --- | --- | --- |
| `intake` | `mode` | `unified_queue` | 统一队列模式 |
| | `max_visible_candidates` | `12` | 候选可见上限 |
| | `dedupe_strategy` | `source_ref_then_fingerprint` | 去重策略 |
| `decision` | `default_human_gate` | `True` | **默认最后一步需人工 gate** |
| | `prefer_small_safe_steps` | `True` | 偏好小而安全的步骤 |
| `execution` | `default_model` | `""` | 空则用全局默认模型 |
| | `max_turns` | `12` | Agent 回合上限 |
| | `permission_mode` | `full_auto` | 工具全自动放行 |
| | `host_mode` | `self_hosted` | 宿主自理 |
| | `use_worktree` | `True` | 用隔离 worktree |
| | `base_branch` | `main` | PR 目标分支 |
| | `max_attempts` | `3` | 最多尝试次数 |
| `github` | `issue_comment_style` | `bilingual` | 中英双语评论 |
| | `pr_branch_prefix` | `autopilot/` | 分支前缀（`_head_branch` 用，1219-1222） |
| | `ci_poll_interval_seconds` | `20` | CI 轮询间隔 |
| | `ci_timeout_seconds` | `1800` | CI 总超时 |
| | `no_checks_grace_seconds` | `60` | 无 check 的宽限期 |
| | `checks_settle_seconds` | `20` | check 出现后的稳定等待 |
| | `auto_merge.mode` | `label_gated` | 合并模式：`label_gated`（默认）/`fully_auto`/`pr_only` |
| | `auto_merge.required_label` | `autopilot:merge` | label-gated 所需标签 |
| `repair` | `max_rounds` | `2` | 修复轮数 |
| | `retry_on` | `[local_verification_failed, remote_ci_failed]` | 可重试的失败阶段 |
| | `stop_on` | `[agent_runtime_error, git_error, permission_error, merge_conflict]` | 不可重试、直接终局的失败阶段 |

### 3.2 `_DEFAULT_VERIFICATION_POLICY`（service.py:90-109）

- `gates`: `[fast_gate, repo_gate, harness_gate]`（命名门标识）；
- `commands`（三条，顺序执行）：
  1. `uv run pytest -q`
  2. `uv run ruff check src tests scripts`
  3. `{command: "cd frontend/terminal && ([ -x ./node_modules/.bin/tsc ] || npm ci --no-audit --no-fund) && ./node_modules/.bin/tsc --noEmit", shell: true}` —— 唯一显式开启 `shell: true` 的条目（含 `&&`、subshell，必须走 shell）；
- `require_tests_before_merge: True`。

> 注意第三条命令的"可用性过滤"触发条件：`_looks_available`（199-209）只在 `cwd/frontend/terminal/package.json` 存在时才放行（service.py:207-208）。

### 3.3 `_DEFAULT_RELEASE_POLICY`（service.py:110-114）

`merge_requires_human: True`、`release_requires_human: True`、`auto_revert_on_failed_verification: False`。当前代码里 `run_card` 尚未消费 release 策略（只把它 dump 进 prompt 供模型参考），是"prompt 级约束"而非"代码级强制"。

### 3.4 辅助工具函数

- `_shorten(text, limit=120)`（117-121）：折叠空白并截断加省略号。
- `_safe_text(value)`（124-127)：`None→""`，否则去首尾空白。
- `_json_default(value)`（130-133)：`Path` 序列化为字符串。
- `_bilingual_lines(zh, en)`（222-223）：`f"{zh}\n{en}"` 拼成双语段落。

---

## 四、RepoAutopilotStore：持久化布局与摄入层

### 4.1 构造与文件布局（service.py:229-251）

```python
def __init__(self, cwd) :                    # service.py:229
    self._cwd        = Path(cwd).resolve()
    self._registry_path = get_project_autopilot_registry_path(self._cwd)   # .openharness/autopilot/registry.json
    self._journal_path   = get_project_repo_journal_path(self._cwd)        # .openharness/autopilot/repo_journal.jsonl
    self._context_path   = get_project_active_repo_context_path(self._cwd) # .openharness/autopilot/active_repo_context.md
    self._runs_dir       = get_project_autopilot_runs_dir(self._cwd)       # .openharness/autopilot/runs/
    self._ensure_layout()
```

四个属性 `registry_path / journal_path / context_path / runs_dir`（237-251）暴露给外部（CLI 打 path 用）。`_ensure_layout`（1888-1899）：三个 YAML 不存在则写默认；`registry.json` 不存在则写空注册表；`active_repo_context.md` 不存在则先重建一次上下文（联动导出仪表盘）。

### 4.2 读/写注册表

- `_load_registry()`（1901-1908）：JSON 缺失或 `JSONDecodeError` 时**容错返回空注册表**（不抛异常）。
- `_save_registry(registry)`（1910-1921）：更新 `updated_at` → `json.dumps(indent=2, ensure_ascii=False)` → `atomic_write_text`。原子写保证崩溃不截断。
- `_read_yaml(path, default)`（2002-2011）：缺失/解析失败/非 dict 全部回退默认。

### 4.3 摄入：`enqueue_card`（service.py:265-332）

签名：`enqueue_card(*, source_kind, title, body="", source_ref="", labels=None, metadata=None) -> tuple[RepoTaskCard, bool]`（第二项 `True`=新建，`False`=刷新）。

流程：
1. 逐字段 trim，算指纹 `_build_fingerprint`（1923-1933）：`basis = source_ref or f"{title}\n{body}"`，`sha1(basis)[:16]`，拼 `"{source_kind}:{digest}"`。
2. 在注册表里按 `fingerprint` 找 `existing`（286 行）。**同指纹 = 同一工作项**：
   - 命中 → **刷新不新增**：非空字段覆盖标题/正文/ref；`_merge_labels` 合并标签（1977-1978，`sorted(set(existing)|set(incoming))`）；`metadata.update()` 合并；`updated_at=now`；重打分（299）；journal 写 `intake_refresh`（301-306）；`rebuild_active_context()`；返回 `(existing, False)`。
   - 未命中 → 建新卡（310-321）：`id=f"ap-{uuid4().hex[:8]}"`，默认标题 `"Untitled intake item"`，`status` 默认 `queued`；打分、追加注册表、journal 写 `intake_added`（含 score）、重建上下文，返回 `(card, True)`。
3. **两个分支都会重打分 + 重建上下文 + 刷新看板**，保证"摄入即反映到仪表盘"。

### 4.4 可解释打分：`_score_card`（service.py:1935-1970）

返回 `(score, score_reasons)`。基础分 `_SOURCE_BASE_SCORES`（43-49）：

| 来源 | 基础分 |
| --- | --- |
| `ohmo_request` | 100 |
| `github_pr` | 85 |
| `manual_idea` | 80 |
| `github_issue` | 75 |
| `claude_code_candidate` | 45 |
| 未知 | 50（`_score_card` 默认兜底，1936） |

加扣分信号（每个都 `append` 一条 reason，**可解释性优先**）：

| 信号 | 分值 | 条件 | reason 文案 |
| --- | --- | --- | --- |
| bug 标签 | +25 | `github_issue` 且 `labels ∩ {bug, regression, failure}` | `bug-labelled issue` |
| bug 关键词 | +15 | 标题/正文含 `_BUG_HINTS`（service.py:50: `bug fix failure broken regression crash error issue`） | `issue looks like a bug/regression` |
| draft PR | −30 | `github_pr` 且 `metadata.is_draft` | `draft pr` |
| 干净合并态 | +20 | `merge_state_status == CLEAN` | `clean merge state` |
| 已 approved | +20 | `review_decision == APPROVED` | `approved review state` |
| 直接人工输入 | +10 | `ohmo_request` / `manual_idea` | `direct user-driven input` |
| 紧急信号 | +20 | 文本含 `_URGENT_HINTS`（`urgent p0 p1 high critical blocker`，51）或相应标签 | `urgent signals` |
| 新鲜度 | 0~10 | `max(0, 10 - age_days)`，`age_days=(now-updated_at)/86400`（1965-1969） | `recently updated` |

### 4.5 查询接口

- `list_cards(*, status=None)`（253-257）：全量排序 `(-score, -updated_at, title.lower())`，可按状态过滤。
- `get_card(card_id)`（259-263）：线性查找，`None` 表示未找着。
- `pick_next_card()`（334-338）：**只取 `status=="queued"`** 的最高分卡；空队列返回 `None`。
- `stats()`（486-490）：`{status: count}` 计数。
- `update_status(card_id, *, status, note=None, metadata_updates=None)`（340-365）：核心状态写入口——改状态、刷新 `updated_at`、写 `metadata["last_note"]`、合并 metadata、**重打分**、journal 写 `status_{status}`、重建上下文。卡不存在抛 `ValueError`。

### 4.6 journal 与活动上下文

- `append_journal(*, kind, summary, task_id=None, metadata=None) -> RepoJournalEntry`（381-398）：`a` 模式追加一行 JSONL（`model_dump_json()`）。
- `load_journal(*, limit=12)`（367-379）：逐行读取，坏行跳过，返回**最后 limit 条**。
- `rebuild_active_context()`（405-484）：**这是对外暴露的"活动仓库上下文"聚合器**：
  1. 按状态分桶：running 组 = `{preparing, running, verifying, waiting_ci, repairing}`，accepted 组 = `{accepted, pr_open}`，还有 queued / completed(`{completed,merged}`) / failed(`{failed,rejected}`)（407-411）。
  2. **focus 优先级 `running组 → accepted组 → queued组`**，组内按 `(-score, -updated_at, title)` 取第一名（412-419）。
  3. 拼 Markdown 六段：`## Current Task Focus` / `## In Progress`(前6) / `## Next Up`(前8) / `## Recently Completed`(前5，按更新倒序) / `## Recent Failures`(前5) / `## Recent Repo Journal`(前8) + `## Policies`（三个 YAML 路径）（421-480）。
  4. `atomic_write_text` 写 `active_repo_context.md`（482），**并随后 `export_dashboard()` 联动刷新快照**（483）。

---

## 五、执行状态机 run_card（核心精读）

### 5.1 状态机全景

13 状态中，`run_card` 实际遍历的活跃子集为：`preparing → running (repairing) → verifying → [repairing ⇄] → waiting_ci → [repairing ⇄] → completed/merged/failed`。转换表：

| 转移 | 触发条件 | 动作 |
| --- | --- | --- |
| 任意→拒绝执行 | `status ∈ {preparing, running, verifying, waiting_ci, repairing}`（655-656） | 抛 `ValueError`（防并发重入） |
| → preparing | 前置检查通过 | 建 worktree、写元数据、发"开始处理"评论 |
| → running（第1轮）/ repairing（attempt>1） | 每轮循环开始（742-747） | 同步 worktree、构造 prompt、跑 Agent |
| → verifying | Agent 正常返回（818-823） | 跑验证门写验证报告 |
| → repairing | 验证失败且 attempt<max / Agent 无改动 / CI 失败且有轮次（848/924/1024） | 记录 `last_failure_*`、发修复评论、`continue` |
| → failed | 修复耗尽（各终局分支 + 1136-1150 `repair_exhausted`） | 写失败 run 报告、发终局评论 |
| → completed | 非 git 仓库（891）/ PR 绿且不可自动合并（1102） | `human_gate_pending=True`、发人工 gate 评论、清 worktree |
| → merged | CI 绿且 `_automerge_eligible`（1070） | squash 合并、发合并评论、清 worktree |

### 5.2 入口方法：`run_next` 与 `run_card`

**`run_next(*, model, max_turns, permission_mode)`（627-642）**：`pick_next_card()`；无卡抛 `ValueError("No queued autopilot cards.")`；否则转 `run_card(card.id, ...)`。

**`run_card(card_id, *, model, max_turns, permission_mode)`（644-1150）**——编排主体，分四段：

#### A. 前置与准备（652-710）
1. `get_card` 判存（652-654）；活跃态拒绝（655-656）。
2. `load_policies()` 解析有效参数（658-670）：
   - `effective_model = model or execution.default_model or None`；
   - `effective_max_turns`；
   - `effective_permission_mode`（默认 `full_auto`）；
   - `max_attempts = self._max_attempts(policies)`（1208-1213）：`max(execution.max_attempts, repair.max_rounds+1, 1)` = `max(3, 3, 1)`；
   - `base_branch`（1215）／`head_branch = f"{prefix}{card.id}"`（1219-1222）／`issue_number`／`linked_pr_number`。
3. **特例短路**（672-673）：`source_kind=="github_pr"` 且已链接 PR 且 `not metadata.autopilot_managed` → `_process_existing_pr_card`（见 6.4），只监控既有 PR，不重跑 Agent。
4. worktree（675-684）：`use_worktree = execution.use_worktree and _is_git_repo(cwd)`；是则 `WorktreeManager().create_worktree(cwd, "autopilot/{card.id}", branch=head_branch)`（`_worktree_slug` 1224-1225），`working_cwd=worktree_info.path`；否则原目录执行。
5. `update_status(preparing)` 落元数据（686-701）：
   `run_started_at / execution_model / max_attempts / worktree_slug / worktree_path / head_branch / base_branch / linked_issue_numbers / linked_pr_number`。
6. 首轮且关联 issue → `_comment_on_issue` 发 `_comment_started`（703-704）。

#### B. 主循环（attempt_count ∈ [已有+1, max_attempts]，712-1134）

每轮：

```
for attempt_count in range(existing_attempts + 1, max_attempts + 1):
  attempt_run_report / attempt_verification_report：{card}-attempt-{NN:02d}-run.md 等（713-714）
  is_first_attempt = attempt_count == 1 and existing_attempts == 0        # 715
  ├─ worktree 同步 _sync_worktree_to_base（716-723；1294-1299）
  │     fetch origin base；首轮 checkout -B head_branch origin/base（reset）
  │     后续轮仅 checkout head_branch
  │     └─ 异常 → failed(git_prepare_failed)，写 report，return
  ├─ update_status: repairing if attempt>1 else running（742-747，note 见代码）
  ├─ prompt = _prepare_repair_prompt(...)（748-755；1595-1626）
  ├─ assistant_summary = await _run_agent_prompt(...)（756-763；见第八章）
  │     └─ 异常 → failed(agent_runtime_error)（764-801）
  │           · _render_run_report(verification_status="not_started") 写双report
  │           · metadata: execution_error / last_failure_stage=agent_runtime_error
  │           · journal run_failed；issue 终局评论；return(failed)
  ├─ pending 版 run 报告（803-810：verification_status="pending"）
  ├─ journal run_finished（811-816）
  ├─ update_status(verifying)（818-823），存 assistant_summary_preview(300字)
  ├─ verification_steps = _run_verification_steps(policies, cwd)（824）
  ├─ verification 报告双写（825-827）
  ├─ failing = 所有 status∈{failed,error} 的步骤（829）
  ├─ final_local_report 双写（830-837，overall=passed/failed）
  ├─ prior_summary = assistant_summary（838）
  ├─ 若 failing：
  │     metadata_updates: verification_failed/verification_steps(完整dump)/last_failure_*（842-847）
  │     ├─ attempt<max → repairing("local verification failed; retrying")（848-865）
  │     │     journal verification_failed；issue 发 _comment_local_failed
  │     │     prior_failure_stage="local_verification_failed" → continue
  │     └─ 否则 → failed（867-889），终局评论，return(failed)
  ├─ 非 git 仓库 → completed("...not a git repo so GitHub automation was skipped",
  │        human_gate_pending=True)（891-911），return(completed)
  ├─ commit_created = _git_commit_all(cwd, f"autopilot({id}): {title}")（913-916；1272-1277）
  ├─ branch_has_progress = commit_created or _git_branch_has_progress(base)（917-920；1282-1292
  │      用 rev-list --count origin/{base}..HEAD > 0 判断分支相对 base 是否有提交）
  ├─ 若无进展：
  │     ├─ attempt<max → repairing("agent produced no changes; retrying")（922-935）
  │     │        last_failure_stage="no_changes"，prior_failure_stage="no_changes"，continue
  │     └─ 否则 → failed(no_changes)（936-954），return(failed)
  ├─ 未新建提交但有进展 → journal existing_progress_detected（955-961，复用既有分支进度）
  ├─ _git_push_branch + _upsert_pull_request（963-971）
  │     └─ 异常 → failed(github_pr_open_failed)（972-991），终局评论，return(failed)
  ├─ update_status(waiting_ci: "waiting for remote CI on PR #{n}")（995-1007）
  │     metadata: linked_pr_number/url/linked_issue_numbers/autopilot_managed=True/
  │               verification_failed=False/verification_steps
  ├─ _comment_on_pr(_comment_pr_opened)（1008）
  ├─ ci_state, ci_summary, pr_snapshot, checks = await _wait_for_pr_ci（1010；见 6.3）
  ├─ update_status(waiting_ci, note=f"remote CI status: {ci_state}")，
  │     metadata: last_ci_conclusion/summary/checks/linked_pr_*（1011-1022）
  │     ※ 注意 1011-1013 的两分支条件相同，恒为 waiting_ci（简化残留）
  ├─ if ci_state == "failed":
  │     ├─ attempt<max → repairing(remote_ci_failed)（1024-1043）
  │     │     journal ci_failed_retry；pr 发 _comment_ci_failed；prior_failure_*=remote_ci_failed；continue
  │     └─ 否则 → failed(remote_ci_failed)（1045-1068）
  │           pr+issue 双终局评论；return(failed, 带 pr_number/pr_url)
  ├─ if _automerge_eligible(pr_snapshot, policies)（1070-1100）:
  │     _merge_pull_request(squash)；update_status(merged, human_gate_pending=False)
  │     journal merged；pr+issue _comment_merged；remove_worktree；return(merged,…)
  └─ else → update_status(completed, human_gate_pending=True)（1102-1134）
        journal human_gate_pending；pr+issue _comment_human_gate；remove_worktree；return(completed,…)

循环自然耗尽（1136-1150）→ failed(repair_exhausted)，return。
```

### 5.3 `metadata` 关键键全表（执行期由 run_card 写入）

| 键 | 来源 | 用途 |
| --- | --- | --- |
| `run_started_at` | preparing | 开始时间 |
| `execution_model` / `max_attempts` | preparing | 本轮模型与上限 |
| `worktree_slug` / `worktree_path` / `head_branch` / `base_branch` | preparing | 隔离环境信息 |
| `linked_issue_numbers: list[int]` | preparing | 关联 issue（PR body 里写 `Closes #N`） |
| `linked_pr_number` / `linked_pr_url` | waiting_ci 后 | 关联 PR |
| `autopilot_managed: bool` | 建 PR 后 | 幂等标记（防重复接管） |
| `attempt_count` | 每轮 | 当前尝试次数 |
| `assistant_summary_preview` | verifying | 模型摘要前 300 字 |
| `verification_failed` / `verification_steps` | 验证后 | 验证结论 + 全量 JSON |
| `last_failure_stage` / `last_failure_summary` | 各失败分支 | **喂给修复 prompt 的关键** |
| `execution_error` | agent_runtime_error | 原始异常串 |
| `last_ci_conclusion` / `last_ci_summary` / `last_ci_checks` | CI 轮询后 | CI 结论摘要 |
| `human_gate_pending: bool` | completed / merged | 看板"待人工"标记 |
| `url` / `updated_at_remote` / `is_draft` / `review_decision` / `merge_state_status` / `head_ref_name` / `base_ref_name` | scan_github_*

### 5.4 修复 prompt：`_prepare_repair_prompt`（1595-1626）

- attempt==1 或没有失败记录 → 直接返回 `_build_execution_prompt` 原样；
- 否则追加两段：`Repair context:`（Attempt、Previous failure stage、Previous failure summary、Previous agent summary≤600字符）＋ `Repair instructions:`（**最小补丁；不要推倒既有分支进度；修复后重跑验证命令**）。
- `_build_execution_prompt(card, policies)`（2013-2039）把三份策略 **YAML 原样 dump 进 prompt**，外加 Goal 五条约束（最小实现、自跑验证、不合并/发布/不可逆外部操作、留可审查状态、总结改动）与"Expected output"三段式（改了什么 / 验证了什么 / 剩余风险与人工跟进）。

### 5.5 Git 基础操作族（service.py:1227-1299）

| 方法 | 行号 | 说明 |
| --- | --- | --- |
| `_run_command(command, *, cwd, timeout, shell, check)` | 1227-1249 | 通用 `subprocess.run`，**对所有外部命令注入 `GIT_TERMINAL_PROMPT=0`、`GIT_ASKPASS=""` 屏蔽 git 挂起等待输入**；`check=True` 时非零码抛 `RuntimeError` |
| `_run_git / _run_gh` | 1251-1255 | 前缀 `git` / `gh` 的薄包装 |
| `_gh_json` | 1257-1262 | `gh ... --json` → 解析 JSON |
| `_git_has_changes` | 1264-1266 | `git status --porcelain` 非空 |
| `_is_git_repo` | 1268-1270 | `git rev-parse --git-dir` 返回 0 |
| `_git_commit_all(cwd, msg)` | 1272-1277 | 有变更才 `add -A` + `commit -m`，返回是否建了新提交 |
| `_git_push_branch` | 1279-1280 | `push -u origin <branch>` |
| `_git_branch_has_progress` | 1282-1292 | `rev-list --count origin/{base}..HEAD` 计数>0 |
| `_sync_worktree_to_base` | 1294-1299 | fetch base；首轮 `checkout -B head origin/base`（丢弃上轮分支），后续 `checkout head` 保留进度 |

---

## 六、GitHub 自动化、CI 轮询与双语评论

### 6.1 仓库/分支/PR 解析

- `_current_repo_full_name()`（1319-1324）：`gh repo view --json nameWithOwner`，失败抛 `RuntimeError`。
- `_issue_number_for_card`（1301-1308）：优先 `metadata.linked_issue_numbers[0]`，否则从 `source_ref`（`issue:N`）解析。
- `_linked_pr_number`（1310-1317）：优先 `metadata.linked_pr_number`，否则 `source_ref`（`pr:N`）。
- `_find_open_pr_for_branch(head_branch)`（1326-1342）：`gh pr list --state open --head <branch> --json number,url,...`，取第一个。
- `_best_effort_add_labels(pr_number, labels)`（1344-1355）：`gh pr edit --add-label ...`，失败只写 `github_warning` journal。

### 6.2 PR 幂等创建：`_upsert_pull_request`（1386-1433）

1. 先查已开 PR（1395-1398）— 有则补 `autopilot` 标签后直接复用（**幂等**）；
2. 否则 `_build_pr_body`（1357-1384）拼正文（`## Autopilot Summary` 含 Task ID/Source/Source ref、`## Reports` 引两个报告路径、`## Notes` 标明"Agent 自报摘要不可信"、关联 issue 时追加 `Closes #{issue}`）→ 写入 `tempfile.NamedTemporaryFile`；
3. `gh pr create --title "Autopilot: {title}" --body-file ... --base base --head head`，`finally` 删临时文件（1406-1427）；
4. 创建后再查证发现 PR（失败抛 RuntimeError），并补 `autopilot` 标签（1429-1433）。

### 6.3 CI 轮询（重点）

**`_pr_status_snapshot(pr_number)`（1497-1513）**：`gh pr view --json number,url,isDraft,labels,headRefName,baseRefName,mergeStateStatus,reviewDecision,statusCheckRollup`，并把 labels 归一化为名字列表。

**`_ci_rollup(pr_snapshot)`（1515-1546）**——把 GitHub 的 `statusCheckRollup` 归一化为 `[{name, status, conclusion, details_url}]`，并判定三态：

| 判定 | 条件（任一） | 返回 |
| --- | --- | --- |
| `pending` | 无任何 check | `"Remote CI checks have not appeared yet."` |
| `pending` | 有 check 处于 `QUEUED/IN_PROGRESS/PENDING/WAITING`，或 `status != COMPLETED` 且无 conclusion（1537） | `"Remote CI is still running."` |
| `failed` | 存在 `conclusion not in {SUCCESS, SKIPPED, NEUTRAL}`（1539-1545） | `"name=conclusion; ..."`（前 4 个） |
| `success` | 其余 | `"All reported remote checks passed."` |

**`_wait_for_pr_ci(pr_number, policies)`（1548-1572）**——异步轮询，细节：

```
timeout  = max(ci_timeout_seconds, 30)          # 默认 1800s
poll     = max(ci_poll_interval_seconds, 5)     # 默认 20s
no_checks_deadline = now + max(no_checks_grace_seconds, poll, 5)   # 默认 60s
checks_seen_at = None
loop:
  snapshot, (state, summary, checks) = _pr_status_snapshot + _ci_rollup
  if checks and checks_seen_at is None: checks_seen_at = now       # 首次看到 check
  if not checks and now >= no_checks_deadline: return success       # 无 check 超宽限判绿
  if state == success and checks and now < checks_seen_at + checks_settle_seconds:
        sleep(poll); continue                                        # 防抖动：刚出现 check 先稳定 20s
  if state in {success, failed}: return (state, summary, snapshot, checks)
  if now >= deadline: return failed("timed out")
  sleep(poll)
```

要点：**无 check 的 PR 在宽限期后视为成功**；有 check 则等其 settle；总超时判失败。`await` 的挂点就是事件循环让出点。

### 6.4 自动合并与既有 PR 监控

- `_automerge_eligible(pr_snapshot, policies)`（1574-1586）：
  - `isDraft` → False；
  - `mode == pr_only` → False（只开 PR 不合并）；
  - `mode == fully_auto` → True；
  - 默认 `label_gated` → PR labels 中含 `required_label`（默认 `autopilot:merge`，**大小写不敏感**）才合并。
- `_merge_pull_request(pr_number)`（1588-1593）：`gh pr merge <n> --squash`（`check=True`，失败抛异常）。
- `_process_existing_pr_card(card, pr_number, policies)`（1628-1700）——**"外部 PR 回收"路径**：不重跑 Agent，只把卡置 `waiting_ci` → 轮询既有 PR 的 CI → CI failed 则 `failed`；绿且可自动合并则 `merged`；否则 `completed(human_gate_pending=True)`；各分支都发对应评论。

### 6.5 双语评论家族（service.py:1435-1495）

所有评论 = `_bilingual_lines`（`"{zh}\n{en}"`）。发送方法统一 ```_comment_on_issue(issue, comment)`（1435-1443）与 `_comment_on_pr(pr, comment)`（1445-1453）```——全部 **try/except，被 GitHub 限流/鉴权失败只写 `github_warning` journal，绝不中断主流程**。

| 生成器 | 行号 | 内容（中/英） |
| --- | --- | --- |
| `_comment_started(card, attempt)` | 1455 | 开始处理第 N 轮 |
| `_comment_pr_opened(pr, url)` | 1461 | 已创建/更新 PR |
| `_comment_ci_failed(attempt, summary)` | 1467 | 远端 CI 失败，准备第 N+1 轮修复 |
| `_comment_local_failed(attempt, summary)` | 1473 | 本地 verification 失败，准备第 N+1 轮修复 |
| `_comment_merged(pr)` | 1479 | 已自动合并，闭环完成 |
| `_comment_human_gate(pr)` | 1485 | 验证+CI 都通过，仍待人工 gate / merge label |
| `_comment_terminal_failure(summary)` | 1491 | 自动化停止，原因：… |

---

## 七、验证门（verification gates）设计

### 7.1 条目解析与 shell 转义防御：`_parse_verification_entry`（155-196）

- 「字符串」或「`{command, shell}` 映射」二选一；空命令、非映射类型 → 生成带 `error="..."` 的 `_VerificationCommand`（不执行，直接产 error 步）。
- **关键防御**：`shell` 未显式开启时，若命令含 `_SHELL_METACHARS = frozenset(";&|`$<>\n\r")`（136 行）中任意字符 → 拒绝并 error，提示 **"use the mapping form {command: '...', shell: true} ... to opt in"**（175-184）。这堵住了 `uv run pytest; rm -rf /` 这类注入式命令。
- `shlex.split` 解析 argv；tokenize 失败或结果为空 → error。
- `_VerificationCommand` frozen dataclass（139-152）：`raw / argv / shell / error`。

### 7.2 可用性过滤：`_looks_available`（199-209）

```
uv → 需要 cwd/pyproject.toml        · ruff check → 需要 pyproject.toml
pytest → 需要 cwd/tests              · tsc/frontend/terminal → 需要 frontend/terminal/package.json
其它 → 恒 True
```

缺依赖的步骤直接跳过 —— **不执行也不算失败**，避免在结构不完整的项目上误报红灯。选择逻辑封装在 `_verification_commands(policies)`（2082-2092）：先 parse，`error` 条目保留（用于报错），`_looks_available` 为假则剔除。

### 7.3 执行：`_run_verification_steps`（2094-2155）

- 同步 `subprocess.run`（在事件循环内会阻塞，但验证命令通常快；`timeout=1800` 兜底）；
- stdout/stderr **仅保留末尾 4000 字符**（观察性 + 体积控制）；
- 异常分类：`FileNotFoundError`→`status="error"`（executable not found）；`TimeoutExpired`→`status="error"`（`Timed out after Ns`）；其它未知异常→`status="error"`（防御）；
- 正常结束 → `status="success" if returncode==0 else "failed"`。
- 每步产出 `RepoVerificationStep`；`failing = [{failed,error}]` 集合驱动 run_card 分支（829 行）。

### 7.4 报告渲染：`_render_verification_report`（2157-2184）

Markdown：头部 `# Verification Report: {id}` + Title；无步骤则 `No verification commands were applicable.`；每步一个 `## {STATUS.upper()} :: {command}` 标题 + Return code + `stdout`/`stderr` 代码块。

---

## 八、与主引擎的交互：_run_agent_prompt 逐行

`_run_agent_prompt(prompt, *, model, max_turns, permission_mode, cwd=None)`（service.py:2041-2080）：

```python
from openharness.ui.runtime import build_runtime, close_runtime, start_runtime

async def _allow(_tool_name, _reason):   # 全自动放行：任何工具都 True
    return True
async def _ask(_question):               # 永不向人提问
    return ""

bundle = await build_runtime(cwd=..., model=..., max_turns=...,
                             permission_prompt=_allow, ask_user_prompt=_ask,
                             permission_mode=permission_mode)     # 全新会话
await start_runtime(bundle)
collected = []
try:
    async for event in bundle.engine.submit_message(prompt):     # QueryEngine.submit_message (query_engine.py:227)
        if isinstance(event, AssistantTextDelta):  collected.append(event.text)
        elif isinstance(event, AssistantTurnComplete):
            text = event.message.text.strip()
            if text and not "".join(collected).strip():  collected.append(text)
        elif isinstance(event, ErrorEvent):          raise RuntimeError(event.message)
finally:
    await close_runtime(bundle)
return "".join(collected).strip()
```

**逐点回答"如何交互"**：
1. **是否开启新会话**：是。`build_runtime`（runtime.py:274）每次调用都构造一套全新的 `RuntimeBundle`——新的 settings 合并、新的 `McpClientManager`（逐个 `connect_all`）、新的默认工具注册表、新的 `AppStateStore`、随机的 `session_id`（runtime.py:377）。**不恢复会话、不注入外部对话历史**；autopilot 的"上下文"就是 prompt 里 dump 的那一份完整策略 + 任务描述。
2. **如何注入任务**：`submit_message(prompt)`，prompt 即 `_build_execution_prompt` 或修复版 `_prepare_repair_prompt` 的输出。工具侧：`create_default_tool_registry(mcp_manager)` 已注册读/写/执行等默认工具；插件工具也注册进去了（runtime.py:324-329）。`cwd` 指向 worktree，所以模型所有 shell 操作都落在隔离分支上。
3. **如何拿工具输出**：autopilot **不订阅工具事件**（`ToolExecutionStarted/Completed` 在 `stream_events` 中存在，但这里只关心三种：`AssistantTextDelta`、`AssistantTurnComplete`、`ErrorEvent`）。工具的执行交由引擎内部完成（全自动权限可见），autopilot 只回收**最终文本摘要**作为 "agent self-report"。
4. **哪些事情会失败**：引擎内部若出现模型错误等会发 `ErrorEvent`，这里立刻 `raise RuntimeError` → run_card 捕获 → `failed(agent_runtime_error)`。
5. **会话生命周期**：`start_runtime`/`close_runtime`（runtime.py:474/482）配对；`finally` 保证异常也关闭 MCP 等资源。

---

## 九、报告、仪表盘导出与 tick/cron 调度

### 9.1 运行报告：`_render_run_report`（2186-2239）

Markdown 三段式：
- `## Agent Self-Reported Summary`：模型自报；
- `## Service-Level Ground Truth`：**信任边界声明**——"上面的段落是模型自报摘要，在下面的 service 级验证完成前视为不可信（untrusted）"（2205-2210）；
- 验证状态节：`not_started`（Agent 自身失败）、`pending`、或 `passed/failed` + 每步 `[{status}] command (rc=N)`。

报告文件按尝试**一式两份**：`{card.id}-attempt-{NN:02d}-run.md`（历史）＋ `{card.id}-run.md`（当前）；验证报告同理 `-verification.md`（713-714、806-807）。

### 9.2 仪表盘导出：`export_dashboard`（1195-1206）

```python
target_dir = cwd/docs/autopilot  （可用 output_dir 覆盖）
mkdir(parents=True, exist_ok=True)
atomic_write_text(target_dir/snapshot.json, json.dumps(snapshot, ensure_ascii=False, indent=2))
atomic_write_text(target_dir/index.html,  _render_dashboard_html(snapshot))   # fallback 页
atomic_write_text(target_dir/.nojekyll, "")                                   # GitHub Pages 直出
```

> `index.html` 为**静态兜底页**：`_render_dashboard_html`（1828-1886）是纯内联 CSS/HTML——只有当输出目录**没有预构建的 React index.html** 才会被覆盖写入（否则 vite build 产物保留）；页上提示 `cd autopilot-dashboard && npm install && npm run build`。`.nojekyll` 让 GitHub Pages 不跑 Jekyll 直接静态托管。

**快照结构** `_build_dashboard_snapshot`（1702-1766）：

| 键 | 来源 |
| --- | --- |
| `generated_at` | `time.time()` |
| `repo_name` / `repo_path` | `cwd.name` / `cwd` |
| `focus` | 焦点卡（优先级 `repairing>waiting_ci>running>verifying>preparing>accepted>queued`，1734-1739 由 columns 桶序取首个） |
| `counts` | 13 状态计数 |
| `status_order` | 13 状态固定顺序（1713-1727） |
| `columns` | `{status: [serialized cards]}` |
| `cards` | 全量序列化卡 |
| `journal` | **最近 30 条** journal |
| `policies` | 三个 YAML 路径 |
| `active_context` | `active_repo_context.md` 全文 |

卡片序列化 `_serialize_card`（1768-1808）——只挑选 UI 需要的字段：基础段 + metadata 扁平化子集（`last_note/url/execution_model/assistant_summary_preview/human_gate_pending/verification_failed/attempt_count/max_attempts/linked_pr_*/last_ci_*/last_failure_*`）+ `verification_steps` 的**精简形态**（仅 command/status/returncode）。排序键 `_status_sort_key`（1810-1826）：`repairing(0) < waiting_ci(1) < running(2) < verifying(3) < preparing(4) < accepted(5) < pr_open(6) < queued(7) < completed(8) < merged(9) < failed(10) < rejected(11) < superseded(12)`，未知状态 99。

### 9.3 调度：tick 与 cron

- `tick(*, model, max_turns, permission_mode, issue_limit, pr_limit) -> RepoRunResult | None`（1152-1172）：
  1. `scan_all_sources(...)` 先扫描；
  2. 若有 active 卡（preparing/running/verifying/waiting_ci/repairing）→ journal `tick_skip`，返回 None（**不去抢占**）；
  3. `pick_next_card()` 为 None → journal `tick_idle`，返回 None；
  4. 否则 `run_next()` 端到端执行并把 `RepoRunResult` 返回。
- `install_default_cron()`（1174-1193）：复用 `openharness.services.cron.upsert_cron_job`，注册两条任务——
  - `autopilot.scan`：每 30 分钟 `oh autopilot scan all --cwd {cwd}`；
  - `autopilot.tick`：每 2 小时 `oh autopilot tick --cwd {cwd}`。
  返回任务名列表。

### 9.4 来源扫描（补充 4.6 输入面）

- `scan_github_issues(limit=10)`（502-534）：`gh issue list --state open --limit N --json number,title,body,labels,updatedAt,url`，逐条 `enqueue_card(source_kind="github_issue", source_ref=f"issue:{number}", metadata={url, updated_at_remote})`。
- `scan_github_prs(limit=10)`（536-573）：`gh pr list --state open ... --json number,title,body,isDraft,reviewDecision,mergeStateStatus,updatedAt,url,labels,headRefName,baseRefName`，metadata 携带 `is_draft/review_decision/merge_state_status/head_ref_name/base_ref_name`（打分用到前三个）。
- `scan_claude_code_candidates(limit=10, root=None)`（575-607）：${HOME}/claude-code/{commands,agents}/*（跳过点文件），source_ref=`command:{path}`/`agent:{path}`，标题 `Evaluate claude-code {label}: {name}`。
- `scan_all_sources(issue_limit=10, pr_limit=10)`（609-625）：三类逐 try/except，**单个来源失败只写 `scan_warning` 不中断**；末尾 journal `scan_all` + `rebuild_active_context()`。

---

## 十、CLI 入口与斜杠命令路由

### 10.1 `oh autopilot ...` 子命令（cli.py:773-1232）

`autopilot_app = typer.Typer(name="autopilot", help="Manage repo autopilot")`（cli.py:773）；`app.add_typer(autopilot_app)`（781）；命令组注册进 units（184）。

| 子命令 | 行号 | 行为 |
| --- | --- | --- |
| `status` | 1011-1043 | 打印 `stats()` 各状态计数 + `pick_next_card()` 候选 + 四个状态文件路径 |
| `list` | 1045-1063 | 按状态过滤列出卡片（id/title/score/status） |
| `add` | 1065-1096 | 手动入队；`--source` 简写映射（idea/ohmo/issue/pr/claude） |
| `context` | 1098-1107 | 打印 `rebuild_active_context()` / `load_active_context()` |
| `journal` | 1109-1124 | 打印最近日志 |
| `scan` | 1126-1150 | `issues/prs/claude-code/all` |
| `run-next` | 1152-1177 | `asyncio.run(run_next(...))` 端到端执行 |
| `tick` | 1179-1210 | 扫描 + 若空闲执行 |
| `install-cron` | 1212-1221 | 安装默认 cron |
| `export-dashboard` | 1223-1232 | `export_dashboard()` 并打印导出路径 |

### 10.2 `/autopilot ...` 斜杠命令（commands/registry.py:2051-2300，注册于 2531-2533）

`_autopilot_handler` 支持 add/list/scan/run/next/journal/context/install-cron/export-dashboard，语义对齐 CLI；`/autopilot run <card-id>`（2303）等直接调 Store 方法。另有一处细节：`/autopilot add` 返回时带 `status_word`（accept/add/import 三态，2193）。

---

## 十一、autopilot-dashboard 前端全解

### 11.1 工程配置

| 文件 | 要点 |
| --- | --- |
| `package.json` | 私有包 `autopilot-dashboard` v0.1.0，`"type":"module"`；脚本 `dev=vite` / `build="tsc -b && vite build"` / `preview`；deps **React 19.1 + react-dom**；devDeps `@vitejs/plugin-react ^4.4.1`、`typescript ~5.8.3`、`vite ^6.3.2`、对应 @types。**零 UI 库、零状态库、零路由**。 |
| `vite.config.ts` | `base: "./"`（相对路径，适配 GitHub Pages 子路径）；`build.outDir: "../docs/autopilot"`；`emptyOutDir: false`（**保留服务端运行时写入的 `snapshot.json`**——"服务端写数据 + 前端写页面"合璧的关键）。 |
| `tsconfig.app.json` | 严格模式全家桶：`strict / noUnusedLocals / noUnusedParameters / noFallthroughCasesInSwitch / noUncheckedSideEffectImports`；`jsx: react-jsx`；`moduleResolution: bundler`；`noEmit`（仅类型检查）。 |
| `index.html` | `<div id="root">` + **JetBrains Mono**（400-700）谷歌字体预连（与终端/接线板视觉一致）。 |
| `src/main.tsx` | `createRoot(...).render(<StrictMode><App /></StrictMode>)`；引入 `./index.css`（546 行，暗色 + 霓虹主题的全部板式样式）。 |

### 11.2 数据契约（types.ts，84 行）

| 定义 | 行号 | 内容 |
| --- | --- | --- |
| `TaskCard` | 1-19 | 镜像后端 `_serialize_card` 的**字段子集**：`id/title/body/status/score/score_reasons/source_kind/source_ref/labels/updated_at/metadata`；`metadata` 内取 `last_note/last_ci_summary/last_failure_summary/human_gate_pending/verification_steps[{status,command}]`。 |
| `JournalEntry` | 21-26 | `timestamp/kind/task_id?/summary`（不含 metadata，够 UI 用即可）。 |
| `Snapshot` | 28-37 | `generated_at/repo_name/focus?/counts/status_order/columns/cards/journal`（policies/active_context 后端有、前端类型未声明——**用不到就不建模**）。 |
| `STATUS_LABELS` | 39-53 | 13 状态文案（Queued…Superseded）。 |
| `STATUS_COLORS` | 55-69 | 13 状态色（`queued #64748b`、`running/completed/merged #00d4aa` 青、`repairing #ff6b35` 橙、`failed/rejected #ff4444` 红、`waiting_ci/verifying #3b82f6` 蓝、`accepted/pr_open #8b5cf6` 紫、`superseded #ffaa00` 琥珀）。 |
| `KanbanGroup` | 72-77 | 列分组结构。 |
| `KANBAN_GROUPS` | 79-84 | **13 状态归并 4 列**：`To Do(queued,accepted)`、`In Progress(preparing,running,repairing)`、`In Review(verifying,pr_open,waiting_ci)`、`Done(completed,merged,failed,rejected,superseded)`。 |

### 11.3 App.tsx（290 行）——页面结构

三个工具函数：
- `fmtAgo(ts)`（9-16）：秒→`42s/5m/3h/2d ago`。
- `statusBadgeClass(status)`（18-26）：六种徽章配色映射（teal/orange/violet/red/blue/amber/gray）。

五大组件：

| 组件 | 行号 | 职责 |
| --- | --- | --- |
| `CardView` | 30-75 | 单卡：顶部 id + 状态徽章；标题；正文前 260 字符；tag 行（labels ∪ source_kind）；footer 展示 `score · updated · ref` + `last_note` + 验证/CI/失败/human-gate 摘要；`--card-accent` 取 `STATUS_COLORS` 做左边框色（36-41）。 |
| `GroupColumnView` | 79-101 | 一列：圆点 + 列名 + 计数；空列显示 `No cards.`。 |
| `JournalView` | 105-136 | `// RECENT JOURNAL`，**倒序**渲染，时间戳转 UTC 可读串。 |
| `App` | 140-290 | 根组件，唯一持有 state 的组件。 |

**数据来源与渲染管线**：
```
useEffect(() => fetch("./snapshot.json", { cache: "no-store" }))   # App.tsx:146-150
  ├─ 失败/缺数据 → LOADING/错误壳（152-168）
  ├─ KANBAN_GROUPS.map → 每列先 flatMap columns[status] 取卡，再过滤器（176-185）
  │    过滤器：id/title/body/source_kind/source_ref/labels/score_reasons 拼串 contains
  ├─ 统计（187-191）：inProgress = preparing+running+verifying+waiting_ci+repairing+accepted+pr_open
  │    completed=completed+merged；failed=failed+rejected（Done 列展示 completed+failed，注意不算 rejected）
  └─ 渲染顺序：Hero(199-231) → Stats Bar(235-258) → Toolbar(261-271) → Board(274-283) → Journal(286)
```

**Hero 区细节**（199-231）：左侧 `// AUTOPILOT_KANBAN` eyebrow + 大标题 `OpenHarness / SELF-EVOLUTION` + `CURRENT_FOCUS` 焦点盒（`[status] title · score=N · source_kind`）；右侧生成时间戳 + `PipelineAnimation`。`HeroBackground` 铺 hero-bg 全底（200-202）。

**Stats Bar**（235-258）四个 stat：`TO DO`(仅 queued，但副标题写 "queued + accepted")、`IN PROGRESS`、`IN REVIEW`(=verifying+pr_open+waiting_ci)、`DONE`(=completed+failed)。

**Toolbar**（261-271）：文本输入过滤 + 硬编码提示 `Reads snapshot.json — no backend required`（269 行，明示静态）。

### 11.4 PipelineAnimation.tsx（223 行）

**5 阶段枚举正是在这里**（15-25 行，极坐标排布、顺时针）：

```
QUEUE → PREP → RUN → CHECK → MERGE
angle_i = π + 2π/5·i  （i=0..4），椭圆 cx=200, cy=105, rx=145, ry=65
```

恰好对应用户可观察的业务链路：**入队 → worktree 准备 → Agent 执行 → 验证/CI 检查 → 合并/人工 gate**。元素清单：

| 元素 | 行号 | 说明 |
| --- | --- | --- |
| defs 四渐变 | 58-77 | `pipe-glow`(径向)、`hub-glow`、`node-glow`、`edge-fade`(线性) |
| 背景网格 | 79-87 | 5 横 × 9 竖 0.04 透明度线 |
| 中心辉光 + 轨道环 | 89-99 | 静态环 + `stroke-dashoffset 0→-24` 的虚线旋转环 |
| 辐条 + 数据脉冲 | 101-118 | 每站一条辐条线，`animateMotion` 数据点以 0.4s 相移往复 |
| 五个阶段节点 | 120-165 | 每节点：辉光→外环(节点专属色 `#64748b/#0f766e/#00d4aa/#3b82f6/#8b5cf6`)→四角 bracket→脉动核心点→扩散环→字母标签 |
| 中心 hub | 167-177 | `∞`（`&#x221E;`）图标 + 扩散光环 |
| 两个巡游数据包 | 179-195 | packet1：5s/顺时/teal；packet2：7s、2.5s 起始偏移/violet，沿 `orbitPath`（五角闭合多边形）`rotate=auto` |
| 环境粒子 | 197-206 | 10 个呼吸粒子（violet/teal 交替） |
| 扫描线 | 208-214 | 4s 自上而下 + 透明度 keyframe |
| 底部标题 | 216-220 | `AUTOPILOT · PIPELINE` |

**结论：全 SVG，所有动效都是 `<animate>`/`<animateMotion>`（SMIL），零 JS 动画循环**（注释 1-8 亦如此声明）。

### 11.5 HeroBackground.tsx（181 行）

全出血 `viewBox="0 0 1200 460"` + `preserveAspectRatio="xMidYMid slice"`，`aria-hidden="true"`。**7 层（back→front）**，逐层（108-178）：

1. 中央径向辉光（110）。
2. **透视网格地板**（112-121）：`mask` 由五档 vertical fade 实现消失→浮现→淡出（98-106）；横线 `gridY=[300,325,355,390,430]` 渐增透明度；放射线从消失点 `(vx,vy)=(600,240)` 向 `y=460` 散射。
3. **数据流虚线**（124-130）：5 条水平虚线（`dash="4 18"` 等），`stroke-dashoffset` 动画模拟流动。
4. **星座连线/节点**（132-146）：12 节点 + 14 边，节点呼吸脉动。
5. **数据片段**（149-156）：8 条短横线（violet/teal 交替）闪烁。
6. **二进制/十六进制雨**（159-169）：20 个 `0/1/A…F` 字形，`translate from 0,0 → 0,480` 下落 + 透明度 keyframe（`0;0.2;0.2;0`）。
7. **扫描线**（171-177）：6s 往复扫过全画面。

### 11.6 是否连接后端？——结论

**纯静态展示页，零后端 API 调用**。数据通路：

```
RepoAutopilotStore.export_dashboard()  (service.py:1195)
  ├─ 原子写 docs/autopilot/snapshot.json      ← React fetch 的数据（{cache:"no-store"}）
  ├─ 写 fallback index.html（仅当目录无预构建产物时）  
  └─ 写 .nojekyll
本地/CI: cd autopilot-dashboard && npm run build   ← vite 覆盖为真 React 产物(emptyOutDir:false 保留 json)
浏览器:  docs/autopilot/index.html → fetch("./snapshot.json")
```

`public/snapshot.json`（59 行）是随源码提交的**示例快照**（repo 名 `OpenHarness-new`、全空计数、策略路径指向 `/home/tangjiabin/...`、`active_context` 为演练文案），供 `vite dev` 直接预览布局；真实部署时由服务端每次写入的快照取代。因此：**没装 Node 也能服务端出数据（fallback 页），装 Node 后由构建产物升级为完整仪表盘**。

---

## 十二、辅助模块 utils（fs/shell/file_lock/helpers）

### 12.1 fs.py — 原子写文件（98 行）

**意图**（docstring 1-26）：`~/.openharness/` 下一切会被反复重写的持久化文件（credentials、settings、session、cron registry、memory index）都**必须原子写**——`write_text` 在崩溃/SIGKILL/断电/磁盘满时留下截断文件，下次读取静默 `{}` 或抛 `JSONDecodeError`，只能人工修复。方案是经典"同目录临时文件 + rename"：

| 函数 | 行号 | 行为 |
| --- | --- | --- |
| `atomic_write_bytes(path, data, *, mode=None)` | 39-66 | `dst.parent.mkdir` → `_resolve_target_mode` → `mkstemp(".{name}." prefix, ".tmp", 同目录)` → 写入 + `flush` + **`fsync`** → `_apply_mode`（文件仍私有时应用权限）→ **`os.replace` 原子换入**；`BaseException` 时 `suppress(OSError)` 清理临时文件再抛 |
| `atomic_write_text(path, data, *, encoding="utf-8", mode=None)` | 69-77 | 编码后委托 bytes 版 |
| `_resolve_target_mode` | 80-89 | 显式 mode 优先；已存在文件保留原 mode；新文件按 `0o666 & ~umask` |
| `_apply_mode` | 92-97 | `os.chmod` 失败（Windows/FAT/网络盘）只吞异常——数据完整性不受影响 |

> Python 3.3 起 `os.replace` 在 Windows 上也有原子语义（docstring 18-20 明示）。

### 12.2 shell.py — 平台感知子进程封装（147 行）

| 函数 | 行号 | 行为 |
| --- | --- | --- |
| `resolve_shell_command(command, *, platform_name=None, prefer_pty=False)` | 17-48 | **按平台挑 shell argv**：Windows→ 先 `WSL bash -lc`（必须 `_bash_is_usable` 实测可用）→ `pwsh/powershell -NoLogo -NoProfile -Command` → `cmd.exe /d /s /c`；POSIX→ `bash -lc`，无 bash 用 `$SHELL`/`/bin/sh`；`prefer_pty=True` 时用 `script -qefc <cmd> /dev/null` 包装（macOS 跳过，108-121） |
| `create_shell_subprocess(command, *, cwd, settings=None, prefer_pty=False, stdin/stdout/stderr/env)` | 51-105 | 异步入口：**Docker 沙箱路由**（`settings.sandbox.enabled and backend=="docker"` → `get_docker_sandbox().exec_command(argv,...)`，沙箱不可用且 `fail_if_unavailable` 抛 `SandboxUnavailableError`，66-83）；否则 `wrap_command_for_sandbox` SRT 包装（87，可能产生 `cleanup_path`）→ `asyncio.create_subprocess_exec`；启动失败清理临时文件；成功后注册退出时清理任务（103-104） |
| `_wrap_command_with_script` | 108-121 | macOS 返回 None（11-12）；非 macOS 且有 `script` 且 argv 形如 `bash -lc cmd` → 包成 `script -qefc cmd /dev/null` |
| `_bash_is_usable(bash)` | 124-140 | 防御 WSL 陷阱：`which bash` 可能命中 WSL 的 `bash.exe` 但无发行版；实测 `bash -lc "exit 0"`（5s 超时），失败回退 PowerShell/cmd |
| `_cleanup_after_exit(process, cleanup_path)` | 143-147 | `await process.wait()` 后删临时文件 |

### 12.3 file_lock.py — 跨平台排他文件锁（86 行）

用途：对共享 JSON 注册表（credentials/settings/cron/memory index/swarm mailbox）的 read-modify-write 临界区加锁，与 `fs.atomic_write_text` 组合出"既无竞态又崩溃安全"的写法。

| 标识 | 行号 | 说明 |
| --- | --- | --- |
| `SwarmLockError(RuntimeError)` | 18 | 基类 |
| `SwarmLockUnavailableError` | 22 | 平台不支持时抛 |
| `exclusive_file_lock(lock_path, *, platform_name=None)` | 27-44 | 上下文管理器；按平台分派：windows→`_exclusive_windows_lock`；macos/linux/wsl→`_exclusive_posix_lock`；其它→Unavailable |
| `_exclusive_posix_lock` | 48-61 | `fcntl.flock(LOCK_EX/LOCK_UN)`；`mkdir(lock 父目录)` + `touch`；`a+b` 打开（**锁 inode/fd 级，跨进程互斥**） |
| `_exclusive_windows_lock` | 65-86 | `msvcrt.locking(LK_LOCK/LK_UNLCK, 1)` 锁首字节；文件为空先写 `\0`（`msvcrt.locking` 要求字节范围存在） |

### 12.4 helpers.py — 通道兼容工具（77 行）

定位"窄且零依赖"，供历史 channel 代码导入（docstring 1-6）：

| 函数 | 行号 | 行为 |
| --- | --- | --- |
| `get_data_path()` | 19-25 | `get_data_dir()` 向后兼容别名 |
| `split_message(text, max_length)` | 28-60 | `max_length<=0` 抛 `ValueError`；空文本→`[]`；≤上限→`[text]`；否则循环：优先 `rfind("\n", 0, max+1)`，其次空格，最后硬切 `max`；`rstrip()/lstrip()` 清理块边界 |
| `safe_filename(value, *, max_length=128)` | 63-77 | 任意值→安全 basename：取 basename→`NFKC` 规范化→`re.sub(r"[^A-Za-z0-9._-]+", "_")`→去首尾 `.`/`_`/`-`→去 `.`/`..`→限长 128 |

---

## 十三、辅助模块（themes / output_styles / keybindings / vim / voice / state / bridge / platforms / \_\_main\_\_）

### 13.1 themes

**schema.py（54 行）**——Pydantic 四层配置模型：

| 模型 | 行号 | 字段（默认值） |
| --- | --- | --- |
| `ColorsConfig` | 10-19 | `primary #5875d4`、`secondary #4a9eff`、`accent #61afef`、`error #e06c75`、`muted #5c6370`、`background #282c34`、`foreground #abb2bf`（一套 One Dark-ish） |
| `BorderConfig` | 22-26 | `style ∈ rounded|single|double|none = rounded`、`char: str\|None = None` |
| `IconConfig` | 29-36 | `spinner ⠋`、`tool ⚙`、`error ✖`、`success ✔`、`agent ◆` 五枚字形 |
| `LayoutConfig` | 39-44 | `compact=False`、`show_tokens=True`、`show_time=True` |
| `ThemeConfig` | 47-54 | `name` + 上述四段（均带默认实例） |

**builtin.py（89 行）**——`BUILTIN_THEMES: dict[str, ThemeConfig]`（13 行起）五主题：

| 主题 | 配色基调 | border | icons 风格 |
| --- | --- | --- | --- |
| `default` | One Dark 蓝紫 | rounded | ⠋⚙✖✔◆ |
| `dark` | Tokyo Night 紫青（`#bb9af7/#7aa2f7/#9ece6a/...`） | single | `· * ! + >` |
| `minimal` | 纯黑白灰，`compact=True`、关 tokens/time | none | `- : E . #` |
| `cyberpunk` | 荧光绿(`#00ff41`)/紫(`#bf00ff`)/青(`#00ffff`)，DPI 红 | **double** | `◈ ◉ ◌ ◍ ◎` |
| `solarized` | 经典深色（`#268bd2` 等） | rounded | ⠋⚙✖✔◆ |

**loader.py（55 行）**：

| 函数 | 行号 | 行为 |
| --- | --- | --- |
| `get_custom_themes_dir()` | 15-19 | `~/.openharness/themes`，自动建目录 |
| `load_custom_themes()` | 22-32 | `*.json` → `ThemeConfig.model_validate`；坏文件仅 `logger.debug` 跳过 |
| `list_themes()` | 35-41 | 内置 + 自定义去重 |
| `load_theme(name)` | 44-55 | **自定义优先、内置兜底**；都无则 `KeyError` 并列出可用主题 |

### 13.2 output_styles（42 + 5 行）

- `OutputStyle` frozen dataclass（11-17）：`name / content / source`。
- `get_output_styles_dir()`（20-24）：`{config_dir}/output_styles`，自动建。
- `load_output_styles()`（27-41）：先拼三条内置样式——`default`(rich 标准输出)、`minimal`(极简纯文本)、`codex`(Codex 风格紧凑 transcript)——再追加该目录里用户放的 `*.md`（`source="user"`，**文件名即样式名**）。

### 13.3 keybindings（四文件管线）

| 文件 | 行号 | 内容 |
| --- | --- | --- |
| `default_bindings.py` | 6-10 | `DEFAULT_KEYBINDINGS = {"ctrl+l": "clear", "ctrl+k": "toggle_vim", "ctrl+v": "toggle_voice", "ctrl+t": "tasks"}` |
| `parser.py` | 8-18 | `parse_keybindings(text)`：`json.loads`；必须 dict；键值必须 str，否则 `ValueError` |
| `resolver.py` | 8-13 | `resolve_keybindings(overrides=None)`：默认表上 `dict.update` 覆盖合并 |
| `loader.py` | 12-22 | `get_keybindings_path()` → `{config_dir}/keybindings.json`；`load_keybindings()`：无文件→纯默认；有文件→parse 后 resolve |

### 13.4 vim（transitions.py，8 行）

`toggle_vim_mode(enabled: bool) -> bool: return not enabled`（6-8）。仅状态切换帮助函数；真正的 Vim 行为在 TUI 层（textual）实现。

### 13.5 voice

| 文件 | 行号 | 内容 |
| --- | --- | --- |
| `voice_mode.py` | 11-17 | `VoiceDiagnostics` frozen dataclass：`available / reason / recorder=None` |
| | 20-22 | `toggle_voice_mode(enabled)` 取反 |
| | 25-43 | **`inspect_voice_capabilities(provider: ProviderInfo)`**：先 `shutil.which("sox") or which("ffmpeg") or which("arecord")` 找录音器；`provider.voice_supported` 为假→不可用（`reason=provider.voice_reason`）；无录音器→不可用（"no supported recorder..."）；否则可用（"voice shell is available"）。**这只是能力探测，不做转写** |
| `keyterms.py` | 8-10 | `extract_keyterms(text)`：`re.findall(r"[A-Za-z0-9_]{4,}")` 抽 4+ 长度词 → 小写 → `sorted` 去重 |
| `stream_stt.py` | 6-8 | `transcribe_stream(_: bytes) -> str` **占位**：恒返回 `"Streaming STT is not configured in this build."`——流式 STT 尚未接入，接口先立 |

### 13.6 state

| 文件 | 行号 | 内容 |
| --- | --- | --- |
| `app_state.py` | 8-30 | `AppState` dataclass 一枚扁平快照：`model / permission_mode / theme / cwd="." / provider="unknown" / auth_status="missing" / base_url="" / vim_enabled / voice_enabled / voice_available / voice_reason / fast_mode / effort="medium" / passes=1 / mcp_connected=0 / mcp_failed=0 / bridge_sessions=0 / output_style="default" / keybindings={}`。初始值由 `build_runtime` 依据解析出的 settings 填充（runtime.py:332-355） |
| `store.py` | 11-40 | `Listener = Callable[[AppState], None]`；`AppStateStore`：`get()` 返回当前快照；`set(**updates)` 用 `dataclasses.replace` 生成**新实例**并通知监听；`subscribe(listener)` 返回取消订阅闭包 |

### 13.7 bridge

| 文件 | 行号 | 内容 |
| --- | --- | --- |
| `types.py` | 9 | `DEFAULT_SESSION_TIMEOUT_MS = 24h` |
| | 12-17 | `WorkData`：`type ∈ session\|healthcheck` + `id` |
| | 20-26 | `WorkSecret`：`version / session_ingress_token / api_base_url`（三个 frozen dataclass） |
| | 29-36 | `BridgeConfig`：`dir / machine_name / max_sessions=1 / verbose=False / session_timeout_ms` |
| `session_runner.py` | 13-29 | `SessionHandle` dataclass：`session_id/process/cwd/started_at`；`kill()` 先 `terminate` 等 3s，超时才 `kill` |
| | 32-46 | `spawn_session(*, session_id, command, cwd)`：`create_shell_subprocess`（`stdout=PIPE`、`stderr=STDOUT` 合并） |
| `manager.py` | 13-23 | `BridgeSessionRecord` frozen dataclass：会话 UI 安全快照 `session_id/command/cwd/pid/status/started_at/output_path` |
| | 26-45 | `BridgeSessionManager.spawn()`：落盘 `{data_dir}/bridge/{session_id}.log`（先清空）+ `create_task(_copy_output)` |
| | 47-68 | `list_sessions()`：按 `started_at` 逆序，状态 running/completed/failed |
| | 70-77 | `read_output(session_id, max_bytes=12000)`：截尾读 |
| | 79-83 | `stop()`：`kill` 杀掉进程 |
| | 85-94 | `_copy_output`：4096 字节块循环追加写日志，`process` 退出后结束 |
| | 97-105 | 模块级单例 `get_bridge_manager()` |
| `work_secret.py` | 11-14 | `encode_work_secret(secret)`：紧凑 JSON → base64url → 去 `=` 填充 |
| | 17-32 | `decode_work_secret`：补 `=` → decode → `version==1`（否则 `ValueError`）→ token 必非空 → `api_base_url` 必为 str |
| | 35-40 | `build_sdk_url(api_base_url, session_id)`：**本地（含 `localhost`/`127.0.0.1`）→ `ws://` + `v2` 路径；远端 → `wss://` + `v1` 路径**；`{protocol}://{host}/{version}/session_ingress/ws/{session_id}` |

### 13.8 platforms.py（86 行）

| 标识 | 行号 | 内容 |
| --- | --- | --- |
| `PlatformName` | 11 | `Literal["macos","linux","windows","wsl","unknown"]` |
| `PlatformCapabilities` frozen dataclass | 14-24 | `name / supports_posix_shell / supports_native_windows_shell / supports_tmux / supports_swarm_mailbox / supports_sandbox_runtime / supports_docker_sandbox` |
| `detect_platform(*, system_name, release, env)` | 27-46 | `darwin→macos`；`windows/win32→windows`；`linux` 且 release 含 `microsoft` 或 env 有 `WSL_DISTRO_NAME`/`WSL_INTEROP` → **wsl**，否则 linux；其它→unknown |
| `get_platform()` | 49-52 | `@lru_cache(maxsize=1)` 进程内探测一次 |
| `get_platform_capabilities(name=None)` | 55-86 | **能力矩阵**：macos/linux/wsl 全量 True（POSIX shell、tmux、mailbox、sandbox、docker）；windows 仅 native shell；unknown 全 False。这是 shell 选型、swarm mailbox、sandbox 开关的决策输入 |

### 13.9 \_\_main\_\_.py（6 行）

`python -m openharness` 入口：`from openharness.cli import app;  if __name__=="__main__": app()` —— Typer 应用从 CLI 模块导入。

---

## 十四、scripts 脚本清单与重点脚本精讲

`scripts/` 共 15 个脚本：`.sh`×3（install.sh、install_dev.sh、sync_nanobot_channels.sh）、`.ps1`×1（install.ps1）、`.py`×11。

### 14.1 用途清单

| 脚本 | 行数 | 用途 |
| --- | --- | --- |
| `install.sh` | 387 | **macOS/Linux/WSL 一键安装器**（详见 14.2） |
| `install.ps1` | 329 | **Windows (PowerShell) 安装器**（详见 14.3） |
| `install_dev.sh` | 181 | 开发者安装：把**当前检出**以 editable 模式装进 venv，注册 oh/ohmo 到 `~/.local/bin`；`--global-venv` 用系统 venv、`--with-channels` 兼容标志 |
| `sync_nanobot_channels.sh` | 90 | 上游 nanobot channels 同步：读 `src/openharness/channels/UPSTREAM` 记录的 commit；无 `--apply` 时仅 diff HEAD 与上游；`--apply` 时复制更新文件、改写 import、更新 UPSTREAM 指针 |
| `e2e_smoke.py` | 807 | **真实模型端到端冒烟**：对 Anthropic 兼容 API 跑一组 `Scenario`（`setup` + `validate` 回调，含 fake MCP 夹具），覆盖引擎/工具/权限/MCP/记忆全链路 |
| `local_system_scenarios.py` | 248 | 本地系统场景（`FakeApiClient`，不调模型）：MCP、插件安装/卸载、bridge（work-secret 编解码 + `spawn_session`）、斜杠命令、工具注册 |
| `migrate_memory.py` | 14 | `openharness.memory.migrate.main()` 的绝薄包装 |
| `react_tui_e2e.py` | 161 | pexpect 驱动真实 CLI（`uv run oh`）的 React TUI 脚本化 E2E |
| `test_cli_flags.py` | 157 | **kimi 模型**（Moonshot Anthropic 端点）验证 CLI flag 组（详见 14.4） |
| `test_docker_sandbox_e2e.py` | 654 | Docker 沙箱全生命周期 E2E：镜像构建、容器生命周期、命令执行、文件/网络隔离、资源限制、清理；无 Docker 自动 skip（pytest 兼容） |
| `test_harness_features.py` | 239 | 真实模型验证 harness 特性：retry、skills、并行工具、路径权限 |
| `test_headless_rendering.py` | 166 | headless REPL 渲染：markdown、工具输出格式化、spinner |
| `test_react_tui_redesign.py` | 187 | React TUI 改版 E2E：对话布局、欢迎横幅、工具展示（pexpect 驱动） |
| `test_real_skills_plugins.py` | 346 | **真实** skill/plugin 仓库（`/tmp/anthropic-skills/skills`、`/tmp/openharness-test-plugins/plugins`）E2E：技能加载、插件加载、命令执行、hooks |
| `test_tui_interactions.py` | 202 | React TUI 交互：命令选择器、权限流、快捷键 |

> 共性：`sys.path.insert(0, src)` 让脚本可直接 `python scripts/xxx.py`；`ANTHROPIC_BASE_URL` / `ANTHROPIC_MODEL` / `ANTHROPIC_AUTH_TOKEN` 环境变量选模型与密钥；输出 `[GREEN/RED] PASS/FAIL`；除 `test_docker_sandbox_e2e.py`（同时兼容 pytest，自动 skip）外，**均不属于 pytest 套件**，是独立的手动/CI 冒烟脚本。

### 14.2 install.sh 详解（387 行）

参数：`--from-source`（git clone 后 `pip install -e`）、`--with-channels`（已废弃，仅提示）、`--help`（35-52）。

| 步骤 | 行号 | 流程 |
| --- | --- | --- |
| 0 着色/日志 | 11-27 | TTY 检测决定是否启用 ANSI 色；`info/success/warn/error/step` 五日志器 |
| 1 探测 OS | 69-85 | `$OSTYPE`：`linux-gnu*` + `/proc/version` 含 `microsoft` → WSL，否则 Linux；`darwin*`→macOS；`msys/cygwin`→Git Bash |
| 2 Python ≥ 3.10 | 90-148 | 依次试 `python3`/`python`，正则取 `X.Y.Z` 判主次版本；不满足则按 OS 给安装建议（brew/apt/dnf/python.org）并退出；pip 缺失回退 `python -m pip`，再不行建议 `ensurepip` |
| 3 Node ≥ 18（可选） | 153-179 | 不足或不装只 `warn`，随后的 npm 安装整体跳过 |
| 4 安装 OpenHarness | 184-238 | ① venv `~/.openharness-venv`（`bin/activate` 缺失视为坏 venv 重建；**避免 PEP 668 externally-managed**）；② `source` 激活后 `python/pip` 全部指向 venv；③ `--from-source`：clone/pull `~/.openharness-src`（`REPO_URL=HKUDS/OpenHarness.git`）→ `pip install -e`；否则 `pip install openharness-ai --upgrade` |
| 5 通道依赖 | 243-246 | `--with-channels` 仅打印"已默认包含" |
| 6 前端依赖 | 251-267 | 按来源模式定位 `frontend/terminal`，`npm install --no-fund --no-audit --silent` |
| 7 配置目录 | 272-278 | 建 `~/.openharness/{,skills,plugins}` |
| 8 注册全局命令 | 283-289 | 软链（`ln -snf`）`~/.local/bin/{oh,ohmo,openharness}` → venv 内脚本 |
| 9 验证 | 294-313 | `oh --version`/`ohmo --help` 实测；链路不可执行再试 `python -m openharness --version` 兜底 |
| 10 shell 集成 | 318-367 | 向 `~/.zshrc` / `~/.bashrc` / `~/.bash_profile` 追加 `export PATH="$HOME/.local/bin:$PATH"`（`grep -q` 去重）；fish 写 `if not contains -- ...` 守卫块；全部失败给出 `export` 建议；结束提示 API key 与启动命令 |

### 14.3 install.ps1 详解（329 行）

与 install.sh 对称的 PowerShell 版。要点差异：

| 项 | 行号 | 差异点 |
| --- | --- | --- |
| 参数 | 5-9 | switch：`-FromSource / -WithChannels / -Help` |
| PowerShell 版本 | 46-57 | 要求 ≥ 5.1 |
| Python 探测 | 62-96 | `python / python3 / py` 三连探测（`py` 启动器），`--version` 正则匹配 |
| Node | 101-122 | 同层可选 |
| venv | 134-155 | `~\.openharness-venv`；激活脚本 `Scripts\Activate.ps1`（**不存在的** `bin/activate`） |
| 前端定位 | 196-221 | **非 source 模式用 `pip show openharness-ai` 的 `Location:` 字段**定位 `{Location}\openharness\_frontend`（201-209） |
| 配置目录 | 226-236 | `~\.openharness\{skills,plugins}` |
| PATH | 241-254 | **`[Environment]::SetEnvironmentVariable("PATH","<venv>\\Scripts;..." ,"User")` 写用户级环境变量**，而非改 shell rc 文件 |
| 启动器选择 | 261-305 | **优先 `openh.exe`，其次 `openharness.exe`，最后 `oh.exe`**（`openh` 是 v0.1.6 后新增别名，老 PyPI 安装只有 `oh`）；并**特别警告 PowerShell 里 `oh` 与内置 Out-Host 别名冲突**（289-293），推荐全名调用 |
| 结束指引 | 310-329 | API key 环境变量 + 按所选启动器给出 `openh`/`openharness`/`oh.exe` 建议 |

### 14.4 test_cli_flags.py 详解（157 行）

用 **kimi 模型**（`https://api.moonshot.cn/anthropic`，`kimi-k2.5`）验证 CLI flag 组的冒烟脚本。环境变量由 `_env()`（18-23）`setdefault` 注入；`_run_oh(*args, timeout=60)`（26-36）跑 `sys.executable -m openharness <args>`。

| 用例 | 行号 | 关键断言 |
| --- | --- | --- |
| `test_help_output` | 39-67 | `--help` 输出必须同时含：品牌 `"Oh my Harness!"`、五个 flag 组（`"Session"/"Model & Effort"/"Output"/"Permissions"/"System & Context"/"Advanced"`）、`"--print"`、`"--model"`、`"--permission-mode"`，以及 `mcp / plugin / auth` 三个子命令 |
| `test_print_mode` | 70-78 | `-p "Say exactly: hello openharness"` 实模型调用：returncode==0 且 stdout 含 `"hello"` |
| `test_print_json` | 81-96 | `-p ... --output-format json`：可 `json.loads`、`type=="result"`、`text` 含 `"test123"` |
| `test_subcommand_mcp_list` | 99-105 | `oh mcp list` 返回码 0 |
| `test_subcommand_plugin_list` | 108-114 | `oh plugin list` 返回码 0 |
| `test_subcommand_auth_status` | 117-123 | `oh auth status` 返回码 0 且输出含 `"provider"` |

`main()`（126-153）顺序跑 6 项、逐项 try/except、汇总 `passed/failed` 并以非零退出码失败（`sys.exit(1 if failed else 0)`）。

---

## 十五、关键类 / 函数清单总表

### 15.1 autopilot（src/openharness/autopilot/）

| 名字 | 位置 | 职责 |
| --- | --- | --- |
| `RepoAutopilotStore` | service.py:226 | 子系统唯一核心类：持久化、摄入、评分、状态机执行、报告、仪表盘导出 |
| `RepoAutopilotStore.enqueue_card` | service.py:265 | 归一化入队/去重刷新（指纹驱动），写日志 + 重建上下文 |
| `RepoAutopilotStore.pick_next_card` | service.py:334 | 最高分 queued 卡 |
| `RepoAutopilotStore.scan_all_sources` | service.py:609 | 容错扫描 issue/PR/claude-code 三来源 |
| `RepoAutopilotStore.run_next` | service.py:627 | pick + run 一趟 |
| `RepoAutopilotStore.run_card` | service.py:644 | **执行状态机主体**（preparing→running/repairing→verifying→waiting_ci→merged/completed/failed） |
| `RepoAutopilotStore.tick` | service.py:1152 | 扫描 + 若空闲执行；cron 调度目标 |
| `RepoAutopilotStore.install_default_cron` | service.py:1174 | 注册 autopilot.scan / autopilot.tick |
| `RepoAutopilotStore.export_dashboard` | service.py:1195 | snapshot.json + fallback index.html + .nojekyll |
| `RepoAutopilotStore.rebuild_active_context` | service.py:405 | active_repo_context.md 聚合 + 联动导出看板 |
| `RepoAutopilotStore._build_dashboard_snapshot` | service.py:1702 | 完整快照（columns/counts/focus/journal/policies/context） |
| `RepoAutopilotStore._serialize_card` | service.py:1768 | 卡 → UI 字段子集 |
| `RepoAutopilotStore._run_agent_prompt` | service.py:2041 | build_runtime + submit_message 跑主引擎，回收摘要 |
| `RepoAutopilotStore._run_verification_steps` | service.py:2094 | 顺序执行验证门（1800s 超时、4000 字截尾） |
| `RepoAutopilotStore._wait_for_pr_ci` | service.py:1548 | CI 轮询（宽限/settle/超时三态） |
| `RepoAutopilotStore._ci_rollup` | service.py:1515 | statusCheckRollup 归一化 + pending/failed/success |
| `RepoAutopilotStore._automerge_eligible` | service.py:1574 | label_gated / fully_auto / pr_only 判断 |
| `RepoAutopilotStore._upsert_pull_request` | service.py:1386 | 幂等 PR 创建 + autopilot 标签 |
| `RepoAutopilotStore._process_existing_pr_card` | service.py:1628 | 既有 PR 只监控 CI 的短路路径 |
| `RepoAutopilotStore._prepare_repair_prompt` | service.py:1595 | 注入失败上下文/最小补丁指令 |
| `RepoAutopilotStore._score_card` | service.py:1935 | 可解释打分（来源基础分 + 各信号） |
| `_DEFAULT_AUTOPILOT_POLICY` | service.py:53 | 默认 autopilot 策略 YAML 种子 |
| `_DEFAULT_VERIFICATION_POLICY` | service.py:90 | 默认验证门命令 |
| `_DEFAULT_RELEASE_POLICY` | service.py:110 | 默认发布策略 |
| `_parse_verification_entry` | service.py:155 | 验证条目解析 + shell 元字符防御 |
| `_looks_available` | service.py:199 | 依赖存在性过滤 |
| `_bilingual_lines` | service.py:222 | 双语评论拼装 |
| `RepoTaskCard/JournalEntry/...` 7 模型 | types.py | 数据模型（见第二章） |

### 15.2 autopilot-dashboard

| 名字 | 位置 | 职责 |
| --- | --- | --- |
| `App` | App.tsx:140 | 根组件：拉快照、过滤、渲染 hero/stats/板/日志 |
| `CardView` | App.tsx:30 | 单卡（徽章/分数/备注/验证摘要） |
| `GroupColumnView` | App.tsx:79 | 归并看板列 |
| `JournalView` | App.tsx:105 | 事件流日志 |
| `PipelineAnimation` | components/PipelineAnimation.tsx:9 | QUEUE→PREP→RUN→CHECK→MERGE 五节点轨道动画（SMIL） |
| `HeroBackground` | components/HeroBackground.tsx:15 | 7 层赛博接线板背景 |
| `STATUS_LABELS/COLORS` | types.ts:39/55 | 状态→文案/色彩 |
| `KANBAN_GROUPS` | types.ts:79 | 13 状态→4 看板列 |

### 15.3 辅助模块

| 名字 | 位置 | 职责 |
| --- | --- | --- |
| `atomic_write_bytes/text` | utils/fs.py:39/69 | 临时文件 + fsync + `os.replace` 原子写 |
| `resolve_shell_command` | utils/shell.py:17 | 平台 shell argv 挑选 |
| `create_shell_subprocess` | utils/shell.py:51 | 异步 shell 子进程（Docker 沙箱 / SRT 包装） |
| `exclusive_file_lock` | utils/file_lock.py:27 | 跨平台排他锁 |
| `split_message` / `safe_filename` | utils/helpers.py:28/63 | 切分 / 安全文件名 |
| `ThemeConfig` + 5 内置主题 | themes/schema.py:47 / builtin.py:13 | 主题模型与 builtin 集 |
| `load_theme` | themes/loader.py:44 | 自定义优先、内置兜底 |
| `load_output_styles` | output_styles/loader.py:27 | 内置 3 样式 + 用户 `*.md` |
| `load_keybindings` | keybindings/loader.py:17 | 默认键位 + 覆盖合并 |
| `toggle_vim_mode` | vim/transitions.py:6 | Vim 状态取反 |
| `inspect_voice_capabilities` | voice/voice_mode.py:25 | 录音器 + provider 能力探测 |
| `transcribe_stream` | voice/stream_stt.py:6 | 流式 STT 占位 |
| `AppState` / `AppStateStore` | state/app_state.py:8 / store.py:14 | 扁平快照 + 迷你订阅存储 |
| `BridgeSessionManager` / `get_bridge_manager` | bridge/manager.py:26/100 | 桥接子会话跟踪 + 输出落盘单例 |
| `encode/decode_work_secret`、`build_sdk_url` | bridge/work_secret.py:11/17/35 | work secret 编解码 + session 接入 URL |
| `detect_platform` / `get_platform_capabilities` | platforms.py:27/55 | 平台归一化 + 能力矩阵 |
| `app()` | \_\_main\_\_.py:6 | `python -m openharness` 入口 |

---

## 附：设计要点小结

1. **"调度器 + 引擎"分层**：Autopilot 不在引擎内部，而是每个任务用 `ui.runtime.build_runtime` 组装**全新会话**复用完整栈（工具/MCP/权限/hooks）；引擎"想 + 做"，autopilot"选、隔离、验证、上报、合并"。
2. **一切皆可审计**：写操作全走 `atomic_write_text`；事件进 append-only JSONL journal；每次尝试双份报告（历史 + 当前）；GitHub 每一步留双语评论；打分 `score_reasons` 逐条可解释。
3. **失败可自修复**：`metadata.last_failure_stage/summary` + 修复 prompt + attempt 循环构成"最小补丁逼近"回路，被 `max_attempts=max(execution.max, repair.rounds+1)` 封顶；`repair.stop_on` 声明不可重试类型（agent_runtime_error/git_error/permission_error/merge_conflict）。
4. **信任边界**：模型摘要被显式标注 "Service-Level Ground Truth … untrusted"；merge/gate 唯一依据 = service 级本地验证 + 远端 CI；`default_human_gate=True`、`auto_merge.mode=label_gated`，避免完全无人值守。
5. **静态仪表盘**：前端零依赖 React，SSG 式消费 `snapshot.json`；`vite.outDir=../docs/autopilot` + `emptyOutDir:false` 让构建产物与运行时数据**共处同一目录**；无 Node 时服务端写 minified fallback 页。
6. **大量"薄壳"扩展模块**（themes/keybindings/vim/voice/state/output_styles）："数据模型 + loader + 占位"三段式，为后续丰富功能预留稳定接口；voice 的 realtime STT、autopilot 的 accepted/rejected/superseded 状态均为"接口先行、实现后置"。
7. **平台自适应**：`platforms.py` 能力矩阵 + `shell.py` 的 WSL 实测防御 + `file_lock.py` 的 fcntl/msvcrt 双实现，让同一套代码覆盖 macOS/Linux/WSL/Windows。

---

## 十六、实际示例：从卡到合并的全链路数据形态

本章把前文抽象描述落到**真实代码形态**：一副卡片 JSON、一份策略 YAML、一次 `run_card` 完整走查、两条双语评论、一个 `snapshot.json`、两组验证门输出、以及 `install.sh` 的一条流。行号均以 `service.py`（2239 行）、`types.py`（91 行）、`types.ts`（84 行）、`autopilot-dashboard/public/snapshot.json`（59 行）、`install.sh`（387 行）实测核对。

### 16.1 一张 autopilot card 的完整 JSON

> 示例：完整 `RepoTaskCard`（字段及其顺序来源 `types.py:33-48`；metadata 键表见 5.3 与 `service.py:686-1007`；JSON 形态对齐 `_save_registry` 的 `model_dump(mode="json")`，`service.py:1910-1921`；内容为示意）

```json
{
  "id": "ap-3f9a2c11",
  "fingerprint": "github_issue:9e2c41b7f0a3d5c6",
  "title": "GitHub issue #42: 修复 README 链接失效",
  "body": "Several README links are broken (404). Please fix them so badges and anchors resolve.",
  "source_kind": "github_issue",
  "source_ref": "issue:42",
  "status": "completed",
  "score": 125,
  "score_reasons": [
    "source=github_issue",
    "bug-labelled issue",
    "issue looks like a bug/regression",
    "recently updated"
  ],
  "labels": ["autopilot", "bug"],
  "metadata": {
    "url": "https://github.com/openharness/OpenHarness/issues/42",
    "updated_at_remote": "2026-08-15T09:12:33Z",
    "run_started_at": 1776338766.0,
    "execution_model": "anthropic/claude-sonnet-4-5",
    "max_attempts": 3,
    "worktree_slug": "autopilot/ap-3f9a2c11",
    "worktree_path": "~/.openharness/worktrees/autopilot+ap-3f9a2c11",
    "head_branch": "autopilot/ap-3f9a2c11",
    "base_branch": "main",
    "linked_issue_numbers": [42],
    "linked_pr_number": 108,
    "linked_pr_url": "https://github.com/openharness/OpenHarness/pull/108",
    "autopilot_managed": true,
    "attempt_count": 1,
    "assistant_summary_preview": "已修复 README 中 3 处失效链接（badge/锚点），替换为仓库内相对路径……",
    "verification_failed": false,
    "verification_steps": [
      {
        "command": "uv run pytest -q",
        "returncode": 0,
        "status": "success",
        "stdout": "248 passed in 12.7s",
        "stderr": ""
      }
    ],
    "last_ci_conclusion": "success",
    "last_ci_summary": "All reported remote checks passed.",
    "last_failure_stage": "",
    "last_failure_summary": "",
    "human_gate_pending": true,
    "last_note": "PR #108 is green; human gate pending"
  },
  "created_at": 1776338700.0,
  "updated_at": 1776342300.0
}
```

说明：`fingerprint = "github_issue:{sha1('issue:42')[:16]}"`（`_build_fingerprint`，1923-1933）；`labels` 经 `_normalize_labels` 排序去重（1972-1975）；`metadata` 从 `scan_github_issues` 的 `url/updated_at_remote`（528-531）起步，随后被 `run_card` 各阶段**逐键叠加**——`score = 75 基础分 + 25 bug 标签 + 15 关键词 + 10 新鲜度`，每条加分都能在 `score_reasons` 里溯因（1935-1970）。

### 16.2 policy YAML 实例

> 示例：`_DEFAULT_AUTOPILOT_POLICY` 的落盘 YAML 形态（内容来源 `service.py:53-89`；由 `_ensure_layout`→`yaml.safe_dump(payload, sort_keys=False)` 写入 `.openharness/autopilot/autopilot_policy.yaml`，`service.py:1888-1895`；内容为示意）

```yaml
intake:
  mode: unified_queue
  max_visible_candidates: 12
  dedupe_strategy: source_ref_then_fingerprint
decision:
  default_human_gate: true
  prefer_small_safe_steps: true
execution:
  default_model: ""
  max_turns: 12
  permission_mode: full_auto
  host_mode: self_hosted
  use_worktree: true
  base_branch: main
  max_attempts: 3
github:
  issue_comment_style: bilingual
  pr_branch_prefix: autopilot/
  ci_poll_interval_seconds: 20
  ci_timeout_seconds: 1800
  no_checks_grace_seconds: 60
  checks_settle_seconds: 20
  auto_merge:
    mode: label_gated
    required_label: autopilot:merge
repair:
  max_rounds: 2
  retry_on:
    - local_verification_failed
    - remote_ci_failed
  stop_on:
    - agent_runtime_error
    - git_error
    - permission_error
    - merge_conflict
```

> 示例：用户自定义扩展示例——`load_policies` 把该文件**原样读回**（`_read_yaml`，2002-2011；不做 schema 校验/深合并），各消费点用 `.get(k, 默认值)` 兜底（如 `_max_attempts`:1208-1213、`_base_branch`:1215-1217、`_head_branch`:1219-1222），**未知键不过滤**、会随 `_build_execution_prompt` 原样 dump 进 Agent prompt（`service.py:2014`）；内容为示意

```yaml
# .openharness/autopilot/autopilot_policy.yaml（用户覆盖）
intake:
  max_visible_candidates: 8          # 覆盖默认 12
execution:
  max_turns: 24                       # 更长回合
  permission_mode: ask                # 更保守的权限
github:
  auto_merge:
    mode: fully_auto                  # 提速：CI 绿即合并
repair:
  max_rounds: 3                       # 多一轮修复
  retry_on:
    - local_verification_failed       # 追加条目（复制整节则整体覆盖）
    - remote_ci_failed
    - no_changes
experimental:
  notify_slack_webhook: ""            # 自定义键：代码不消费，但会进 prompt 给模型参考
```

注意：覆盖是以**整个键块为单位**——写了 `retry_on` 就会覆盖默认清单，因此扩展示例里把原有两条也一起写出。

### 16.3 run_card 一次完整运行走查

> 示例：假想 issue「修复 README 链接失效」（GitHub `issue:42`，卡 `ap-3f9a2c11`）的端到端走查；状态机主循环来源 `service.py:644-1150`（全景见第五章），下面每个阶段的触发代码行均已实测；内容为示意

| # | 阶段 | 触发（service.py 行号） | 副作用 |
| --- | --- | --- | --- |
| 1 | **pick** | `oh autopilot tick`→`scan_all_sources` 后由 `pick_next_card` 选中分数最高 queued 卡（`tick`:1152-1172；`run_next`:627-642；`pick_next_card`:334-338） | 队列里 `ap-3f9a2c11`（score 125）被取出；journal 记 `tick_idle` 或 `run_finished` 等 |
| 2 | **preparing** | 活跃态检查（655-656）→ `use_worktree` 计算（670）→ `WorktreeManager.create_worktree(slug="autopilot/ap-3f9a2c11", branch=head)`（675-684；worktree.py:150，路径 `~/.openharness/worktrees/autopilot+ap-3f9a2c11`，`/`→`+`，worktree.py:78）→ `update_status(preparing)`（686-701）→ 首轮且关联 issue 时发 `_comment_started`（703-704） | 分支 `autopilot/ap-3f9a2c11` 基于 `origin/main` checkout 出来；metadata 写入 `run_started_at/worktree_path/head_branch/…`；issue #42 收到第一条双语评论（1455-1459） |
| 3 | **running** | `_sync_worktree_to_base`（716-723；1294-1299）→ `update_status(running)`（742-747）→ 构造 prompt：attempt=1 直接用 `_build_execution_prompt`（748-755；2013-2039，三份策略 YAML dump 进 prompt）→ `_run_agent_prompt`（756-763；2041-2080，`build_runtime` 全新会话 + `permission_mode=full_auto`） | Agent 在 worktree 里改掉了 `README.md` 的坏链接；`{id}-attempt-01-run.md` 与 `{id}-run.md` 双写（803-810/830-837）；journal 记 `run_finished`（811-816） |
| 4 | **verifying** | `update_status(verifying)`（818-823，存 `assistant_summary_preview`）→ `_run_verification_steps(policies, cwd)`（824；2094-2155）→ 验证报告双写（825-827；2157-2184）→ `failing` 为空 → `final_local_report` 写 `passed`（829-837） | `uv run pytest -q`（248 passed）、`uv run ruff check …`、`tsc --noEmit` 依次执行；`metadata.verification_steps` 记录每步 JSON；若无 `frontend/terminal/package.json` 则第三条被 `_looks_available` 跳过（199-209） |
| 5 | **预演提交与 PR**（在 waiting_ci 之前） | `_git_commit_all(cwd, "autopilot(ap-3f9a2c11): GitHub issue #42: 修复 README 链接失效")`（913-916）→ `_git_branch_has_progress`（917-920）→ `_git_push_branch`→`_upsert_pull_request`（963-971；1386-1433，PR body 含 `Closes #42`）| 本地 commit + `git push -u origin autopilot/ap-3f9a2c11`；幂等创建 PR #108（含 `## Autopilot Summary`/`## Reports`/`## Notes`，1357-1384），补 `autopilot` 标签（1432/1397） |
| 6 | **waiting_ci** | `update_status(waiting_ci)（995-1007，写 linked_pr_*、autopilot_managed=True）→ `_comment_pr_opened`（1008）→ `_wait_for_pr_ci`（1010；1548-1572，每 20s 轮询 `gh pr view --json statusCheckRollup`）→ 结果回写 `last_ci_conclusion/summary/checks`（1011-1022） | PR #108 收到"已创建或更新 PR"评论（1461-1465）；远端 CI 全绿（`All reported remote checks passed.`）；无 check 时宽限 60s 判绿（1563-1564） |
| 7 | **completed** | `_automerge_eligible` 为假（默认 `label_gated` 且 PR 无 `autopilot:merge` 标签，1070/1574-1586）→ `update_status(completed, human_gate_pending=True)`（1102-1111）→ journal `human_gate_pending`（1112-1117）→ PR+issue 双发 `_comment_human_gate`（1118-1120；1485-1489）→ `remove_worktree`（1121-1122） | 卡停在 `completed` 待人工；worktree 被清理；PR #108 等待人工 gate。若人之后补 `autopilot:merge` 标签，下一次 `tick`/`run_card` 会走 merged 路径 squash 合并（1070-1100，`_comment_merged`:1479-1483）；若 CI 失败且 attempt<max，则进 `repairing` 新一轮（1024-1043） |

> 每步 `update_status` 末尾都会 `rebuild_active_context()` → `export_dashboard()`（483；1195-1206），因此每一步态都实时反映到 `docs/autopilot/snapshot.json` 与 React 看板（`counts`/`columns`/`focus` 同步刷新）。

### 16.4 双语 GitHub 评论示例

> 示例：`_bilingual_lines(zh, en)`（`service.py:222-223`）生成的评论文本；发送走 `_comment_on_issue`/`_comment_on_pr`（1435-1453，底层 `gh issue comment` / `gh pr comment` `--body`），**GitHub 限流/鉴权失败只写 `github_warning` journal，绝不中断主流程**（1439-1442/1449-1452）；内容为示意

评论 1 —— 开跑时发到 issue #42（`_comment_started`：1455-1459）：

```text
OpenHarness autopilot 已开始处理 `ap-3f9a2c11`，当前第 1 轮执行。
OpenHarness autopilot started processing `ap-3f9a2c11`. Attempt 1 is now running.
```

评论 2 —— PR #108 绿后停在人工 gate 时，PR 与 issue 双发（`_comment_human_gate`：1485-1489）：

```text
PR #108 的本地验证和远端 CI 都已通过，但仍需人工 gate 或 merge label。
PR #108 passed local verification and remote CI, but still requires a human gate or merge label.
```

评论族共 7 个生成器（1455-1495）：`_comment_started` / `_comment_pr_opened` / `_comment_ci_failed` / `_comment_local_failed` / `_comment_merged` / `_comment_human_gate` / `_comment_terminal_failure`，全部是 `f"{zh}\n{en}"` 两行拼接。

### 16.5 dashboard snapshot.json 结构

> 示例：`_build_dashboard_snapshot`（`service.py:1702-1766`）产生的完整快照。实测顶层键共 **11 个**，与真实 `autopilot-dashboard/public/snapshot.json`（59 行）完全同构；卡为 `_serialize_card` 的**精简形态**（1768-1808，metadata 只留 UI 子集 + verification_steps 仅 command/status/returncode）；`focus` 优先级 `repairing>waiting_ci>running>verifying>preparing>accepted>queued`（1734-1739）；内容为示意

```json
{
  "generated_at": 1776342300.005,
  "repo_name": "OpenHarness",
  "repo_path": "/home/dev/OpenHarness",
  "focus": {
    "id": "ap-3f9a2c11",
    "title": "GitHub issue #42: 修复 README 链接失效",
    "body": "Several README links are broken (404). Please fix them.",
    "status": "waiting_ci",
    "source_kind": "github_issue",
    "source_ref": "issue:42",
    "score": 125,
    "score_reasons": ["source=github_issue", "bug-labelled issue"],
    "labels": ["autopilot", "bug"],
    "created_at": 1776338700.0,
    "updated_at": 1776339060.0,
    "metadata": {
      "last_note": "remote CI status: pending",
      "url": "https://github.com/openharness/OpenHarness/issues/42",
      "execution_model": "anthropic/claude-sonnet-4-5",
      "assistant_summary_preview": "已修复 README 中 3 处失效链接……",
      "human_gate_pending": null,
      "verification_failed": false,
      "attempt_count": 1,
      "max_attempts": 3,
      "linked_pr_number": 108,
      "linked_pr_url": "https://github.com/openharness/OpenHarness/pull/108",
      "last_ci_conclusion": "pending",
      "last_ci_summary": "Remote CI is still running.",
      "last_failure_stage": "",
      "last_failure_summary": "",
      "verification_steps": [
        { "command": "uv run pytest -q", "status": "success", "returncode": 0 }
      ]
    }
  },
  "counts": {
    "queued": 2, "accepted": 0, "preparing": 0, "running": 0, "verifying": 0,
    "pr_open": 0, "waiting_ci": 1, "repairing": 0, "completed": 1, "merged": 1,
    "failed": 1, "rejected": 0, "superseded": 0
  },
  "status_order": [
    "queued", "accepted", "preparing", "running", "verifying", "pr_open",
    "waiting_ci", "repairing", "completed", "merged", "failed", "rejected",
    "superseded"
  ],
  "columns": {
    "waiting_ci": [ { "id": "ap-3f9a2c11", "title": "GitHub issue #42: 修复 README 链接失效", "status": "waiting_ci", "score": 125, "updated_at": 1776339060.0 } ],
    "queued": [],
    "accepted": [],
    "preparing": [],
    "running": [],
    "verifying": [],
    "pr_open": [],
    "repairing": [],
    "completed": [],
    "merged": [],
    "failed": [],
    "rejected": [],
    "superseded": []
  },
  "cards": [
    { "id": "ap-3f9a2c11", "title": "GitHub issue #42: 修复 README 链接失效", "status": "waiting_ci", "score": 125, "updated_at": 1776339060.0 },
    { "id": "ap-8d71be02", "title": "GitHub issue #11: 升级依赖版本", "status": "queued", "score": 90, "updated_at": 1776335000.0 },
    { "id": "ap-5c0a9f33", "title": "GitHub PR #99: 修复历史回放竞态", "status": "merged", "score": 85, "updated_at": 1776338000.0 }
  ],
  "journal": [
    {
      "timestamp": 1776338700.4,
      "kind": "intake_added",
      "summary": "Queued github_issue: GitHub issue #42: 修复 README 链接失效",
      "task_id": "ap-3f9a2c11",
      "metadata": { "source_ref": "issue:42", "score": 125 }
    },
    {
      "timestamp": 1776339061.1,
      "kind": "status_waiting_ci",
      "summary": "waiting_ci: GitHub issue #42: 修复 README 链接失效 (waiting for remote CI on PR #108)",
      "task_id": "ap-3f9a2c11",
      "metadata": {}
    }
  ],
  "policies": {
    "autopilot": "/home/dev/OpenHarness/.openharness/autopilot/autopilot_policy.yaml",
    "verification": "/home/dev/OpenHarness/.openharness/autopilot/verification_policy.yaml",
    "release": "/home/dev/OpenHarness/.openharness/autopilot/release_policy.yaml"
  },
  "active_context": "# Active Repo Context\n\nGenerated at: 2026-08-15 12:31:00 UTC\n\n## Current Task Focus\n- [waiting_ci] GitHub issue #42: 修复 README 链接失效 (github_issue, score=125)\n...\n## Recent Repo Journal\n- 08-15 12:31 status_waiting_ci: waiting_ci ... "
}
```

说明：`types.ts` 的 `Snapshot`（28-37）只对**前端用到的 8 个键**建模，`repo_path/policies/active_context` 后端有、前端不消费所以不建模（服务端超集、前端子集）；`cards` 与 `columns` 同源但一为全量、一为看板桶。

### 16.6 验证门输出示例

> 示例：`_run_verification_steps`（`service.py:2094-2155`）对 `_DEFAULT_VERIFICATION_POLICY.commands`（`service.py:90-109`）逐条执行后产出的 `RepoVerificationStep`（`types.py:69-76`）JSON；`stdout`/`stderr` 仅保留末尾 4000 字符（2123-2124），`timeout=1800`（2116），异常映射为 `status="error"`；内容为示意

`uv run pytest -q` 通过：

```json
{
  "command": "uv run pytest -q",
  "returncode": 0,
  "status": "success",
  "stdout": "248 passed in 12.7s",
  "stderr": ""
}
```

`uv run ruff check src tests scripts` 失败（returncode=1 → `status=failed`）：

```json
{
  "command": "uv run ruff check src tests scripts",
  "returncode": 1,
  "status": "failed",
  "stdout": "",
  "stderr": "README.md:1:1: D100 Missing docstring in public module (TRY-300, E501)"
}
```

两种不会被执行 / 不进成功列的边界情形：

```json
{ "command": "uv run pytest; rm -rf /", "returncode": -1, "status": "error",
  "stderr": "verification policy error: command contains shell metacharacters; use the mapping form {command: '...', shell: true} in verification_policy.yaml to opt in" }
{ "command": "cd frontend/terminal && ([ -x ./node_modules/.bin/tsc ] || npm ci --no-audit --no-fund) && ./node_modules/.bin/tsc --noEmit",
  "returncode": -1, "status": "error", "stderr": "Timed out after 1800s" }
```

第一种是 `_parse_verification_entry` 的 shell 注入防御（175-184），第二种是 `subprocess.TimeoutExpired` 兜底（2136-2145）；若仓库缺 `pyproject.toml`/`tests/`，对应条目不入库（`_looks_available` 跳过，199-209），`failing` 判定只认 `{failed, error}`（829 行）。

### 16.7 install.sh 端到端示例

> 示例：用户执行 `curl -fsSL https://raw.githubusercontent.com/HKUDS/OpenHarness/main/scripts/install.sh | bash`（`install.sh:3`）后发生的 10 步流程；每步对应的真实命令已核对 `install.sh` 行号；内容为示意

1. **Step 1 检测系统**（`install.sh:69-85`）：按 `$OSTYPE` 归一到 `Linux / WSL（grep /proc/version 含 microsoft）/ macOS / Windows (Git Bash)`，打印 `[INFO] OS detected: macOS`。
2. **Step 2 检查 Python ≥ 3.10**（88-128）：依次试 `python3`、`python`，解析 `--version` 取主次版本号，不满足直接 `exit 1` 并给出 `brew install python@3.12` 等提示。
3. **Step 3 检查 Node ≥ 18**（153-179）：`command -v node` + 版本比对，决定 `NODE_OK`；无 Node 只 `warn`（React TUI 跳过，不阻断）。
4. **Step 4 安装 OpenHarness**（184-238）：先建 venv（`$PYTHON_CMD -m venv "$HOME/.openharness-venv"`，194-208 并处理残缺 venv）→ `source ~/.openharness-venv/bin/activate`（205）→ 默认 `pip install openharness-ai --quiet --upgrade`（235）；带 `--from-source` 则克隆 `https://github.com/HKUDS/OpenHarness.git` 到 `~/.openharness-src` 后 `pip install -e`（210-232）。
5. **Step 5 通道依赖**（243-246）：`--with-channels` 已是兼容占位，打印提示不做事。
6. **Step 6 前端 npm 依赖**（251-267）：`NODE_OK` 且存在 `frontend/terminal/package.json` 时跑 `(cd "$FRONTEND_DIR" && npm install --no-fund --no-audit --silent)`（262）；否则跳过。
7. **Step 7 配置目录骨架**（272-278）：`mkdir -p ~/.openharness ~/.openharness/skills ~/.openharness/plugins`。
8. **Step 8 注册全局命令**（283-289）：`mkdir -p ~/.local/bin` + `ln -snf` 软链 `~/.openharness-venv/bin/{oh,ohmo,openharness}` → `~/.local/bin/`。
9. **Step 9 验证安装**（294-313）：`~/.local/bin/oh --version` 与 `ohmo --help`；软链不可执行时退化为 `python -m openharness --version` 并提示 PATH。
10. **Step 10 Shell 集成与收尾**（318-387）：向 `~/.zshrc`/`~/.bashrc`/`~/.bash_profile`/fish `config.fish` 追加 `export PATH="$HOME/.local/bin:$PATH"`（去重判断 337-347），最后打印下一步指引：重启 shell、`export ANTHROPIC_API_KEY=...`、`oh`/`ohmo` 启动、文档链接。