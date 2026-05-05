# ADR-0002: 叙事Prompt替代通用决策Prompt

## Status
Proposed

## Date
2026-05-05

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Python/FastAPI (DeepSeek Anthropic API) |
| **Domain** | Scripting |
| **Knowledge Risk** | LOW |
| **References Consulted** | `static/locales/zh-CN/templates/ai.txt`, demo v1-v12 |
| **Verification Required** | v12 three-agent prompt templates validated |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (回合制) |
| **Enables** | S6 PromptBuilder, S10 AgentSystem |
| **Blocks** | None |

## Context

模拟器的`ai.txt`是通用NPC决策prompt——LLM输出"下一步行动链"。恋情模块需要第一人称叙事prompt——LLM输出"内心独白+对话+身体反应"。11轮demo验证了叙事型prompt的可行性。

## Decision

用三个角色专用的第一人称叙事prompt模板替代模拟器的通用`ai.txt`。每个模板注入角色档案、目标、事件链和素材库上下文。

## GDD Requirements Addressed

| GDD | Requirement | How |
|-----|-------------|-----|
| S6 prompt-builder | 三个Agent各有独立的第一人称prompt模板 | 本ADR确认S6的架构方向 |

## Validation Criteria
- 三个Agent的prompt输出风格区分明显（v12已验证）
- Agent能从素材库中选择场景/体位/道具
