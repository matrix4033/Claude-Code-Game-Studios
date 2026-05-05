# 恋情模块 — 架构蓝图

**Date**: 2026-05-05 | **Stack**: Python/FastAPI + Vue 3 + Pixi.js + DeepSeek API
**Based on**: cultivation-world-simulator 改造 | 12 MVP GDDs | 12 demo versions

---

## System Layer Map

```
┌─────────────────────────────────────────────┐
│  PRESENTATION (S13-S17)                     │
│  Vue 3 + Pixi.js 复 模拟器                   │
│  StatusBar(视角切换) / EventPanel(视角过滤)   │
│  MapLayer(都市贴图) / RoleplayDock(介入)      │
├─────────────────────────────────────────────┤
│  FEATURE (S10-S12)                          │
│  S10 Agent — 三Agent协调，回合制              │
│  S11 认知隔离 — 在场=involved规则            │
│  S12 事件链 — 每角色独立结构化历史            │
├─────────────────────────────────────────────┤
│  CORE (S5-S9)                               │
│  S5 LLM ──── DeepSeek Anthropic格式         │
│  S6 Prompt ─── 三Agent叙事模板               │
│  S7 目标链 ─── lt_goal/st_goal自驱推进       │
│  S8 碰撞 ──── 命名位置同地点检测              │
│  S9 引力 ──── prompt注入对方位置              │
├─────────────────────────────────────────────┤
│  FOUNDATION (S1-S4)                         │
│  S1 素材库 ── YAML加载+缓存+格式化            │
│  S2 角色模型 ── 5维参数+目标+事件链          │
│  S3 Delta ─── 事件→属性查表                  │
│  S4 时间 ──── 5时段日循环                    │
├─────────────────────────────────────────────┤
│  PLATFORM                                    │
│  FastAPI / WebSocket / Pixi.js               │
│  (cultivation-world-simulator)               │
└─────────────────────────────────────────────┘
```

---

## Data Flow (per turn)

```
S4.advance()
  ↓
S10 Agent系统
  ├→ S6 build_prompt()
  │   ├→ S1.get_all_context(stage)  ← 素材文本块
  │   ├→ S2 role state              ← 参数+目标
  │   ├→ S4 current time            ← "第N天·午後"
  │   ├→ S12 fmt_events()           ← 该角色事件链
  │   └→ S8+S9 位置+引力           ← 其他角色位置
  ├→ S5 call_llm(prompt) ×3         ← 三Agent并发
  ├→ S10 parse JSON responses
  ├→ S7 update_goals()              ← 目标链更新
  ├→ S8 update positions            ← 位置更新
  ├→ S3 apply_delta()               ← 属性变化
  ├→ S11 distribute_events()        ← 按在场规则分发
  └→ S12 append to each chain       ← 追加事件
  ↓
WebSocket 推送 → 前端渲染
```

---

## S11 认知隔离规则

每个角色永远只看到自己的视角。碰撞决定"还知道谁的行动"：

```
partner==bull     → 依伊+奥丁互知对方行动
partner==cuckold  → 林晨知道依伊在场（公开版本）
bull==cuckold     → 林晨知道奥丁在场

不在场 = 不知道 = 事件链中无对方行动
```

---

## Module Ownership

| System | Owns | Exposes |
|--------|------|---------|
| S1 MaterialLibrary | 9 YAML缓存, 5状态 | `get_all_context(stage)` |
| S2 CharacterState | 5维参数+目标+事件链 | 完整dataclass |
| S3 DeltaTable | 12条事件→属性映射 | `apply_delta(char, event)` |
| S4 DayCycle | (day, bi) | `current()`, `advance()` |
| S5 LLMClient | API调用+重试 | `call_llm(prompt)` |
| S6 PromptBuilder | 3个prompt模板 | `build_*_prompt()` |
| S7 GoalChain | 目标更新逻辑 | `update_goals()` |
| S8 Collision | 碰撞检测 | `check(game) → types` |
| S10 AgentSystem | 回合执行流程 | 主循环 |
| S11 KnowledgeScope | 事件分发 | `distribute_events()` |
| S12 EventChain | 结构化历史 | `append()`, `format()` |

---

## Simulator Integration

| 模拟器 | 恋情 | 策略 |
|--------|------|------|
| `src/utils/llm/client.py` | S5 | 🔄 复用 |
| `src/server/loop_runtime.py` | S10 | 🔧 回合制替代 |
| `src/server/api/websocket.py` | — | 🔄 复用 |
| `src/classes/ai.py` | S10 | 🔧 叙事Agent替代 |
| `src/classes/long_term_objective.py` | S7 | 🔧 短期目标替代 |
| `src/systems/time.py` | S4 | 🔧 日循环替代月步进 |
| `src/romance/models/character_state.py` | S2 | 🔧 加目标字段 |
| `src/romance/config/*.yaml` | S1 | 🔄 复用 |
| `web/src/components/game/MapLayer.vue` | S13 | 🔧 换都市贴图 |
| `web/src/components/game/EventPanel` | S14 | 🔧 加视角过滤 |
| `web/src/components/game/RoleplayDock.vue` | S15 | 🔄 复用 |
| `web/src/components/layout/StatusBar` | S16 | ✨ 加视角切换 |
| `src/server/auto_save.py` | S17 | 🔄 复用 |

---

## Open Questions

- WebSocket推送节奏：回合制下从每秒推送改为每回合推送
- 存档兼容恋情新增字段（目标、事件链）
