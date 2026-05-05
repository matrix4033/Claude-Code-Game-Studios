# ADR-0005: 日循环替代月步进

## Status
Proposed

## Date
2026-05-05

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (回合制) |
| **Enables** | S4 DayCycle |
| **Blocks** | None |

## Context

模拟器`src/systems/time.py`以月(`MonthStamp`)为单位步进。恋情模块的叙事需要日级别的粒度——清晨上课、午后约会、深夜秘密。

## Decision

用5时段日循环（清晨→上午→午后→傍晚→深夜）替代月步进。系统每回合自动推进一个时段，LLM不管理时间。

## GDD Requirements Addressed

| GDD | Requirement | How |
|-----|-------------|-----|
| S4 day-cycle | 每回合自动推进一个时间块 | 本ADR确认日循环方案 |

## Validation Criteria
- 5时段循环推进正确（v10已验证连续无跳跃）
- 每回合一个时间块
