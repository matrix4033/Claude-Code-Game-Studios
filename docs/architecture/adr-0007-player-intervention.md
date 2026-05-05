# ADR-0007: 玩家介入系统设计

## Status
Accepted

## Date
2026-05-02

> **TD-ADR Review**: APPROVED — Pure GDScript logic, no engine API concerns (2026-05-02)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | Core (Feature Logic) |
| **Knowledge Risk** | LOW — pure GDScript logic, no engine API dependencies |
| **References Consulted** | None — no engine-specific APIs used |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | None |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (state_changed — interventions only available in DISCOVERY/CONFRONTATION), ADR-0002 (get_perceived_relationships for option generation, apply_delta for action results), ADR-0003 (advance_clock after intervention, notify_player_action to agents), ADR-0004 (autoload position 9) |
| **Enables** | Player-facing intervention UI story |
| **Blocks** | None downstream |
| **Ordering Note** | Feature layer. All Foundation and Core ADRs must be written first. |

## Context

### Problem Statement
玩家只能在DISCOVERY和CONFRONTATION状态中介入——每天最多2次。每个介入选项必须基于玩家已知的信息(PERCEIVED)生成——无法对未知秘密采取行动。每次介入产生不可逆的关系delta和连锁反应(Agent感知→行为调整→新事件)。核心原则: 没有"正确"选择——只有不同方向的后果。

### Constraints
- 仅DISCOVERY和CONFRONTATION状态可用
- 每日2次介入限制
- 选项基于PERCEIVED生成(非REAL)
- 旁观永远是合法选项且不消耗介入次数
- 调查成功率: base 0.3 + info_gap 0-0.3 + suspicion 0-0.2 → max 0.8

### Requirements
- 7种介入行动: investigate, expose, support, sabotage, confront, defuse, observe
- 连锁反应3层: 直接关系delta→Agent感知→叙事连锁
- 挑拨可被识破(双重背叛)
- 连续调查同一角色3次→触发对峙

## Decision

**PlayerIntervention autoload在DISCOVERY/CONFRONTATION状态中基于当前PERCEIVED值生成介入选项。每日2次窗口。每次介入调用apply_delta(写入REAL)和advance_clock(消耗时间)，并通知Agent。**

### Key Interfaces

```gdscript
# === PlayerIntervention (Feature Autoload) ===

func get_available_actions(state: GameState) -> Array[Dictionary]:
    # 返回: [{id, type, description, targets, risk_level, available}]
    # 过滤: 基于get_perceived_relationships() — 不基于REAL
    # 例: PERCEIVED显示desire="???" → 无法选择"expose_desire"行动
    # 每日>2次介入 → 仅返回 [{type: "observe"}]

func execute_action(action_id: String, target_id: String) -> Dictionary:
    # 验证: 介入次数 < 2 && state ∈ {DISCOVERY, CONFRONTATION}
    # 执行: apply_delta() → advance_clock(0.15-0.3) → notify_player_action()
    # 返回: {success, deltas, chain_reactions, intervention_count_remaining}

signal intervention_completed(result: Dictionary)
# { action_type, target, deltas_applied, clock_advanced, agent_notified }

# === Action Types & Delta Ranges ===

const ACTION_DELTAS := {
    "investigate": {
        "success_formula": "base_chance(0.3) + info_gap_bonus(0-0.3) + suspicion_bonus(0-0.2)",
        "on_success": {"target_trust": -0.05, "perceived_update": "one_dimension"},
        "on_fail": "target may notice (+0 suspicion for target)"
    },
    "expose":   {"trust_damage": [-0.10, -0.30], "revealed_to": "receiver"},
    "support":  {"affection": [0.05, 0.15], "trust": [0.05, 0.10], "others_affection": -0.05},
    "sabotage": {"trust_damage": [-0.05, -0.15], "detection_risk": true},
    "confront": {"trust_damage": [-0.10, -0.20], "perceived_update": "bidirectional"},
    "defuse":   {"affection": [0.0, 0.10], "success_inverse_with_physical_intimacy": true},
    "observe":  {"no_deltas": true, "event_proceeds_uninfluenced": true}
}

# Daily limit
var _interventions_today: int = 0
const MAX_INTERVENTIONS_PER_DAY: int = 2

# Agent notification
func _notify_agents(action: String, target: String) -> void:
    # Agents adjust behavior in next decision cycle (ADR-0003)
    MultiAgentOrchestrator.notify_player_action(action, target)
```

### Edge Cases

| Scenario | Behavior |
|----------|----------|
| Player investigates same target 3× consecutively | Target notices → triggers confrontation event |
| Player sabotages but is detected | Both targets' trust toward player drops → "double betrayal" |
| Player supports both sides in confrontation | Both feel dismissed → affection drops for both |
| Interventions exhausted + critical event occurs | "You can no longer intervene. You can only watch." |
| Player observes 3 consecutive days | "Observer ending" route — three characters' fates unfold without player input |

## Alternatives Considered

### A: PERCEIVED-based option generation (CHOSEN)
Options filtered by what the player actually knows. Aligns with Pillar 2.

### B: REAL-based option generation (REJECTED)
Player can act on information they haven't discovered. Violates Pillar 2 — "信息即权力" would be meaningless if all options always visible.

### C: No daily limit (REJECTED)
Unlimited interventions dilute each action's weight. Pillar 3 ("每次选择都有代价") requires scarcity.

## Consequences
- **Positive**: PERCEIVED filtering enforces information asymmetry — player cannot act on secrets they haven't discovered
- **Negative**: 2/day limit means some dramatic moments pass without player agency — by design ("not choosing IS a choice")
- **Risk**: Players may feel frustrated when they want to intervene but can't → mitigated by "observe" always being free and narrative framing

## GDD Requirements Addressed

| GDD | Section | Requirement | How Addressed |
|-----|---------|-------------|---------------|
| player-intervention | Rules 1-3 | 7 action types, PERCEIVED-based, state-gated | `get_available_actions()` + ACTION_DELTAS table |
| player-intervention | Rule 4 | Chain reaction: delta→Agent→narrative | `execute_action()` → apply_delta → notify_player_action |
| player-intervention | Rule 5 | Daily limit + clock advance | `_interventions_today` counter + `advance_clock(0.15-0.3)` |
| player-intervention | Formulas | investigate_success_chance, sabotage_effectiveness | Formula variables from GDD encoded in ACTION_DELTAS |

## Validation Criteria
- [ ] DISCOVERY state → 2-4 intervention options generated (excluding observe)
- [ ] PERCEIVED desire="???" → "expose desire" action NOT available
- [ ] 3rd intervention attempt in one day → rejected, only observe available
- [ ] `execute_action("expose", "hw")` → trust_damage applied, agent notified, clock advanced
- [ ] Sabotage detected → both targets' trust toward player drops
- [ ] Continuous observe for 3 days → observer ending triggered

## Related Decisions
- ADR-0001: state_changed gates intervention availability
- ADR-0002: get_perceived_relationships() for option filtering; apply_delta() for results
- ADR-0003: advance_clock() after intervention; notify_player_action() to agents
