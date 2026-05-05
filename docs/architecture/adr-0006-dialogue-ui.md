# ADR-0006: 对话界面架构

## Status
Accepted

## Date
2026-05-02

> **TD-ADR Review**: APPROVED — Dual-focus handling and CanvasLayer + Control tree architecture confirmed (2026-05-02)

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.6 |
| **Domain** | UI (Control Nodes, RichTextLabel, Input) |
| **Knowledge Risk** | HIGH — Godot 4.6 dual-focus system separates mouse/touch focus from keyboard/gamepad focus; `RichTextLabel.push_meta` added `tooltip` parameter in 4.4 |
| **References Consulted** | `docs/engine-reference/godot/modules/ui.md`, `docs/engine-reference/godot/breaking-changes.md`, `docs/engine-reference/godot/current-best-practices.md` |
| **Post-Cutoff APIs Used** | `Control.grab_focus()` (4.6 dual-focus: kb focus ≠ mouse focus); `RichTextLabel.push_meta()` with `tooltip` parameter (4.4+); `FoldableContainer` (4.5+ — optional for collapsible panels) |
| **Verification Required** | (1) 4.6 dual-focus: Tab navigation on ChoiceButton with `focus_neighbor_*` explicitly set — confirm keyboard focus and mouse hover don't produce dual visual feedback; (2) RichTextLabel CJK font rendering on Web export — Noto Serif SC loading test; (3) `Timer` 40ms accuracy for typewriter on backgrounded browser tab |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001 (`state_changed` — drives layout switching), ADR-0004 (autoload position 10 — after LLMDialogue and PlayerIntervention), ADR-0005 (`text_token` signal from LLM pipeline) |
| **Enables** | Player-facing UI implementation story |
| **Blocks** | None downstream |
| **Ordering Note** | Core-layer ADR. Depends on ADR-0005 (text_token signal contract). DialogueUI autoload initializes at position 10 — after all data sources. |

## Context

### Problem Statement
对话界面是玩家与「浮世」的唯一交互窗口——它必须同时处理流式AI文本（打字机效果）、非对称选择按钮（反映"选择的重角"）、7种状态的布局切换和桌面/平板/移动端响应式适配。Godot 4.6引入了一个关键变更：dual-focus系统——键盘焦点和鼠标焦点分离。如果Tab导航不显式设置`focus_neighbor_*`，键盘用户将无法在选择按钮之间切换。

### Constraints
- Godot 4.6 / GDScript / Web导出
- 4.6 dual-focus: 鼠标悬停和键盘焦点可以同时激活不同控件
- RichTextLabel: 4.4中`push_meta`增加了`tooltip`参数
- Web平台: CJK字体加载可能失败 → 降级路径必需
- 移动端: viewport <768px → 堆叠布局
- 对话文本色 #E8E0CC, 18-20px, Noto Serif SC, 行高1.8

### Requirements
- Control节点树: CanvasLayer → 5个子层(Background/Character/Dialogue/Choice/Status)
- 流式文本: 40ms/字打字机效果，点击/空格跳过
- 非对称选择按钮: 左圆弧/右锐切，HOVER/SELECTED/DISABLED状态
- 状态→布局映射: 7种游戏状态各有不同面板/按钮配置
- 响应式: ≥1366桌面, ≥768平板, <768移动
- Tab导航: 4.6 dual-focus适配
- 对话历史回滚

## Decision

**CanvasLayer为根，5子层Control节点树。状态→布局由`state_changed`信号驱动切换。非对称按钮使用ThemeBox样式+九宫格缩放。4.6 dual-focus通过`focus_neighbor_*`显式设置解决。打字机使用ADR-0005的`Timer`管道。**

### Control节点树

```
CanvasLayer (root)
├── BackgroundLayer
│   ├── ColorRect (夜墨基底 #0D1117)
│   └── LightOverlay (GradientTexture2D — 单光源情绪光照)
├── CharacterLayer
│   ├── PortraitFrame (active_speaker — TextureRect + 光晕)
│   └── PortraitFrame (other — 可选显示)
├── DialogueLayer
│   ├── DialoguePanel (PanelContainer — 暗底+单边光渗色)
│   │   ├── SpeakerTag (Label — 角色名 #D4A56A, 左上角)
│   │   └── DialogueText (RichTextLabel — 正文 #E8E0CC, 18-20px, 行高1.8, Noto Serif SC)
│   └── ScrollContainer (对话历史回滚)
├── ChoiceLayer (visible only in DISCOVERY/CONFRONTATION)
│   └── VBoxContainer/HBoxContainer
│       └── ChoiceButton × N (非对称形状)
├── StatusLayer (top bar)
│   ├── DayLabel ("第N天")
│   ├── TimeBlockLabel ("早晨"/"下午"/"夜晚")
│   └── StateIndicator
└── TransitionLayer (日转覆盖)
    ├── ColorRect (全屏夜墨)
    └── GoldLine (金线动画)
```

### State → Layout Mapping

| GameState | DialoguePanel | ChoiceLayer | RelationshipPanel | LightOverlay |
|-----------|--------------|-------------|-------------------|--------------|
| TITLE | hidden | Start/Settings buttons | hidden | 孤灯光晕 |
| OBSERVATION | active, left-aligned | hidden | 30% opacity | 纸窗晨光 |
| DISCOVERY | active | 2-4 buttons | 100% opacity | 烛火摇曳 |
| CONFRONTATION | active | 1-2 buttons | 100% opacity | 底部冷光 |
| INTIMACY | active (erotic mode) | hidden | 100% opacity | 烛火低垂+粒子 |
| REFLECTION | review mode | hidden | 100% opacity | 暮色入窗 |
| TRANSITION | hidden | hidden | hidden | 月隐星沉 |

### ChoiceButton States

```gdscript
# ThemeBox样式 — 非对称形状
# DEFAULT:  夜墨底(#1A1E24) + 纸白1px边框(#E8E0CC)
# HOVER:    烛焰光晕左扩散(0.2s ease-out) + 文字变烛焰(#D4A56A)
# SELECTED: 墨渍扩散动画(0.5s ease-in) → 褪色残红(#3A1C1C)
# DISABLED: 灰色(#5A5A5A) + opacity 50%

# Layout rules:
# 2 options → horizontal (left + right)
# 3 options → 2 left + 1 right (or vertical stack on mobile)
# 4 options → 2×2 grid
# "旁观" option always rightmost or bottom — visually separated
```

### 4.6 Dual-Focus Adaptation

```gdscript
# Godot 4.6: grab_focus() only affects keyboard/gamepad focus.
# Mouse hover can activate a DIFFERENT control simultaneously.
# Fix: explicitly set focus neighbors for Tab navigation:
func _setup_focus_navigation(buttons: Array[ChoiceButton]) -> void:
    for i in range(buttons.size()):
        if i > 0:
            buttons[i].focus_neighbor_left = buttons[i-1].get_path()
        if i < buttons.size() - 1:
            buttons[i].focus_neighbor_right = buttons[i+1].get_path()
        buttons[i].focus_mode = Control.FOCUS_ALL

# State change: clear all focus before layout switch
func _on_state_changed(old_state: GameState, new_state: GameState) -> void:
    _clear_all_focus()
    _apply_layout(new_state)
    if new_state in [GameState.DISCOVERY, GameState.CONFRONTATION]:
        _choice_buttons[0].grab_focus()  # focus first choice

func _clear_all_focus() -> void:
    get_viewport().gui_release_focus()
```

### Responsive Breakpoints

```gdscript
func _on_resize() -> void:
    var vw := get_viewport().get_visible_rect().size.x
    if vw >= 1366:
        _layout_desktop()   # 肖像左 | 文本中 | 关系右
    elif vw >= 768:
        _layout_tablet()    # 关系面板折叠, 文本 90% 宽度
    else:
        _layout_mobile()    # 垂直堆叠: 肖像上→文本中→按钮下
```

### Typewriter Pipeline

```gdscript
# 监听ADR-0005的text_token信号
func _on_text_token(char: String) -> void:
    _dialogue_text.append_text(char)
    _scroll_to_bottom()

func _on_text_token_complete() -> void:
    _show_continue_indicator()

# Player skip: click/Space
func _input(event: InputEvent) -> void:
    if event.is_action_pressed("ui_accept") or (event is InputEventMouseButton and event.pressed):
        if _typewriter_active:
            _skip_typewriter()  # Timer → complete → emit full text
        elif _text_complete:
            _advance_dialogue()
```

### Fallback Font

```gdscript
func _load_font() -> void:
    var font := FontFile.new()
    if font.load_dynamic_font("res://assets/fonts/NotoSerifSC-Regular.otf") != OK:
        # Fallback: system default — preserve font size and line height
        font = ThemeDB.get_default_theme().get_default_font()
        push_warning("CJK font failed to load — using system default")
    _dialogue_text.add_theme_font_override("normal_font", font)
```

## Alternatives Considered

### Alternative 1: CanvasLayer + Control node tree (CHOSEN)
- **Description**: 标准Godot UI模式。CanvasLayer为独立渲染层根。Control节点树管理布局和输入。
- **Pros**: Godot惯用——任何UI开发者都能理解。内建`anchor_*`+`container`响应式。`Theme`/`ThemeBox`样式系统。信号驱动的布局切换与ADR-0001一致。
- **Cons**: 深层节点树（~20 controls）。4.6 dual-focus需要显式管理。
- **Rejection Reason**: N/A — chosen

### Alternative 2: 自定义_draw()手动渲染
- **Description**: 单个Control节点——所有文本、按钮、面板通过`draw_string()`/`draw_rect()`/`draw_texture()`手动渲染。
- **Pros**: 完全控制每个像素。零节点开销。无dual-focus问题。
- **Cons**: 手动实现所有交互（hit testing, focus, scroll, selection）。无法重用Godot的`Theme`/`Container`/`RichTextLabel`。开发时间是Control节点方案的3-5倍。
- **Rejection Reason**: 牺牲Godot UI系统的全部优势（自动布局、主题、焦点管理、RichTextLabel的BBCode）来换取控制权。在文本为主的游戏中不合理。

### Alternative 3: Theme override per state
- **Description**: 不使用布局切换（显示/隐藏节点），而是为每个状态定义完整的Theme，在`state_changed`时应用。
- **Pros**: 更少的显示/隐藏逻辑。Theme是Godot的标准样式机制。
- **Cons**: 7个Theme需维护。布局不仅关乎样式（DISCOVERY有按钮，OBSERVATION无按钮——不能仅通过Theme切换）。Theme切换和节点显示/隐藏需要同时管理。
- **Rejection Reason**: 布局不仅关乎样式——节点可见性（DISCOVERY中按钮显示/隐藏、关系面板透明度）更适合直接显示/隐藏+Theme组合，而非纯Theme切换。

## Consequences

### Positive
- CanvasLayer提供独立渲染层——文本在背景和角色立绘之上始终可见
- RichTextLabel提供BBCode支持（颜色、粗体、斜体）——无需自定义文本解析器
- Control节点继承完整的Godot输入事件处理——键盘/鼠标/触摸免费支持
- 状态→布局映射集中于`_on_state_changed()`——新状态只需添加一行映射
- 响应式断点在`_on_resize()`中单一处理——新添加的Control自动适应

### Negative
- 节点树约20个Control——需要管理所有节点的`_exit_tree()`断开连接（ADR-0001强制要求）
- 4.6 dual-focus显式管理增加了每个ChoiceButton的`focus_neighbor_*`代码
- 7个状态×3个响应式断面=21种布局组合——测试矩阵需覆盖

### Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| 4.6 dual-focus: Tab聚焦按钮+鼠标悬停另一个按钮→双重视觉反馈 | Medium | 状态切换时`gui_release_focus()`清除所有焦点。HOVER样式不影响`focus`样式——两者使用不同的ThemeBox属性 |
| CJK字体Web加载失败→方块字或无文本 | Medium | 字体加载失败→立即回退到系统默认字体。保留字号和行高 |
| `Timer` 40ms打字机在后台标签页被节流→文本显示延迟 | Low | 玩家返回时文本已完成或接近完成。不影响正确性 |
| 移动端 `<768px` 堆叠布局中关系面板占用过多空间 | Low | 移动端关系面板折叠到顶栏小图标——点击展开为悬浮面板 |

## GDD Requirements Addressed

| GDD System | Section | Requirement | How This ADR Addresses It |
|------------|---------|-------------|--------------------------|
| dialogue-intervention-ui | Core Rules, Rule 1 | CanvasLayer节点树: Background/Character/Dialogue/Choice/Status层 | 5-layer Control tree rooted at CanvasLayer |
| dialogue-intervention-ui | Core Rules, Rule 2 | 文本管道: text_queue → 逐字渲染 → 跳过 → 推进 | `_on_text_token()` + Timer typewriter + player skip via `_input()` |
| dialogue-intervention-ui | Core Rules, Rule 3 | 选择按钮: DEFAULT/HOVER/SELECTED/DISABLED + 非对称形状 + 布局规则 | ThemeBox styles + `_setup_focus_navigation()` + layout rules per option count |
| dialogue-intervention-ui | Core Rules, Rule 4 | 状态→布局映射表 (7行) | `_on_state_changed()` — layout mapping table |
| dialogue-intervention-ui | Core Rules, Rule 5 | 输入: 空格/Enter=推进, 数字键=选择, Tab=焦点, Esc=暂停 | `_input()` — key mapping + mouse click |
| dialogue-intervention-ui | Core Rules, Rule 6 | 对话历史: 回滚+50条+自动加载 | ScrollContainer + `_dialogue_entries: Array[50]` |
| dialogue-intervention-ui | Formulas | 响应式断点: ≥1366=Desktop, ≥768=Tablet, <768=Mobile | `_on_resize()` — 3 breakpoint layout switch |
| dialogue-intervention-ui | Edge Cases | 文本>500字自动分页, 打字中切换窗口不暂停, 连续点击0.1s冷却 | Pagination at 500 chars, click_cooldown=100ms, no pause on defocus |

## Performance Implications
- **CPU**: `_on_state_changed()` layout switch <0.5ms. Typewriter Timer tick <0.01ms. Input processing <0.1ms
- **Memory**: Control node tree ~20 controls × ~2KB = 40KB. Font texture ~1-2MB (Noto Serif SC)
- **Load Time**: Font load ~50-200ms (async). Control tree instantiation ~2ms
- **Network**: N/A

## Migration Plan
N/A — 新系统。

## Validation Criteria
- [ ] 7个状态×3个断点 → 21种布局全部正确渲染（可见/隐藏匹配映射表）
- [ ] Tab键在ChoiceButton之间导航 — 焦点按`focus_neighbor_*`设置顺序移动
- [ ] 鼠标悬停HOVER按钮 + 键盘聚焦另一个按钮 → 双重视觉状态不产生冲突
- [ ] 打字机40ms/字 → 字符逐字出现在RichTextLabel中
- [ ] 打字中点击/空格 → 立即显示完整文本
- [ ] 对话历史回滚 → 最多50条可滚动条目, 不在底部时新文本不自动滚动
- [ ] 连续快速点击 → 0.1s冷却防止跳过关键对话
- [ ] CJK字体加载失败 → 系统默认字体回退, 游戏不崩溃
- [ ] 移动端(<768px) → 垂直堆叠布局, 关系面板折叠到顶栏图标
- [ ] TRANSITION状态 → 日转覆盖2-3s夜墨渐变+金线动画

## Related Decisions
- ADR-0001: `state_changed` 驱动 `_on_state_changed()` 布局切换
- ADR-0004: DialogueUI在autoload位置10 — 所有数据源已就绪
- ADR-0005: `text_token`/`text_token_complete`/`text_error` 信号 — 打字机管道输入
