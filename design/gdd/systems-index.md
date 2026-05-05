# Systems Index — 恋情模块

*Created: 2026-05-05*
*Status: Confirmed*
*基于: game-concept.md, 11 demo iterations, cultivation-world-simulator 改造方案*

---

## 改造策略

| 标记 | 含义 |
|------|------|
| 🔄 复用 | 模拟器已有，配置/接入即可 |
| 🔧 改造 | 模拟器有基础，需重大修改 |
| ✨ 新建 | 模拟器没有，全新实现 |

---

## Foundation Layer

| # | System | 策略 | 模拟器现有资产 | 改造说明 |
|---|--------|------|-------------|---------|
| S1 | 素材库加载器 | ✨ 新建 | `src/romance/config/romance_config.py`(部分) | 完善加载器：按stage过滤、全量素材格式化输出、服装库接入 |
| S2 | 角色数据模型 | 🔧 改造 | `src/romance/models/character_state.py` | 增加 lt_goal/st_goal/goal_done 目标字段 |
| S3 | Delta映射表 | ✨ 新建 | 无 | 事件类型→5维属性变化查表，demo DELTA已验证 |
| S4 | 日周期时间系统 | 🔧 改造 | `src/systems/time.py`(MonthStamp月步进) | 改为5时段日循环，回合制推进 |

## Core Layer

| # | System | 策略 | 模拟器现有资产 | 改造说明 |
|---|--------|------|-------------|---------|
| S5 | LLM客户端 | 🔄 复用 | `src/utils/llm/client.py` | 已有Anthropic格式+重试+并发控制，配DeepSeek API |
| S6 | 叙事Prompt构建器 | ✨ 新建 | `static/locales/zh-CN/templates/ai.txt`(通用决策prompt) | 替为三Agent专用的第一人称叙事prompt模板 |
| S7 | 短期目标链 | 🔧 改造 | `src/classes/long_term_objective.py`(5-10年期) | 改为每回合更新的st_goal+goal_done机制 |
| S8 | 位置碰撞检测 | 🔧 改造 | `src/classes/core/avatar/core.py`(pos_x/pos_y) | 改为命名位置+同地点碰撞触发 |
| S9 | 目标→位置引力 | ✨ 新建 | 无 | 目标涉及他人时Agent倾向移动到对方位置 |

## Feature Layer

| # | System | 策略 | 模拟器现有资产 | 改造说明 |
|---|--------|------|-------------|---------|
| S10 | Agent决策系统 | 🔧 改造 | `src/classes/ai.py`(LLMAI._decide，决策型) | 改为叙事型Agent：奥丁先手→依伊回应，苦主并行 |
| S11 | 认知隔离引擎 | 🔧 改造 | `src/romance/models/event.py`(KnowledgeScope已定义) | 接入Agent决策流程，按角色认知过滤事件 |
| S12 | 事件链记忆系统 | ✨ 新建 | `src/classes/event.py`(基础Event类) | 结构化事件条目，全量注入prompt，角色独立记忆链 |

## Presentation Layer

| # | System | 策略 | 模拟器现有资产 | 改造说明 |
|---|--------|------|-------------|---------|
| S13 | 地图渲染 | 🔧 改造 | `web/src/components/game/MapLayer.vue`(修仙世界Pixi.js) | 换成都市贴图，位置使用命名地点 |
| S14 | EventPanel | 🔧 改造 | `web/src/components/game/EventPanel` | 增加视角过滤(current_perspective→filter events) |
| S15 | 玩家介入 | 🔄 复用 | `web/src/components/game/RoleplayDock.vue` + `roleplay_service.py` | 复用RoleplayDock自由文本输入，后端解析意图 |
| S16 | 视角切换 | ✨ 新建 | `web/src/components/layout/StatusBar`(无此功能) | StatusBar增加三视角切换按钮 |

## Polish Layer

| # | System | 策略 | 模拟器现有资产 | 改造说明 |
|---|--------|------|-------------|---------|
| S17 | 存档系统 | 🔄 复用 | `src/server/auto_save.py` + `save_load_control.py` | 复用，存储恋情角色状态+事件链 |

---

## Dependency Map

```
Foundation (零依赖)
  ✨ S1 素材库 ──→ 独立
  🔧 S2 角色模型 ──→ 独立
  ✨ S3 Delta表 ──→ 独立
  🔧 S4 时间系统 ──→ 独立

Core
  🔄 S5 LLM客户端 ──→ 已有(SettingsService)
  ✨ S6 Prompt构建器 ──→ S1 + S2 + S12
  🔧 S7 目标链 ──→ S2 + S12
  🔧 S8 碰撞检测 ──→ S2
  ✨ S9 位置引力 ──→ S7 + S8

Feature
  🔧 S10 Agent系统 ──→ S5 + S6 + S7 + S8 + S9
  🔧 S11 认知隔离 ──→ S10 + S12
  ✨ S12 事件链 ──→ S10 + S2

Presentation
  🔧 S13 地图 ──→ S8 (复用Pixi.js)
  🔧 S14 EventPanel ──→ S11 + S12
  🔄 S15 玩家介入 ──→ S10 (复用RoleplayDock)
  ✨ S16 视角切换 ──→ S11

Polish
  🔄 S17 存档 ──→ S2 + S12 + S4 (复用auto_save)
```

---

## Priority Assignment

| Tier | Systems | Count | 新建 | 改造 | 复用 |
|------|---------|-------|------|------|------|
| **MVP** | S1-S12 | 12 | 5 | 6 | 1 |
| **VS** | S13-S17 | 5 | 1 | 3 | 1 |

### MVP 设计顺序
1. ✨ S1 素材库加载器 → 2. 🔧 S2 角色数据模型 → 3. ✨ S3 Delta映射表 → 4. 🔧 S4 日周期时间系统 → 5. 🔄 S5 LLM客户端 → 6. ✨ S6 叙事Prompt构建器 → 7. 🔧 S7 短期目标链 → 8. 🔧 S8 位置碰撞检测 → 9. ✨ S9 目标→位置引力 → 10. 🔧 S10 Agent决策系统 → 11. 🔧 S11 认知隔离引擎 → 12. ✨ S12 事件链记忆系统

---

## Progress Tracker

| # | System | 策略 | Tier | Status |
|---|--------|------|------|--------|
| S1 | 素材库加载器 | ✨ | MVP | Not Started |
| S2 | 角色数据模型 | 🔧 | MVP | Not Started |
| S3 | Delta映射表 | ✨ | MVP | Not Started |
| S4 | 日周期时间系统 | 🔧 | MVP | Not Started |
| S5 | LLM客户端 | 🔄 | MVP | Not Started |
| S6 | 叙事Prompt构建器 | ✨ | MVP | Not Started |
| S7 | 短期目标链 | 🔧 | MVP | Not Started |
| S8 | 位置碰撞检测 | 🔧 | MVP | Not Started |
| S9 | 目标→位置引力 | ✨ | MVP | Not Started |
| S10 | Agent决策系统 | 🔧 | MVP | Not Started |
| S11 | 认知隔离引擎 | 🔧 | MVP | Not Started |
| S12 | 事件链记忆系统 | ✨ | MVP | Not Started |
| S13 | 地图渲染 | 🔧 | VS | Not Started |
| S14 | EventPanel | 🔧 | VS | Not Started |
| S15 | 玩家介入 | 🔄 | VS | Not Started |
| S16 | 视角切换 | ✨ | VS | Not Started |
| S17 | 存档系统 | 🔄 | VS | Not Started |

---

## Summary

- **17 systems**: 4 Foundation + 5 Core + 3 Feature + 4 Presentation + 1 Polish
- **12 MVP** (5 new + 6 modify + 1 reuse), 3-4 weeks
- **5 VS** (1 new + 3 modify + 1 reuse)
- **改造占比**: 新建 6 · 改造 9 · 复用 2
