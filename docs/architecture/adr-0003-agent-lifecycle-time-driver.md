# ADR-0003: Agent生命周期与时间驱动

## Status
Accepted

## Date
2026-05-02

> **TD-ADR Review**: CONCERNS resolved — added `block_end(previous_block, next_block)` signal for intra-day block transitions (2026-05-02)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | Core (Agent Scheduling + Time System) |
| **Knowledge Risk** | LOW — `randf()`, `Array.sort_custom()`, `await`, custom clock logic are all GDScript-level patterns stable since Godot 4.0 |
| **References Consulted** | `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/deprecated-apis.md`, `docs/engine-reference/godot/current-best-practices.md` |
| **Post-Cutoff APIs Used** | None — weighted random sort, coroutine `await`, and custom clock logic are pre-cutoff GDScript patterns |
| **Verification Required** | (1) `Array.sort_custom(Callable)` on Web export — confirm performance for ≤3 agent sort; (2) `AsyncTaskQueue` coroutine chain error propagation — confirm failed LLM calls don't stall the queue; (3) weighted sort tiebreaker determinism — same seed produces same ordering; (4) INTIMACY state `base_tick` halving — verify clock slowdown is perceptible |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (`state_changed` — agents toggle active based on game state; `day_end`/`day_begin`/`skip_*` signals to GameStateMachine), ADR-0002 (`get_perceived_relationships()` for agent decisions, `apply_delta()` for relationship writes) |
| **Enables** | ADR-0005 (LLM client — agent decisions call LLM via `AgentContext`), ADR-0007 (player intervention — `advance_clock` called after intervene) |
| **Blocks** | LLM integration epic, player intervention implementation |
| **Ordering Note** | Core-layer ADR. Foundation ADRs (0001, 0002) must be written first. This ADR defines the clock contract that ADR-0005's LLM calls depend on. |

## Context

### Problem Statement
每个Agent具有不同的性格参数（desire_drive、sociability、trust_inclination），应该以不同的频率和顺序行动。一个高欲望的HOME_WRECKER应该比一个被动CUCKOLD更早行动。同时，游戏需要事件驱动时钟——Agent的每次行动消耗时间，时间到达块边界触发状态转换。

如果没有统一的Agent生命周期管理：
1. Agent的行动顺序变成固定轮询——所有Agent看起来同样活跃，破坏Pillar 1（"AI真实自主性"）。
2. 每个Agent行动后直接调用LLM——每一个小行动都需要网络往返，一个游戏天可达27次LLM调用（3 agent × 3块 × 3行动）。行动链机制将此减少到每个角色每天约1-2次LLM调用。
3. 游戏时钟和Agent行动是两个独立的循环——块边界可能在Agent执行到一半时到达，没有协同的"时钟→跳过"逻辑。

### Constraints
- Godot 4.6 / GDScript / Web WASM单线程（无Semaphore/Thread）
- 3个Agent角色 — 调度复杂度极低
- LLM API调用是成本和延迟瓶颈 — 需要减少调用频率
- Agent使用PERCEIVED值做决策（ADR-0002） — 不能访问REAL值
- 天数循环和Agent调度分属Foundation和Core层 — 必须是单独的autoload

### Requirements
- desire_drive加权随机排序确定Agent行动顺序
- 行动链（3-5行动/LLM决策），仅在链为空时调用LLM
- 事件驱动时钟：每次Agent行动完成后推进块时钟
- 无事件日跳过逻辑（block_time ≥ 50% → skip_to_reflection）
- INTIMACY状态：块时钟推进减半（亲密时刻更漫长）
- LLM不可用时的fallback行为（纯数值模式）
- 3层容错（重试→降级→纯数值）

## Decision

**DayCycleManager（Foundation层autoload）和MultiAgentOrchestrator（Core层autoload）保持为独立模块，共享 `advance_clock()` / `get_current_time_context()` 合约。Agent按desire_drive加权随机排序调度。每个Agent维护一个行动链——仅当链耗尽时触发LLM决策。时钟由Agent行动事件驱动推进。**

### 模块边界决策

| 维度 | DayCycleManager (M3, Foundation) | MultiAgentOrchestrator (M6, Core) |
|------|----------------------------------|-----------------------------------|
| **Owns** | day_number, current_block, block_time, base_tick, tick_multiplier, max_days | Agent调度器、行动链、目标管理、Suspicion、fallback表 |
| **向对方提供** | `get_current_time_context()` → TimeContext | `advance_clock(action_weight)` 调用 |
| **不拥有** | Agent状态、LLM决策、关系逻辑 | 时间状态、块切换逻辑、skip判定 |

**为什么分开**: DayCycleManager是纯时间逻辑——不依赖角色数据、Agent行为或LLM。MultiAgentOrchestrator依赖角色Profile、PERCEIVED关系值和LLM API。合并将把Core层依赖拖入Foundation层——违反架构原则2（数据所有权单向）。

**为什么在一个ADR中**: `advance_clock` 是两个模块之间最紧密的跨层合约。任何一个变化必然影响另一个。在单个ADR中定义此合约可以防止接口偏离。

### Architecture Diagram

```
MultiAgentOrchestrator (Core Autoload)
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ┌──────────────────────┐    ┌─────────────────────┐   │
│  │ Scheduler            │    │ Action Chain Engine  │   │
│  │ _sort_by_desire()    │───▶│ tick_agents()        │   │
│  │ desire_drive × randf │    │ if chain_empty → LLM │   │
│  │ + tiebreaker (id)    │    │ else → chain.pop()   │   │
│  └──────────────────────┘    └──────────┬──────────┘   │
│                                         │               │
│  ┌──────────────────────┐               │               │
│  │ Context Assembly     │◀──────────────┘               │
│  │ Profile + PERCEIVED  │    agent_action_completed     │
│  │ + Memory + Events    │───▶ signal                    │
│  └──────────────────────┘                               │
│                                         │               │
│  ┌──────────────────────┐               ▼               │
│  │ Suspicion Engine     │    ┌─────────────────────┐   │
│  │ body_leak detection  │    │ Fallback Table       │   │
│  │ level 0.0–1.0        │    │ per role × block     │   │
│  └──────────────────────┘    └─────────────────────┘   │
│                                         │               │
└─────────────────────────────────────────┼───────────────┘
                                          │
                    advance_clock(weight) │  get_time_context()
                                          │
┌─────────────────────────────────────────┼───────────────┐
│ DayCycleManager (Foundation Autoload)   │               │
│                                          ▼               │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Clock                                            │   │
│  │ block_time = min(block_time + base_tick          │   │
│  │   + action_weight × tick_multiplier, 1.0)        │   │
│  │                                                  │   │
│  │ base_tick: M=0.05, A=0.03, E=0.02               │   │
│  │ tick_mult: M=0.03, A=0.04, E=0.06                │   │
│  │ INTIMACY: base_tick × 0.5                         │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │ Skip Logic                                       │   │
│  │ block_time ≥ 0.5 && no_events → skip_reflection  │   │
│  │ full_day_no_events → skip_transition              │   │
│  │ 3_consecutive_idle_blocks → base_tick × 3         │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  signals: day_end | day_begin | skip_to_reflection      │
└─────────────────────────────────────────────────────────┘
```

### Key Interfaces

```gdscript
# === DayCycleManager (Foundation Autoload) ===

func get_current_time_context() -> TimeContext:
    # TimeContext { day_number: int, current_block: String, block_time: float }

func advance_clock(action_weight: float) -> void:
    # Core operation: block_time = min(block_time + base_tick + action_weight × tick_multiplier, 1.0)
    # action_weight: 0.0 (idle/no action) to 1.0 (major event like confrontation)
    # INTIMACY override: base_tick halved internally
    # block_time ≥ 1.0 → emit block_end signal (mid-day) or day_end (EVENING→day boundary)

signal day_begin()              # TRANSITION → OBSERVATION confirmed, day_number incremented
signal day_end()                # REFLECTION → TRANSITION trigger
signal block_end(previous_block: String, next_block: String)  # mid-day block transition: MORNING→AFTERNOON, AFTERNOON→EVENING
signal skip_to_reflection()     # block_time ≥ 0.5 with no events → trigger
signal skip_to_transition()     # full day no events → trigger

# === MultiAgentOrchestrator (Core Autoload) ===

func tick_agents(time_context: TimeContext) -> void:
    # Main loop called per time block:
    # 1. Sort: _sort_by_desire()
    # 2. For each agent: tick one action from chain
    # 3. If chain empty → flag agent as "idle"
    # 4. After all agents: gather idle agents → LLM decision (AsyncTaskQueue)
    # 5. Each completed action: DayCycle.advance_clock(action_weight)

signal agent_action_completed(result: Dictionary):
    # { agent_id, action_type, target_id, deltas, events_generated }

# === Agent Scheduling ===

# Internal sort function — pure (no randf() inside comparator)
func _sort_by_desire(agents: Array[Agent]) -> Array[Agent]:
    var weighted: Array = []
    for agent in agents:
        var weight: float = agent.desire_drive * (0.8 + randf() * 0.4)
        # weight range: 0.0–1.0 × 0.8–1.2(random_jitter per GDD)
        weighted.append({"agent": agent, "weight": weight, "id": agent.id})
    weighted.sort_custom(func(a, b):
        if a.weight != b.weight: return a.weight > b.weight  # descending
        return a.id < b.id  # tiebreaker: deterministic by ID
    )
    return weighted.map(func(w): return w.agent)

# === Action Chain Rules ===

# Chain length by block type:
# MORNING: 2-3 actions (low intensity)
# AFTERNOON: 3-4 actions (social active)
# EVENING: 2-3 actions (high intensity — each action is weightier)
# Emergency (confront/surprised): 1 action

# Agent cycle:
# if action_chain.is_empty():
#     agent is "idle" → triggers LLM decision in next batch
#     LLM returns JSON: { thinking, emotion, action_chain: [3-5 actions] }
# else:
#     current_action = action_chain.pop_front()
#     current_action.execute() → deltas → advance_clock → event propagation

# === Fallback Action Table (Pure Numeric Mode) ===

const FALLBACK_ACTIONS: Dictionary = {
    "VICTIM": {
        "MORNING": ["maintain:cuckold"],
        "AFTERNOON": ["maintain:cuckold"],
        "EVENING": ["investigate:self"]
    },
    "HOME_WRECKER": {
        "MORNING": ["observe:victim"],
        "AFTERNOON": ["seduce:victim"],
        "EVENING": ["seduce:victim"]
    },
    "CUCKOLD": {
        "MORNING": ["maintain:victim"],
        "AFTERNOON": ["maintain:victim"],
        "EVENING": ["maintain:victim"]
    }
}

# === Suspicion System ===

# Cuckold only. Tracks observed body_leak events.
# suspicion.level: 0.0–1.0
#   <0.2: normal behavior
#   0.2–0.4: subtle dialogue probes ("are you okay?")
#   0.4–0.6: active observation (increased PERCEIVED update frequency)
#   0.6–0.8: investigate actions
#   0.8–1.0: confront
# suspicion decays: -0.05/day after 2+ days without anomalies
```

**concurrency limits** / **signal timing** / **seed control** / **Web export caveats**

### Alternatives Considered

### Alternative 1: Separate DayCycle + AgentOrch, shared clock contract (CHOSEN)
- **Description**: DayCycleManager (Foundation autoload) = pure time。MultiAgentOrchestrator (Core autoload) = agent scheduling。`advance_clock()` + `get_current_time_context()` 是共享合约。
- **Pros**: 尊重架构层边界（Foundation不依赖Core）。DayCycle可独立单元测试（无Agent）。符合架构文档Phase 2模块所有权。现有信号合约不变（`day_end`/`day_begin` 等源于 DayCycle）。
- **Cons**: 两个autoload而非一个——多一个全局节点。`advance_clock`需要跨层调用。
- **Rejection Reason**: N/A — chosen

### Alternative 2: 合并的单一Orchestrator Autoload
- **Description**: 将DayCycleManager和MultiAgentOrchestrator合并为一个autoload。Agent调度和时钟逻辑共存于一个`_process()`循环。
- **Pros**: 更简单——一个模块，一个`tick_agents()`循环。无跨autoload调用。
- **Cons**: 跨Foundation/Core边界——Core依赖（LLM、AgentProfile）被拖入Foundation层。违反架构原则2（数据所有权单向）。Foundation和Core无法独立测试。God Object风险：一个Autoload承载了时间管理、Agent调度和事件通信。
- **Rejection Reason**: 架构原则5（MVP不阻碍完整版路径）——独立模块更容易扩展（添加更多Agent角色或更复杂的时间系统时不需要修改Agent代码）。

### Alternative 3: 轮询调度（无desire_drive加权）
- **Description**: 固定顺序（VICTIM → HOME_WRECKER → CUCKOLD），每Agent每块一个行动。无加权随机。
- **Pros**: 完全确定性——相同初始条件产生相同结果。更简单（无排序逻辑、无随机种子管理）。
- **Cons**: 破坏Pillar 1（"AI真实自主性"）——高欲望角色与低欲望角色行为频率相同，感觉不自然。玩家很快看穿固定顺序模式。
- **Rejection Reason**: 加权随机是GDD规定的方案——它是Pillar 1的核心交付。3个Agent规模下`sort_custom`的复杂度微不足道。

### Alternative 4: 每行动一次LLM调用（无行动链）
- **Description**: 每个Agent行动都触发单独的LLM调用。无行动链状态管理。
- **Pros**: 最大响应性——Agent可以在每次行动后根据新事件调整行为。无"承诺然后无法反应"问题。
- **Cons**: 3 agent × 3块 × 3行动 = 最多27次LLM调用/天。按$0.002/FAST调用估算，每天成本~$0.05，5天游戏~$0.25（仅Agent决策——不包含对话渲染）。行动链可将此减少到每个角色每天1-2次LLM调用（~$0.02/天）。
- **Rejection Reason**: 成本降低3-5倍胜出。紧急响应（对峙、被发现）作为1行动链处理——保持关键路径的响应性。

## Consequences

### Positive
- 行动链将Agent决策的LLM调用减少3-5倍——5天游戏周期从~$1.25降至~$0.25
- desire_drive加权排序使高欲望Agent自然更活跃——Pillar 1的具体体现
- 事件驱动时钟对标签页失焦免疫——在`_process()`被节流时不累积偏差
- DayCycleManager可独立进行单元测试（输入TimeContext，验证触发信号）
- fallback行为表保证LLM API完全不可用时游戏仍可运行

### Negative
- 行动链中的Agent无法对链执行过程中发生的中间事件做出反应（除非是紧急响应——遵循1行动规则）。这对日常（MORNING/AFTERNOON）块来说是可接受的，但EVENING块可能存在在欺骗场景中反应迟钝的风险。缓解措施：EVENING链长度为2-3（非3-5），每个行动权值更大。
- 两个autoload通过`advance_clock`耦合——DayCycleManager必须在MultiAgentOrchestrator之前初始化（启动顺序由ADR-0004定义）
- 加权排序不保证"最活跃"Agent总是先行动——这符合设计（人类的不可预测性），但使Agent行为模式更难调试

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| `sort_custom`比较函数非纯（如果在比较函数内部调用randf()）→ 排序结果未定义 | Low | randf()在排序前缓存到weight。Comparator仅比较缓存值+tiebreaker。 |
| 加权利余平局时排序不稳定（两个Agent的weight相同）→ 顺序在不同运行之间变化 | Low | id-based tiebreaker确保确定性降级。 |
| AsyncTaskQueue协程错误静默失败（await的task抛出异常 → 协程停止但Queue不恢复）| Medium | 所有task结果通过Dictionary返回（含error字段）。失败的task由调用方显式处理。看门狗计数器检测队列冻结。 |
| 3个连续空闲块 → 加速因子×3可能超过1.0 | Low | `advance_clock()`在内部对block_time执行clamp，加速因子针对base_tick而非最终值。 |
| Web标签页失焦期间白天可能在单个帧中结束（批量缓冲agent决策） | Low | 架构已经通过`_process`内部的触发器缓冲（ADR-0001）处理此问题。DayCycle信号以正确的顺序消费。 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| multi-agent-orchestration | Core Rules, Rule 1 | Agent四层决策输入: 背景+目标+性格+信息(PERCEIVED) | `tick_agents()` 组装来自角色数据模型的Profile + PERCEIVED关系值 + 来自GDD的预设目标配置 |
| multi-agent-orchestration | Core Rules, Rule 2 | 目标结构: ShortTermGoal + LongTermGoal | 短期目标由LLM JSON返回（`action_chain`字段）。长期目标以预设配置加载（首次: maintain:苦主 / seduce:受害女性 / keep:受害女性） |
| multi-agent-orchestration | Core Rules, Rule 3 | 行动选择: 目标→行动类型→可行性检查→执行 | action_chain机制: LLM一次决策返回3-5行动。每个行动执行通过可行性检查(time block匹配) |
| multi-agent-orchestration | Core Rules, Rule 4 | Agent调度: desire_drive加权随机排序 | `_sort_by_desire()` — weight = desire_drive × random_jitter(0.8-1.2) + id tiebreaker |
| multi-agent-orchestration | Core Rules, Rule 5 | 信息约束: Agent使用PERCEIVED值(非REAL值)做决策 | PERCEIVED约束——Agent使用`get_perceived_relationships()`(ADR-0002)。body_response值永远不可见 |
| multi-agent-orchestration | Appendix F | Agent间事件驱动通信协议 | Agent通过`agent_action_completed`信号 + EventLog (ADR-0002) 间接感知其他Agent行为 |
| multi-agent-orchestration | Appendix G | LLM Prompt模板注入 | Context Assembly在`tick_agents()`中——Profile+PERCEIVED+记忆+事件+身体状态→完整Prompt上下文传递给ADR-0005 |
| multi-agent-orchestration | Appendix H | 失败容错: 3层（重试→降级→纯数值） | AsyncTaskQueue重试(2次), fallback动作表激活, 通知LLM不可用 |
| day-cycle-manager | Core Rules, Rules 1-3 | 3块结构 + 块时钟(0.0–1.0) + day推进 | `advance_clock()` + base_tick/tick_multiplier per block |
| day-cycle-manager | Core Rules, Rule 4 | day计数器(1–N) | `day_number`在收到`day_begin`时递增 |
| day-cycle-manager | Core Rules, Rule 5 | 事件触发时钟: advance_clock(action_weight) | `advance_clock()`公式: block_time = min(block_time + base_tick + action_weight × tick_multiplier, 1.0) |
| day-cycle-manager | Edge Cases | INTIMACY时钟减半 + 连续3空闲块加速 | INTIMACY: base_tick × 0.5. 空闲: 第3块 base_tick × 3 |
| day-cycle-manager | Edge Cases | Skip逻辑: block_time≥0.5无事件→skip_reflection, 全天无事件→skip_transition | Skip逻辑在`advance_clock()`内部——0.5阈值检查+事件计数器 |

## Performance Implications
- **CPU**: `_sort_by_desire()`: 3个agent × 排序 + 权重计算 < 0.01ms。`tick_agents()`: 每agent一次循环 < 0.1ms/agent × 3 = 0.3ms/块。总计 <0.5ms/块 — 可忽略
- **Memory**: Agent状态（行动链、目标、Suspicion级别）< 5KB/agent × 3 = <15KB
- **Load Time**: Fallback动作表加载 < 1ms。Agent预设目标配置< 1ms
- **Network**: 行动链将LLM调用减少3-5倍——每游戏天从~27次降至~6-9次（3 agent × ~2-3 LLM调用/天）

## Migration Plan
N/A — 新系统，无现有代码需要迁移。

## Validation Criteria
- [ ] desire_drive=0.9 agent的weight > desire_drive=0.5 agent在超过70%的排序运行中（100次试验，相同种子）
- [ ] 行动链长度: MORNING=2-3, AFTERNOON=3-4, EVENING=2-3, 紧急=1
- [ ] LLM仅在action_chain为空时调用——非每个tick调用
- [ ] `advance_clock(0.8)` 在EVENING块，base_tick=0.02, tick_mult=0.06 → block_time += 0.02 + 0.8×0.06 = 0.068
- [ ] INTIMACY状态: base_tick=0.02 → 0.01（减半），块持续时间×2
- [ ] 无事件块到block_time≥0.5 → `skip_to_reflection` 信号触发
- [ ] 全天无事件 → `skip_to_transition` 信号触发
- [ ] 3个连续空闲块 → base_tick三倍加速
- [ ] LLM API连续3次失败 → 纯数值模式激活 → fallback动作按角色+块加载
- [ ] 相同种子+相同初始条件 → 相同的Agent排序顺序（确定性tiebreaker）
- [ ] AsyncTaskQueue: 3个并发LLM调用中的1个失败 → 其他2个正常完成，失败任务报告error

## Related Decisions
- ADR-0001: `state_changed` 信号 — Agent在OBSERVATION/DISCOVERY/CONFRONTATION中活跃；`day_end`/`day_begin`/`skip_*` 信号到GameStateMachine
- ADR-0002: `get_perceived_relationships()` 用于Agent决策；`apply_delta()` 用于写入关系变更
- ADR-0004: Autoload启动顺序 — DayCycleManager必须在MultiAgentOrchestrator之前初始化
- ADR-0005: LLM客户端 — `AsyncTaskQueue` + Prompt组装合约（AgentContext → LLM response）
- ADR-0007: 玩家介入调用`advance_clock()`并通知Agent（`notify_player_action()`）
