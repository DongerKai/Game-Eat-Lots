# 输入（Input）

> **Status**: In Design
> **Author**: user + agents
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 时间在走，手不用快；数字看得见地涨

## Summary

输入把鼠标、键盘和手柄翻译成一组固定动作，并由上下文状态机决定当前界面响应哪些动作。它保证每个界面与局内操作都能只用键盘或手柄完成，且没有任何操作依赖反应速度。玩家不会直接看到这个系统，但每一次点菜、跳过和翻页都要经过它。

> **Quick reference** — Layer: `Foundation` · Priority: `MVP` · Key deps: `None`

## Overview

输入系统把鼠标、键盘和手柄翻译成一组固定的动作，并把"当前该响应哪些动作"交给上下文状态机（Menu / Run / Paused / Modal / Blocked）。所有其他系统只问"accept 是否刚按下"，从不读取设备状态或原始事件。

它保证每个界面和局内操作都能只用键盘或手柄完成——手柄支持是完整的，鼠标只是可选加速器——同时确保没有任何操作依赖反应速度（支柱 3「时间在走，手不用快」）。

Godot 4.6 起鼠标焦点与键盘/手柄焦点分离，因此输入层负责分别记录两种焦点，由界面分别绘制；Godot 4.7 又改了设备 ID，所以设备一律按事件类型识别，不比较 `device == 0`。

## Detailed Design

### Core Rules

1. **固定动作集**：全局只有下表这些动作。其他系统只能查询动作，不得自行监听原始事件。

   | 动作 | 键盘 | 手柄 | 鼠标 |
   |---|---|---|---|
   | `ui_up` / `ui_down` / `ui_left` / `ui_right` | 方向键、WASD | 左摇杆、十字键 | — |
   | `ui_accept` | Enter、Space | A | 左键（悬停的控件） |
   | `ui_cancel` | Backspace | B | — |
   | `ui_pause` | Esc（Run 上下文）、P | Start | — |
   | `game_skip_dish` | S | X | 右键（悬停的菜品卡） |

2. **上下文决定语义**：Esc 同时绑定 `ui_cancel` 与 `ui_pause`；Run 上下文读 `ui_pause`，Menu / Modal 上下文读 `ui_cancel`。
3. **初始焦点**：每个界面打开时把键盘/手柄焦点放在主操作上（第一道菜、第一项升级、主按钮）。
4. **完整可玩**：每个界面与局内操作都必须能只用键盘或手柄完成；鼠标是可选加速器。
5. **触发时机**：动作在按下瞬间触发一次。忽略操作系统按键回显（`event.echo`）；长按不重复触发 `ui_accept` 或 `game_skip_dish`。
6. **导航重复**：按住方向超过延迟后开始重复（公式见 Formulas）。
7. **摇杆回滞**：摇杆导航进入阈值为 0.60、释放阈值为 0.40，防止漂移造成连续导航。
8. **双焦点**：鼠标悬停与键盘/手柄焦点分开记录、分开绘制。鼠标移动不抢键盘焦点，键盘导航不清除悬停。
9. **设备识别**：按事件类型判断，绝不比较 `device == 0`（Godot 4.7 起鼠标/键盘使用 `DEVICE_ID_MOUSE` / `DEVICE_ID_KEYBOARD`）。
10. **无重绑（MVP）**：绑定集中在一张映射表里；重绑是以后替换这张表，不改变动作集。
11. **输入只读**：局内系统只查询"某动作是否刚触发"，不持有设备或原始事件对象。

### States and Transitions

| 上下文 | 何时进入 | 响应动作 | 屏蔽 |
|---|---|---|---|
| Blocked | 加载、首局引导、结算动画 | `ui_pause`（若该界面允许） | 其余全部 |
| Menu | 主菜单、结算、升级、设置 | 导航、`ui_accept`、`ui_cancel` | `game_skip_dish`、`ui_pause` |
| Run | 一局进行中且无弹窗 | 导航、`ui_accept`、`game_skip_dish`、`ui_pause` | — |
| Paused | Run 中按 `ui_pause` | `ui_pause`（恢复）、`ui_accept`（激活焦点控件，默认「继续」）、`ui_cancel`（打开退出确认） | `game_skip_dish` |
| Modal | 任意弹窗（设置、确认框） | 导航、`ui_accept`、`ui_cancel` | 其余全部 |

**转换**：Run → Paused（`ui_pause`）；Paused → Run（`ui_pause`，或界面流程消费 `ui_accept` 后请求）；任意 → Modal（界面请求）；Modal → 来源上下文（`ui_accept` 确认或 `ui_cancel` 关闭）；Run → Menu（结算完成）；Menu → Run（开始一局）。

### Interactions with Other Systems

| 系统 | 方向 | 数据/接口 | 硬/软 |
|---|---|---|---|
| 界面流程 | 下游读取 | 焦点、导航、`ui_accept`、`ui_cancel` | 硬 |
| 餐局流程 | 下游读取 | 每回合的 `ui_accept`、`game_skip_dish`、`ui_pause` | 硬 |
| 局内 HUD | 下游读取 | 菜品卡的悬停、点击、右键 | 硬 |
| 首局引导 | 下游读取 | 请求进入 Blocked 上下文，只放行引导需要的动作 | 软 |
| 音效反馈 | 下游读取 | 动作触发事件（用于 UI 音效） | 软 |

**接口所有权**：输入拥有动作映射、上下文状态机与双焦点记录；下游只查询动作与焦点，不修改映射表。

## Formulas

### 导航重复步数

The `nav_steps` formula is defined as:

`nav_steps = 1 + max(0, floor((T − D) / I) + 1)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `T` | float | 0–∞ 秒 | calculated | 方向键/摇杆按住时长 |
| `D` | float | 0.25–0.60 秒 | data file（`repeat_delay`） | 首次重复前的延迟 |
| `I` | float | 0.08–0.20 秒 | data file（`repeat_interval`） | 重复间隔 |
| `nav_steps` | int | 1–∞ | calculated | 含首次按下的总步数 |

**Output Range:** 最小 1（`T < D` 时恒为 1），无上界。重复在 `t = D + k·I ≤ T` 时触发；边界统一取 `≤`，避免两套实现差一步。

**Example:** `T=1.0`、`D=0.35`、`I=0.12` → `floor(0.65 / 0.12) = 5`，总步数 `1 + 5 + 1 = 7`；触发点 0.35 / 0.47 / 0.59 / 0.71 / 0.83 / 0.95 秒。

### 摇杆进入判定

The `stick_engaged` formula is defined as:

`stick_engaged = (m ≥ E) ∨ (stick_engaged_prev ∧ m ≥ R)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `m` | float | 0.0–1.0 | calculated | 摇杆合成幅度 |
| `E` | float | 0.50–0.75 | data file（`stick_engage`） | 进入阈值 |
| `R` | float | 0.25–0.50 | data file（`stick_release`） | 释放阈值 |
| `stick_engaged_prev` | bool | true / false | calculated | 上一帧的判定结果 |
| `stick_engaged` | bool | true / false | calculated | 本帧是否算作导航输入 |

**Output Range:** `{false, true}`。回滞带为 `[R, E)`，释放发生在 `m < R`。约束 `R ≤ E − 0.10`：`E = R` 会抖动，`R ≥ E` 则永远无法释放。

**Example:** `prev=false, m=0.55` → false；`prev=true, m=0.55` → true；`prev=true, m=0.35` → false。

### 双击抑制：不设

动作只在按下沿触发、已忽略操作系统回显、长按不重复，因此不存在"一按两触发"。在回合制里，再次按下就是玩家的意图，计时器抑制反而会像丢输入。防重复靠"事件在消费它的上下文内一次性消费"，不靠冷却时间。

若日后确实需要，`accept_cooldown` 默认 `0.00` 秒，安全范围 `0.00–0.12` 秒；超过 0.12 秒开始有丢手感的风险。

### 调参起点

| 参数 | 起点值 | 备注 |
|---|---|---|
| `D`（repeat_delay） | 0.35 秒 | 倾向 0.35–0.45，宽容优先（支柱 3） |
| `I`（repeat_interval） | 0.12 秒 | 倾向 0.12–0.15 |
| `E`（stick_engage） | 0.60 | 与 `R` 的差值不小于 0.10 |
| `R`（stick_release） | 0.40 | 倾向 0.30–0.35 更防误触 |

以上全部需要在垂直切片阶段由试玩确定，不凭公式定死。

## Edge Cases

- **If 窗口失去焦点**（Alt-Tab、点击别的窗口）：输入层不生成任何动作，上下文保持不变。是否暂停由场景与流程管理决定（当前设计：**不自动暂停**——回合制没有实时压力，见 scene-flow.md）。
- **If 手柄在局中拔出**：输入层停止接受该设备的事件，按下中的动作按释放处理，上下文保持不变。是否暂停由场景与流程管理决定（当前设计：**不自动暂停**）；重连后继续，键盘仍可操作。
- **If 手柄在局中接入**：不改变当前状态，新设备立即开始工作。
- **If 鼠标与键盘/手柄同时操作**：双焦点各自记录；以最近一次输入的方式决定当前绘制哪种焦点样式，任何动作都不会丢失。
- **If 长按 `ui_accept` 或 `game_skip_dish`**：只触发一次，不重复。
- **If 操作系统发出按键回显**（`event.echo == true`）：忽略，不产生第二次动作。
- **If 弹窗打开时按 Esc**：按 `ui_cancel` 关闭弹窗，不触发暂停。
- **If 当前没有任何控件获得焦点时按 `ui_accept`**：无操作，不报错。
- **If 点击落在任何菜品卡之外**：无操作，不消耗时间格。
- **If 鼠标未悬停在菜品卡上时按跳过**（右键 / X / S）：无操作。
- **If 摇杆幅度停在 0.40–0.60 之间**：保持上一次的进入/释放状态（回滞），不来回抖动。
- **If 两个设备同时触发同一动作**：只产生一次动作。
- **If 输入发生在 Blocked 上下文**：除该界面允许的暂停外一律丢弃。
- **If 上下文切换时方向键正被按住**：旧按键不会延续到新上下文，必须重新按下才产生导航。
- **If 输入事件的 `device` 为 0**：不得据此判断设备类型——4.7 起鼠标/键盘使用专用常量，一律按事件类型识别。

## Dependencies

**上游依赖：无。** 输入是基础层系统，只依赖 Godot 的 Input 与焦点机制。

**下游依赖方：**

| 系统 | 硬/软 | 接口 | 状态 |
|---|---|---|---|
| 界面流程 | 硬 | 导航、`ui_accept`、`ui_cancel`、初始焦点、双焦点状态 | GDD 未写 |
| 餐局流程 | 硬 | 每回合的 `ui_accept`、`game_skip_dish`、`ui_pause` | GDD 未写 |
| 局内 HUD | 硬 | 菜品卡的悬停、点击、右键 | GDD 未写 |
| 首局引导 | 软 | 请求 Blocked 上下文，只放行指定动作 | GDD 未写 |
| 音效反馈 | 软 | 动作触发事件 | GDD 未写 |
| 场景与流程管理 | 硬 | 顶层状态切换时通知进入默认上下文并清空临时栈；由它订阅 `ui_pause` 驱动 `Run ⇄ Paused` | GDD 已写 |

**双向一致（已完成）**：索引第 5、15、16 行的 `Depends On` 已分别记录「输入」；场景与流程管理的 GDD 也已完成并把「输入」列为上游依赖，索引第 3 行已记为「数据配置, 输入」。以上系统编写各自 GDD 时必须同样列出「输入」为上游依赖。

## Visual/Audio Requirements

输入本身不绘制、不播放任何内容，但它定义了界面必须表现的两种焦点状态：

- **键盘/手柄焦点**：必须有清晰、持续的描边或高亮，不依赖颜色单一通道（色盲可用）。
- **鼠标悬停**：使用与焦点不同的样式，两者可以同时出现且可区分（Godot 4.6 双焦点）。
- **无效操作**：点击空白处不产生视觉反馈，也不消耗时间格。

音效由「音效反馈」系统负责；输入只发出动作事件，不自己选音。

## Game Feel

- **即时**：按下到反馈不超过 1 帧（60 FPS 下 16.7 ms），不做输入缓冲或取消窗口——回合制里缓冲会像丢输入。
- **可预测**：焦点移动是离散的，不加速、不滑动；每次移动一格。
- **重复节奏**：首次重复要慢到不会误跳（0.35 s），之后要快到不烦（0.12 s）。
- **无惩罚**：点空白、无焦点时按确认都不扣时间、不扣钱，只当作没发生。
- **设备一致**：键盘和手柄走同一套焦点与动作逻辑，手感不应因设备不同而改变。

## UI Requirements

输入向下游界面提出以下要求：

- 每个可交互控件都必须能被键盘/手柄焦点到达；禁用控件不可获焦。
- 焦点顺序按屏幕位置（上/下/左/右），不按节点树顺序。
- 每个界面必须有初始焦点；弹窗打开时焦点移入弹窗，关闭后返回原来的控件。
- 界面不得自行监听原始输入事件，只能查询动作与焦点状态。
- 不提供虚拟摇杆或触屏操作（见 Open Questions 第 5 条）。

## Cross-References

**None — this system references no other GDD.** 输入不依赖数据配置或其他已写文档；它被界面流程、餐局流程、局内 HUD 引用。场景与流程管理的 GDD 已完成并已回指本系统；其余文档写好后需要在本表补充反向链接。

## Acceptance Criteria

**测试夹具**（一次定义，下文引用）：F1 输入探针（记录动作名、原始事件、时间戳）；F2 InputMap 导出；F3 回显注入器；F4 重复时钟；F5 摇杆合成器；F6 双焦点测试界面；F7 设备事件合成器；F8 上下文 harness；F9 全界面清单（逐屏记录主节点路径）。

1. **GIVEN** F1 + F2，**WHEN** 启动，**THEN** InputMap 恰有 8 个动作（`ui_up` / `ui_down` / `ui_left` / `ui_right` / `ui_accept` / `ui_cancel` / `ui_pause` / `game_skip_dish`），没有其他游戏动作。
2. **GIVEN** F8 处于 Run，**WHEN** 按 Esc，**THEN** 只触发 `ui_pause`，`ui_cancel` 计数为 0；**WHEN** 打开 Modal 后按 Esc，**THEN** 只触发 `ui_cancel`，不进入 Paused。
3. **GIVEN** F9，**WHEN** 每个界面首次显示，**THEN** `focus_owner` 等于该屏清单里写死的主节点路径，手柄按 A 立即生效。
4. **GIVEN** F9 且鼠标已拔出，**WHEN** 只用键盘或只用柄走完一局，**THEN** 能到达结算，全程不需要鼠标。
5. **GIVEN** F3，**WHEN** 按住 `ui_accept` 0.5 秒并注入 `event.echo=true` 五次，**THEN** 动作计数为 1；`game_skip_dish` 同理。
6. **GIVEN** F4，`D=0.35`、`I=0.12`，**WHEN** `T=1.00`，**THEN** `nav_steps=7`，触发点为 0.35 / 0.47 / 0.59 / 0.71 / 0.83 / 0.95 秒；`T=0.35` 时 **THEN** 为 2（边界取 `≤`）；`T=0.34` 时 **THEN** 为 1。
7. **GIVEN** F5，**WHEN** `(prev=false, m=0.55)` **THEN** false；`(prev=true, m=0.55)` **THEN** true；`(prev=true, m=0.35)` **THEN** false；`(prev=true, m=0.40)` **THEN** true；`(prev=false, m=0.60)` **THEN** true。
8. **GIVEN** F6，**WHEN** 鼠标悬停在卡片 B 后按方向下，**THEN** 悬停仍在 B、键盘焦点已移动，两套样式同时绘制；**WHEN** 移动鼠标，**THEN** 键盘焦点不变。
9. **GIVEN** F7，**WHEN** 合成 `device=0` 的 `InputEventKey` 与 `InputEventMouseMotion`，**THEN** 按事件类型判为键盘/鼠标；**GIVEN** `device=0` 的手柄事件，**THEN** 不误判。源码搜索 `device ==` 命中 0 次。
10. **GIVEN** F2，**WHEN** 扫描 InputMap 的写入点，**THEN** 只有中央表 1 处、8 个动作，重绑入口为 0。
11. **GIVEN** 下游探针场景，**WHEN** 调用输入 API，**THEN** 只暴露 `is_action_just_pressed(action) -> bool`；静态扫描中 `InputEvent` 引用只出现在输入层。
12. **GIVEN** F8，**WHEN** Run → Paused → Run、Run → Modal → 来源上下文，**THEN** 动作白名单与状态表一致，Blocked 上下文全部丢弃；**WHEN** 窗口失焦或手柄拔出，**THEN** 上下文不变、不产生任何动作；手柄拔出后键盘仍能完成 `ui_accept`。
13. **GIVEN** 方向键保持按住，**WHEN** 切换上下文再切回，**THEN** 不产生导航，必须释放后重按。
14. **GIVEN** 同一帧按下手柄 A 与 Enter，**WHEN** 两者映射到同一动作，**THEN** 动作计数为 1。
15. **GIVEN** 60 FPS 目标与同帧轮询，**WHEN** 从 F1 时间戳到消费者查询，**THEN** 延迟 ≤ 1 帧（16.7 ms）。该预算依赖 `project.yaml` 的目标帧率与输入轮询方式，不测主观手感。
16. **GIVEN** 任意界面停在 `ui_accept` 可用状态，**WHEN** 等待 10 秒后再按，**THEN** 结果与立即按下相同——没有任何接受倒计时。
17. **GIVEN** F6 的双焦点绘制，**WHEN** 界面渲染，**THEN** 保留截图到 `production/qa/evidence/`，由视觉 gate 确认两种焦点样式可区分。

**无法直接断言、已改写或转为视觉门的规则**：
- 「输入只读」：黑盒不可判 → 第 11 条改为 API 形状 + 静态扫描。
- 「完整可玩」：无清单不可判 → 第 4 条依赖 F9 逐屏清单并拔掉鼠标。
- 「初始焦点在主操作」：主操作未定义 → 第 3 条要求逐屏写死节点路径。
- 「不依赖反应速度」：主观 → 第 16 条改为"等待 10 秒结果不变"。
- 「双焦点分开绘制」：属于视觉 → 第 17 条走截图 + 视觉 gate。

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | 是否要做按键重绑？首版不做，倾向 Full Vision 或上线前再加。 | user | 发布前 |
| 2 | 手柄按钮图标（A/B/X/Start）是否需要按设备类型替换？ | ux-designer | 界面流程 GDD 编写时 |
| 3 | 导航重复的 0.35s / 0.12s 是否需要 playtest 调整？ | game-designer | 垂直切片 |
| 4 | 是否需要"按住连续点菜"？回合制下倾向不做。 | user | 垂直切片 |
| 5 | Steam Deck 触控与虚拟摇杆是否在目标范围内？首版不做。 | user | Full Vision 之前 |
