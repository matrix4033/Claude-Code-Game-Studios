# ADR-0002: REAL/PERCEIVED双层数据架构

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
| **Domain** | Core (Data Architecture) |
| **Knowledge Risk** | LOW — `Dictionary`, `Array[T]`, `JSON` are stable Godot 4.x APIs since 4.0 |
| **References Consulted** | `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/deprecated-apis.md`, `docs/engine-reference/godot/current-best-practices.md` |
| **Post-Cutoff APIs Used** | `Dictionary.duplicate(true)` (Godot 4.0+ stable, not post-cutoff). No APIs from 4.4–4.6 are used. Note: `duplicate_deep()` is a `Resource` method only — NOT used here since we chose Dictionary storage. |
| **Verification Required** | (1) `Dictionary.duplicate(true)` deep copy behavior on nested Dictionaries with `float` values; (2) `JSON.stringify()` with nested Dictionaries on Web export `FileAccess` writing to `user://` (IndexedDB-backed); (3) `JSON.parse_string()` deserialization time for ~200 event entries on Web — test for >50ms pause on load |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (`state_changed` signal — visibility rules change per game state) |
| **Enables** | ADR-0003 (Agent reads PERCEIVED for decisions), ADR-0005 (LLM reads REAL+PERCEIVED for prompt injection), ADR-0007 (player intervention reads PERCEIVED for option generation) |
| **Blocks** | All Core/Feature ADRs that read or write relationship data |
| **Ordering Note** | Write after ADR-0001 (needs `state_changed` for visibility rules), before ADR-0003 (Agent needs PERCEIVED data) |

## Context

### Problem Statement
「浮世」的核心玩法——信息不对称——依赖于两个平行的"真相"：角色真实的感受（AI行为的基础）和玩家通过观察推断的感受（玩家决策的基础）。如果这两个版本不分离：

1. **玩家关系面板会显示body_response数值** — 破坏Pillar 2（"信息即权力"）。body_response是角色最私密的身体信号——玩家永远不应该通过一个数值条看到它。如果REAL和PERCEIVED混在同一数据结构中，任何UI开发者都可能意外读取REAL值而不经过可见性过滤器。

2. **玩家认知无法与真实关系脱节** — PERCEIVED值与REAL值之间的差距本身就是核心玩法。玩家在OBSERVATION中看到affection="高"（模糊）——但REAL值是0.55还是0.95？这个模糊区间是玩家做介入决策时的"不确定空间"。如果PERCEIVED被直接写入而非派生，玩家认知会与真实关系脱节。

3. **秘密暴露后无法追踪"玩家知道了什么"** — 当一个DesireTag从`hidden: true`变为`hidden: false`，需要立即更新多个PERCEIVED条目。如果没有集中管理的暴露机制，这些更新会散落各处。

**为什么必须现在决定**: 这是Foundation层数据架构——每个Core/Feature模块都依赖它。在Agent开始读关系值或UI开始显示关系面板之前，必须明确定义REAL和PERCEIVED的边界。

### Constraints
- Godot 4.6 / GDScript / Web导出平台
- 3个角色 × 4个关系维度 × 2层 = 24个有向值对 — 数据量极小
- Web WASM内存限制（~500MB有效，数据层<70KB — 可忽略）
- body_response永远不可见数值 — 这是Pillar 2的架构约束，不能靠开发者纪律保证
- 所有数据变更必须可追溯（不可变事件日志）
- 角色数据文件损坏时必须有fallback路径

### Requirements
- 双层关系数据结构：REAL (AI行为基础) + PERCEIVED (玩家所见)
- 4维度有向关系: affection, trust, desire, body_response
- body_response的PERCEIVED永远返回"???" — 架构强制执行
- 信息可见性按游戏状态动态切换（OBSERVATION→模糊, DISCOVERY→精确, CONFRONTATION→targeted, INTIMACY→全维度但body仍仅文字）
- 角色记忆列表(MemoryLog) + 事件日志(EventLog) — 不可变追加
- 秘密暴露机制：PERCEIVED在暴露时从"???"更新
- 存档完整序列化/反序列化（Web `user://` → IndexedDB）

## Decision

**使用嵌套Dictionary结构存储REAL和PERCEIVED关系数据。双层物理分离。PERCEIVED由REAL经过状态驱动的可见性过滤器自动派生——外部模块不直接写入PERCEIVED。`apply_delta()` 是唯一的REAL值写入入口。**

### 为什么选择Dictionary而非Resource子类

此决策经过对Godot `Resource`子类的明确评估后做出：

| 维度 | Dictionary (chosen) | Resource subclass |
|------|-------------------|-------------------|
| **序列化** | `JSON.stringify()` → 可读、可调试、可直接写入Web `user://` | `ResourceSaver.save()` → `.tres`格式，Web导出路径复杂 |
| **动态维度** | 新增维度只需加key — 无代码变更 | 需修改`@export`属性 + 重编译 |
| **调试** | `print(my_dict)` 直接可读 | 需遍历属性 |
| **编译期安全** | 无 — 依赖运行时验证 | `@export`类型安全 |

4个维度(affection/trust/desire/body_response)在MVP中固定，但完整版计划扩展更多维度。Dictionary的灵活性支持完整版在不重写数据结构的情况下添加维度。对于MVP的3角色规模，编译期类型安全的价值不足以抵偿JSON序列化的简单性优势。

**回退路径**: 如果扩展到20+角色且有预制角色配置需求，可迁移到`Resource`子类 —— `Dictionary`→`Resource`的转换是机械的（key→`@export`属性）。

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│              RelationshipInfoManager (Autoload)               │
│                                                              │
│  ┌──────────────────────┐    ┌──────────────────────────┐    │
│  │   REAL Layer         │    │   PERCEIVED Layer        │    │
│  │   (写: Agent+Player)  │───▶│   (只读: UI+PlayerInterv) │    │
│  │                      │过滤│                          │    │
│  │ victim→hw:{          │    │ victim→hw:{              │    │
│  │   affection: 0.2     │    │   affection: "低"        │    │
│  │   trust: 0.1         │    │   trust: "低"            │    │
│  │   desire: 0.6        │    │   desire: "???"          │    │
│  │   body_response:0.4  │    │   body_response: "???"   │    │
│  │ }                    │    │ }                        │    │
│  └──────────────────────┘    └──────────────────────────┘    │
│                                                              │
│  ┌──────────────────────┐  ┌────────────────────────────┐    │
│  │  MemoryLog           │  │  EventLog (不可变追加)       │    │
│  │  (per-character)     │  │  [_event_log.append(entry)] │    │
│  └──────────────────────┘  └────────────────────────────┘    │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  Visibility Matrix (state-driven)                    │    │
│  │  OBSERVATION: aff/trust=fuzzy, desire/body=hidden    │    │
│  │  DISCOVERY: aff/trust=precise, desire=fuzzy          │    │
│  │  CONFRONTATION: targeted full (body=never)            │    │
│  │  INTIMACY: participants full (body=never+text hint)   │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Key Interfaces

```gdscript
# === REAL Layer ===

# 有向关系查询
func get_real_relationship(source_id: String, target_id: String, dimension: String) -> float
# dimension: "affection" | "trust" | "desire" | "body_response"

# 批量查询（供Agent+LLM上下文)
func get_all_real_relationships(character_id: String) -> Dictionary

# 唯一的REAL值写入入口
func apply_delta(source_id: String, target_id: String, dimension: String, delta: float) -> void
# delta范围: -0.30 to +0.30
# 内部: new = clamp(current_real + delta, 0.0, 1.0) → 更新REAL → 运行可见性过滤器 → 更新PERCEIVED → 写入EventLog → 发射信号
# 拒绝: delta超出范围 → push_error + return

signal relationship_changed(source_id: String, target_id: String, dimension: String, new_real_value: float, delta: float)

# === PERCEIVED Layer ===

func get_perceived_relationships(from_character_id: String) -> Dictionary
# 返回: { "cuckold_01": {affection: "高", trust: "中", desire: "???", body_response: "???"}, ... }
# body_response: 永远返回"???" 或触发body_hint标记

# 身体暗示检查
func should_emit_body_hint(character_id: String, target_id: String) -> bool
# body_response REAL ≥ 0.3 且 cooldown 已过 → true → LLM Prompt收到标记

# === Memory & Event Log ===

func get_memories(character_id: String, n: int = 5) -> Array[Dictionary]
func add_memory(character_id: String, entry: Dictionary) -> void
# entry: { event_type, description, involved_characters, day, emotional_impact, timestamp }

func get_event_log(day: int = -1) -> Array[Dictionary]
func _append_event(entry: Dictionary) -> void  # 不可变追加 — 封装强制执行
# _event_log仅供内部追加。外部模块通过get_event_log()只读访问

signal secret_discovered(secret: Dictionary)  # { tag, target_id, exposed_by }

# === Visibility Control ===

func _apply_visibility_rules(state: GameState) -> void
# 由 state_changed 信号驱动 — 不在外部调用
# 遍历所有PERCEIVED条目，按当前状态的可见性类型更新

func check_secret_exposure(character_id: String) -> void
# expose_chance = base_probability + player_investigate_bonus + threshold_bonus
# 成功 → secret_discovered 信号发射 → 相关DesireTag hidden→false → PERCEIVED更新

# === Serialization ===

func serialize() -> String
# JSON.stringify({real: _real, perceived: _perceived, memories: _memory_logs, events: _event_log})

func deserialize(json_string: String) -> void
# JSON.parse_string → 重建所有内部结构 → 验证必需键 → 缺失键回退至role默认值
# 失败时: push_error + 使用fallback数据 + 不崩溃
```

**内部存储**:
```gdscript
var _real_relationships: Dictionary = {
    "victim_01": {
        "cuckold_01": {"affection": 0.7, "trust": 0.5, "desire": 0.3, "body_response": 0.1},
        "home_wrecker_01": {"affection": 0.2, "trust": 0.1, "desire": 0.4, "body_response": 0.35}
    }
}

var _perceived_relationships: Dictionary = {
    "victim_01": {
        "cuckold_01": {
            "affection": {"value": 0.7, "visibility": "precise"},
            "trust": {"value": 0.5, "visibility": "fuzzy"},
            "desire": {"value": 0.3, "visibility": "hidden"},
            "body_response": {"value": 0.1, "visibility": "never"}
        }
    }
}

# 可见性类型:
# "precise" → 返回精确浮点值
# "fuzzy" → 返回 "低"(0–0.3) / "中"(0.3–0.7) / "高"(0.7–1.0)
# "hidden" → 返回 "???"
# "never" → body_response专属 — 永远"???"，但触发文字暗示

var _event_log: Array[Dictionary] = []    # 不可变追加 (封装强制)
var _memory_logs: Dictionary = {}          # character_id → Array[Dictionary]
```

**写入规则（架构强制执行）**:
- `apply_delta()` 是 **唯一** 写入REAL关系值的入口
- 外部模块 **禁止** 直接操作 `_real_relationships` Dictionary
- 外部模块 **禁止** 写入 `_perceived_relationships` — PERCEIVED仅由可见性过滤器更新
- `_event_log` 为只读外部访问 — 追加通过 `_append_event()` 封装强制
- 每次 `apply_delta()` 后自动: (1)更新REAL, (2)运行过滤器更新PERCEIVED, (3)写入EventLog, (4)发射 `relationship_changed` 信号

**body_response特殊规则**:
- 初始值0.0（GDD规定）
- 首次非零body_response → 记录 `first_body_response_event` 到EventLog（"点燃"标记）
- ≥ 0.3 → `should_emit_body_hint()` 返回true
- 冷却: 每(source, target)对每时间块最多1次body_hint
- PERCEIVED中永远标记为 `"never"` visibility — 任何状态均不例外

**Web持久化策略**:
- 存档时: `serialize()` → `JSON.stringify()` → `FileAccess.open("user://save_%d.json" % slot, FileAccess.WRITE)` → 浏览器IndexedDB后端
- 读档时: `FileAccess.open("user://save_%d.json", FileAccess.READ)` → `JSON.parse_string()` → `deserialize()`
- 验证: 反序列化后检查所有必需键存在；缺失键→role默认值回退；角色数据文件损坏→full fallback
- `dict.duplicate(true)` 用于存档预览的内存拷贝（深拷贝嵌套Dictionary）；持久化路径直接使用JSON序列化

## Alternatives Considered

### Alternative 1: 双层Dictionary分离存储 (CHOSEN)
- **Description**: REAL和PERCEIVED为独立的嵌套Dictionary。PERCEIVED通过从REAL运行可见性过滤器自动派生。
- **Pros**: 清晰分离——Agent/LLM永远读REAL, UI永远读PERCEIVED。无法意外泄露body_response。序列化直接（`JSON.stringify()` + Web `user://`）。零额外引擎依赖。新维度只需添加Dictionary key。
- **Cons**: 两倍内存（3角色×4维度×2层 ≈ <1KB — 可忽略）。无编译期类型检查（key拼写错误为运行时bug）。过滤器代码需维护。Dictionary内层无类型约束（`Array[Dictionary]`只保证元素是Dictionary）。
- **Rejection Reason**: N/A — chosen

### Alternative 2: 单层存储+访问控制标记
- **Description**: 一份关系数据，每个维度附加 `visibility` 标记。`get_perceived()` 和 `get_real()` 从同一存储读取，通过标记控制返回。
- **Pros**: 一份拷贝 — 无同步风险。内存减半（已可忽略）。
- **Cons**: 紧密耦合 — UI开发者可能误用 `get_real()` 而非 `get_perceived()`。body_response"永不可见"约束需在每个读取路径显式检查 — 易遗漏。调试时不确定看到的是REAL还是PERCEIVED。
- **Rejection Reason**: body_response硬约束需要物理分离作为安全保证。双层分离使"body_response永不可见"由架构强制执行 — 而非纪律。

### Alternative 3: Resource子类 (typed Godot Resource)
- **Description**: `class_name RelationshipData extends Resource`，`@export var affection: float` 等。REAL和PERCEIVED为独立的Resource实例。
- **Pros**: 编辑器可见和可配置。编译期类型检查。`ResourceSaver` 内置序列化。
- **Cons**: `.tres`/`.res` 序列化格式在Web导出下不如JSON直接；`ResourceSaver.save()` 需要wrapper。动态添加维度需要修改类定义。模板代码多于Dictionary。
- **Rejection Reason**: MVP维度固定但完整版计划扩展——Dictionary灵活性支持在不重写数据结构的情况下添加维度。`JSON.stringify()` 对Web `user://`持久化的简单性优势在当前规模下胜出。如果扩展到20+角色且有预制配置需求，可迁移到Resource。

## Consequences

### Positive
- body_response"永不可见"由架构强制执行 — 不存在"意外泄露数值"的代码路径
- REAL和PERCEIVED物理分离 — 调试时可明确知道读取的版本
- `JSON.stringify()`/`JSON.parse_string()` 序列化直接 — Web `user://`文件 = 浏览器IndexedDB
- 可见性过滤器集中在一处 — `_apply_visibility_rules(state)` 是唯一的PERCEIVED更新路径
- 不可变EventLog追加通过 `_append_event()` 封装强制执行 — 无外部直接push

### Negative
- 两倍内存 — 3角色 × 4维 × 2层 ≈ 可忽略 (<1KB)
- 无编译期类型检查 — Dictionary key拼写错误为运行时bug（缓解: 反序列化时验证所有必需键）
- PERCEIVED不能直接写入 — 外部模块不能"设置玩家认知"，只能通过改变REAL值间接影响
- 每角色-角色对的PERCEIVED是在查询时计算的 — 非全局缓存。CONFRONTATION中targeted角色看到更多，第三方保持OBSERVATION级别（但本MVP中玩家只有一个视角——此风险在多人扩展中才出现）

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| PERCEIVED与REAL差距过大 — 玩家感觉被系统欺骗 | Low | body_response≥0.3触发body_hint — 即使数值不可见，文字暗示让玩家感知"有事发生"。GDD设计意图 |
| 嵌套Dictionary缩放到20+角色时内存线性增长 | Low | MVP 3角色。迁移路径: `_real[N][M]` → `Resource` 子类保留相同API签名 |
| Dictionary key拼写错误产生运行时bug | Low | `deserialize()` 启动时验证所有必需键。缺失键→fallback默认值。`apply_delta()` 验证dimension参数在合法集合中 |
| `_event_log` 不可变追加在GDScript中无法语言级别强制 | Low | 封装强制: `_event_log` 外部只读, `_append_event()` 为唯一写入路径。代码审查检查点 |
| Web标签页失效期间IndexedDB写入中断 | Low | 自动存档在TRANSITION状态触发 — 每个游戏天写入一次。写入失败→记录错误，不阻断游戏 |
| `dict.duplicate(true)` 在Godot 4.6嵌套float值上的深拷贝行为未验证 | Low | 持久化路径使用JSON序列化(不依赖duplicate)。`duplicate()`仅用于内存拷贝(存档预览)。测试验证 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| relationship-info-management | Core Rules, Rule 1 | 双层关系数据结构: REAL + PERCEIVED | 分离的 `_real_relationships` 和 `_perceived_relationships` Dictionary |
| relationship-info-management | Core Rules, Rule 2 | 4维度关系变更流程: 事件→delta→REAL→可见判定→PERCEIVED→日志 | `apply_delta()` 封装完整5步流程 — 单一入口 |
| relationship-info-management | Core Rules, Rule 3 | 信息可见性矩阵（按状态）: OBS/DISC/CONF/INTIM各有可见性规则 | `_apply_visibility_rules(state)` — 4种visibility类型映射 |
| relationship-info-management | Core Rules, Rule 4 | 角色记忆系统 MemoryEntry{event_type, description, ...} | `get_memories()` / `add_memory()` — per-character Array[Dictionary] |
| relationship-info-management | Core Rules, Rule 5 | 秘密发现: expose_chance = base_prob + investigate_bonus + threshold_bonus | `check_secret_exposure()` + `secret_discovered`信号 |
| relationship-info-management | Core Rules, Rule 6 | 事件日志 EventEntry{type, data, day, block, player_visible} | `_event_log: Array[Dictionary]` — 不可变追加(封装强制) |
| character-data-model | Core Rules, Block C | 4维度: affection/trust/desire/body_response, 不对称有向 | 嵌套Dictionary: `_real[source][target][dim]` |
| character-data-model | Formulas | body_response初始=0.0, 独立于affection | `apply_delta()` 支持所有4维度。body_response PERCEIVED固定为"never" |
| character-data-model | Edge Cases | 角色数据文件损坏→role默认值fallback | `deserialize()` 验证+缺失键回退 |
| game-state-machine | Detailed Design, Rule 4 | 状态→可见性: OBSERVATION模糊/DISCOVERY精确/CONFRONTATION targeted/INTIMACY全维度(body除外) | `_apply_visibility_rules()` 按状态切换visibility类型 |

## Performance Implications
- **CPU**: `apply_delta()` ~0.02ms（clamp + 过滤器 + 日志追加 + 信号发射）。每时间块~6-9次行动 × 0.02ms = <0.2ms/块 — 可忽略
- **Memory**: REAL~24 floats, PERCEIVED~24值+标记, EventLog~200条×200B=40KB, MemoryLog~90条×300B=27KB. **总计<70KB**
- **Load Time**: JSON反序列化~200条日志 < 1ms。存档加载 <5ms — 可忽略
- **Network**: N/A — 纯本地数据。LLM Prompt注入的数据由LLM客户端模块管理(ADR-0005)

## Migration Plan
N/A — 首个数据层ADR，无现有代码需要迁移。

## Validation Criteria
- [ ] `apply_delta("victim", "hw", "body_response", 0.15)` → REAL body_response=0.15, PERCEIVED body_response="???"
- [ ] `get_perceived_relationships("victim")` 在OBSERVATION状态 → affection/trust返回"低"/"中"/"高", desire/body_response="???"
- [ ] `get_perceived_relationships("victim")` 在DISCOVERY状态 → affection/trust返回精确数值, desire返回"低"/"中"/"高", body_response="???"
- [ ] body_response从0.0首次变为非零 → EventLog记录 `first_body_response_event`
- [ ] body_response≥0.3 → `should_emit_body_hint()` 返回true
- [ ] `check_secret_exposure()` 成功 → `secret_discovered`发射 → DesireTag.hidden→false → PERCEIVED更新
- [ ] `apply_delta()` delta=0.50 → 拒绝(push_error), 关系值不变
- [ ] 序列化→`JSON.stringify()`→`JSON.parse_string()`→反序列化 → 所有值完全一致
- [ ] 角色数据文件损坏 → `deserialize()` 使用role默认值fallback, 游戏不崩溃
- [ ] `dict.duplicate(true)` 深拷贝嵌套Dictionary → 修改拷贝不影响原始REAL值

## Related Decisions
- ADR-0001: `state_changed` 信号驱动 `_apply_visibility_rules()`
- ADR-0003: Agent使用 `get_perceived_relationships()` 做决策（非REAL值）
- ADR-0005: LLM Prompt注入使用 `get_all_real_relationships()` + `should_emit_body_hint()`
- ADR-0007: 玩家介入选项使用 `get_perceived_relationships()` 生成
