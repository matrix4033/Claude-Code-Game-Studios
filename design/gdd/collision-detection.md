# 位置碰撞检测 (Collision Detection)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Strategy**: 🔧 改造 `src/classes/core/avatar/core.py` (pos_x/pos_y → 命名位置)

## Overview

将模拟器的xy坐标碰撞改为命名位置碰撞。每回合检查三个角色是否在同一地点，标记碰撞类型。

## Detailed Design

**碰撞规则**: 逐对比较 `pos[role1] == pos[role2]`。

| 碰撞 | 含义 |
|------|------|
| partner==bull | 依伊和陈深同处——可能触发互动 |
| partner==cuckold | 依伊和林晨同处——日常 |
| bull==cuckold | 陈深和林晨同处——罕见 |

**位置来源**: S1素材库中的locations.yaml(8约会地点)+校园地点(6个)。

## Interactions

← S2 角色模型(位置) → S10 Agent系统

## Acceptance Criteria

1. **GIVEN** 依伊和陈深都在"咖啡厅", **WHEN** check_collision(), **THEN** 返回["partner_bull"]
2. **GIVEN** 三人都在"教室", **WHEN** check_collision(), **THEN** 返回["partner_bull","partner_cuckold","bull_cuckold"]
