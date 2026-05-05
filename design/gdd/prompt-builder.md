# 叙事Prompt构建器 (Prompt Builder)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Implements Pillar**: 1(信息差叙事), 2(目标驱动), 3(视角即玩法)
> **Strategy**: ✨ 新建

## Overview

为三个Agent构建第一人称叙事prompt。每个prompt注入角色档案、事件链、当前状态、素材库上下文和位置信息。替代模拟器的通用决策prompt（ai.txt），改为角色专用的叙事型prompt。

## Player Fantasy

无。

## Detailed Design

### Core Rules

**三个Prompt模板**:

| Agent | 模板 | 关键差异 |
|-------|------|---------|
| 依伊(partner) | `build_partner_prompt()` | 注入严依伊外显档案、内心矛盾、身体感受 |
| 陈深(bull) | `build_bull_prompt()` | 策略思维、观察依伊状态、位置引力 |
| 林晨(cuckold) | `build_cuckold_prompt()` | 朴素日常、隐约不安、不知道出轨 |

**每个Prompt包含的区块**:

1. 创作说明(SN) — 专业情色文学作家身份
2. 角色档案 — 从角色状态读取，依伊额外注入外显档案
3. 目标(lt_goal/st_goal)
4. 当前状态 — 时间、位置、参数值
5. 其他角色位置 — 知道对方在哪（用于位置引力）
6. 事件历史 — 最近N条结构化事件（全量）
7. 素材库上下文 — S1的 `get_all_context(stage)` 输出
8. JSON输出格式 — 每个Agent有不同的输出schema

**输出**：字符串，直接传给S5 LLM客户端。

### Interactions

← **S1 素材库**: `get_all_context(stage)`
← **S2 角色模型**: 读取所有参数+目标+事件链
← **S4 时间系统**: 读取当前时间
← **S8 碰撞检测**: 读取其他角色位置
→ **S5 LLM客户端**: 输出构建好的prompt字符串

## Dependencies

| 依赖 | 类型 |
|------|------|
| S1 素材库 | 硬 |
| S2 角色模型 | 硬 |
| S4 时间系统 | 软（读取但不写入） |
| S12 事件链 | 硬（格式化历史） |

## Acceptance Criteria

1. **GIVEN** READY状态角色+素材库, **WHEN** build_partner_prompt(), **THEN** 返回包含外显档案+事件链+素材+JSON schema的prompt字符串
2. **GIVEN** 三个角色, **WHEN** 分别构建prompt, **THEN** 三个prompt注入不同角色档案和不同事件链
3. **GIVEN** DEGRADED素材库, **WHEN** 构建prompt, **THEN** prompt包含降级提示文本，系统不crash

## Open Questions

- 事件链截断策略：当前全量注入。token过多时是否需要只保留最近N条？（暂搁置，观察token消耗）