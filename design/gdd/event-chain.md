# 事件链记忆系统 (Event Chain)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Strategy**: ✨ 新建

## Overview

每个角色维护独立的结构化事件链。每回合结束后追加新事件条目。Agent的prompt中注入全量事件链（不截断），确保决策连贯性。

## Detailed Design

**事件条目结构**: round, time, location, actions(每个角色的行动+对话), emotional(tags), goal_update, player_intervention。

**存储**: 每角色 `events: list[dict]`。三个角色的事件链长度始终相同（同回合追加），但内容不同（认知隔离）。

**全量注入**: 不截断。10回合≈5000字事件链，远在DeepSeek 64K上下文内。

## Interactions

← S10 Agent(每回合输出) → S6 Prompt(注入历史) → S11 认知隔离(过滤)

## Acceptance Criteria

1. **GIVEN** 回合结束后, **WHEN** 追加事件, **THEN** 三角色链长度+1
2. **GIVEN** 10回合后, **WHEN** 构建prompt, **THEN** 包含完整10条事件历史
