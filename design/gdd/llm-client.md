# LLM客户端 (LLM Client)

> **Status**: In Design | **Last Updated**: 2026-05-05
> **Strategy**: 🔄 复用 `src/utils/llm/client.py`

## Overview

复用模拟器LLM客户端。已支持Anthropic Messages API格式、urllib直连、重试+指数退避、并发信号量控制、错误分类处理。配置DeepSeek API即可使用。

## Player Fantasy

无。

## Detailed Design

### Core Rules

**配置**: `base_url=https://api.deepseek.com/anthropic/v1/messages`, `api_format=anthropic`, `model=deepseek-chat`, `max_tokens=4096`

**接口**: `call_llm(prompt, mode=NORMAL) → str` — 同步调用。NORMAL用于叙事Agent，FAST保留给轻量任务。

**复用点**: 不新建代码。修改`data/llm_settings.json`中的base_url/model_name/api_format指向DeepSeek。

### States

已有。不增加。

### Interactions

→ **S6 Prompt构建器**: 接收构建好的prompt文本
← **S10 Agent系统**: 每回合调用3次（依伊/奥丁/苦主）

## Acceptance Criteria

1. **GIVEN** DeepSeek API配置, **WHEN** call_llm("Hello"), **THEN** 返回非空字符串
2. **GIVEN** API返回429, **WHEN** 调用, **THEN** 自动重试，最多3次后降级
3. **GIVEN** API完全不可达, **WHEN** 调用超时, **THEN** 返回降级JSON不crash

## Open Questions

- 无