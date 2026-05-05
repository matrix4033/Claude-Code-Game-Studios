# 日周期时间系统 (Day Cycle)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Implements Pillar**: 2 — 目标驱动
> **Strategy**: 🔧 改造 `src/systems/time.py`

## Overview

将模拟器的MonthStamp月步进改为5时段日循环。系统自动推进时间（清晨→上午→午后→傍晚→深夜→清晨...），每回合一个时间块。LLM不管理时间，只读取当前时段作为叙事上下文。

## Player Fantasy

时间的连续流动给玩家"这一天在真实发生"的感觉。清晨到深夜的自然过渡创造叙事节奏——日常在白天、秘密在深夜。

## Detailed Design

### Core Rules

**时间块**: 5个固定时段，系统按顺序循环。

| 索引 | 时段 | 氛围暗示 |
|------|------|---------|
| 0 | 清晨 | 新的一天开始，校园苏醒 |
| 1 | 上午 | 上课、工作、正常社交 |
| 2 | 午后 | 自由时间、约会窗口 |
| 3 | 傍晚 | 暧昧时段、酒吧/咖啡厅 |
| 4 | 深夜 | 秘密、私密空间、风险场景 |

**推进逻辑**: `advance()` — bi+1；bi==5时bi=0, day+1。每回合结束后调用。

**查询**: `current()` → (day, time_block)。Agent prompt中使用 `f"第{day}天 · {time_block}"`。

**存档**: 序列化 (day, bi)。

### States

无复杂状态。只有(day, bi)两个整数。

### Interactions

→ **S6 Prompt构建器**: 读取当前时间作为叙事上下文
→ **S10 Agent系统**: 回合结束后调用advance()
→ **S12 事件链**: 事件条目记录时间戳
→ **S17 存档**: 序列化(day, bi)

## Formulas

无公式。

## Edge Cases

- **bi溢出**: advance()中bi+1后若==5则归零，day+1。防御性：bi>5时强制归零。
- **day溢出**: 无上限。展示层超过阈值时可显示"第N天"。

## Dependencies

零系统依赖。

## Tuning Knobs

| Knob | 默认 | 说明 |
|------|------|------|
| blocks_per_day | 5 | 时段数。改此值需同步更新TIME_BLOCKS列表 |
| start_day | 1 | 起始天数 |
| start_block | 1(上午) | 起始时段 |

## Acceptance Criteria

1. **GIVEN** bi=0(清晨), **WHEN** advance(), **THEN** bi=1(上午), day不变
2. **GIVEN** bi=4(深夜), **WHEN** advance(), **THEN** bi=0, day+1
3. **GIVEN** day=1, bi=1, **WHEN** 序列化再反序列化, **THEN** day=1, bi=1
4. **GIVEN** 任意状态, **WHEN** current(), **THEN** 返回 (day, time_block_name)

## Open Questions

- 无