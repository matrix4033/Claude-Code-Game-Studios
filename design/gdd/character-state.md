# 角色数据模型 (Character State)

> **Status**: In Design
> **Last Updated**: 2026-05-05
> **Implements Pillar**: 2 — 目标驱动
> **Strategy**: 🔧 改造 (`src/romance/models/character_state.py`)

## Overview

角色数据模型是所有Agent决策和叙事系统的基础数据结构。定义三个角色的身份信息、5维恋情参数、目标状态和事件历史。改造自 `src/romance/models/character_state.py`，新增 lt_goal/st_goal/goal_done 目标驱动字段。

## Player Fantasy

纯基础设施——玩家不与数据结构交互。但目标字段的加入使Agent能够自主驱动叙事，玩家在视角切换时看到的是目标驱动的角色行为，而非静态NPC。

## Detailed Design

### Core Rules

**角色类型**：

| 角色 | role | 独有参数 |
|------|------|---------|
| 依伊(伴侣) | partner | love, attraction, submission, physical_memory, arousal |
| 陈深(黄毛) | bull | 共享partner的attraction/submission/physical_memory（观察值） |
| 林晨(苦主) | cuckold | awareness(怀疑度), trust(信任度) |

**通用字段**（所有角色）：

| 字段 | 类型 | 范围 | 说明 |
|------|------|------|------|
| name | str | — | 角色名 |
| role | str | partner/bull/cuckold | 角色类型 |
| personality | str | — | 性格描述 |
| background | str | — | 背景故事 |
| appearance | str | — | 外貌描述 |
| lt_goal | str | — | 长期目标（缓慢变化） |
| st_goal | str | — | 短期目标（每回合可更新） |
| goal_done | bool | false | 短期目标是否已达成 |
| events | list[dict] | — | 事件链（全量历史） |

**恋情5维参数**（partner/bull共享观察）：

| 字段 | 类型 | 范围 | 说明 |
|------|------|------|------|
| love | float | 0.0–1.0 | 对伴侣的爱（可双向变化） |
| attraction | float | 0.0–1.0 | 对黄毛的性吸引力 |
| submission | float | 0.0–1.0 | 对黄毛的服从度 |
| physical_memory | float | 0.0–1.0 | 身体记忆（累积型，缓慢衰减） |
| arousal | float | 0.0–1.0 | 当前性兴奋度（每次事件后自然衰减） |

**苦主参数**（仅cuckold）：

| 字段 | 类型 | 范围 | 说明 |
|------|------|------|------|
| awareness | float | 0.0–1.0 | 怀疑度（发现线索时上升） |
| trust | float | 0.0–1.0 | 对伴侣的信任度（与awareness此消彼长） |

### States

目标有三种状态：`goal_done=false`（推进中）、`goal_done=true`（已达成待更新）。由S7目标链系统管理。

参数无显式状态机——所有值是连续的float，通过S3 Delta表修改。

### Interactions

→ **S3 Delta表**: 写入参数变化
→ **S6 Prompt构建器**: 读取所有字段构建Agent prompt
→ **S7 目标链**: 读写 lt_goal/st_goal/goal_done
→ **S8 碰撞检测**: 读取 role 判定碰撞类型
→ **S10 Agent系统**: 读写全部字段
→ **S12 事件链**: 读取 events 构建历史上下文
→ **S17 存档**: 序列化全部字段

## Formulas

无公式。纯数据结构。

## Edge Cases

- **参数溢出**: 所有float字段clamp到[0.0, 1.0]。Delta应用前不做预检查——应用后统一clamp。
- **love=0.0且attraction=1.0且submission=1.0**: 极端状态。合法。表示伴侣已完全沦陷。
- **awareness > trust**: 合法。表示苦主已不信任伴侣。此时他的叙事应该充满怀疑。
- **goal_done=true但new_st_goal为空**: LLM输出异常。代码应保留旧st_goal，goal_done保持true，下一回合LLM会重新处理。

## Dependencies

| 依赖 | 方向 | 类型 |
|------|------|------|
| 无 | — | 纯数据结构，零系统依赖 |

## Tuning Knobs

所有初始参数值。通过角色配置文件（YAML/JSON）设定，无需改代码。

| Knob | 默认范围 | 说明 |
|------|---------|------|
| love初始值 | 0.7–0.95 | 伴侣初始对苦主的爱 |
| attraction初始值 | 0.0–0.1 | 伴侣初始对黄毛的吸引力 |
| submission初始值 | 0.0 | 初始服从度 |
| awareness初始值 | 0.0 | 初始怀疑度 |
| trust初始值 | 0.8–0.95 | 初始信任度 |

## Acceptance Criteria

1. **GIVEN** 新角色实例化，**WHEN** 读取所有字段，**THEN** 所有float在[0.0, 1.0]范围内，str字段非空
2. **GIVEN** 任意float参数，**WHEN** 应用delta后值超出[0.0, 1.0]，**THEN** 自动clamp到边界
3. **GIVEN** PartnerState和BullState，**WHEN** 比较attraction/submission/physical_memory，**THEN** 两者读取相同值（共享观察）
4. **GIVEN** goal_done=true且无new_st_goal，**WHEN** 下一回合开始，**THEN** 保留旧st_goal，系统不crash
5. **GIVEN** 角色实例，**WHEN** 序列化为JSON再反序列化，**THEN** 所有字段值一致

## Open Questions

- 无
