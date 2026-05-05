# ADR-0005: LLM客户端架构与容错

## Status
Accepted

## Date
2026-05-02

> **TD-ADR Review**: APPROVED — AsyncTaskQueue + full HTTP response + Timer typewriter architecture confirmed sound for Web WASM (2026-05-02)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | Core (Networking / HTTP) |
| **Knowledge Risk** | MEDIUM — `HTTPRequest` on Web export has CORS/cross-origin constraints; `Semaphore`/`Mutex`/`Thread` are UNAVAILABLE on Web WASM |
| **References Consulted** | `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/deprecated-apis.md`, `docs/engine-reference/godot/current-best-practices.md`, `docs/engine-reference/godot/modules/networking.md` |
| **Post-Cutoff APIs Used** | None directly — `HTTPRequest` and `JSON` are pre-4.0 stable APIs. `Timer`/`Tween` for typewriter effect are 4.0+ stable. |
| **Verification Required** | (1) `HTTPRequest` CORS preflight on Web export — may need proxy if LLM API doesn't return `Access-Control-Allow-Origin`; (2) `HTTPRequest.request()` timeout behavior on Web WASM — confirm 30s timeout fires correctly; (3) `Timer` 40ms interval accuracy on Web export when tab is backgrounded; (4) `JSON.parse_string()` with full LLM response (~4KB) — confirm < 1ms parse time |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0003 (AgentContext — MultiAgentOrchestrator provides context for LLM decisions), ADR-0004 (LLMDialogue autoload initialization at position 6 in autoload order) |
| **Enables** | ADR-0006 (DialogueUI receives text_token signals from this ADR's pipeline) |
| **Blocks** | Dialogue rendering implementation, agent decision integration |
| **Ordering Note** | First Core-layer ADR after Foundation. Write after ADR-0004 (autoload order). LLM autoload must be initialized before DialogueUI (ADR-0006). |

## Context

### Problem Statement
LLM API调用是「浮世」的技术瓶颈——每个游戏天需要6-9次Agent决策+对话渲染调用。API不可靠（超时、无效JSON、速率限制），需要统一的容错策略。同时Godot 4.6 Web导出强加了两个硬约束：

1. **无Semaphore/Mutex/Thread** — Web WASM是单线程的。无法使用Python参考项目中`asyncio.gather + Semaphore`的并发控制模式。
2. **HTTPRequest不支持SSE流式** — LLM的逐token流式输出无法通过Godot标准HTTP节点实现。

### Constraints
- Godot 4.6 / GDScript / Web WASM单线程
- HTTPRequest不支持分块传输编码(SSE/streaming)
- 无Semaphore/Mutex/Thread
- LLM API可能是OpenAI兼容端点、Anthropic，或两者混合——客户端必须支持可切换后端
- 每局LLM成本控制在~$0.25（MVP: 3角色×5天）
- 对话渲染延迟感知应 <2秒（从行动到文本开始出现的延迟）

### Requirements
- 并发控制：最多3个并行LLM调用（Web平台兼容方案）
- JSON响应解析：自动重试2次无效JSON
- 3层降级：重试→使用上次成功响应→纯数值fallback模式
- 两种模型层级：FAST（agent decision, ~$0.002/call）和NORMAL（dialogue render, ~$0.01/call）
- 打字机效果：40ms/字符逐字显示
- 对话历史：缓冲区50条，Prompt注入最近10条+高impact标记条
- 30秒超时（单次LLM请求）

## Decision

**使用协程AsyncTaskQueue（await+计数器，非Semaphore）+ HTTPRequest完整响应 + Godot端Timer逐字打字机。MVP完全不依赖SSE流式。**

### 为什么选择这个方案

LP-FEASIBILITY在架构阶段发现了两个阻塞项。此ADR直接采纳推荐的缓解措施：

1. **Semaphore → AsyncTaskQueue**: 协程任务队列通过`await`+active_count计数器控制并发——纯GDScript，Web WASM完全兼容。
2. **SSE streaming → full response + Timer typewriter**: `HTTPRequest`返回完整JSON响应体。Godot端`Timer`以40ms间隔逐字推送到RichTextLabel。玩家感知体验完全相同——初始延迟略长（~0.5-2s等待LLM完整响应），但打字机动画本身不变。

### API Provider Selection

**Current (2026-05-03): MiniMaxi** — `MiniMax-M2.7-highspeed` via `https://api.minimaxi.com/anthropic` (Anthropic-compatible endpoint).

Rationale:
- Anthropic-compatible Messages API — `x-api-key` auth, `/v1/messages` endpoint, content blocks
- High-quality CJK dialogue generation
- The `api_format` config field now drives runtime branching at four touchpoints:
  1. Endpoint construction (`/v1/messages` vs `/chat/completions`)
  2. Auth headers (`x-api-key` + `anthropic-version` vs `Authorization: Bearer`)
  3. Payload shape (`system` as top-level field, content as `[{type, text}]` vs messages array)
  4. Response extraction (`content[0].text` vs `choices[0].message.content`)

Backend switching:
Switching between `"openai"` and `"anthropic"` api_format requires only `data/llm_settings.json` changes — no code changes.

**Previous (superseded): DeepSeek API** (`deepseek-chat` via OpenAI-compatible `/v1/chat/completions`).

### Architecture Diagram

```
LLMDialogue (Core Autoload)
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  ┌──────────────────────┐    ┌──────────────────────┐   │
│  │ Prompt Assembler     │    │ AsyncTaskQueue        │   │
│  │ 7-layer injection:   │    │ max_concurrent: 3     │   │
│  │ Profile→PERCEIVED→   │    │ _active_count: int    │   │
│  │ Events→Body→State→   │    │ _queue: Array[Callable]│   │
│  │ Actions→World        │    │ enqueue()/_try_proc()  │   │
│  └──────────┬───────────┘    └───────────┬───────────┘   │
│             │                            │               │
│             ▼                            ▼               │
│  ┌──────────────────────┐    ┌──────────────────────┐   │
│  │ HTTP Client           │    │ Response Parser       │   │
│  │ HTTPRequest node      │    │ JSON.parse_string()   │   │
│  │ POST {endpoint}       │    │ 2 retries on failure  │   │
│  │ timeout: 30s          │    │ fallback on 3rd fail  │   │
│  └──────────┬───────────┘    └───────────┬───────────┘   │
│             │                            │               │
│             ▼                            ▼               │
│  ┌──────────────────────────────────────────────────┐    │
│  │ Typewriter Pipeline                              │    │
│  │ Timer(40ms) → char by char → text_token signal   │    │
│  │ Player click = skip → emit full text immediately  │    │
│  └──────────────────────────────────────────────────┘    │
│                                                         │
│  ┌──────────────────────┐    ┌──────────────────────┐   │
│  │ Dialogue History      │    │ Fallback Table        │   │
│  │ buffer: 50 entries    │    │ Numeric-only mode     │   │
│  │ inject: last 10       │    │ (no LLM available)    │   │
│  └──────────────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Key Interfaces

```gdscript
# === LLMDialogue (Core Autoload) ===

# 三种文本模式
func request_agent_decision(agent_context: Dictionary) -> Dictionary:
    # 模式: agent_decision | 模型: FAST | 不显示给玩家
    # 返回: { thinking, emotion, short_term_goal, action_chain: [{action, target, intent, ...}] }
    # 如果失败: enqueue retry → degraded → return fallback action from table

func request_dialogue_render(action_context: Dictionary) -> void:
    # 模式: dialogue_render | 模型: NORMAL | 显示给玩家
    # 触发 text_token 信号管道
    # 管道: 完整HTTP响应 → JSON解析 → Timer逐字推送(40ms/字)
    # 玩家跳过: 立即发射完整文本

func request_body_reflection(context: Dictionary) -> Dictionary:
    # 模式: body_reflection | 模型: NORMAL | 仅受害女性
    # 返回: { self_perception, denial_adjustment, shame_adjustment, ... }

# 流式文本管道
signal text_token(char: String)      # 单字符推送 → DialogueUI接收
signal text_token_complete()         # 当前段落完成 → UI显示"继续"指示符
signal text_error(message: String)   # API错误 → UI显示fallback文本

# === AsyncTaskQueue (内部类) ===

class AsyncTaskQueue:
    var _max_concurrent: int = 3
    var _active_count: int = 0
    var _queue: Array[Callable] = []

    func enqueue(task: Callable) -> void:
        _queue.append(task)
        _try_process()

    func _try_process() -> void:
        if _active_count >= _max_concurrent or _queue.is_empty():
            return
        _active_count += 1
        var task: Callable = _queue.pop_front()
        _run_task(task)

    func _run_task(task: Callable) -> void:
        var result: Dictionary = await task.call()
        _active_count -= 1
        if result.get("error", false):
            push_warning("Async task failed: ", result.get("message", ""))
        _try_process()  # 处理队列中下一个

# === HTTP Client ===

func _post_llm(endpoint: String, payload: Dictionary) -> Dictionary:
    var http := HTTPRequest.new()
    add_child(http)
    http.timeout = 30.0
    var error := http.request(endpoint, _headers, HTTPClient.METHOD_POST,
        JSON.stringify(payload))
    if error != OK: return {"error": true, "message": "HTTP request failed: %d" % error}
    var response := await http.request_completed
    http.queue_free()
    var result: int = response[0]
    var body: PackedByteArray = response[3]
    if result != HTTPRequest.RESULT_SUCCESS:
        return {"error": true, "message": "HTTP %d" % result}
    return {"success": true, "body": body.get_string_from_utf8()}

# === JSON Parser with Retry ===

func _parse_with_retry(raw_json: String, max_retries: int = 2) -> Dictionary:
    var last_error: String = ""
    var request_count: int = 0
    var consecutive_failures: int = 0
    for attempt in range(1 + max_retries):
        request_count += 1
        if attempt > 0:
            raw_json = await _retry_llm_call(payload, error_context)
        var parsed: Variant = JSON.parse_string(raw_json)
        if parsed != null and parsed is Dictionary:
            # 一致性验证
            if _validate_response(parsed): return {"success": true, "data": parsed}
            else: last_error = "validation failed"; consecutive_failures += 1
        else:
            last_error = "JSON parse null or wrong type"; consecutive_failures += 1
        if consecutive_failures >= 3:
            return _activate_numeric_fallback()
    return {"error": true, "message": last_error}

# === Typewriter Pipeline ===

func _start_typewriter(full_text: String) -> void:
    var index: int = 0
    var timer := Timer.new()
    add_child(timer)
    timer.wait_time = 0.04  # 40ms
    timer.start()
    timer.timeout.connect(func():
        if index < full_text.length():
            text_token.emit(full_text[index])
            index += 1
        else:
            timer.queue_free()
            text_token_complete.emit()
    )
    # 玩家跳过: 点击/空格 → timer.queue_free() → emit完整文本 + text_token_complete

# === Model Tier Config ===

const MODEL_CONFIG := {
    "FAST": {
        "model": "gpt-4o-mini",
        "endpoint": "https://api.openai.com/v1/chat/completions",
        "max_tokens": 500,
        "temperature": 0.8,
        "budget_tokens": {"input": 2000, "output": 500}
    },
    "NORMAL": {
        "model": "claude-sonnet-4-6",
        "endpoint": "https://api.anthropic.com/v1/messages",
        "max_tokens": 1000,
        "temperature": 0.9,
        "budget_tokens": {"input": 4000, "output": 1000}
    }
}
```

### Fallback策略

```
Layer 1: JSON Retry
├── JSON.parse_string() 失败 → 重试 (max 2次, 指数退避)
├── 必需字段缺失(thinking/emotion/action_chain) → 重试
└── emotion不在枚举中 → 默认映射到"calm" (不重试)

Layer 2: Degrade
├── 重试耗尽 → 使用上次成功的response模板
├── timeout(>30s) → 跳过本次decision，下个time block再试
└── API rate limit → 退避30s + 通知player

Layer 3: Numeric-Only Mode
├── 连续3次API失败 → 进入纯数值模式
│   └── Agent: 使用FallbackActionTable (per role × per block)
│   └── Dialogue: 使用预设文本模板
└── 通知Player: "AI服务暂时不可用，角色行为由算法驱动"
```

### Prompt组装管道

```
模板(来自多Agent编排Appendix G)
  ↓ + 角色Profile (ADR-0002: CharacterData)
  ↓ + PERCEIVED关系值 (ADR-0002: RelationshipInfo)
  ↓ + 最近事件 (ADR-0002: EventLog, 最近5-10条)
  ↓ + 身体状态 (ADR-0003: body_hint标记, 仅受害女性)
  ↓ + 游戏状态 (ADR-0001: get_current_state())
  ↓ + 可用行动列表 (ADR-0003: MULTI_AGENT_ORCH)
  → 完整Prompt → HTTPRequest POST
```

## Alternatives Considered

### Alternative 1: AsyncTaskQueue + full response + typewriter (CHOSEN)
- **Description**: 协程任务队列(max 3 active) + `HTTPRequest`获取完整JSON + Godot `Timer`以40ms间隔逐字推送。无SSE依赖。
- **Pros**: Web WASM完全兼容（纯GDScript协程+标准HTTP）。实现简单——Godot内建节点。打字机动画与流式方案感知相同。
- **Cons**: 初始延迟更长（等待完整LLM响应——典型500-1500ms，而非流式首token 200ms）。LLM响应时间变化时打字机不做中间调整。
- **Rejection Reason**: N/A — chosen

### Alternative 2: JavaScriptBridge + fetch + ReadableStream
- **Description**: 使用`JavaScriptBridge.eval()`直接调用浏览器`fetch()`+`ReadableStream` API。真正的SSE流式——LLM token在到达时逐字渲染。
- **Pros**: 真正的流式——首token 200-400ms。更低感知延迟。浏览器原生——无Godot HTTPRequest的CORS限制。
- **Cons**: 绑定Web平台——无法在桌面编辑器/导出中运行。JS→GDScript桥接产生序列化开销（每个token需要eval→callback）。调试困难——错误发生在JS端。安全——`eval()`执行任意JS。
- **Rejection Reason**: 平台绑定不可接受——MVP需在Godot桌面编辑器中可测试。首token延迟差异（200ms vs 500ms）在打字机40ms/字的速度下不可感知。

### Alternative 3: 代理服务器缓冲
- **Description**: 部署Python/Node中间代理——代理连接LLM的SSE流式，缓冲为分块JSON。Godot通过`HTTPRequest`轮询代理获取下一块。
- **Pros**: Godot端仍使用标准HTTP。支持真正的流式（通过轮询模拟）。
- **Cons**: 增加运维复杂度——需要部署和维护代理服务器。代理宕机=游戏瘫痪。轮询引入额外延迟。MVP阶段过度架构。
- **Rejection Reason**: MVP仅需3角色×5天——代理运维成本不匹配项目规模。如果扩展为实时对话（20+用户并发），代理方案值得重新评估。

## Consequences

### Positive
- Web WASM完全兼容——无Semaphore、无SSE、无JavaScrip桥接依赖
- 打字机视觉效果与流式方案感知相同（每40ms一个字符——人眼无法区分源是SSE还是Timer）
- 基于协程的AsyncTaskQueue比Semaphore模式更简单——纯GDScript逻辑
- 3层降级保证LLM API完全不可用时游戏仍可运行（纯数值模式）
- 两种模型层级控制成本——FAST调用$0.002 vs NORMAL调用$0.01

### Negative
- 初始延迟~0.5-2s（等待完整LLM响应）vs 流式首token~200-400ms
- 打字机速度是固定的——LLM响应无论长短都以相同速度显示
- 零并发（Web WASM）限制了吞吐量——但3个agent规模下这不是瓶颈

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| LLM API CORS拒绝Web导出请求（`Access-Control-Allow-Origin`缺失） | High | 如果LLM提供商不支持CORS → 需要简单的CORS代理。记录为open question——实现前验证 |
| `HTTPRequest.timeout`在Web WASM中行为未验证（浏览器fetch超时可能不同） | Medium | 添加应用层超时——如果30s内无`request_completed`信号，手动取消+标记失败 |
| `Timer`在浏览器标签页隐藏时被节流——打字机可能卡顿 | Low | 节流不影响正确性——仅延迟。玩家恢复标签页时文本已完成 |
| `_active_count`竞态——协程完成和`_try_process()`之间的小时间窗口 | Low | Web WASM单线程——无真正竞态。`await`协程在`_try_process()`的同一事件循环迭代中不中断 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| llm-dialogue-generation | Core Rules, Rule 1 | 三种文本模式: agent_decision, dialogue_render, body_reflection | `request_agent_decision()` / `request_dialogue_render()` / `request_body_reflection()` — 对应FAST/NORMAL/NORMAL模型层级 |
| llm-dialogue-generation | Core Rules, Rule 2 | 7层Prompt组装管道 | Prompt Assembler — Profile→PERCEIVED→Events→Body→State→Actions→World |
| llm-dialogue-generation | Core Rules, Rule 3 | LLM API调用: endpoint, key, model, semaphore(3), 2 retries, 30s timeout | `_post_llm()` + AsyncTaskQueue(3 concurrent) + `_parse_with_retry(max_retries=2)` + timeout=30s |
| llm-dialogue-generation | Core Rules, Rule 4 | 流式文本管道: token→缓冲→逐字显示(40ms/字) | Typewriter Pipeline: `Timer(40ms)` → `text_token(char)` signal。Player skip → emit full immediately |
| llm-dialogue-generation | Core Rules, Rule 5 | 对话历史管理: 缓冲50条, Prompt注入10条 | `_dialogue_history: Array[50]` — `_inject_recent(n=10)` |
| llm-dialogue-generation | Edge Cases | LLM返回无效JSON → 重试2次。失败→fallback。超时→skip+fallback | `_parse_with_retry()` — 2 retries + degraded + numeric fallback |
| llm-dialogue-generation | Edge Cases | 连续3次API失败→纯数值模式 | `_consecutive_failures >= 3` → `_activate_numeric_fallback()` |
| llm-dialogue-generation | Appendix A | 7种对话模式: daily_talk/flirt/confrontation/intimate/inner_thought/narration/body_description | 模式选择由GameStateMachine当前状态驱动 — `_select_dialogue_mode(state)` |
| multi-agent-orchestration | Appendix G | Agent决策Prompt: JSON输出{thinking, emotion, action_chain} | `request_agent_decision()` 使用Appendix G模板。FAST模型。action_chain长度验证（2-3/3-4/1 urgent） |
| multi-agent-orchestration | Appendix H | 3层容错: 重试→降级→纯数值。Fallback行为表 | 完整实现——重试(2次,指数退避)→ degraded(上次成功模板)→ numeric(per role×block表) |
| multi-agent-orchestration | Appendix H | 并发LLM: asyncio.gather + Semaphore → GDScript等价 | AsyncTaskQueue(coroutine-based, max 3 concurrent, Web WASM compatible) |

## Performance Implications
- **CPU**: `HTTPRequest` async — 无CPU占用。JSON解析 ~0.5ms。打字机Timer ~0.01ms/tick。可忽略
- **Memory**: 对话历史缓冲 50条 × ~500B = 25KB。Prompt组装 ~4KB。总计 <50KB
- **Load Time**: Prompt模板加载 <1ms。API连接验证 ~5ms（第一帧异步）
- **Network**: 每局~6-9次Agent决策(FAST) + ~15次对话渲染(NORMAL) → ~$0.25/局。每请求~1-5KB上行, ~500-2000B下行

## Migration Plan
N/A — 新系统。

## Validation Criteria
- [ ] AsyncTaskQueue: 4个任务入队，max_concurrent=3 — 前3个启动，第4个等待 — 一个完成后第4个开始
- [ ] `_parse_with_retry()`: 无效JSON → 重试(最多2次) → 3次全部无效 → 返回error（不崩溃）
- [ ] `request_completed` 30s超时 → 标记失败 → fallback，不阻塞游戏循环
- [ ] 连续3次API失败 → 纯数值模式激活 + fallback动作加载
- [ ] `text_token` 信号以40ms间隔发射 — 字符在RichTextLabel中逐字出现
- [ ] 玩家点击/空格 → 打字机立即完成 — 完整文本显示
- [ ] FAST模型用于`request_agent_decision`(max_tokens=500) — NORMAL用于`request_dialogue_render`(max_tokens=1000)
- [ ] Prompt组装: 7层注入全部执行 — 无漏层
- [ ] `text_error` 信号在API失败时发射 → DialogueUI显示fallback文本
- [ ] `_active_count` 在任务完成后正确递减 — 队列恢复处理

## Related Decisions
- ADR-0003: AgentContext通过MultiAgentOrchestrator组装 — 传递给`request_agent_decision()`
- ADR-0004: LLMDialogue在autoload位置6 — 在MultiAgentOrchestrator(5)之后、DialogueUI(10)之前初始化
- ADR-0006: DialogueUI监听`text_token`/`text_token_complete`/`text_error`信号
