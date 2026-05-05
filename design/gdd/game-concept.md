# Game Concept: 恋情模块

*Created: 2026-05-02*
*Updated: 2026-05-05 (基于11轮demo验证重写)*
*Status: Confirmed*

---

## Elevator Pitch

> 三个角色——一个被欲望撕裂的女大学生、一个掌控欲极强的学长、一个暗恋她的普通同学——在同一个校园里各自追逐自己的目标。AI实时驱动每个人的行动和对话。你切换视角，看到不同版本的"真相"。介入，或者只是旁观——故事都在继续。

---

## Core Identity

| Aspect | Detail |
| ------ | ------ |
| **Genre** | AI驱动成人叙事 / 视角切换 / 调教NTR |
| **Platform** | Web (浏览器) |
| **Tech Stack** | Python/FastAPI + Vue 3 + Pixi.js + DeepSeek API |
| **Target Audience** | 成人向叙事爱好者、NTR/调教题材玩家 |
| **Player Count** | 单人 |
| **Session Length** | 30-60 分钟 |
| **Estimated Scope** | 小-中 (MVP 3-4周) |
| **Comparable Titles** | era游戏系列 (文字调教), AI Dungeon (AI叙事), School Days (情感复杂度) |

---

## Core Fantasy

你不在故事之外——你在故事之间。每切换一次视角，你就看到此前被隐藏的一面。作为苦主，你感到隐约不安。切到依伊，你看到她的身体在背叛理智。切到奥丁，你看到他的策略在一步步收紧。

核心幻想不是"攻略角色"，而是**在三重视角之间穿梭，体验信息差带来的戏剧张力**——你比任何一个角色都知道得更多，但你也比任何人都更无力改变。

---

## Unique Hook

> 目标驱动的AI角色自主叙事——不是剧本，不是分支树。每个角色有自己的长期目标和短期目标，LLM决定他们每天做什么、和谁互动、关系如何演化。视角切换揭示信息差：同一个事件，苦主只知道"她又在加班"，伴侣知道"我在体育馆跪着"。

---

## Player Experience (MDA)

### Target Aesthetics

| Aesthetic | Priority | How |
| ---- | ---- | ---- |
| **Narrative** | 1 | LLM生成的目标驱动叙事，无预设剧本 |
| **Discovery** | 2 | 切换视角发现隐藏信息、"如果介入会怎样" |
| **Submission** | 2 | 沉浸式旁观+支配/被支配体验 |
| **Fantasy** | 3 | 扮演三个截然不同的视角 |
| **Expression** | 3 | 每次介入定义你是谁 |

### Key Dynamics
- 玩家反复切换视角对比"不同版本的故事"
- 玩家在"介入"和"旁观"之间抉择
- 对特定角色产生情感依附或排斥

### Core Mechanics
1. **目标驱动Agent系统** — 每角色有lt_goal/st_goal，LLM自主设定和更新
2. **事件链记忆** — 结构化历史全量注入prompt
3. **视角切换** — 三按钮，EventPanel按认知过滤
4. **玩家介入** — 自由文本，LLM解析意图影响delta
5. **系统驱动时间** — 清晨→上午→午后→傍晚→深夜连续流

---

## Player Motivation Profile

| Need | How | Strength |
| ---- | ---- | ---- |
| **Autonomy** | 每次介入是自由选择，无"正确"路线 | Core |
| **Relatedness** | 与AI角色建立真实情感连接 | Core |
| **Competence** | 精读角色行为、把握介入时机 | Supporting |

### Player Type (Bartle)
- **Explorers** — 核心：挖掘隐藏关系、试探"如果...会怎样"
- **Socializers** — 核心：与AI角色建立深度情感连接
- **Achievers** — 不适用
- **Killers/Competitors** — 不适用

---

## Core Loop

```
系统驱动时间（每回合一个时间块）
    │
    ▼
三Agent并行决策（目标驱动，自主选择位置/行动）
    │
    ▼
    依伊：去哪？做什么？内心的撕裂？
    奥丁：观察还是接近？线上还是线下？
    林晨：搭话还是沉默？注意到什么异常？
    │
    ▼
碰撞检测 → 位置相同则触发互动
    │
    ▼
生成本回合演出文本
    │
    ▼
玩家：读 / 切换视角 / 输入介入 / 点继续 → 下一回合
```

### 时间系统
- 每回合一个时间块：清晨→上午→午后→傍晚→深夜→清晨...
- 系统自动推进，LLM不管理时间
- 三天约15回合为一个自然故事周期

### 位置系统
- 14个位置：8个约会地点(locations.yaml) + 6个校园地点
- 每个位置有type(public/semi_public/private)、atmosphere、risk_level
- 位置由LLM自主选择，目标驱动移动

---

## Game Pillars

### Pillar 1: 信息差即叙事
故事不是写出来的，是在三个角色各自知道/不知道的事之间自然产生的。
*Design test*: 选"玩家看到片面的真相"而非"全知旁白解释一切"。

### Pillar 2: 目标驱动
每个角色的lt_goal和st_goal是叙事的唯一引擎。目标冲突自动产生戏剧。
*Design test*: 选"让角色追求自己的目标"而非"安排他们该做什么"。

### Pillar 3: 视角即玩法
玩家不是固定角色，是视角切换器。三条线都体验才是完整游戏。
*Design test*: 选"信息按认知隔离"而非"让玩家看到全部"。

### Pillar 4: 每个选择留下痕迹
介入改变delta。不介入也是一种选择（默认走向目标驱动的方向）。
*Design test*: 选"delta反映真实后果"而非"装饰性选择"。

### Anti-Pillars
- **不是恋爱模拟** — 不存在"追到幸福结局"的路线
- **不是视觉小说** — 地图驱动，不是场景驱动
- **不是多人游戏**
- **三角结构不可变**（苦主/伴侣/黄毛），但角色设定完全自定义

---

## Characters

| 角色 | 身份 | 目标驱动 |
|------|------|---------|
| 严依伊 | 女大学生 | lt: "找到穿透乖乖女外壳的人" st: 可变 |
| 陈深(奥丁) | 大四学长 | lt: "彻底掌控一个对象" st: 可变 |
| 林晨 | 同班暗恋者 | lt: "和她在一起或知道她在想什么" st: 可变 |

每个角色有完整外显档案（依伊已有，详见`素材/1.外显`）。

---

## Technical Validation (11 demo iterations)

| 验证项 | 结果 |
|--------|------|
| LLM API | DeepSeek Anthropic格式直连，零拒绝 |
| 性爱/调教内容 | LLM自主生成，无需硬编码指令 |
| 外显档案 | 身体细节精确使用（瓷白肌肤/D罩杯/小狗眼等） |
| 素材库 | YAML场景/体位/道具/服装正确引用 |
| 事件链记忆 | 全量注入，决策连贯 |
| 目标驱动 | 目标链自驱推进 |
| 系统时间 | 连续无跳跃 |
| 位置引力 | 碰撞率80% |

---

## MVP Definition

### Core hypothesis
3个目标驱动的AI角色在封闭校园环境中自主互动 + 玩家视角切换与介入 = 涌现叙事体验，支撑30-60分钟session。

### Required for MVP
1. 3 Agent系统：目标驱动 + 事件链记忆 + 顺序互动
2. 素材库：8个YAML + 外显档案
3. 位置系统：14个地点，碰撞检测
4. Delta系统：静态查表
5. 前端：Pixi.js地图(换贴图) + EventPanel + 视角切换 + 介入输入
6. DeepSeek API集成
7. 基础存档（单槽）

### NOT in MVP
- 多CG/立绘
- 音效/音乐
- 手机适配
- 跨周目记忆
- 成就系统

### Scope Tiers

| Tier | Content | Timeline |
| ---- | ---- | ---- |
| **MVP** | 3角色 × 基础循环 × 14地点 | 3-4周 |
| **V Slice** | 完整调教弧线 + 素材库全覆盖 | +2周 |
| **Full** | 多角色自定义 + 存档 + 完善UI | +1-2月 |

---

## Risks

| Risk | Level | Mitigation |
|------|-------|------------|
| LLM API费用 | 中 | DeepSeek ≈$0.14/1M token |
| 叙事质量控制 | 中 | 外显档案+素材库约束LLM输出 |
| 参数演进停滞 | 低 | 调Delta或增加回合数 |
| 内容分发限制 | 中 | Web部署，不走应用商店 |

---

## Next Steps

1. `/map-systems` — 将概念分解为独立系统
2. `/design-system` — 逐系统编写GDD
3. `/create-architecture` — 架构蓝图
4. `/architecture-decision` — 关键决策ADR
5. `/gate-check` — 阶段门验证
6. `/sprint-plan new` — 第一个Sprint
