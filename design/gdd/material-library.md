# 素材库加载器 (Material Library)

> **Status**: In Design
> **Last Updated**: 2026-05-05
> **Implements Pillar**: 2 — 目标驱动
> **Strategy**: ✨ 新建

## Overview

素材库加载器在服务启动时一次性加载全部YAML配置文件（场景/体位/道具/服装/调教方法/堕落阶段/性癖/对白风格/约会地点），缓存为内存数据结构。Agent决策时，按当前堕落阶段过滤可用素材，并格式化为可直接注入LLM prompt的文本上下文。没有它，Agent无法从素材库中选择任何场景、体位或道具——叙事将失去具体的物质基础。

## Player Fantasy

玩家不会直接与素材库交互。但素材库的丰富度直接决定了叙事多样性的上限。每个场景、体位、道具的加入，都扩大了Agent可生成故事的可能性空间。设计目标：素材库应该让玩家在10次游戏中看到10种不同的调教路径，而不是重复相同的场景组合。

## Detailed Design

### Core Rules

**1. 加载时机**：服务启动时一次性加载全部9个YAML → 内存缓存。不逐回合读磁盘。

**2. 素材结构**：每个素材保留原始YAML字段，不做截断。

| YAML | 关键字段 |
|------|---------|
| scenes | name, stage_min, description, privacy, atmosphere, tags |
| positions | name, stage_min, description, intensity, intimacy_level, tags |
| props | name, stage_min, type, description, use_effect, tags |
| clothing | name, style, description |
| training_methods | name, type, description |
| corruption_stages | stage, name, description, unlock_conditions |
| fetishes | name, category, description, intensity, tags |
| dialogue_styles | name, tone, description |
| locations | name, type, atmosphere, risk_level, description |

**3. 过滤规则**：`filter_by_stage(stage)` — 返回 `stage_min <= stage` 的所有条目。无上限限制——高阶段可以看到低阶段所有内容。

**4. 格式化输出**：`get_all_context(stage)` → 字符串，可直接拼入prompt。格式：
```
【可用场景 — 阶段2+】
  · 咖啡厅 — 安静的独立咖啡厅 [氛围:轻松 私密:low]
  · 酒店 — 市中心的精品酒店 [氛围:情欲 私密:private]

【可用体位 — 阶段2+】
  · 传教士 — 经典的面对面体位 [强度:3/5 basic]

【可用道具 — 阶段2+】
  · 眼罩(blindfold) — 丝绸眼罩 [遮蔽视线后触觉加倍敏感]

【可用调教方法】
  · 言语引导(verbal) — 通过语言引导对方心理变化

【可用服装】
  · 性感内衣(lingerie) — 性感的蕾丝内衣

【可用地点】
  · 咖啡厅 [public] — 安静的咖啡厅，适合日常社交 (风险:0)
  · 酒吧 [semi_public] — 昏暗灯光下的酒吧 (风险:1)
  · 酒店 [private] — 市中心的精品酒店 (风险:2)
```

**5. 重载**：`reload()` 清空缓存重新加载。新游戏开始时调用。

### Error Handling

| 级别 | 条件 | 行为 |
|------|------|------|
| **INFO** | 非关键YAML缺失（如fetishes.yaml） | 日志警告，该类别返回空列表，系统正常运行 |
| **ERROR** | 关键YAML缺失（scenes/positions/clothing/corruption_stages） | 系统进入ERROR状态，降级为纯LLM自由生成 |
| **FATAL** | 所有YAML不可读（目录不存在/权限拒绝） | 服务启动失败，明确报错退出 |

**降级策略**：ERROR状态下 `get_all_context()` 返回：
```
【素材库当前不可用。请根据角色档案和历史自由决定场景、体位、道具。】
```

### States

| 状态 | 条件 | 行为 |
|------|------|------|
| UNLOADED | 未调用load | 返回空列表 |
| LOADING | 正在读YAML | 阻塞等待 |
| READY | 全部加载成功 | 正常查询 |
| DEGRADED | 非关键YAML缺失 | 正常查询，缺失类别返回空列表 |
| ERROR | 关键YAML缺失 | 返回降级提示文本 |

### Interactions

→ **S6 Prompt构建器**: 调用 `get_all_context(stage)` 获取当前阶段可用的素材文本块注入Agent prompt。ERROR状态下注入降级提示文本。

## Formulas

无公式。纯数据结构加载和过滤。

## Edge Cases

- **YAML字段缺失**: 使用默认值（`stage_min`默认0，`description`默认空字符串）。不crash。
- **stage为负数**: `filter_by_stage(-1)` 等价于 `stage_min <= -1`，返回空列表。不抛异常。
- **YAML包含重复条目**: 同名条目保留后加载的。依赖dict key天然去重。
- **reload()与查询并发**: reload先加载到临时变量，完成后原子赋值。查询不中断、不读到半份数据。

## Dependencies

| 依赖 | 方向 | 类型 | 接口 |
|------|------|------|------|
| 文件系统 (YAML目录) | 入 | 硬 | `src/romance/config/*.yaml` 必须存在且可读 |
| → S6 Prompt构建器 | 出 | 硬 | `get_all_context(stage)` → 格式化文本块 |

纯Foundation层，零系统级依赖。

## Tuning Knobs

无运行时调整参数。素材内容本身是tuning knob——修改YAML文件后重启服务或调用 `reload()` 生效。

## Acceptance Criteria

1. **GIVEN** 服务启动且YAML目录完好，**WHEN** 调用 `load()`，**THEN** 状态变为READY，9个类别全部有数据
2. **GIVEN** READY状态，**WHEN** `get_scenes(2)`，**THEN** 返回所有 `stage_min <= 2` 的场景
3. **GIVEN** READY状态，**WHEN** `get_all_context(2)`，**THEN** 返回包含场景/体位/道具/服装/调教/地点6个类别的格式化文本
4. **GIVEN** 非关键YAML缺失（如fetishes.yaml），**WHEN** `load()`，**THEN** 状态为DEGRADED，缺失类别返回空列表，其他类别正常可用
5. **GIVEN** 关键YAML缺失（scenes/positions/clothing/corruption_stages任一），**WHEN** `load()`，**THEN** 状态为ERROR，`get_all_context()` 返回降级提示文本
6. **GIVEN** READY状态，**WHEN** 调用 `reload()`，**THEN** 缓存刷新，进行中的查询使用旧数据完成
7. **GIVEN** YAML中某个字段缺失（如scene缺少atmosphere），**WHEN** `load()`，**THEN** 缺失字段使用默认值（0或空字符串），不crash

## Open Questions

- 素材数量上限（每类最多选几条注入prompt）— 暂搁置，观察阶段5的token消耗后再决定
- 素材随机化（是否每回合随机抽样增加多样性）— 暂搁置，先验证LLM自主选择的多样性
