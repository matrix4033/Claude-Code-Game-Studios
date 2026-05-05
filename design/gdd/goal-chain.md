# 短期目标链 (Goal Chain)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Implements Pillar**: 2 — 目标驱动 | **Strategy**: 🔧 改造 `src/classes/long_term_objective.py`

## Overview

将模拟器的长期目标系统（5-10年维度）改为每回合可更新的短期目标链。LLM在每次输出中自检goal_done，达成后自动推进到下一目标。

## Detailed Design

**核心逻辑**: `update_goals(char, llm_output)` → 读取llm_output中的 `goal_done` 和 `new_st_goal` → 如果goal_done=true且有new_st_goal，写入新st_goal并将goal_done重置为false。如果goal_done=true但无new_st_goal，保持goal_done=true（下回合LLM会重试）。

**lt_goal更新**: LLM可输出new_lt_goal。通常不更新——只在叙事重大转折时变化。

## Interactions

← S2 角色模型 → S10 Agent系统

## Acceptance Criteria

1. **GIVEN** goal_done=true, new_st_goal="约她喝咖啡", **WHEN** update_goals(), **THEN** st_goal更新, goal_done=false
2. **GIVEN** goal_done=true, 无new_st_goal, **WHEN** update_goals(), **THEN** st_goal不变, goal_done保持true
