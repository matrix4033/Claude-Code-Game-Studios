# ADR-0001: 回合制驱动替代模拟器每秒循环

## Status
Proposed

## Date
2026-05-05

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Python/FastAPI (cultivation-world-simulator reuse) |
| **Domain** | Core |
| **Knowledge Risk** | LOW |
| **References Consulted** | `src/server/loop_runtime.py`, `src/server/runtime/session.py`, demo v1-v12 |
| **Post-Cutoff APIs Used** | None |
| **Verification Required** | v12 demo validated per-turn execution chain |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None |
| **Enables** | ADR-0002 (叙事Prompt), ADR-0003 (短期目标链), ADR-0004 (命名位置), ADR-0005 (日循环) |
| **Blocks** | None |
| **Ordering Note** | Foundation decision — must be implemented before all other systems |

## Context

### Problem Statement
模拟器的`loop_runtime.py`是无限自动循环（`while True: asyncio.sleep(1); sim.step()`），每秒一个tick。恋情模块需要每回合停等玩家：生成演出文本→玩家阅读/切换视角/介入→点"继续"→下一回合。

### Constraints
- 复用模拟器FastAPI/WebSocket基础设施
- 不与模拟器原有循环冲突（两者是独立deployment）
- 回合执行链已被11轮demo验证可行

### Requirements
- 每回合执行S1-S12完整系统链
- 回合结束后WebSocket推送前端渲染
- 等待玩家"继续"信号后才推进

## Decision

将驱动模式从"无限自动循环"改为"回合制"。每回合执行：

```
S4.advance() → S10.run() → WebSocket广播 → 等待 POST /romance/continue → 下一回合
```

复用模拟器`runtime.pause()`机制实现回合间的同步等待。前端"继续"按钮调用`POST /romance/continue`，后端`runtime.resume()`解锁下一回合。

## Alternatives Considered

### A) 保留无限循环+暂停
- **Description**: 沿用模拟器模式，roleplay_auto_paused控制
- **Pros**: 零改动
- **Cons**: 文本需要阅读时间，暂停时机不可控；不适合回合制叙事节奏
- **Rejection Reason**: 核心玩法是回合制视角切换，自动循环不匹配

### B) 回合制（采用）
- **Description**: 每回合执行后自动停止，等玩家确认
- **Pros**: 匹配核心循环设计；v12 demo验证执行链正确
- **Cons**: 需改动loop_runtime

### C) 纯WebSocket事件驱动
- **Description**: 无主循环，前端trigger触发
- **Pros**: 灵活
- **Cons**: 需重写整个后端，工作量过大
- **Rejection Reason**: 不符合MVP时间约束

## Consequences

### Positive
- 玩家有完整的阅读和决策时间
- 回合边界清晰，便于存档和调试

### Negative
- 修改模拟器核心循环逻辑
- WebSocket推送节奏从每秒改为每回合

### Risks
- 回合与模拟器原循环冲突：独立deployment隔离

## GDD Requirements Addressed

| GDD | Requirement | How This ADR Addresses It |
|-----|-------------|--------------------------|
| S4 day-cycle | 每回合自动推进一个时间块 | advance()在回合开始时调用 |
| S10 agent-system | 三Agent每回合并行决策 | run()执行完整S1-S12链 |

## Validation Criteria
- 10回合正常运行，每回合后WebSocket推送
- 玩家不点"继续"时系统不自动推进
- `POST /romance/continue`正确解锁下一回合
