# Interaction Pattern Library（交互模式库）

> **Status**: In Design — 首版目录与 15 条模式已定义；`Gaps & Patterns Needed` 列出尚未出规格的屏幕所需模式。
> **Author**: user + ux-designer
> **Last Updated**: 2026-11-05
> **Template**: Interaction Pattern Library
> **Accessibility Tier**: Standard（`design/accessibility-requirements.md`）
> **Platform**: PC；键盘/鼠标为主、手柄部分支持、无触屏（`.claude/docs/technical-preferences.md`）
> **Sources**: `design/ux/hud.md`（唯一已完成的屏幕规格）、`design/gdd/ui-flow.md` 屏幕契约与规则 1–28、`design/gdd/input.md` 规则 1–12、各系统 GDD 的 `## UI Requirements`
> **Sibling Specs**: `design/ux/hud.md`（局内 HUD）。其余屏幕规格尚未创建，见 Gaps。

---

## Overview

本库解决的问题是：**同一个交互在 13 个屏幕根上不要各写一遍、也不要各写错一遍**。每个模式在这里定义一次行为与验收口径，屏幕规格按名字引用，story 与测试按 `PAT-NN` 编号引用。

三条使用规则：

1. **屏幕规格引用模式名，不复制规格正文。** 屏幕级只写「本屏用到 PAT-03，原因文案见下表」，不重述模式定义。
2. **模式不发明 GDD 没有的规则。** 每条模式的数值与语义都能追溯到具体 GDD 规则或 AC；无法追溯的写进 `Gaps`，不在这里拍板。
3. **冲突时 GDD 优先。** 模式是 GDD 的实现约定，不是对 GDD 的修订；若某屏的 GDD 与模式冲突，先改模式，不改 GDD。

**编号稳定性**：`PAT-NN` 一经分配不再复用；模式废弃时标 `Deprecated` 并保留编号与替代项。

---

## Pattern Catalog

| ID | 模式 | 类别 | 使用屏幕 | 一句话 |
|---|---|---|---|---|
| PAT-01 | 焦点 / 悬停双轨 | Input / Feedback | HUD、全部菜单与模态 | 鼠标悬停与键盘焦点分开记录、分开绘制，可同屏并存 |
| PAT-02 | 焦点陷阱与焦点归还 | Modal / Input | HUD 确认面板、全部模态 | 模态内循环焦点，关闭后回到打开它的控件 |
| PAT-03 | 带原因的禁用态 | Data Display / Input | HUD 卡片、升级、解锁、开始按钮 | 禁用不是灰掉就完事，必须写「为什么」；区分数据态与流程禁用 |
| PAT-04 | 同帧数字落定 | Data Display / Feedback | HUD 顶栏、收据、升级页 | 数字在提交那一帧到位，不做倒计时与中间插值 |
| PAT-05 | 就地拒绝反馈 | Feedback | HUD 卡片、按钮 | 拒绝发生在发起处：抖动 + 一行原因，不弹窗、不扣值 |
| PAT-06 | 破坏性确认（默认取消） | Modal | 退出、放弃、购买、删除生涯 | 不可逆动作先确认，默认焦点在「取消」 |
| PAT-07 | 强制模态与出口矩阵 | Modal / Navigation | 存档 `Blocked`、`Error` | 阻断性状态按原因给有限出口，`ui_cancel` 无效，不给「覆盖保存」 |
| PAT-08 | 非模态提示 | Overlay / Feedback | 保存指示、恢复说明、新达成、完成横幅 | 不阻塞、不移动布局、不重复弹、可关闭 |
| PAT-09 | 显式空状态 | Data Display | 收据、纪录、腻度、下一波预览、档口 | 空数据写一句人话，不显示全 0 表或空白 |
| PAT-10 | 分段刻度条 + 整数读数 | Data Display | 容量条、回合槽、波次条 | 图形是冗余通道，整数数字才是判据 |
| PAT-11 | 锚点高亮与受保护区域 | Overlay / Guidance | 首局引导、HUD 锚点契约 | 引导只拿矩形、只压暗，不移动、不遮挡关键读数 |
| PAT-12 | 只读快照与口径所有权 | Data Display | 所有跨系统读数 | 界面只画别人给的值，不重算、不累加、不缓存第二份 |
| PAT-13 | 屏幕根替换与返回决策 | Navigation | 主菜单、收据、子屏、暂停层 | `ui_cancel` 的落点由固定优先级表决定，永不直接退出 |
| PAT-14 | 目录行与内联操作 | Data Display / Navigation | 升级、纪录、解锁、里程碑 | 一行一条记录，状态、价格、原因同屏，操作不跳出上下文 |
| PAT-15 | 设置滑杆行 | Input / Settings | 设置模态 | 滑杆必须可键盘/手柄操作，默认值与范围可见，立即生效 |

---

## Patterns

### PAT-01 焦点 / 悬停双轨

**Category**: Input / Feedback
**Used In**: `hud.md` §4.2.12、§10.6；`ui-flow` 规则 6；全部屏幕

**Description**: 鼠标悬停与键盘/手柄焦点是**两个独立指针**，分开记录、分开绘制，允许同时落在不同控件上。它解决的问题是：玩家用鼠标指着一张卡、同时用键盘操作另一张卡时，界面必须同时表达「我将要按的」和「我的鼠标在哪」，而不是猜玩家想用哪个设备。

**Specification**:
- 焦点：2 px 实线外框 + 2 px 间距，交互蓝；无发光、无脉动、无缩放。悬停：浅色底填充，不描边。两者同时命中同一控件时**叠加绘制**，不互相取代（`HUD-AC38`）。
- 鼠标移动不改键盘焦点；键盘导航不清除悬停（`INP-R8`、`HUD-R14`）。
- 焦点顺序按**屏幕位置**（上→下、左→右），不按节点树顺序（`INP` UI Requirements 2）。
- 设备切换不重置焦点；手柄断开时焦点停在原控件（`HUD-AC39`）。
- 悬停只是加速器：任何信息与操作都不得只在悬停态存在。
- 音频：焦点移动可发轻 tick，悬停不发；提交才发 `ui_button_activated`（`ui-flow` 规则 25）。

**When to Use**: 任何有焦点概念的屏幕；尤其是有卡片网格、列表行、按钮并列的界面。

**When NOT to Use**: `BootWaiting`、`ErrorScreen` 的只读文字区、HUD 的只读读数块——这些不进入焦点链，也不应有悬停样式。

**Reference**: `hud.md` §4.2.12 双轨表；§10.6 焦点链。

---

### PAT-02 焦点陷阱与焦点归还

**Category**: Modal / Input
**Used In**: `hud.md` §4.2.10 结束取餐确认、`ui-flow` 规则 5 / 6 / 7；全部 `ConfirmModal` / `SettingsModal` / `BlockedModal`

**Description**: 模态打开时，焦点移入模态并在其内部循环；`ui_cancel` 关闭模态并把焦点**还给打开它的那个控件**，而不是回到屏幕的第一个焦点位。

**Specification**:
- 打开：记录 `focus_restore` 目标 → 焦点落到模态内的默认焦点 → 形成循环（首项 ↑ 到尾项、尾项 ↓ 回首项）。
- 关闭：`ui_cancel` / 取消 / 确认任一路径都必须归还焦点；`Esc` 不得穿透到下层屏幕。
- 默认焦点：破坏性确认 = 「取消」；非破坏性模态 = 主操作（`ui-flow` 规则 6）。
- 模态深度上限 2（`ui-flow` 规则 5）；第三层请求被拒绝并记 WARNING，**不排队**。
- `Modal` 上下文必须成对 `push_context` / `pop_context`；任何离开路径都不得泄漏栈（`ui-flow` 规则 27）。
- 例外：HUD 的结束取餐面板**不**调用 `push_context(Modal)`（`HUD-R16`），但同样实现面板内循环与焦点归还——模式的行为要求不因上下文实现方式而降低。

**When to Use**: 任何阻断下层交互的浮层：确认框、设置、强制说明、HUD 瞬时面板。

**When NOT to Use**: 非模态提示（PAT-08）——它们不夺焦点、不拦截输入。

**Reference**: `hud.md` §4.2.10；`ui-flow` 规则 5、规则 6。

---

### PAT-03 带原因的禁用态

**Category**: Data Display / Input
**Used In**: HUD 不可吃卡片与跳过按钮、升级购买按钮、解锁按钮、主菜单「开始营业」、所有写盘依赖动作

**Description**: 控件不可用时必须同时回答两个问题：「现在能不能用」和「为什么不能用」。灰掉而不写原因，等于让玩家猜；猜错一次就会以为游戏坏了。本模式同时强制区分两种「灰」：

| 维度 | 数据态（例如放不下） | 流程禁用（例如 `Ending` / `frozen`） |
|---|---|---|
| 视觉 | 40% 不透明度 + 2 px 左侧提示线 | 40% + 无提示线 |
| 原因文案 | **100% 不透明**，写作「需要 6 格，剩 4 格」 | 不写局内原因，或写「本局已结束」「已暂停」 |
| 可获焦 | **是**——它是信息 | **否**——它是不可操作 |
| 可悬停 | 是 | 否 |
| 提交 | 拒绝并给原因，不改变任何数值 | 无响应（焦点本来就不在那） |

**Specification**:
- 禁用原因必须来自拥有该判定的系统：容量、回合、存档可写性、前置解锁、满级。界面不自行推断（PAT-12）。
- 同一屏内同类原因使用同一句式；原因句长度上限写进该屏文案表。
- 禁止只靠颜色表达禁用/锁定；锁形图标必须配文字（`unlocks` UI Requirements、`ACC-VIS`）。
- 可用性回复时（例如容量变化使卡可吃），样式与可获焦性同帧恢复，不播放「解锁」动画。

**When to Use**: 任何有前置条件的控件。

**When NOT to Use**: 只读数据块（它们不是控件，不该有禁用态）；空数据用 PAT-09。

**Reference**: `hud.md` §4.2.5、§4.2.9；`input` UI Requirements 1；`meta-upgrades` UI Requirements；`unlocks-and-stations` UI Requirements。

---

### PAT-04 同帧数字落定

**Category**: Data Display / Feedback
**Used In**: HUD 顶栏（回合、容量、净利、差额）、收据合计、升级页等级与钱包

**Description**: 动作提交的那一帧，读数就是新值；不做逐秒递减、不做假倒计时、不插值中间态。玩家要靠这些数字做下一次决策，动画只能**加在新值之上**，不能延迟新值出现。

**Specification**:
- 时间源：只由 `action_committed` 一类事件驱动；不存在按 `delta` 递减的读数（`TURN-UI`、`HUD-R12`）。
- 可选表现：≤ 250 ms 的位移/放大回落，叠加在**已经正确**的值上；`reduce_motion` 时关闭（`hud.md` §10.3）。
- 位数变化不引起容器重排：`min-width` 按最大位数预留。
- 收据是唯一允许「滚动」的地方，且滚动终点必须逐字段等于 `RunResult`，滚动不产生新数值（`ui-flow` 规则 14）；滚动中第一次 `ui_accept` / `ui_cancel` 只跳过滚动，不激活控件。
- 禁止乐观更新：动作被拒绝时不预扣、不预占、不改值（见 PAT-05）。

**When to Use**: 所有由离散动作改变的读数。

**When NOT to Use**: 纯装饰性进度（本作没有）；`Loading` 骨架态用 `—` 占位（PAT-09）。

**Reference**: `hud.md` §8 数字落定行、§10.3；`time-and-turns` UI Requirements；`ui-flow` 规则 14。

---

### PAT-05 就地拒绝反馈

**Category**: Feedback
**Used In**: HUD 菜品卡、跳过按钮、升级购买、解锁、开始营业

**Description**: 操作被拒绝时，反馈出现在**发起操作的地方**：控件抖动/描边变化 + 一行原因；不弹窗、不跳转、不扣任何值。拒绝是信息，不是错误屏。

**Specification**:
- HUD：卡片横向抖动 2 px ≤ 150 ms + D 行写「放不下 / 没有回合了 / 本局已结束」，150 ms 后恢复常态；同类拒绝在 500 ms 内合并，只刷新计时（`hud.md` §8）。
- 非 HUD 屏幕：按钮旁一行原因，保持控件可聚焦，不改变钱包/等级显示。
- 原因码到文案的映射由拥有该系统的一方给出；界面不翻译错误码之外的含义。
- `reduce_motion`：取消抖动，改为静态 2 px 左线加深。
- 音频：由 `AUD` 订阅 `dish_action_rejected(reason)` 播 `sfx.eat.reject`；界面自身不播音频。
- 绝不出现「点了没反应」：任何被拒绝的提交都必须有可见结果（`run-flow` UI Requirements 2）。

**When to Use**: 校验失败、前置条件不满足、事务失败的可恢复错误。

**When NOT to Use**: 不可恢复的系统故障（用 PAT-08 横幅）；需要玩家决策才能继续的冲突（用 PAT-07）。

**Reference**: `hud.md` §4.2.5、§8；`run-flow` UI Requirements 2；`meta-upgrades` UI Requirements（`COMMIT_FAILED`）。

---

### PAT-06 破坏性确认（默认取消）

**Category**: Modal
**Used In**: 退出游戏、放弃本局、购买升级、解锁档口、删除生涯、结束取餐

**Description**: 不可逆或直接扣钱的动作先打开确认框，说明后果；默认焦点在「取消」。确认框**不是二次校验**——真正的系统仍须完整校验一次，确认只表达玩家意图。

**Specification**:
- 标题为动作名，正文写后果与代价（价格、扣后余额、将放弃什么）；不写营销话术。
- 默认焦点「取消」；`ui_cancel` = 取消并归还焦点（PAT-02）。
- 确认后才调用业务接口；业务接口返回失败时按 PAT-05 就地反馈，不关框装成功。
- 进行中忽略重复确认，不发起第二笔（`ui-flow` 规则 22）。
- 无时限：等待 60 秒不自动关闭、不自动确认（`HUD-R17`、`ACC-COG-02`）。
- 破坏性动作不得用单次点击完成；「删除生涯」需两次确认，第二次默认「取消」（`ui-flow` 规则 24）。

**When to Use**: 扣钱、丢局、删档、退出、结束取餐。

**When NOT to Use**: 非末波跳过（`WAVE-OQ8` 已答：立即提交，不弹确认）；可撤销的操作。

**Reference**: `ui-flow` 规则 6、22、24；`hud.md` §4.2.10；`save-system` UI Requirements 6。

---

### PAT-07 强制模态与出口矩阵

**Category**: Modal / Navigation
**Used In**: 存档 `Blocked`、`ErrorScreen`

**Description**: 有些状态不允许玩家「继续凑合」——例如存档被另一个实例锁住、数据校验失败。此时给一个强制模态，**按原因列出有限的、明确的出口**，不提供危险选项。

**Specification**:
- 出口按原因查表，不自由组合；锁优先于原因（`save-system` UI Requirements 4）：

| 条件 | 出口 |
|---|---|
| `lock_live = true`（任意原因） | 退出 / 重试 |
| 存档属于更新版本 | 退出 / 新建生涯（旧档保留） |
| 存档与备份都无法读取 | 退出 / 新建生涯（旧档保留） |
| 迁移失败 | 退出 / 重试（不提供新建生涯） |
| 另一实例在用且暂不可读 | 退出 / 重试 |

- 默认焦点「退出」；`ui_cancel` **无效**（显式消费为空操作）；窗口关闭仍按场景规则直接退出，不算第三个按钮。
- **不得提供「覆盖保存」**；未解决前主菜单其它控件不可操作。
- `ErrorScreen` 只提供「退出」，release 构建中 debug 的「重新加载数据」不是存在后被拒绝，而是根本不存在（`ui-flow` 规则 18）。
- 出口按钮文案写动作结果（「新建生涯」「重试」），不写「确定」。

**When to Use**: 数据不可信、存档不可写、必须退出才能继续的状态。

**When NOT to Use**: 可恢复的校验失败（PAT-05）；需要玩家选择方案的普通确认（PAT-06）。

**Reference**: `save-system` UI Requirements 3–5 与规则 20；`ui-flow` 规则 18、20。

---

### PAT-08 非模态提示

**Category**: Overlay / Feedback
**Used In**: 保存指示、恢复说明、只读说明、进入失败/放弃提示、新达成条、完成横幅、错误横幅

**Description**: 不阻塞输入、不移动布局、不抢焦点的一条提示。它告知事实，然后消失或等玩家关闭；不要求确认，也不打断阅读。

**Specification**:
- 位置固定（`NoticeLayer` 一层），不推移任何控件；出现不改变焦点。
- 时长：短暂型（「已保存」）约 1.5 秒后淡出；持久型（恢复说明、错误横幅）保留到关闭或屏幕变化。
- 同一事件不重复弹出；同类短暂提示合并，不排队堆积（`hud.md` §8 队列规则）。
- 有声音则只发语义 cue；「已保存」**不发声**（`ui-flow` 规则 21）。
- `reduce_motion` 时淡入淡出为 0 ms。
- 关闭方式：可关闭型接受 `ui_cancel`（在返回优先级中排第 3 位，见 PAT-13）；不可关闭型（错误横幅）只随状态变化消失。

**When to Use**: 状态变化的事实通知、一次性说明、轻量结果。

**When NOT to Use**: 需要玩家做决定的事（PAT-06/07）；动作被拒绝（PAT-05）。

**Reference**: `ui-flow` 规则 19、20、21、23；`hud.md` §8；`save-system` UI Requirements 1–3。

---

### PAT-09 显式空状态

**Category**: Data Display
**Used In**: 收据空局、纪录页空生涯、腻度面板、下一波预览、档口列表、`Loading` 首帧

**Description**: 没有数据时显示**一句说明为什么没有**，而不是空白、全 0 表格或编造的默认值。空状态和错误态是两件事，文案不能混用。

**Specification**:
- 每一条数据依赖元素都必须在屏幕规格里登记空状态（`ux-review` 逐条检查）。
- 空状态文案写事实，不写安慰：「这一局什么都没吃」「还没有账本」「暂无」「下一波：4 道未揭晓」。
- 空状态不得伪装成 0 值：`0 / 0` 只用于真实的容量 0 并同时给出原因；`—` 只用于「数据未到/不可用」。
- 空状态不得提供不存在的操作（例如空生涯页不显示「清空纪录」）。
- 查询失败进入错误态或 `—`，不显示旧值。

**When to Use**: 首次使用、零数据、过滤后为空、加载首帧。

**When NOT to Use**: 数据真实为零且零本身有意义（此时显示 `0`，不是空状态）。

**Reference**: `hud.md` §5 空状态清单；`settlement-and-entry-fee` UI Requirements；`records-and-milestones` UI Requirements 2、6；`ui-flow` 规则 11。

---

### PAT-10 分段刻度条 + 整数读数

**Category**: Data Display
**Used In**: 容量条、回合槽、波次条

**Description**: 条状图形只做冗余通道，**整数读数才是判据**。数字永远可读、等宽、落在高对比底上；条可以被压缩、简化或省掉，数字不能。

**Specification**:
- 数字与条同屏；条不替代数字。150% 字号下条可以减格或退化为比例条，数字保持精确值（`hud.md` §10.2）。
- 分段数有下限（容量格宽 ≥ 6 px）；放不下时减少**视觉**分段，不改变读数。
- 不使用颜色单独表达状态：已用/剩余用填充形状 + 加粗边界；逼近上限不变红（`HUD` Visual）。
- 不做「危险区」动画、不脉动、不倒计时。
- 数值为 0 是合法状态，不做除零；`capacity_total = 0` 显示 `0 / 0` 并写原因。

**When to Use**: 有总量与已用量的资源、进度、波次。

**When NOT to Use**: 无总量概念的自由数值（直接给数字）。

**Reference**: `hud.md` §4.2.1、§4.2.2、§4.2.4、§10.2；`stomach-capacity` UI Requirements；`time-and-turns` UI Requirements。

---

### PAT-11 锚点高亮与受保护区域

**Category**: Overlay / Guidance
**Used In**: 首局引导（气泡、聚光遮罩、锚点）、HUD 锚点契约

**Description**: 引导层向 HUD **要矩形**，不自己算坐标；拿到矩形后只做压暗与高亮，不移动焦点、不改变布局、不遮挡受保护读数。

**Specification**:
- 接口：`get_anchor_rect(anchor_id) -> Rect2` 与 `get_protected_rects() -> Rect2[3]`；每次请求重新计算，不缓存（`ONB-R11`、`ONB-R12`）。
- 锚点不存在时返回空矩形，调用方走降级路径；受保护矩形取不到时按「不相交」处理并记断言失败。
- 三个受保护读数（回合、容量、净利/差额）在任何布局下都保持原亮度；引导改变的是可操作性，不是规则（`run-flow` UI Requirements 7）。
- 引导层不绘制 HUD、不设置焦点、不隐藏读数。
- 颜色高亮必须同时有文字标签，不能只靠色相（`first-run-onboarding` UI Requirements）。
- 气泡组件单实例，同一时刻最多一条；文本行数超限即为文案缺陷。

**When to Use**: 新手引导、功能发现、一次性教学高亮。

**When NOT to Use**: 常规提示（PAT-08）；需要阻断操作的模态（PAT-02/07）。

**Reference**: `hud.md` §4.2.13；`first-run-onboarding` UI Requirements；`run-flow` UI Requirements 7。

---

### PAT-12 只读快照与口径所有权

**Category**: Data Display
**Used In**: HUD 全部读数、收据、升级页钱包、纪录页、档口进度

**Description**: 每个数值有且只有一个系统拥有其口径；界面只画快照里的值，**不重算、不累加、不缓存第二份**。这条模式是防止「HUD 自己加一遍，和收据差一块钱」的根因。

**Specification**:
- 净利/差额来自 `RunResult` 或运行期快照；HUD 不得把菜品价格相加（`HUD-R6`、`HUD-AC1`、`run-flow` UI Requirements 1）。
- 收据逐字段读 `RunResult`，不排序、不合并、不补算，不显示 `opt_value` 或「本可以多赚」（`ui-flow` 规则 11）。
- 回本率、完成判定、解锁进度分母、可购买性都取拥有者的返回值；界面不做等价计算。
- 界面不得自行推算状态（例如「是不是该结束了」）；只读 `run_state`。
- 快照到边界即停：收据按引用保留到关闭，不重复入账、不为同一 `run_id` 开第二张（`ui-flow` 规则 9、13）。
- 跨屏一致性：同一数值在 HUD、收据、纪录页出现时必须同源；出现差异按缺陷处理。

**When to Use**: 一切跨系统读数。

**When NOT to Use**: 纯展示派生量（例如「还差 N 格」的措辞）——但必须由拥有者提供文案生成器，界面不自行判定。

**Reference**: `hud.md` §4.2.3、§4.2.13；`run-flow` UI Requirements 1、3；`ui-flow` 规则 10、11、23；`settlement-and-entry-fee` UI Requirements。

---

### PAT-13 屏幕根替换与返回决策

**Category**: Navigation
**Used In**: 主菜单、收据、升级/纪录子屏、暂停覆盖层、设置/确认模态

**Description**: `ui_cancel` 不猜意图，按一张固定优先级表决定落点；任何状态下都**不直接退出游戏**。

**Specification**:
- 优先级（`ui-flow` 规则 7）：①可取消模态 → 关闭；②强制模态 → 无动作；③可关闭非模态提示 → 关闭；④收据滚动中 → 跳过；⑤菜单子屏 → 弹回主菜单；⑥收据根屏 → 主菜单；⑦主菜单根屏 → 退出确认；⑧`Paused` → 退出确认；⑨`Error` / `Boot` / `Run` / `Quit` → 无动作。
- 屏幕栈最多 2 层：根 + 一个子屏；收据是**替换根**，不叠在菜单上（`ui-flow` 规则 4）。
- 顶层状态切换时丢弃旧屏幕栈与模态栈，按新状态重建，不保留跨状态界面（`ui-flow` 规则 3）。
- 暂停是覆盖层：不卸载、不重建局内界面（`ui-flow` 规则 15）。
- 每个屏幕必须有确定的返回路径；`ErrorScreen` / `BlockedModal` 是显式例外。

**When to Use**: 所有屏幕与浮层的进入/退出。

**When NOT to Use**: 局内动作键（吃菜、跳波）不参与返回决策。

**Reference**: `ui-flow` 规则 3、4、7、15、18；`scene-flow` UI Requirements。

---

### PAT-14 目录行与内联操作

**Category**: Data Display / Navigation
**Used In**: 升级页、纪录页、解锁档口区、里程碑列表

**Description**: 一条记录一行，左侧是身份与状态，右侧是数值与动作；操作不离开当前上下文，不为一条记录打开新屏。适合本作这种「清单 + 少量可执行项」的菜单。

**Specification**:
- 行内固定字段顺序：名称 → 状态/等级/进度 → 数值 → 原因（不可用时）→ 操作。金额与单位价值右对齐、等宽。
- 行高随字号增长，150% 时不换行错位、金额列不重叠（各 GDD `## UI Requirements`）。
- 不可用行按 PAT-03 给原因；已满级/已达成用文字，不只靠颜色或对勾图标。
- 整行可聚焦；行内操作按钮是焦点链的一部分，焦点顺序按屏幕位置。
- 列表较长时滚动，不压缩字号；滚动区域与固定钱包/标题区分离。
- 首局引导可以高亮一行并标「推荐」，但不得自动购买或替玩家确认（`meta-upgrades` UI Requirements）。

**When to Use**: 目录、纪录、里程碑、档口、升级等重复结构。

**When NOT to Use**: 需要并排比较多个维度的表格（本作没有）；单条详情页（直接用屏幕规格）。

**Reference**: `meta-upgrades` UI Requirements；`records-and-milestones` UI Requirements 2–4；`unlocks-and-stations` UI Requirements；`ui-flow` 规则 22、23。

---

### PAT-15 设置滑杆行

**Category**: Input / Settings
**Used In**: `SettingsModal`（主音量、音效音量、文本缩放、HUD 不透明度）

**Description**: 设置项一行：标签、当前值、可操作控件。滑杆必须能只用键盘/手柄调到位，且修改立即生效、可感知。

**Specification**:
- 范围与默认值可见；当前值以文字显示（例如 `−12 dB`、`125%`、`80%`），不要求玩家读刻度位置。
- 键盘/手柄：左右键步进、按住重复；步长与加速曲线必须在屏幕规格里写死（当前为 Gaps 项，见下）。
- 修改立即生效；关闭模态或退出时写入机器级设置文件，失败显示「设置可能未保存」并保留内存值（`ui-flow` 规则 24）。
- 设置不属于生涯档，删除生涯不影响设置。
- 音量为 0 dB / −80 dB 时文案写「静音」，不写 `-∞`。
- 设置项不与游戏状态冲突：暂停菜单内打开设置时，局内数字冻结不变。

**When to Use**: 音频、无障碍、HUD 显示等全局偏好。

**When NOT to Use**: 局内即时操作（那属于玩法输入）；需要立即看到结果的玩法参数。

**Reference**: `ui-flow` 规则 24、`SettingsModal` 行；`audio-feedback` UI Requirements；`hud.md` §10.5。

---

## Prohibited Patterns（本作明确不采用）

这些不是「暂时没做」，而是**设计禁止**；屏幕规格里出现即为缺陷：

| 禁止 | 原因 | 出处 |
|---|---|---|
| 悬浮提示作为字段的唯一通道 | 手柄/键盘无法悬停；信息必须同屏 | `dish-values` UI Requirements、`HUD-R8` |
| 按单位价值排序 / 可吃性过滤 | 等于把最优解交给玩家 | `dish-values`、`stomach-capacity`、`cloying-and-taste` |
| 颜色单独承载语义 | 色觉缺陷 + 灰度可读是 ship-blocking | `ACC-VIS-08`、`hud.md` §10.1 |
| 限时确认 / 自动关闭 | 违反 Standard 档无计时承诺 | `ACC-COG-02`、`HUD-R17` |
| 乐观更新 | 拒绝后数值会跳回去，破坏信任 | `hud.md` §8 |
| 提示队列堆积 | 本作只有瞬时槽 + 横幅，不补播过期消息 | `hud.md` §8 |
| 写死键位图标 | 手柄/键鼠切换后会说谎 | `hud.md` §9、`INP-OQ2` |
| HUD 元素重排 / 全隐 | 会破坏「三块读数永不遮挡」判据 | `hud.md` §9、§10.5 |
| 无确认的扣钱/删档 | 不可逆动作必须确认 | `ui-flow` 规则 6、24 |
| 用确认框代替业务校验 | 确认只是意图，校验仍须完整 | `ui-flow` 规则 22 |

---

## Gaps & Patterns Needed

### 已有模式但屏幕规格未写

以下屏幕已由 `ui-flow` 给出内容契约，但尚无 `design/ux/*.md` 规格。它们落地时应直接引用本库，不再发明新交互；若发现契约缺口，先补本库再写屏幕：

| 屏幕 | 将复用 | 待定 |
|---|---|---|
| `MainMenu` | PAT-03、08、13、14 | 入场费/钱包的常驻位置与空钱包文案 |
| `ReceiptScreen` | PAT-04、09、12、13 | 滚动与跳过的视觉表现；长收据的分段/折叠；印章与旧最佳的并排布局 |
| `UpgradeScreen` | PAT-03、06、14 | 目录滚动、`BUSY` 进行中态 |
| `RecordsScreen` | PAT-09、14 | 里程碑分组折叠、完成横幅与「新达成」条的避让 |
| `SettingsModal` | PAT-15、06 | 滑杆步进/加速曲线；分区标题；删除生涯的次级分区 |
| `PauseOverlay` | PAT-06、13 | `Ending` 后隐藏「放弃本局」时的布局稳定性 |
| `ConfirmModal` | PAT-02、06 | 长后果文案的换行与最大宽度 |
| `ReadOnlyNotice` / `BlockedModal` | PAT-07、08 | 双出口/三出口的按钮排列 |
| 首局引导层 | PAT-11、02 | 气泡版式、聚光遮罩边缘、跳过入口位置 |
| `ErrorScreen` | PAT-07、09 | debug 信息区的折叠 |

### 尚未定义的新模式

- **收据滚动与跳过**：`ui-flow` 规则 14 定义了时序与事件，但「滚动中」的视觉、跳过提示、焦点保持未定。
- **设置滑杆的键盘/手柄步进**：`ui-flow` 规则 24 只给了范围，没有步长与重复曲线。
- **列表滚动与焦点可见性**：当目录超过一屏时，焦点移出可视区如何滚动、滚动条是否显示，未定。
- **档口选择器（未来）**：MVP 关闭；开启时需要「选择器 + 解锁」组合模式，届时新增而非改写现有模式。

### Open Questions

1. `hud.md` 的确认面板不压 `Modal` 上下文（`HUD-R16`），而其它屏幕的确认框都压；PAT-02 只在行为层统一，实现层是否要在未来统一为同一套焦点陷阱组件？——待 `ui-programmer` 评估。
2. 滑杆步进曲线归谁定：`ui-flow`（设置模型）还是各屏幕规格？建议前者，避免每个设置页各写一套。
3. `receipt_roll` 的跳过提示是否需要一行「按任意键跳过」文字；若加，必须同时满足无计时器与可静音原则。
4. 本库是否需要在 `/ux-review` 中作为必查输入——目前 `/architecture-review` 只查文件存在；建议在 Pre-Production gate 前补一次模式一致性检查。
