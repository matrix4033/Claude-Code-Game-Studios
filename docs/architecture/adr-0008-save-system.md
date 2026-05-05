# ADR-0008: 存档序列化策略

## Status
Accepted

## Date
2026-05-02

> **TD-ADR Review**: APPROVED — JSON + FileAccess save architecture confirmed. Note: ADR-0004 autoload table should be updated to include SaveSystem at position 11 (2026-05-02)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | Core (Persistence / FileAccess) |
| **Knowledge Risk** | MEDIUM — `FileAccess.store_*` returns `bool` since Godot 4.4 (was `void`). Verify return check pattern. |
| **References Consulted** | `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/deprecated-apis.md` |
| **Post-Cutoff APIs Used** | `FileAccess.store_string()` returns `bool` (4.4) — must check return value; `duplicate_deep()` (4.5) for Resource deep copy — NOT used here (JSON path) |
| **Verification Required** | (1) `FileAccess.store_string()` return value check on Web export (IndexedDB write); (2) JSON.stringify/parse round-trip integrity for nested Dictionary with float values |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0002 (relationship_values, event_log, memory_logs state ownership), ADR-0003 (day_state — day_number, current_block) |
| **Enables** | Save/load implementation story (Alpha tier) |
| **Blocks** | None — Alpha tier, can be deferred |
| **Ordering Note** | Final ADR. Alpha tier — MVP does not require save functionality (single 30-60 min session) |

## Context

### Problem Statement
「浮世」的游戏状态（角色关系、记忆、事件、天数、游戏状态）需要在中断时保存和恢复。存档系统是Alpha tier——MVP不阻塞此功能，但数据结构必须设计为可序列化（架构原则5: MVP不阻碍完整版路径）。Godot 4.4变更了`FileAccess.store_*`方法——从`void`变为返回`bool`的失败指示——现有代码如果忽略返回值可能静默写入失败。

### Constraints
- MVP: 单档位自动存档（TRANSITION状态触发）
- Web导出: `user://` → 浏览器IndexedDB（~50MB-1GB配额）
- JSON格式: 可读、可调试、Web友好
- Godot 4.4+: `FileAccess.store_string()` 返回`bool` — 必须检查

### Requirements
- 序列化: REAL值、PERCEIVED值、secret_desires、记忆列表、事件日志
- 反序列化: 验证所有必需键，缺失→回退
- 自动存档: TRANSITION状态进入时触发
- 写入失败: 记录错误，不阻断游戏

## Decision

**JSON序列化格式 + `FileAccess`写入`user://save_%d.json`。自动存档在TRANSITION状态触发。写入失败静默记录——不阻断游戏。MVP单档位。**

### Key Interfaces

```gdscript
# === SaveSystem (Alpha Autoload) ===

func save_game(slot: int = 0) -> bool:
    var data := {
        "version": 1,
        "timestamp": Time.get_unix_time_from_system(),
        "game_state": GameStateMachine.get_current_state(),
        "day_cycle": {
            "day_number": DayCycleManager.day_number,
            "current_block": DayCycleManager.current_block,
            "block_time": DayCycleManager.block_time
        },
        "relationships": RelationshipInfoManager.serialize()  # ADR-0002 serialize()
    }
    var json := JSON.stringify(data, "\t")
    var file := FileAccess.open("user://save_%d.json" % slot, FileAccess.WRITE)
    if file == null: return false
    # Godot 4.4+: store_string returns bool
    var ok := file.store_string(json)
    file.close()
    if not ok:
        push_error("SaveSystem: write failed for slot %d" % slot)
    return ok

func load_game(slot: int = 0) -> bool:
    if not FileAccess.file_exists("user://save_%d.json" % slot):
        return false
    var file := FileAccess.open("user://save_%d.json" % slot, FileAccess.READ)
    if file == null: return false
    var json := file.get_as_text()
    file.close()
    var data := JSON.parse_string(json)
    if data == null: return false
    # Validate required keys
    if not data.has_all(["version", "game_state", "day_cycle", "relationships"]):
        push_error("SaveSystem: corrupt save data, missing required keys")
        return false
    # Restore state
    DayCycleManager.restore(data["day_cycle"])
    RelationshipInfoManager.deserialize(data["relationships"])
    GameStateMachine.request_transition("game_start")  # → OBSERVATION
    return true

# Auto-save trigger
func _on_state_changed(old_state: GameState, new_state: GameState) -> void:
    if new_state == GameState.TRANSITION:
        save_game(0)  # auto-save to slot 0
```

### Web Export Storage

| Path | Backend | Quota | Use |
|------|---------|-------|-----|
| `user://` | IndexedDB | ~50MB-1GB | Save files, config |
| `res://` | preloaded (read-only) | bundled | Character data, prompt templates |

### Fallback

- `FileAccess.open()` 失败 → 返回false，调用方处理（不崩溃）
- `JSON.parse_string()` null → 不恢复状态，重新开始
- 必需键缺失 → `push_error`，重新开始
- 角色数据文件损坏 → ADR-0002的fallback逻辑（role默认值）

## Alternatives Considered

### A: JSON + FileAccess (CHOSEN)
Simple, readable, debuggable. Web export backed by IndexedDB. Human-editable for testing.

### B: ResourceSaver + .tres (REJECTED)
Godot-native Resource serialization. `.tres` format is Godot-specific — not easily inspected. Web export path through ResourceSaver is more complex than direct JSON.

### C: ConfigFile API (REJECTED)
Godot's built-in INI-style. Suitable for settings, not for nested relationship data (4 dimensions × N×N matrix).

## Consequences
- **Positive**: JSON is human-readable — debugging saves is trivial. Web IndexedDB is transparent.
- **Negative**: No compression — full save ~50-70KB per slot. Not a concern for MVP.
- **Risk**: `store_string` return value ignored → silent write failures on Web. Mitigated by explicit return check.

## GDD Requirements Addressed

| GDD | Requirement | How Addressed |
|-----|-------------|---------------|
| relationship-info-management | serialize()/deserialize() for full state | `save_game()` → `RelationshipInfoManager.serialize()` → JSON |
| day-cycle-manager | get_save_state() → {day_number, current_block, block_time} | `day_cycle` key in save dict |
| game-state-machine | TRANSITION auto-save | `_on_state_changed()` trigger |

## Validation Criteria
- [ ] `save_game(0)` → `user://save_0.json` exists, valid JSON
- [ ] `load_game(0)` → all relationships, day state, game state restored
- [ ] Corrupt JSON → `load_game()` returns false, game does not crash
- [ ] Missing key in save → `push_error`, return false
- [ ] `store_string()` returns false → error logged, game continues
- [ ] Auto-save triggers on TRANSITION state entry

## Related Decisions
- ADR-0002: `serialize()`/`deserialize()` methods on RelationshipInfoManager
- ADR-0003: DayCycleManager state serialization (day_number, block, block_time)
- ADR-0004: SaveSystem as final autoload (position 11)
