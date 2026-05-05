# Systems Index — 恋情模块

*Created: 2026-05-05*
*Status: Confirmed*
*Based on: game-concept.md, 11 demo iterations*

---

## System Enumeration

### Foundation Layer

| # | System | Description |
|---|--------|-------------|
| S1 | 素材库加载器 | 加载9个YAML，按堕落阶段过滤场景/体位/道具/服装/调教方法 |
| S2 | 角色数据模型 | 5维恋情参数(lv/at/sb/pm/ar) + 苦主参数(aw/tr) + 目标状态 + 事件链 |
| S3 | Delta映射表 | 事件类型→属性变化静态查表 |
| S4 | 日周期时间系统 | 清晨→上午→午后→傍晚→深夜循环，每回合自动推进 |

### Core Layer

| # | System | Description |
|---|--------|-------------|
| S5 | LLM客户端 | DeepSeek Anthropic格式直连，重试+错误处理 |
| S6 | 叙事Prompt构建器 | 三Agent的第一人称叙事prompt模板，注入素材库+角色档案+事件链 |
| S7 | 短期目标链 | lt_goal/st_goal/goal_done，LLM自设自更新 |
| S8 | 位置碰撞检测 | 同地点角色自动标记碰撞，触发互动 |
| S9 | 目标→位置引力 | 目标涉及他人时Agent主动移向对方位置 |

### Feature Layer

| # | System | Description |
|---|--------|-------------|
| S10 | Agent决策系统 | 三Agent协同：奥丁先手→依伊回应，苦主并行 |
| S11 | 认知隔离引擎 | 按角色认知范围过滤事件，信息差叙事 |
| S12 | 事件链记忆系统 | 结构化事件条目，全量注入prompt，不截断 |

### Presentation Layer

| # | System | Description |
|---|--------|-------------|
| S13 | 地图渲染(Pixi.js) | 复用模拟器GameCanvas，都市贴图替换 |
| S14 | EventPanel | 按当前视角过滤事件流，复用模拟器组件 |
| S15 | 玩家介入 | 自由文本输入，LLM解析意图，复用RoleplayDock |

### Polish Layer

| # | System | Description |
|---|--------|-------------|
| S16 | 存档系统 | 角色状态+事件链+时间序列化，复用模拟器auto_save |

---

## Dependency Map

```
Foundation (零依赖)
  S1 素材库 ──→ 独立
  S2 角色模型 ──→ 独立
  S3 Delta表 ──→ 独立
  S4 时间系统 ──→ 独立

Core (依赖 Foundation)
  S5 LLM客户端 ──→ 已有(模拟器SettingsService)
  S6 Prompt构建器 ──→ S1 + S2 + S12(事件链)
  S7 目标链 ──→ S2 + S12
  S8 碰撞检测 ──→ S2(位置字段)
  S9 位置引力 ──→ S7(目标) + S8(碰撞)

Feature (依赖 Core)
  S10 Agent系统 ──→ S5 + S6 + S7 + S8 + S9
  S11 认知隔离 ──→ S10 + S12
  S12 事件链 ──→ S10(Agent输出) + S2

Presentation (依赖 Feature)
  S13 地图渲染 ──→ S8(位置), 复用模拟器Pixi.js
  S14 EventPanel ──→ S11 + S12
  S15 玩家介入 ──→ S10, 复用模拟器RoleplayDock

Polish (依赖 Presentation)
  S16 存档 ──→ S2 + S12 + S4, 复用模拟器存档
```

### Bottleneck Systems
- **S6 Prompt构建器**: 三个Agent都依赖它，叙事质量的关键瓶颈
- **S10 Agent决策系统**: Feature层的核心，所有上层系统依赖它

---

## Priority Assignment

| Tier | Systems | Count |
|------|---------|-------|
| **MVP** | S1-S12 | 12 |
| **VS** | S13-S16 | 4 |

### MVP Design Order
1. S1 素材库加载器 → 2. S2 角色数据模型 → 3. S3 Delta映射表 → 4. S4 日周期时间系统 → 5. S5 LLM客户端 → 6. S6 叙事Prompt构建器 → 7. S7 短期目标链 → 8. S8 位置碰撞检测 → 9. S9 目标→位置引力 → 10. S10 Agent决策系统 → 11. S11 认知隔离引擎 → 12. S12 事件链记忆系统

---

## Progress Tracker

| # | System | Tier | Status |
|---|--------|------|--------|
| S1 | 素材库加载器 | MVP | Not Started |
| S2 | 角色数据模型 | MVP | Not Started |
| S3 | Delta映射表 | MVP | Not Started |
| S4 | 日周期时间系统 | MVP | Not Started |
| S5 | LLM客户端 | MVP | Not Started |
| S6 | 叙事Prompt构建器 | MVP | Not Started |
| S7 | 短期目标链 | MVP | Not Started |
| S8 | 位置碰撞检测 | MVP | Not Started |
| S9 | 目标→位置引力 | MVP | Not Started |
| S10 | Agent决策系统 | MVP | Not Started |
| S11 | 认知隔离引擎 | MVP | Not Started |
| S12 | 事件链记忆系统 | MVP | Not Started |
| S13 | 地图渲染 | VS | Not Started |
| S14 | EventPanel | VS | Not Started |
| S15 | 玩家介入 | VS | Not Started |
| S16 | 存档系统 | VS | Not Started |

---

## Summary

- **16 systems**: 4 Foundation + 5 Core + 3 Feature + 3 Presentation + 1 Polish
- **12 MVP**, estimated 3-4 weeks
- **4 VS**, reuse existing simulator infrastructure
