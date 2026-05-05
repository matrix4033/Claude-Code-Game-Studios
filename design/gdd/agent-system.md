# Agent决策系统 (Agent System)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Strategy**: 🔧 改造 `src/classes/ai.py`

## Overview

三Agent协同决策系统。替代模拟器的LLMAI._decide()通用NPC决策。交互模式：陈深先行动→依伊看到后回应，林晨并行执行。

## Detailed Design

**每回合执行顺序**:
1. 并行构建三个prompt(S6)
2. 并行调用LLM(S5) — 三个Agent同时决策
3. 解析返回JSON(S10)
4. 更新目标和位置(S7+S8)
5. 应用delta(S3)

**顺序互动**（v7验证）：陈深先手→依伊的prompt中包含陈深本轮行动，她的回应有直接针对性。

**输出解析**: 从LLM返回的文本中提取JSON。处理markdown包裹、括号不完整等情况。

## Interactions

→ S3 Delta表 ← S5 LLM ← S6 Prompt → S7 目标 → S8 碰撞 → S9 引力

## Acceptance Criteria

1. **GIVEN** 三个角色状态, **WHEN** 执行一回合, **THEN** 三个Agent各自输出合法JSON
2. **GIVEN** 陈深输出", **WHEN** 依伊prompt构建, **THEN** 包含陈深本轮对话