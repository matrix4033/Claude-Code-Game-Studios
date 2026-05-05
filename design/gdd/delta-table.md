# Delta映射表 (Delta Table)

> **Status**: In Design
> **Last Updated**: 2026-05-05
> **Implements Pillar**: 4 — 每个选择留下痕迹
> **Strategy**: ✨ 新建

## Overview

Delta映射表定义事件类型到5维恋情属性变化的静态映射关系。Agent输出event_category，代码查表计算delta，应用到角色状态。是"每个选择留下痕迹"（Pillar 4）的数值基础。

## Player Fantasy

玩家不会直接感知Delta数值。但每次介入后参数的可见变化（信任下降、服从上升）是玩家获得"我的选择真的改变了什么"反馈的核心来源。

## Detailed Design

### Core Rules

**查表逻辑**：`apply_delta(char, event_category)` → 读取DELTA表中对应行的[love, attraction, submission, physical_memory]偏移量 → 加法应用到角色参数 → 统一clamp到[0.0, 1.0]。

**Delta表**：

| event_category | love | attraction | submission | physical_memory | 额外效果 |
|---------------|------|------------|------------|-----------------|---------|
| sex_event | -0.06 | +0.08 | +0.06 | +0.10 | — |
| training_session | -0.04 | +0.06 | +0.07 | +0.05 | — |
| bull_force_push | -0.08 | +0.05 | +0.07 | +0.08 | — |
| bull_verbal_seduce | -0.02 | +0.04 | +0.03 | +0.02 | — |
| partner_active_cheat | -0.10 | +0.08 | +0.06 | +0.10 | — |
| partner_passive_yield | -0.05 | +0.04 | +0.05 | +0.06 | — |
| partner_resist_fail | -0.02 | +0.02 | +0.03 | +0.03 | — |
| partner_resist_success | +0.02 | 0 | -0.02 | 0 | — |
| partner_lies | -0.04 | +0.02 | 0 | 0 | awareness+0.02 |
| cuckold_discovers_clue | 0 | 0 | 0 | 0 | awareness+0.10, trust-0.08 |
| player_intervention | +0.03 | -0.02 | -0.01 | 0 | — |
| daily_idle | 0 | 0 | 0 | 0 | — |

**应用后处理**：所有值clamp到[0.0, 1.0]。arousal在每回合结算后自动衰减0.05。

### States

无状态机。静态表。可热更新（修改表值后下一回合生效）。

### Interactions

→ **S2 角色模型**: 写入 love/attraction/submission/physical_memory/arousal/awareness/trust
← **S10 Agent系统**: 读取Agent输出的event_category

## Formulas

无公式。加法查表。

## Edge Cases

- **event_category不在表中**: 等效于daily_idle，参数不变。日志警告。
- **多次叠加后溢出**: clamp到[0.0, 1.0]。不报错。
- **同一回合多个事件**: 按出现顺序依次应用，每次应用后独立clamp。

## Dependencies

| 依赖 | 方向 | 类型 |
|------|------|------|
| S2 角色模型 | 写入 | 硬 |

## Tuning Knobs

表中每个delta值都是tuning knob。通过修改表值调整游戏节奏。

| Knob | 默认值 | 调高效果 | 调低效果 |
|------|--------|---------|---------|
| sex_event.love | -0.06 | 出轨后爱消散更快 | 慢 |
| sex_event.submission | +0.06 | 服从上升快 | 慢 |
| player_intervention.love | +0.03 | 介入挽回效果强 | 弱 |

## Acceptance Criteria

1. **GIVEN** event_category="sex_event"，**WHEN** 调用apply_delta，**THEN** love=原值-0.06, attraction=原值+0.08, submission=原值+0.06, physical_memory=原值+0.10
2. **GIVEN** love=0.02且delta=-0.10，**WHEN** 应用delta，**THEN** love=0.0（clamp不溢出）
3. **GIVEN** event_category="unknown_event"，**WHEN** 调用apply_delta，**THEN** 参数不变，日志含警告
4. **GIVEN** 同一回合["sex_event","training_session"]，**WHEN** 依次应用，**THEN** 两次delta独立叠加后clamp
5. **GIVEN** cuckold_discovers_clue，**WHEN** 应用到苦主，**THEN** awareness+0.10, trust-0.08

## Open Questions

- 无