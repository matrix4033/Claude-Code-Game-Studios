# 目标→位置引力 (Position Gravity)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Implements Pillar**: 2 — 目标驱动 | **Strategy**: ✨ 新建

## Overview

当Agent的短期目标涉及其他角色时，在prompt中展示对方当前位置，促使Agent主动移动。解决demo v10中"碰撞率0%"问题（v11修复后碰撞率80%）。

## Detailed Design

**实现方式**: Prompt中注入"其他角色位置"行。不修改代码逻辑——LLM自主决定是否移动。

```
【当前状态】你在教室。
依伊在咖啡厅——她离你多远？你需要接近她。
陈深在图书馆。
```

## Interactions

← S2 角色模型 ← S7 目标链 → S6 Prompt构建器

## Acceptance Criteria

1. **GIVEN** Agent的st_goal涉及另一角色, **WHEN** 构建prompt, **THEN** 包含对方当前位置
2. **GIVEN** 碰撞率, **WHEN** 多回合运行, **THEN** 碰撞率>50%
