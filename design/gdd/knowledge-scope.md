# 认知隔离引擎 (Knowledge Scope)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Implements Pillar**: 1 — 信息差叙事 | **Strategy**: 🔧 改造 `src/romance/models/event.py`

## Overview

按角色认知范围过滤事件。同一事件：苦主只知道"她说加班"，伴侣知道"实际在体育馆"。KnowledgeScope已定义但未接入主循环。

## Detailed Design

**事件分发**: 每个事件标注visibility（仅自己可见/对指定角色可见/全可见）。EventPanel按当前玩家视角角色过滤。

**信息差产生**: Prompt中只注入角色认知范围内的事件。苦主的prompt不含出轨细节；伴侣的prompt不含苦主的内心挣扎。

## Interactions

← S10 Agent系统 ← S12 事件链 → S14 EventPanel

## Acceptance Criteria

1. **GIVEN** 依伊和陈深私下见面事件, **WHEN** 苦主读取事件链, **THEN** 不包含该事件
2. **GIVEN** 切换视角到伴侣, **WHEN** EventPanel刷新, **THEN** 显示此前隐藏的事件
