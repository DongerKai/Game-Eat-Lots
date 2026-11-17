# UX Spec: 设置模态（SettingsModal）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Journey Phase(s)**: 全流程（主菜单、子屏、暂停覆盖层之上）
> **Screen / Flow Name**: `SettingsModal`
> **Platform Target**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持（`Gamepad Support: Partial`）、无触屏；参考分辨率 1280×720，支持至 3840×2160 与 21:9
> **Related GDDs**: `design/gdd/ui-flow.md`（规则 5、15、19–21、24–28、屏幕契约）、`design/gdd/input.md`、`design/gdd/save-system.md`（规则 20、21、23；UI Requirements 5–6）、`design/gdd/audio-feedback.md`（UI Requirements 音频四项）、`design/gdd/in-game-hud.md`（文本缩放与 HUD 不透明度消费方）、`design/accessibility-requirements.md`
> **Sources**: 上述 GDD 的 `## UI Requirements`；`design/ux/interaction-patterns.md`；`design/ux/hud.md`（结构与质量基线，未复制内容）；`design/accessibility-requirements.md`；`.claude/docs/technical-preferences.md`
> **Sibling Specs**: `main-menu.md`（入口 1、焦点归还、设置写入失败提示落点）、`PauseOverlay` 规格（入口 2，由另一批规格负责，文件名以那批交付为准）、`upgrade-screen.md`、`records-screen.md`、`receipt-screen.md`、`hud.md`；删除生涯的 `ConfirmModal` 两步变体版式归本文件定义
> **Accessibility Tier**: Standard（承诺与逐条要求见 `design/accessibility-requirements.md`）
> **Style Reference**: `design/art/art-bible.md` § 4 Color System；**§ 7 UI/Visual Direction 尚未撰写**（见 Open Questions）
> **Template**: UX Spec

> **范围边界**：本文件覆盖设置模态本体、四个分区的控件行为、关闭写盘与删除生涯的两步确认。保存指示、只读说明、`Blocked` 模态、暂停菜单、主菜单提示层各有归属；本文件只规定它们的交互边界。判断标准：玩家是否在改**机器级偏好**或**删档**，而不是在改这一局的玩法。

## 引用代号

| 代号 | 出处 |
|---|---|
| `UIF-Rn` / `UIF-ACn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 / Acceptance Criteria 第 n 条 |
| `UIF-UI` | 同文件 `## UI Requirements` 的屏幕契约表（含 SettingsModal 行与初始焦点） |
| `SAVE-Rn` / `SAVE-UI` / `SAVE-ACn` | `design/gdd/save-system.md` |
| `AUD-UI` | `design/gdd/audio-feedback.md` UI Requirements 设置项表 |
| `INP-Rn` / `INP-UI` / `INP-OQn` | `design/gdd/input.md` |
| `PAT-NN` | `design/ux/interaction-patterns.md` 模式编号 |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. Purpose & Player Need

设置是玩家的**调音台与台账抽屉**：把声音、字、动静、HUD 浓淡调到自己能舒服读完一局的程度，并在最底部保留一个「把账本撕掉重来」的抽屉。它必须满足两个相反的期待——**改一下立刻感觉得到**（不等「应用」），**关掉之后下次还在**（写进机器级设置，不随生涯删除）。删除生涯则必须慢：两次确认、默认取消、说清旧档被隔离而非销毁（`SAVE-R23`、`SAVE-UI6`）。

**The player goal**：把音量/字号/动态/HUD 透明度调到能舒服玩，并在需要时安全地删掉当前生涯。

**The game goal**：让偏好即时生效且可持久，让不可逆的删档只发生在玩家两次明确同意之后。

---

## 2. Player Context on Arrival

| Question | Answer |
|---|---|
| What was the player just doing? | 在主菜单点「设置」；或局内按 `ui_pause` 后在暂停菜单点「设置」 |
| What is their emotional state? | 被某件事打断：听不清、字太小、画面晃、HUD 太淡；或准备彻底重来 |
| What cognitive load are they carrying? | 低；但局内打开时仍记着当前回合/净利，担心改设置会丢进度 |
| What information do they already have? | 「设置不在生涯档里」「局内数字不会因设置改变」 |
| What are they most likely trying to do? | 调音量或字号；少数情况下来删档 |
| What are they likely afraid of? | 改坏了、点了没反应、删档误触、设置把存档弄坏 |

**Emotional design target**：像调收音机旋钮——每一下都有即时反馈，关掉面板后世界还是原来的世界，只是更清楚。

---

## 3. Navigation Position

```
任意有界面的状态（Menu 根屏 / 子屏 / Paused）
  └── SettingsModal（模态，modal_depth = 1）
        └── ConfirmModal（删除生涯变体；模态，modal_depth = 2；同一模态内两步，不叠第三层）
```

**Modal behavior**：真模态，阻断下层输入；`push_context(Modal)` 打开、`pop_context()` 关闭（`UIF-R5/27`）。可用 `ui_cancel` 关闭；点击模态外区域**不**关闭（避免误触丢失未写盘改动）；无计时自动关闭（`ACC-COG-03`）。`Blocked` 强制模态期间本屏不可打开（主菜单其它控件不可操作，`PAT-07`）。

**Reachability — all entry points**：

| Entry Point | Triggered By | Notes |
|---|---|---|
| 主菜单「设置」 | 点击 / `ui_accept` | 设置是主菜单一级入口 |
| 暂停菜单「设置」 | 局内 `Paused` 时打开 | 打开设置不恢复 `Run`；关闭后回暂停菜单（`UIF-R15`、`UIF-AC21`） |
| 升级 / 纪录 / 收据 | 不提供入口 | 子屏不叠模态入口；收据是根屏，无设置入口 |

---

## 4. Entry & Exit Points

**Entry table**：

| Trigger | Source Screen / State | Transition Type | Data Passed In | Notes |
|---|---|---|---|---|
| 激活「设置」 | `MainMenu` | 模态打开（淡入 ≤ 180 ms） | 记录 opener 控件 id，用于焦点归还 | 初始焦点「主音量」（`UIF-UI`） |
| 激活「设置」 | `PauseOverlay`（`Paused`） | 同上 | opener id；不改变 `run_state` | 局内数字冻结不变（`PAT-15`） |

**Exit table**：

| Exit Action | Destination | Transition Type | Data Returned / Saved | Notes |
|---|---|---|---|---|
| 「关闭」/ `ui_cancel` | 打开前的屏幕 / 暂停菜单 | 模态关闭（淡出 ≤ 180 ms） | 内存设置值；写机器级设置文件 | 失败 → 「设置可能未保存」，内存值保留 |
| 删除生涯成功 | `MainMenu` | 关闭两层模态，刷新新生涯 | `delete_career(token)` 已执行 | 设置不变；金币 0、纪录空、升级归零 |
| 删除生涯取消 | 本屏删除分区 | 关闭确认模态 | 无 | 焦点回「删除生涯」 |
| 顶层切换 / 退出游戏 | 不经过本屏 | — | 设置写入（UIF-R24） | 模态栈清空 |

> 关闭设置**不等**写盘成功才关闭：写盘失败不阻塞离开，但必须说出「设置可能未保存」（`UIF-R24`、`UIF` Edge Cases）。

---

## 5. Layout Specification

### 5.1 ASCII Wireframe

```
        ┌──────────────────────────────────────────────────────┐
        │ 设置                                                 │
        │                                                      │
        │ 音频                                                 │
        │ 主音量          −12 dB     [──────●────────]         │
        │ 音效音量         −6 dB     [───────●───────]         │
        │ 静音            关         [ 关 ]                    │
        │ 减少重复音效    关         [ 关 ]                    │
        │                                                      │
        │ 无障碍                                               │
        │ 文本缩放        [ 100% ][ 125% ][ 150% ]             │
        │ 减少动态        关         [ 关 ]                    │
        │                                                      │
        │ 显示                                                 │
        │ HUD 不透明度    80%        [────────●──────]         │
        │                                                      │
        │ ─────────────────────────────────────────────────    │
        │ 删除生涯        删除当前生涯与全部纪录，旧档保留为   │
        │                 隔离文件            [ 删除生涯… ]    │
        │                                                      │
        │                                      [ 关闭 ]        │
        └──────────────────────────────────────────────────────┘
```

### 5.2 Layout Zones

| Zone Name | Description | Approximate Size | Scrollable? | Overflow Behavior |
|---|---|---|---|---|
| A 标题 | 「设置」 | 高 8% | 否 | 不截断 |
| B 音频分区 | 两条滑杆 + 两个开关 | 高 30% | 否 | 150% 时标签与控件可换行，控件不缩 |
| C 无障碍分区 | 文本缩放三档 + 减少动态 | 高 20% | 否 | 三档按钮等宽；150% 可换行成两行 |
| D 显示分区 | HUD 不透明度 | 高 12% | 否 | 同 B |
| E 删除分区 | 分隔线 + 说明 + 删除按钮 | 高 15% | 否 | 说明最多三行 |
| F 底栏 | 「关闭」 | 高 10% | 否 | 按钮增高 |

模态宽 720 px（1280 下）、最大高 85% 视口；内容超出时整块 B–F 纵向滚动，标题与「关闭」固定。安全边距 `m = clamp(round(0.05×H), 32, 96)`。

### 5.3 Component Inventory

| Component Name | Type | Zone | Purpose | Required? | Reuses Existing Component? |
|---|---|---|---|---|---|
| 标题 | Label | A | 定位 | Yes | Yes — 模态标题样式 |
| 滑杆行 ×3 | Slider row（PAT-15） | B/D | 音量 ×2、HUD 不透明度 | Yes | No — 新组件 |
| 开关行 ×3 | Toggle row | B/C | 静音、减少重复音效、减少动态 | Yes | No — 新组件 |
| 文本缩放三档 | Segmented control | C | 100/125/150 | Yes | No |
| 删除说明 | Text | E | 隔离文件说明 | Yes | No |
| 「删除生涯…」 | Destructive button | E | 打开两步确认 | Yes | Yes — 危险按钮样式 |
| 「关闭」 | Button | F | 写盘并离开 | Yes | Yes — 菜单按钮 |
| 写入失败条 | Banner | F | 「设置可能未保存」 | 条件 | Yes — `NoticeLayer` 变体 |
| 删除确认模态 | Modal | 覆盖层 | 两步确认 | Yes | Yes — `ConfirmModal`（两步变体） |

**Primary focus element on open**：`master_volume_db` 滑杆（`UIF-UI` SettingsModal 行）。打开时滑杆即获焦，但不改变当前值。

### 5.4 Information Hierarchy

1. **音量**——最高频调整，置顶。
2. **文本缩放与减少动态**——无障碍刚需，紧随其后。
3. **HUD 不透明度**——情境性偏好。
4. **删除生涯**——低频、不可逆，沉底并用分隔线隔离，不与日常调节混排。

---

## 6. States & Variants

| State Name | Trigger | What Changes Visually | What Changes Behaviorally | Notes |
|---|---|---|---|---|
| Default | 打开 | 各控件显示模型当前值 | 初始焦点「主音量」 | 无「未保存」标记，避免焦虑 |
| Dirty | 任一值改变 | 值文本即时更新；无应用按钮 | 关闭时写入；`ui_cancel` 同样写入 | 即时生效（`UIF-R24`） |
| SaveFailed | 关闭时写设置文件失败 | 底栏出现「设置可能未保存」 | 仍关闭；内存值保留；下次修改/退出重试 | 不显示生涯「已保存」指示（`UIF-R21`） |
| DeleteReady | 可写且不在局内 | 删除分区按钮可用 | 点击进入确认第 1 步 | `LoadReport.writable = true` |
| DeleteLocked | 只读 / `lock_live` / 局内 | 按钮灰 + 2 px 左线 + 原因（「本局进行中，结算后可删除生涯」/「另一个实例正在使用存档，无法删除」） | 仍可获焦；提交被拒并重复原因，不弹确认 | `PAT-03`、`SAVE-R21/23`；见 OQ4 |
| ConfirmStep1 | 点击「删除生涯…」 | 覆盖层：标题「删除生涯」、后果说明、「取消」「继续」 | 模态深度 2；默认焦点「取消」 | `PAT-06` |
| ConfirmStep2 | 第 1 步「继续」 | 覆盖层第二步：标题「确认删除生涯」、加粗后果、「取消」「删除生涯」 | 默认焦点「取消」；不叠第三层 | `PAT-06`、`UIF-R24` |
| DeleteBusy | 第 2 步确认 | 两个按钮禁用 | 忽略重复确认，不发起第二笔 | 本地操作，通常一帧内返回 |
| DeleteRefused | `delete_career` 拒绝 / 存储失败 | 确认框内就地显示原因 | 不关闭确认框、不刷新主菜单；按钮恢复 | `PAT-05`；见 OQ9 |
| DeleteSuccess | `delete_career` 成功 | 两层模态关闭 | 主菜单刷新新生涯；设置不变 | `UIF` Edge Cases |
| TextScalePreview | 选择 125% / 150% | 本屏立即重排；焦点保持在原控件 | 所有界面随后按新档排版 | 焦点不丢（`UIF-AC35`） |

> `Blocked` 不是本屏状态：强制模态期间设置不可达；`ReadOnly` 下设置仍可改、可写（设置不属于生涯档，`SAVE-R20`），只有删除被锁。

---

## 7. Interaction Map

### 7.1 Navigation Inputs

| Input | Platform | Action | Visual Response | Audio Cue | Notes |
|---|---|---|---|---|---|
| ↑ / ↓ | 键盘、手柄 | 在控件行之间移动（滑杆/开关/三档组/删除/关闭） | 焦点环 | 无（焦点移动不发声） | 按屏幕位置顺序；重复 `D = 0.35 s`、`I = 0.12 s`（`INP`） |
| ← / → | 键盘、手柄 | 滑杆：步进；文本缩放组：切换档位并立即应用 | 值文本与填充同帧变化 | 无（焦点移动不发声） | 文本缩放切换同时移动焦点到新档 |
| 按住 ← / → | 键盘、手柄 | 连续步进 | 连续更新 | 同上 | 不加速；无模拟量精度要求 |
| 滚轮 | 鼠标 | 内容超出时滚动；不调值 | 内容滚动，焦点不动 | 无 | 防止误调（见 OQ1） |
| 鼠标点击滑杆 | 鼠标 | 跳到点击位置的值 | 聚焦该滑杆 + 值更新 | 无（焦点移动不发声） | 命中区 ≥ 44 px 高 |
| 鼠标点击开关/档位 | 鼠标 | 切换并立即生效 | 状态文本翻转 | 激活 cue | 悬停与焦点形状可区分 |

### 7.2 Action Inputs

| Input | Platform | Context | Action | Response | Animation | Audio Cue | Notes |
|---|---|---|---|---|---|---|---|
| `ui_accept` | 键盘、手柄 | 「关闭」 | 写设置并关闭 | 模态淡出，焦点归还 | ≤ 180 ms | 按钮激活一次 | 发 `ui_button_activated(settings_close)` |
| `ui_accept` | 键盘、手柄 | 「删除生涯…」可写 | 打开确认第 1 步 | 覆盖层出现，焦点「取消」 | ≤ 180 ms | 按钮激活一次 | 深度 2 |
| `ui_accept` | 键盘、手柄 | 确认第 1 步「继续」 | 进入第 2 步 | 文案加强，焦点仍「取消」 | 0（同框换内容） | 激活 cue | 不叠第三个模态 |
| `ui_accept` | 键盘、手柄 | 第 2 步「删除生涯」 | 调 `delete_career(confirm_token)` | 成功关闭两层；失败就地给原因 | 0 | 激活 cue | 令牌由第 2 步生成 |
| `ui_cancel` | 键盘、手柄 | 任一确认步骤 | 取消并回设置 | 覆盖层关闭，焦点回「删除生涯…」 | ≤ 180 ms | 无 | 第 1 步取消 = 不删 |
| `ui_cancel` | 键盘、手柄 | 设置本体 | 写设置并关闭 | 回打开前屏幕，焦点归还 | ≤ 180 ms | 无 | 从暂停打开则回暂停菜单，不恢复 `Run` |
| `ui_accept` | 键盘、手柄 | 删除按钮（Locked） | 拒绝 | 原因句保持，无确认框 | 无 | 无 | `PAT-03` 数据态禁用 |

### 7.3 State-Specific Behaviors

| State | Input Restriction | Reason |
|---|---|---|
| DeleteLocked | 删除按钮不打开确认；其余设置可改 | 只读/持锁/局内不允许删档（`SAVE-R23`） |
| DeleteBusy | 确认与取消都禁用 | 不发起第二笔删除 |
| SaveFailed | 无输入锁；提示常驻到关闭 | 失败不阻塞离开 |
| Confirm 两步 | 焦点陷阱在覆盖层内；下层设置不可操作 | `UIF-R5`、`PAT-02` |

---

## 8. Data Requirements

| Data Element | Source System | Update Frequency | Who Owns It | Format | Null / Missing Handling |
|---|---|---|---|---|---|
| `master_volume_db` | 设置模型（界面流程） | 每次 ←/→、点击、拖拽 | 界面流程 | −80…0，步长 5；`−80` 显示「静音」 | 越界钳制；不显示 `−∞` |
| `sfx_volume_db` | 设置模型 | 同上 | 界面流程 | −40…0，步长 5 | 同上 |
| `muted` | 设置模型 | 开关切换 | 界面流程 | 开 / 关 | — |
| `reduce_repetitive_cues` | 设置模型 | 开关切换 | 界面流程 | 开 / 关，默认关 | — |
| `text_scale_percent` | 设置模型 | 选档 | 界面流程 | 100 / 125 / 150，无第四档 | 非法值回退 100 并记 WARNING |
| `reduce_motion` | 设置模型 | 开关切换 | 界面流程 | 开 / 关 | — |
| `hud_opacity_percent` | 设置模型 | 滑杆 | 界面流程 | 60–100，步长 10；文字 α 恒 1.0 | 越界钳制（`ACC-VIS-14`） |
| 生涯可写性 | `save.LoadReport.writable` | 打开本屏时 | 存档 | 布尔 | `false` → 删除禁用 + 原因 |
| 写锁活性 | `LoadReport.lock_live` | 打开本屏时 | 存档 | 布尔 | `true` → 删除禁用 + 原因 |
| 删除结果 | `save.delete_career(token)` | 第 2 步确认时 | 存档 | `Result` | 拒绝 → 就地原因；不刷新主菜单 |
| 设置写盘结果 | 设置文件写入 | 关闭 / 退出时 | 界面流程（设置模型） | 成功 / 失败 | 失败 → 「设置可能未保存」 |

**禁止**：本屏不读钱包、纪录、升级目录；不把设置写进生涯档；不调用 `commit`；不直接操作 `Master` 总线（音量由 `音效反馈` 消费，`AUD-UI`）。

---

## 9. Events Fired

| Player Action | Event Fired | Payload | Receiver System | Notes |
|---|---|---|---|---|
| 打开设置 | `modal_opened(SettingsModal, 1)` | `modal_id`、`modal_depth` | 音效反馈 / QA | 由界面流程发 |
| 关闭设置 | `modal_closed(SettingsModal, restored_focus_id)` | 模态 id、归还焦点 id | 音效反馈 / QA | `pop_context()` 配对 |
| 点击「关闭」 | `ui_button_activated(settings_close)` | `control_id` | 音效反馈 | 每次提交恰好一次 |
| 打开删除确认 | `modal_opened(ConfirmModal, 2)` | 同上 | 音效反馈 / QA | 删除生涯变体；两步在同一模态内 |
| 第 2 步确认 | `delete_career(confirm_token)` | `confirm_token` | 存档 | 仅此路径可调用；无令牌拒绝 |
| 删除成功 | `modal_closed(...)` ×2 | 模态 id | 音效反馈 / QA | 随后主菜单刷新 |
| 值改变 | 设置模型 setter（内部） | `key`、`value` | 音效反馈 / HUD / 全部界面 | 立即消费；不写生涯档 |
| 关闭写盘失败 | 无公开事件（提示层） | — | — | 见 OQ6 |

---

## 10. Transitions & Animations

| Transition | Trigger | Direction / Type | Duration (ms) | Easing | Interruptible? | Skipped by Reduced Motion? |
|---|---|---|---|---|---|---|
| 模态打开 | 主菜单 / 暂停菜单 | 淡入 + 焦点移入 | ≤ 180 | ease-out | 否 | 是 → 0 ms |
| 模态关闭 | 关闭 / `ui_cancel` | 淡出 + 焦点归还 | ≤ 180 | ease-in | 否 | 是 → 0 ms |
| 打开删除确认 | 点击删除 | 覆盖层淡入 | ≤ 180 | ease-out | 否 | 是 |
| 第 1 → 第 2 步 | 「继续」 | 同框换文案，无滑动 | 0 | — | — | — |
| 改值 | 滑杆/开关/档位 | 值文本与填充同帧 | 0 | — | — | — |
| 文本缩放重排 | 选 125/150% | 同帧重排，焦点保留 | 0 | — | — | — |

> 数值与状态必须同帧落定，动画只叠加在新值上（`PAT-04`）；任何过渡不得推迟设置生效或焦点出现（`UIF-R26`）。

---

## 11. Input Method Completeness Checklist

**Keyboard**
- [ ] 全部九个控件（三条滑杆、三个开关、三档文本缩放、删除、关闭）都可用 ↑/↓ 到达，←/→ 调值
- [ ] 只用键盘可完成调音量、改字号、开减少动态、关设置，以及两步删档
- [ ] 焦点始终可见；打开时焦点在「主音量」，关闭后回打开它的按钮
- [ ] `ui_cancel` 先取消确认框，再关闭设置；从不退出游戏
- [ ] 确认框内焦点循环，不逃到下层设置

**Gamepad**
- [ ] 十字键/左摇杆可达全部控件；A 确认、B 取消
- [ ] 滑杆与三档组均可只用十字键调整；无模拟量精度要求
- [ ] 手柄断开时焦点停在原控件
- [ ] 不绘制键位图标，只写动作语义

**Mouse**
- [ ] 每条滑杆、开关、档位、按钮有悬停态，且与焦点态形状可区分
- [ ] 可点目标 ≥ 44×44 px；滑杆行高 ≥ 56 px；按钮 ≥ 56 px 高
- [ ] 点击模态外不关闭；滚轮滚动内容而不调值
- [ ] 右键无动作

**Touch**：不适用——无触屏目标，不提供虚拟摇杆（`INP-OQ5`）。

---

## 12. Accessibility

**Text contrast requirements**（实测值，来自 `ACC` 对比度表）：

| Text Element | Background Context | Required Ratio | Current Ratio | Pass? |
|---|---|---|---|---|
| 标签、值文本、分区标题 | 米白 `#F5F1E6` | ≥ 4.5:1 | 墨黑 14.6:1 | 是 |
| 次级说明（删除后果） | 米白 | ≥ 4.5:1 | 深灰 `#4A4E4A` 7.5:1 | 是 |
| 禁用删除原因句 | 米白 | ≥ 4.5:1 | 墨黑 100% 不透明 | 是 |
| 选中档位文字 | 墨黑填充 `#1F1F1C` | ≥ 4.5:1 | 米白 14.6:1 | 是 |
| 滑杆填充 / 焦点环 | 米白 | ≥ 3:1 | 交互蓝 4.9:1；轨道用 2 px 墨黑边（14.6:1） | 是 |

> 冷灰 `#8C918E` 在米白上 2.8:1，**不承载文字、不做滑杆轨道**（`ACC-VIS-04/05`）。

**Colorblind-safe elements**：

| Element | Colorblind Risk | Mitigation |
|---|---|---|
| 开关开/关 | 绿/灰 | 文字「开 / 关」+ 滑块位置（左/右） |
| 文本缩放选中档 | 仅底色 | 反白填充块 + 2 px 边框；灰度下仍是实心块 |
| 禁用删除 | 灰 | 原因句 + 2 px 左线；仍可获焦 |
| 焦点 vs 选中 | 蓝 vs 墨黑 | 焦点环在外、选中块在内，形状不同（`PAT-01`） |
| 「设置可能未保存」 | 颜色 | 文字整句 |

**Focus visibility**：2 px 实线外框 + 2 px 间距，永不被裁切；蓝环落在反白档位上时外加 2 px 墨黑隔边（`ACC-VIS-05`）。

**Text scaling**：100% / 125% / 150%。本屏自身即时重排：滑杆标签与值可换行、三档按钮可折成两行、删除说明最多三行、内容超出时纵向滚动且焦点不丢。三档均不得横向滚动、截断或重叠（`ACC-VIS-09/10`）。

**Cognitive**：7 个偏好 + 1 个破坏性入口，按三分区呈现；无计时器、无自动关闭、无「应用」确认步骤；确认框 60 s 不操作仍保持（`ACC-COG-02/03`）。

**Grayscale**：去色后仍能读出每个开关状态、选中档位、滑杆值、删除是否可用及原因（`ACC-VIS-08`）。

**Screen reader**：本作不声称设置可被 OS 读屏使用（`ACC` Known Limitations）；补偿为文字化状态、固定初始焦点、全键盘/手柄可达。

---

## 13. Localization

- 所有玩家可见字符串走 `tr()`；分区标题、开关状态、原因句、确认文案都来自文本表（`ACC-COG-08`）。
- 中文为基准；允许 40% 扩写。标签上限 8 字；删除说明允许 3 行，超长按原文换行不截断。
- 数值格式：`−12 dB`（U+2212、整数、无小数）、`125%`、`80%`；`−80 dB` 写「静音」，不写 `−∞`（`PAT-15`）。
- 确认文案必须包含「旧档会被保留为隔离文件，可在文件目录中找回」（`SAVE-UI6`），不得改写成语义更弱的「删除后不可恢复」。
- 不拼接句子式数字；「当前 80%」中的数字来自读数，便于语序变化。

---

## 14. Acceptance Criteria

**入口、焦点与模态**

- [ ] `SET-AC01` 从主菜单与暂停菜单打开时初始焦点均为「主音量」滑杆（`UIF-AC37`）；打开不改变任何设置值。
- [ ] `SET-AC02` 设置在 `modal_depth = 1`；删除确认在 `modal_depth = 2`；不存在第三层；每次 `push_context` 都有配对 `pop_context`（`UIF-AC39`）。
- [ ] `SET-AC03` 关闭设置后焦点归还打开它的按钮；从暂停菜单打开时关闭只回暂停菜单，不恢复 `Run`（`UIF-AC21`）。
- [ ] `SET-AC04` 点击模态外区域不关闭；无计时器、无自动关闭；`ui_cancel` 先取消确认、再关闭设置，从不退出游戏。

**设置值与即时生效**

- [ ] `SET-AC05` 七个设置项与范围逐项符合 `UIF-R24` / `AUD-UI`：主音量 −80…0、音效 −40…0、静音、减少重复音效（默认关）、文本缩放 100/125/150、减少动态、HUD 不透明度 60–100；不多出任何选项（无重绑、无色盲开关）。
- [ ] `SET-AC06` 改值同帧生效：音频下一次起播即用新值；文本缩放立即重排本屏；`reduce_motion` 立即把界面过渡与收据滚动置 0；HUD 不透明度立即被 HUD 消费，文字 α 仍为 1.0。
- [ ] `SET-AC07` `master_volume_db = −80` 显示「静音」，其他值显示 `−n dB`；`muted` 另有开关状态；不显示 `−∞`。
- [ ] `SET-AC08` 滑杆 ←/→ 步进为主音量/音效 5 dB、HUD 10%，按住重复不加速；键盘与手柄均可调到两端；鼠标点击/拖拽可达同一值域。
- [ ] `SET-AC09` 文本缩放只有三档；选中态用反白块 + 边框，不只靠颜色；切换后焦点仍在文本缩放控件上（`UIF-AC35`）。

**写盘与删档**

- [ ] `SET-AC10` 关闭或退出时写机器级设置文件；失败显示「设置可能未保存」并保留内存值；**不**显示生涯「已保存」指示（`UIF-AC27`）。
- [ ] `SET-AC11` 设置不进入生涯档：删除生涯后设置文件逐字节不变；设置写入不调用 `commit`（`SAVE-AC25`）。
- [ ] `SET-AC12` 「删除生涯」需两次确认，两步默认焦点均为「取消」；单次点击绝不删除；第 2 步按钮文案为「删除生涯」。
- [ ] `SET-AC13` 删除说明写明旧档被改名隔离保留、可在文件目录找回（`SAVE-UI6`）。
- [ ] `SET-AC14` 只读、持锁或局内时删除按钮不可用且给出原因（仍可获焦）；提交只重复原因，不打开确认框（`SAVE-R23`）。
- [ ] `SET-AC15` 删除成功：两层模态关闭、主菜单刷新新生涯（钱包 0、纪录空、升级归零），设置保持原值（`UIF-AC34`）。
- [ ] `SET-AC16` 删除被拒或失败：确认框不关闭、就地显示原因、按钮恢复、不刷新主菜单、不改任何数据（`PAT-05`）。
- [ ] `SET-AC17` `delete_career` 只在第 2 步确认后调用恰好一次，且携带确认令牌；其余路径零调用。

**无障碍与本地化**

- [ ] `SET-AC18` 所有文字/背景组合实测 ≥ 4.5:1；滑杆填充与焦点环 ≥ 3:1；开关与选中态均有颜色之外的通道（`ACC-VIS-06/08`）。
- [ ] `SET-AC19` 100/125/150% 三档无横向滚动、无截断、无重叠；内容超出时纵向滚动且焦点不丢（`UIF-AC35`）。
- [ ] `SET-AC20` 焦点环完整可见；命中区 ≥ 44×44 px；滑杆与按钮 ≥ 56 px 高；所有字符串来自 `tr()`。
- [ ] `SET-AC21` 打开到可交互 ≤ 200 ms；调值无可感知掉帧；删除为本地操作，无加载态。

---

## 15. Open Questions

| # | 问题 | 归属 | 何时解决 | 状态 |
|---|---|---|---|---|
| 1 | **滑杆步进/加速曲线归属**：`interaction-patterns` 把「滑杆步进与加速曲线」列为 Gaps（其 OQ2）。本规格暂定主音量/音效 5 dB、HUD 10%，按住不加速。若模式库后续定义统一曲线，本屏跟随。 | ux-designer | Phase 4 | 未决 |
| 2 | **设置写入时机与文件**：`UIF-OQ3` 未定格式/路径/每次改即写还是关闭写；本规格按「关闭或退出时写、失败保留内存值」执行。 | lead-programmer + ux-designer | 架构阶段 | 未决 |
| 3 | **静音与 −80 dB 的关系**：模型同时有 `muted` 与 `master_volume_db = −80`，两者都可静音；本规格让两者独立，滑杆拉到 −80 不自动改 `muted`。是否需要在静音开关旁显示「主音量已为 −80」？ | audio + ui-flow | 写 epic 前 | 未决 |
| 4 | **局内删除生涯**：设置可从暂停菜单打开，但 `UIF` Edge 只描述「删除后主菜单刷新」。本规格在 `Paused` 时禁用删除并给原因，避免删档后继续一局。需确认此限制是否写入 `UIF-R24`。 | game-designer + ui-flow | 写 epic 前 | 未决 |
| 5 | **两次确认的形态**：本规格用同一模态内两步（深度 2 不变），避免第二次 `push_context(Modal)` 的语义争议（`UIF-OQ14`）。若输入层要求两次独立压栈，则本屏需改。 | input + ui-flow | 架构阶段 | 未决 |
| 6 | **「设置可能未保存」的落点**：关闭后回主菜单有 `NoticeLayer`，可显示；从暂停菜单关闭时 `PauseOverlay` 是否也有提示层未定义。暂定：在主菜单用非模态提示，暂停中用底栏条保留到回主菜单。 | ui-flow | 写 epic 前 | 未决 |
| 7 | **文本缩放控件形态**：模式库只定义滑杆行（PAT-15），未定义三档分段控件。本规格用三按钮反白选中；若实现成本高可退回「三档滑杆」，但必须显示 `100/125/150` 三值。 | ux-designer + ui-programmer | 垂直切片前 | 未决 |
| 8 | **Standard 缺口在设置中无入口**：重绑（`ACC-MOT-02`）、色盲模式（设计上不需要）、教学回看（`ACC-COG-05`）都不在设置里。本规格不添加灰色占位项；需确认「设置里没有」是可接受的缺口表达。 | accessibility + ux-designer | 上线前评审 | 未决 |
| 9 | **删除被拒的反馈层级**：本规格用确认框内就地原因（`PAT-05`），而不是关闭后主菜单提示。若拒绝原因是「另一实例持锁」，玩家可能需要退出重试；是否需要给「退出游戏」出口？本规格不提供（避免从设置直接退出）。 | ux-designer + save 作者 | 写 epic 前 | 未决 |
| 10 | **是否提供「重置设置」**：`SAVE` Edge Cases 提到「重置设置」但无 UI Requirements。本规格不提供；若试玩发现玩家调乱后无法恢复，再评估加入（需单次确认）。 | ux-designer + producer | 首轮试玩后 | 未决 |
