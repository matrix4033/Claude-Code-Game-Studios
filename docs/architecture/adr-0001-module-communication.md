# ADR-0001: 模块间通信与状态管理

## Status
Accepted

## Date
2026-05-02

## Last Verified
2026-05-02

## Decision Makers
User + Claude Code Game Studios (Technical Director review via TD-ARCHITECTURE gate)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | Core (Signals, State Machine, Autoload) |
| **Knowledge Risk** | LOW — `signal`, `enum`, Autoload are stable Godot 4.x APIs since 4.0 |
| **References Consulted** | `docs/engine-reference/godot/modules/ui.md`, `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/deprecated-apis.md` |
| **Post-Cutoff APIs Used** | None — all APIs used (`signal`, `enum`, `autoload`, `_ready()`) are pre-4.3 stable features |
| **Verification Required** | (1) Web export: autoload `_ready()` order and signal connection timing; (2) 4.6 dual-focus: `focus_neighbor_*` on ChoiceButton nodes during state transitions; (3) Tab-away buffer: verify signal arbitration produces correct results when browser tab regains focus after being backgrounded |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None |
| **Enables** | ADR-0002 (REAL/PERCEIVED data architecture), ADR-0003 (Agent lifecycle), ADR-0004 (Autoload order), ADR-0006 (Dialogue UI) |
| **Blocks** | All Core/Feature/Presentation layer ADRs — Foundation must be stable before downstream decisions |
| **Ordering Note** | First Foundation ADR to be written. ADR-0004 (Autoload order) depends on the signal connection contracts defined here. |

## Context

### Problem Statement
「浮世」有11个模块跨5个架构层。没有统一的通信机制会导致:
- 模块间直接调用 → 紧耦合，单元测试无法隔离
- 状态变更无通知 → UI与游戏状态不同步（例如OBSERVATION中不应显示介入按钮，但UI不知道状态已切换）
- 多个触发器同时到达 → 无仲裁规则导致竞态（例如`day_end`和`intimacy_trigger`同时触发——应该先处理亲密还是先日转？）

### Constraints
- Godot 4.6 / GDScript / Web导出平台
- Web WASM单线程 — 无多线程竞争，但也无`Semaphore`/`Mutex`
- 11个模块，依赖关系已在架构文档 `docs/architecture/architecture.md` 中明确
- 所有跨模块通信必须可追踪、可调试
- 性能预算: 状态切换处理 <2ms（文本游戏无帧预算压力，但应保持轻量）

### Requirements
- 支持7个游戏状态(TITLE→OBSERVATION→DISCOVERY→CONFRONTATION→INTIMACY→REFLECTION→TRANSITION)的完整转换图（12条合法转换，3条禁止转换）
- 同一帧多个触发器到达时按优先级仲裁
- 非法转换必须拒绝并记录日志（不静默崩溃）
- Presentation层模块（3个）必须同步响应状态变更
- 任何模块不得直接修改另一个模块拥有的状态

## Decision

**使用 Godot 信号作为主要的跨模块通信机制。`state_changed(old_state, new_state)` 是架构的中心信号。直接 Autoload 方法调用和 groups 作为辅助手段。**

### 通信手段治理规则

| 手段 | 适用场景 | 不适用场景 |
|------|---------|-----------|
| **信号 (Primary)** | 通知事件: 状态变更、行动完成、文本到达、关系变化 | 同步查询（信号是异步推送，不适合请求-响应） |
| **直接Autoload方法调用** | 同步查询: `get_current_state()`, `get_current_time_context()`, `get_character_profile()` | 写入操作: 不得通过直接调用修改其他模块拥有的状态 |
| **Groups** | 临时广播: "系统暂停"、"全局设置变更" 等不频繁的全系统通知 | 常规跨模块通信（groups不支持类型安全参数） |

**规则**: 查询（读）用同步方法调用。命令/事件（写/通知）用信号。

### Architecture Diagram

```
                      GameStateMachine (Autoload)
                      ═══════════════════════════
                      signal state_changed(old_state, new_state)
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   VisualPres.          DialogueUI          RelationPanel
   (光效切换)            (布局切换)            (透明度控制)
          │                   │                   │
          └───────────────────┼───────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   DayCycleMgr         PlayerInterv.       MultiAgentOrch
   (day_begin递增)      (行动可用性)         (活跃/非活跃)
```

### 状态转换图

```
TITLE ──→ OBSERVATION                   (game_start)
OBSERVATION ──→ DISCOVERY                (secret_discovered)
OBSERVATION ──→ REFLECTION               (skip_to_reflection)
OBSERVATION ──→ TRANSITION               (skip_to_transition)
DISCOVERY ──→ CONFRONTATION              (player_intervene)
DISCOVERY ──→ INTIMACY                   (intimacy_trigger)
DISCOVERY ──→ REFLECTION                 (发现但不介入)
CONFRONTATION ──→ INTIMACY               (intimacy_trigger)
CONFRONTATION ──→ REFLECTION              (confrontation_end)
INTIMACY ──→ REFLECTION                   (intimacy_end — 强制)
REFLECTION ──→ TRANSITION                 (day_end)
TRANSITION ──→ OBSERVATION                (day_begin)
```

**禁止的转换（静默拒绝+错误日志）**:
- TITLE → 非OBSERVATION
- INTIMACY → 非REFLECTION（亲密后必须沉淀）
- TRANSITION → 非OBSERVATION

### 触发-转换映射

| 触发器 | 来源模块 | 目标转换 |
|--------|---------|---------|
| `game_start` | DialogueUI (开始按钮) | TITLE → OBSERVATION |
| `secret_discovered(secret)` | RelationshipInfo | OBSERVATION → DISCOVERY |
| `player_intervene(action)` | PlayerIntervention | DISCOVERY → CONFRONTATION |
| `intimacy_trigger` | RelationshipInfo (阈值) 或 MultiAgent (AI行动) | DISCOVERY/CONFRONTATION → INTIMACY |
| `confrontation_end` | PlayerIntervention | CONFRONTATION → REFLECTION |
| `intimacy_end` | LLMDialogue | INTIMACY → REFLECTION |
| `day_end` | DayCycle | REFLECTION → TRANSITION |
| `day_begin` | DayCycle (过渡动画完成) | TRANSITION → OBSERVATION |
| `skip_to_reflection` | DayCycle (≥50%块时间无事件) | OBSERVATION → REFLECTION |
| `skip_to_transition` | DayCycle (全天无事件) | OBSERVATION → TRANSITION |

### Key Interfaces

```gdscript
# === GameStateMachine (Autoload) ===

enum GameState {
    TITLE, OBSERVATION, DISCOVERY, CONFRONTATION,
    INTIMACY, REFLECTION, TRANSITION
}

# 中心信号 — 所有Presentation层模块监听
signal state_changed(old_state: GameState, new_state: GameState)

# 查询接口（同步方法调用 — 合法的辅助通信手段）
func get_current_state() -> GameState

# 唯一的转换入口 — 任何模块不得直接设置current_state
func request_transition(trigger: String) -> void

# === 触发缓冲与仲裁（内部实现） ===
# request_transition() 将触发追加到 _pending_triggers[]
# _process() 中调用 _resolve_triggers() 选择最高优先级触发
# 状态锁定标志 _state_locked 防止执行转换时的重入

# 仲裁优先级常量
const TRIGGER_PRIORITY: Dictionary = {
    "intimacy_trigger": 100,
    "player_intervene": 90,
    "secret_discovered": 80,
    "day_end": 70,
    "skip_to_transition": 60,
    "skip_to_reflection": 50,
}
```

**调用方约束**:
- 不得直接赋值 `current_state` — 只能通过 `request_transition()`
- 非法转换被静默拒绝（`push_error()` 记录日志），保持当前状态
- `state_changed` 信号在 `on_enter()` 中发射，不在 `request_transition()` 中
- 过渡动画未完成时新触发排队（`_state_locked=true` 时缓冲）

**GameStateMachine保证**:
- 每次状态变更恰好发送一次 `state_changed` 信号
- 同一帧多触发按仲裁优先级处理
- 三个强制路径不可被跳过打断: INTIMACY→REFLECTION, TRANSITION→OBSERVATION, TITLE→OBSERVATION
- 触发缓冲在 `_process()` 帧中解析——不在触发被调用的同一位置立即执行

**信号连接生命周期**:
- 所有消费模块在 `_ready()` 中连接 `state_changed`
- 所有消费模块在 `_exit_tree()` 中断开连接（防止Web导出场景切换时的内存泄漏）
- `connection_timing`: 下游Autoload的 `_ready()` 在GameStateMachine的 `_ready()` 之后执行（Godot保证Autoload的`_ready()`顺序与`project.godot`配置顺序一致）

### 模块级信号规范

| Producer | Signal | Consumers |
|----------|--------|-----------|
| DayCycle | `day_end()`, `day_begin()`, `skip_to_reflection()`, `skip_to_transition()` | GameStateMachine |
| PlayerIntervention | `intervention_completed(result: ActionResult)` | DialogueUI, MultiAgent |
| RelationshipInfo | `relationship_changed(from: String, to: String, dim: String, delta: float)` | RelationPanel, VisualPres. |
| RelationshipInfo | `secret_discovered(secret: DesireTag)` | PlayerIntervention, GameStateMachine |
| MultiAgent | `agent_action_completed(result: ActionResult)` | DayCycle, LLMDialogue |
| LLMDialogue | `text_token(char: String)`, `text_token_complete()` | DialogueUI |

## Alternatives Considered

### Alternative 1: Godot信号中枢模式 (CHOSEN)
- **Description**: `state_changed` 为GameStateMachine发出的中心信号。所有Presentation层模块监听此信号。模块间通信通过专用信号。直接方法调用和groups作为辅助。
- **Pros**: 符合Godot习惯——信号是一等公民；编辑器可视化连接；类型安全；零额外基础设施；调用图可追踪（看信号连接即可理解数据流）；支持一对多通知
- **Cons**: 信号是单向的——请求-响应需两个信号；模块需要Autoload引用（`GameStateMachine.request_transition(...)`）；需显式管理连接生命周期
- **Rejection Reason**: N/A — chosen

### Alternative 2: EventBus全局事件总线
- **Description**: 全局 `EventBus` autoload，所有模块向总线发布/订阅事件。模块不直接连接彼此的信号。
- **Pros**: 最大解耦——生产者不知道消费者的存在；动态订阅/取消灵活
- **Cons**: 隐式依赖——难追踪"谁在监听这个事件"；调试困难（栈追踪经过Bus）；容易退化为God Object（Bus承载了所有通信逻辑）；增加一个在11模块规模下不必要的基础设施层
- **Rejection Reason**: 增加中间层带来的灵活性价值在当前11模块、已知依赖的项目中不成立。Godot原生信号已提供足够解耦。在Godot中，EventBus本质上是信号的包装器——增加间接层不增加能力。

### Alternative 3: 纯直接调用+接口
- **Description**: 每个模块暴露公共方法。模块间通过方法调用传递数据。无信号、无总线。
- **Pros**: 最简单——直接调用、直接调试；无异步复杂度；请求-响应天然支持
- **Cons**: 紧耦合——修改一个模块接口需修改所有调用方；无法支持一对多通知（状态变更时需逐个调用每个监听者）；单元测试无法隔离模块；Presentation层需每帧轮询状态（而非被动接收）
- **Rejection Reason**: 违背架构原则1（信号是主要的跨模块通信机制）。一对多通知（如state_changed有6+个监听器）在直接调用下会变成脆弱的硬编码调用链。Web WASM单线程已排除线程安全问题——信号的异步解耦优势在此项目中是纯粹的架构收益。

### Alternative 4: 纯信号——禁止任何直接调用
- **Description**: 连 `get_current_state()` 都通过信号实现（请求信号+回复信号）。
- **Pros**: 绝对的解耦——所有跨模块交互都经过信号
- **Cons**: 简单查询（"现在是什么状态？"）需要请求信号+等待回复信号——增加不必要的复杂度和延迟。不符合Godot的实用主义习惯。
- **Rejection Reason**: 务实折中——查询用直接调用（同步、简单），事件用信号（异步、解耦）。治理规则清晰记录了边界。

## Consequences

### Positive
- 所有跨模块事件通信可追踪——在Godot编辑器中查看信号连接即可理解数据流
- 新Presentation层子系统只需连接 `state_changed` 即可获得状态感知
- 单元测试可隔离——将信号连接到mock监听器，无需实例化整个模块链
- 符合Godot习惯——任何Godot开发者都能立即理解该模式
- 查询与事件有明确的治理边界——开发者知道何时用方法调用、何时用信号

### Negative
- 信号是单向的（生产者→消费者）——请求-响应模式需要两个信号
- 模块初始化时需确保信号连接完成——启动顺序由ADR-0004定义
- GDScript信号在调试器中不如方法调用直观（调用栈显示`emit_signal`而非具体的处理函数名）

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| `state_changed` 监听器过多（6+个），处理时间累积导致帧尖峰 | Low | 每个 `_on_state_changed` 必须 ≤0.5ms（只设标志位、启动Tween，不做重计算） |
| Web标签页失焦后恢复时，缓冲的触发被批量消费，可能产生不正确的仲裁结果 | Low | `_pending_triggers` 只在 `_process()` 中消费。仲裁基于优先级而非时序——即使批量消费，最高优先级触发仍然胜出 |
| 4.6 dual-focus系统中，`state_changed` 触发UI布局切换时，鼠标悬停和键盘焦点可能同时激活两个控件 | Low | 状态切换时清除所有焦点再重新分配；`ChoiceButton` 设置 `focus_neighbor_*` |
| 场景切换时信号连接泄漏（Web导出内存限制低） | Low | 强制所有消费者在 `_exit_tree()` 中 `disconnect()` |
| 信号参数命名与架构文档/GDD不一致导致实现者写出错误的连接代码 | Low | ADR为权威来源；architecture.md和GDD在写入时同步更新 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| game-state-machine | Detailed Design, Rule 1 | 7 GameState枚举定义 | `GameState` enum with 7 values |
| game-state-machine | Detailed Design, Rule 2 | 12条合法状态转换 + 3条禁止转换 | 完整转换图 + 触发-转换映射表。非法转换在`request_transition()`中拒绝+日志 |
| game-state-machine | Formulas | 多触发器冲突仲裁 `intimacy > intervene > discovered > day_end > skip` | `TRIGGER_PRIORITY` 字典 — `_resolve_triggers()` 按优先级选择 |
| game-state-machine | Core Rules, Rule 4 | 状态生命周期: `on_enter()` 发信号, `on_exit()` 保存上下文, `on_update()` 检查转换 | `state_changed` 在`on_enter()`中发射。`request_transition()`+`_process()` 协作提供 `on_update` |
| multi-agent-orchestration | Appendix F | Agent间事件驱动通信协议（不直接发消息） | 模块级信号规范 — Agent通过`agent_action_completed`通知下游，通过`relationship_changed`/`secret_discovered`感知世界 |
| dialogue-intervention-ui | Detailed Design, Rule 4 | 状态→布局映射表（7状态各有不同面板/按钮/透明度配置） | `state_changed` 驱动UI的 `on_state_changed(old, new)` → 按映射表切换布局 |
| dialogue-intervention-ui | Detailed Design, Rule 5 | 输入处理: 键盘/鼠标/触摸 + Tab焦点切换 | 4.6 dual-focus适配 — `focus_neighbor_*` 显式设置。状态切换时清除所有焦点再重分配 |

## Performance Implications
- **CPU**: `state_changed` 信号发射 ~0.05ms，6个监听器各 ~0.2ms → 总开销 <1.5ms/状态切换。状态切换频率 ~10-20次/游戏天 → 可忽略
- **Memory**: 信号连接 ~20-30个，每个连接 ~100 bytes → <3KB。触发缓冲 `_pending_triggers` 最大 <10个 → <1KB
- **Load Time**: 无影响——信号在运行时通过 `connect()` 连接。
- **Network**: N/A

## Migration Plan
N/A — 首个ADR，无现有代码需要迁移。

## Validation Criteria
- [ ] `request_transition()` 接受10个合法触发器，拒绝未知触发器和非法转换（如 INTIMACY→OBSERVATION）
- [ ] 同一帧同时发射 `day_end` 和 `intimacy_trigger` → 仅 `intimacy_trigger` 执行（优先级100 > 70）
- [ ] `state_changed` 信号在每次状态变更时恰好发射一次（测试: 从TITLE到OBSERVATION再到DISCOVERY → 恰好2次信号）
- [ ] INTIMACY→非REFLECTION 转换被拒绝，错误写入日志
- [ ] 6个 `state_changed` 监听器回调总耗时 <2ms
- [ ] Web标签页失焦30秒后恢复 → 缓冲触发被正确仲裁（不受挂起时间影响）
- [ ] 场景切换（如从游戏场景到菜单场景）→ 所有 `state_changed` 连接在旧场景 `_exit_tree()` 中断开
- [ ] `ChoiceButton` 在 `state_changed` 切换布局时 Tab 导航正常工作（4.6 dual-focus）

## Related Decisions
- ADR-0004: 引擎启动顺序与Autoload依赖链 — 定义本ADR中GameStateMachine的autoload顺序和初始化时机
- ADR-0006: 对话界面架构 — DialogueUI对 `state_changed` 的具体响应行为（状态→布局映射表实现）
