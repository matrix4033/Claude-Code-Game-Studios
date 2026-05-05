# ADR-0004: 命名位置替代xy坐标碰撞

## Status
Proposed

## Date
2026-05-05

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (回合制) |
| **Enables** | S8 Collision, S9 Gravity |
| **Blocks** | None |

## Context

模拟器使用`Avatar.pos_x/pos_y`二维连续坐标+Manhattan距离判断角色距离。恋情模块只有3个角色+14个命名地点，不需要连续坐标系。"同地点"的二元判定就够了。

## Decision

用命名位置（locations.yaml 8点+校园6点）替代xy坐标。碰撞检测改为`pos[role1]==pos[role2]`。

## GDD Requirements Addressed

| GDD | Requirement | How |
|-----|-------------|-----|
| S8 collision | 同地点角色自动标记碰撞 | 本ADR确认命名位置方案 |

## Validation Criteria
- 碰撞检测正确识别同地点角色（v11已验证：碰撞率80%）
- 位置从locations.yaml+校园地点中选择
