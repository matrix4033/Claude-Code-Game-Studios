# ADR-0003: 短期目标链替代长期目标

## Status
Proposed

## Date
2026-05-05

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (回合制) |
| **Enables** | S7 GoalChain |
| **Blocks** | None |

## Context

模拟器`long_term_objective.py`是5-10年维度的长期目标。恋情模块需要在每回合维度驱动叙事——LLM自设st_goal，达成后推进到下一个。

## Decision

用lt_goal/st_goal/goal_done三级目标状态替代模拟器的单一长期目标。LLM自设自更新，代码只读不写。

## GDD Requirements Addressed

| GDD | Requirement | How |
|-----|-------------|-----|
| S7 goal-chain | 短期目标每回合可更新，达成自动推进 | 本ADR确认S7架构 |

## Validation Criteria
- 10回合内目标链自驱演进（v10已验证）
- goal_done=true时有new_st_goal
