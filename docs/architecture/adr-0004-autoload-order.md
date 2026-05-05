# ADR-0004: 引擎启动顺序与Autoload依赖链

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
| **Domain** | Core (Autoload Configuration) |
| **Knowledge Risk** | LOW — Autoload registration order and `_ready()` ordering are stable Godot 4.x features since 4.0 |
| **References Consulted** | `docs/engine-reference/godot/breaking-changes.md` |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | Web export: confirm autoload `_ready()` order matches `project.godot` registration order on WASM runtime |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (GameStateMachine autoload + signal contracts), ADR-0002 (RelationshipInfoManager autoload + data interfaces), ADR-0003 (DayCycleManager + MultiAgentOrchestrator autoloads + advance_clock contract) |
| **Enables** | ADR-0005 (LLM autoload initialization), ADR-0006 (DialogueUI autoload initialization), ADR-0007 (PlayerIntervention autoload initialization) |
| **Blocks** | Any code that calls autoload methods or connects to autoload signals — if autoload order is wrong, signals fire before listeners connect |
| **Ordering Note** | Final Foundation ADR — write after ADRs 1-3. Must be written before any implementation begins (incorrect autoload order causes silent "missed signal" bugs) |

## Context

### Problem Statement
Godot的autoload系统按照`project.godot`中的注册顺序初始化——先注册的autoload的`_ready()`先执行。如果下游autoload在其`_ready()`中连接上游autoload的信号，而下游autoload先初始化，连接将发生在信号发射*之后*——导致静默的信号丢失。

具体来说：
- GameStateMachine在`_ready()`中设置current_state=TITLE——不发射信号（无状态变更）
- 第一个`state_changed(TITLE, OBSERVATION)`在玩家点击"开始"后发射
- 但如果DialogueUI的`_ready()`在GameStateMachine的`_ready()`之后执行，第一个状态变更信号可能在UI的connect之前发射...

实际上这不是问题——第一个状态变更是用户触发的（点击按钮）。真正的风险是：
- MultiAgentOrchestrator在`_ready()`中连接DayCycleManager的`day_end`信号
- 如果MultiAgentOrchestrator在DayCycleManager之前初始化→连接在DayCycleManager发出任何信号之前建立→没问题
- 但如果DayCycleManager在初始化期间触发`day_end`（不应该，但防御性设计需要保证）

核心风险：**初始化顺序错误导致静默失败**。Godot不报告"信号已发射但无监听器"。

### Constraints
- Godot 4.6 autoload系统——顺序由`project.godot`中`[autoload]`section的注册顺序决定
- Godot保证autoload的`_ready()`按注册顺序执行
- 11个autoload跨5个架构层
- Web WASM单线程——初始化期间无并发问题

### Requirements
- Foundation autoload必须先于Core autoload初始化
- Core autoload必须先于Feature autoload初始化
- Presentation autoload可以与其他层并行初始化（仅需静态资源）
- 任何autoload在其`_ready()`完成前不得被其他autoload调用
- 信号连接在`_ready()`中建立——下游连接上游

## Decision

**`project.godot` autoload注册顺序按照架构层依赖排序：Foundation → Core → Feature → Presentation。同一层内，按无依赖→有依赖排序。**

### Autoload Registration Order (`project.godot`)

```
[autoload]
# === FOUNDATION LAYER (零或最少依赖) ===

CharacterData="*res://src/foundation/character_data.gd"
# 依赖: 无
# 在_ready()中: 从JSON文件加载3个角色Profile
# 不连接任何信号

RelationshipInfoManager="*res://src/foundation/relationship_info_manager.gd"
# 依赖: CharacterData (读取初始关系值)
# 在_ready()中: 初始化REAL/PERCEIVED矩阵、连接state_changed(ADR-0002)

DayCycleManager="*res://src/foundation/day_cycle_manager.gd"
# 依赖: 无
# 在_ready()中: 设置day_number=1, current_block=MORNING, block_time=0.0
# 连接state_changed以接收day_begin回调(ADR-0003)

GameStateMachine="*res://src/foundation/game_state_machine.gd"
# 依赖: 无(但DayCycle和RelationshipInfo必须在之前初始化——它们在_ready()中连接到此autoload的信号)
# 在_ready()中: 设置current_state=TITLE, 不发射信号(ADR-0001)

# === CORE LAYER (依赖Foundation) ===

MultiAgentOrchestrator="*res://src/core/multi_agent_orchestrator.gd"
# 依赖: CharacterData, DayCycleManager, RelationshipInfoManager, GameStateMachine
# 在_ready()中: 加载Agent预设目标、fallback表、连接agent_action_completed信号
# 连接state_changed以确定活跃/非活跃(ADR-0003)

LLMDialogue="*res://src/core/llm_dialogue.gd"
# 依赖: MultiAgentOrchestrator, CharacterData, RelationshipInfoManager
# 在_ready()中: 加载Prompt模板、验证API连接
# 连接agent_action_completed信号以触发dialogue_render(ADR-0005)
# 连接state_changed以切换对话模式(daily_talk/flirt/confrontation/intimate)

# === FEATURE LAYER (依赖Core) ===

PlayerIntervention="*res://src/feature/player_intervention.gd"
# 依赖: GameStateMachine, RelationshipInfoManager, MultiAgentOrchestrator, DayCycleManager
# 在_ready()中: 重置每日介入计数=0
# 连接state_changed以启用/禁用介入选项(ADR-0007)

# === PRESENTATION LAYER (可并行——仅需静态资源) ===

VisualPresentation="*res://src/presentation/visual_presentation.gd"
# 依赖: GameStateMachine, CharacterData, RelationshipInfoManager
# 在_ready()中: 加载光效资源、角色立绘路径
# 连接state_changed以切换光照(Vertical Slice)

DialogueUI="*res://src/presentation/dialogue_ui.gd"
# 依赖: GameStateMachine, PlayerIntervention, LLMDialogue
# 在_ready()中: 初始化Control节点树、加载字体
# 连接state_changed以切换布局、连接text_token以逐字渲染(ADR-0006)

RelationshipGraphPanel="*res://src/presentation/relationship_graph_panel.gd"
# 依赖: GameStateMachine, RelationshipInfoManager
# 在_ready()中: 初始化空关系图
# 连接state_changed以控制透明度、连接relationship_changed以更新节点(Vertical Slice)

# === PERSISTENCE (独立于层——最后初始化，仅读取/写入已就绪的系统) ===

SaveSystem="*res://src/core/save_system.gd"
# 依赖: GameStateMachine, DayCycleManager, RelationshipInfoManager, CharacterData (读取状态以保存/恢复)
# 在_ready()中: 无——等待显式save_game()/load_game()调用
# 连接state_changed以监听TRANSITION自动存档(ADR-0008)
```

### 初始化时间线

```
帧1 (_ready()顺序):
  CharacterData._ready()        ~2ms   (JSON解析3个角色文件)
  RelationshipInfoManager._ready() ~0.5ms (初始化REAL/PERCEIVED矩阵)
  DayCycleManager._ready()      ~0.1ms
  GameStateMachine._ready()     ~0.1ms

  MultiAgentOrchestrator._ready() ~1ms  (加载fallback表+预设目标)
  LLMDialogue._ready()          ~5ms   (验证API连接——可异步化)

  PlayerIntervention._ready()   ~0.1ms

  VisualPresentation._ready()   ~3ms   (加载光效纹理)
  DialogueUI._ready()           ~2ms   (初始化Control节点树)
  RelationshipGraphPanel._ready() ~1ms

  SaveSystem._ready()           ~0.1ms (仅注册信号连接，无初始化操作)

帧1 _process():
  GameStateMachine: current_state=TITLE
  所有autoload已就绪 → 等待玩家输入

帧2+ (玩家点击"开始"):
  GameStateMachine.request_transition("game_start")
  TITLE → OBSERVATION → state_changed(TITLE, OBSERVATION) 发射
  → 所有6个监听器已连接(在_ready()中建立) ✅
```

### 信号连接时机保证

Godot保证autoload的`_ready()`按注册顺序执行。这意味着：
- GameStateMachine在Autoload列表的第4个——其`_ready()`在第4个执行
- DialogueUI在Autoload列表的第10个——其`_ready()`在第10个执行
- 当DialogueUI在`_ready()`中调用`GameStateMachine.state_changed.connect(_on_state_changed)`时，GameStateMachine已经完全初始化
- 第一个`state_changed`信号在用户点击"开始"后发射——此时所有autoload的`_ready()`都已执行完毕

**无需在`_process()`第一帧中延迟连接——Godot的`_ready()`顺序提供了这个保证。**

### Autoload间的方法调用规则

| 调用方向 | 允许？ | 示例 |
|---------|--------|------|
| 下游→上游 (查询) | ✅ | DialogueUI调用`GameStateMachine.get_current_state()` |
| 下游→上游 (写入) | ✅ | MultiAgentOrchestrator调用`DayCycleManager.advance_clock(0.8)` |
| 上游→下游 (查询) | ❌ | GameStateMachine调用`DialogueUI.xxx()`——Foundation不应依赖Presentation |
| 上游→下游 (写入) | ❌ | DayCycleManager调用`MultiAgentOrchestrator.xxx()`——Foundation不应依赖Core |
| 同层调用 | ✅ | 仅查询——写入通过信号 |

## Alternatives Considered

### Alternative 1: 依赖排序注册 + _ready()连接 (CHOSEN)
- **Description**: `project.godot` autoload按依赖顺序注册。信号连接在各自的`_ready()`中建立。Godot保证`_ready()`按注册顺序执行。
- **Pros**: 利用Godot内建保证——无需手动管理初始化阶段。简单——排序列表即可实现。新autoload只需插入正确位置。常规Godot模式。
- **Cons**: 循环依赖在排序时就会暴露（无法确定先后顺序）——但这迫使解决设计问题而非掩盖它。如果`_ready()`中包含长时间操作（如LLM API连接验证），会阻塞后续autoload的初始化。
- **Rejection Reason**: N/A — chosen

### Alternative 2: 延迟初始化管理器
- **Description**: 单一`InitManager` autoload管理所有模块的初始化阶段。提供一个"阶段完成"回调系统。下游模块注册其依赖关系，InitManager在所有依赖就绪后调用其初始化函数。
- **Pros**: 最大控制——可处理循环依赖（如果设计允许）。清晰的初始化阶段日志。支持异步初始化。
- **Cons**: 过度工程——11个autoload不需要专门的初始化管理器。增加一层间接——InitManager成为新的God Object。隐藏依赖关系（在代码中注册而非在`project.godot`中可见）。
- **Rejection Reason**: 在11个autoload的规模下过度设计。Godot的`_ready()`排序已经提供了所需保证。

### Alternative 3: 按字母顺序排列（无显式管理）
- **Description**: autoload按名称的字母顺序注册。无显式排序规则。模块在`_ready()`中检查依赖是否已初始化。
- **Pros**: 无——仅避免"决定顺序"的决策成本。
- **Cons**: 隐式依赖——名称变更可能改变初始化顺序。静默失败——依赖未初始化时需手动检查。新开发者无法从`project.godot`理解初始化顺序。
- **Rejection Reason**: 在关键路径上创造不确定性以换取微不足道的便利。当生产代码中autoload重命名导致初始化顺序破坏时，调试成本极高。

## Consequences

### Positive
- 初始化顺序在`project.godot`中一目了然——任何开发者都能理解启动顺序
- Godot保证的`_ready()`执行顺序消除了手动协调的需要
- Foundation/Core/Feature/Presentation的分层注册强化了架构文档定义的边界
- 循环依赖在注册阶段就会暴露——两个autoload互相依赖无法排序
- 新autoload只需插入正确的层位置——不需要调用复杂的注册API

### Negative
- `LLMDialogue._ready()`中的API连接验证（~5ms）会阻塞后续autoload的`_ready()`——虽短但确实是阻塞的
- autoload名称固定——重命名autoload需要在`project.godot`中重新排序（但重命名本身就很少发生）
- 如果未来autoload数量增长到20+，单层的`_ready()`时间会累积——但目前11个总量<15ms可接受

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| LLMDialogue API验证超时（>5s）阻塞整个游戏启动 | Medium | 将API验证移到后台——`_ready()`中只做快速可达性检查，完整验证在`_process()`第一帧异步执行 |
| autoload重命名后`project.godot`自动重新排序破坏初始化顺序 | Low | 在`project.godot`中保存注释标记每层的开始位置。重命名后手动验证排序 |
| Web WASM运行时autoload `_ready()`顺序与编辑器不同 | Low | ADR的验证标准要求Web导出测试。差异极低——Godot 4.x WASM运行时遵循与桌面相同的autoload顺序 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| game-state-machine | Core Rules, Rule 1 | 7状态枚举在游戏启动时初始化 | GameStateMachine在第4个autoload初始化——所有消费者在其后连接 |
| day-cycle-manager | Core Rules, Rule 3 | day推进: day_begin → day_number递增 | DayCycleManager(第3)在GameStateMachine(第4)之前初始化——可以在GSM就绪前设置初始状态 |
| multi-agent-orchestration | Appendix H | 并发LLM调用管理 | MultiAgentOrchestrator(第5)在LLMDialogue(第6)之前——LLM autoload初始化时Agent语境已可用 |
| 架构文档 Phase 2 | Module Ownership | Foundation→Core→Feature→Presentation依赖方向 | `project.godot`注册顺序强制执行此方向 |

## Performance Implications
- **CPU**: 全部11个autoload的`_ready()`执行 <15ms总耗时 → 可忽略。在Web导出第一帧中不可感知
- **Memory**: autoload注册本身不消耗额外内存——每个autoload在`_ready()`中自行分配内存
- **Load Time**: <15ms增加到第一帧——在1-2秒的Web加载时间中可忽略
- **Network**: N/A

## Migration Plan
N/A — 新项目，无现有autoload需要重新排序。

## Validation Criteria
- [ ] Godot编辑器: 从`project.godot`按列出的顺序注册所有12个autoload，编辑器无错误
- [ ] 所有autoload的`_ready()`按注册顺序执行（日志验证: 打印每个autoload的名称+时间戳）
- [ ] DialogueUI在GameStateMachine之后调用`_ready()`——`state_changed`连接在第一个信号发射前建立
- [ ] MultiAgentOrchestrator可以调用`DayCycleManager.advance_clock()`而不产生null引用错误
- [ ] LLMDialogue API验证失败不阻塞其他autoload的`_ready()`——游戏仍启动，触发fallback模式
- [ ] Web导出: autoload `_ready()`顺序与桌面编辑器一致
- [ ] Foundation层autoload不引用Core/Feature/Presentation层autoload（代码审查检查点）

## Related Decisions
- ADR-0001: GameStateMachine autoload + `state_changed`信号——所有Presentation层autoload连接此信号
- ADR-0002: RelationshipInfoManager autoload——CharacterData之后初始化
- ADR-0003: DayCycleManager和MultiAgentOrchestrator的`advance_clock()`合约——排序强制执行此合约
- ADR-0006: DialogueUI autoload——在LLMDialogue和PlayerIntervention之后初始化
