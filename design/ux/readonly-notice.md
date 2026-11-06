# ReadOnlyNotice / BlockedModal Design（只读说明与阻塞模态）

> **Status**: In Design
> **Author**: user + ux-designer（autonomous 模式）
> **Last Updated**: 2026-11-06
> **Game**: 回本 (Break Even)
> **Platform Targets**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主，手柄部分支持；无触屏。参考 1280×720
> **Related GDDs**: `save-system.md`（规则 10/21/26、UI Requirements 3/4）、`ui-flow.md`（UIF-R20）、`scene-flow.md`（退出）、`input.md`（Modal / Menu 上下文）
> **Accessibility Tier**: Standard（逐条要求见 `design/accessibility-requirements.md`）
> **Sources**: `design/ux/interaction-patterns.md`（PAT-02/03/07/08/12）、`design/ux/hud.md`（焦点与色板基线）、ADR-0002（存档路径与隔离文件）、ADR-0005（`Modal` 栈与 `ui_cancel`）
> **Sibling Specs**: `design/ux/pause-overlay.md`、`design/ux/confirm-modal.md`、`design/ux/error-screen.md`、`design/ux/onboarding-layer.md`、`design/ux/hud.md`、`design/ux/interaction-patterns.md`
> **Template**: UX Screen Spec

> **范围边界**：本文件同时覆盖**同一份 `LoadReport` 的两个界面**——`ReadOnly` 的非模态说明（`ReadOnlyNotice`）与 `Blocked` 的强制模态（`BlockedModal`）。两者共享数据契约，但交互等级相反：前者可关闭、不挡主菜单；后者不可取消、挡一切。本文件不拥有主菜单布局、不拥有存档读写、不拥有 `retry_load()` / `delete_career()` 的业务实现。

## 引用代号

| 代号 | 出处 |
|---|---|
| `SAV-Rn` / `SAV-UI n` | `design/gdd/save-system.md` Core Rules 第 n 条 / UI Requirements 第 n 条 |
| `SAV-ACn` | 同文件 Acceptance Criteria 第 n 条 |
| `UIF-Rn` | `design/gdd/ui-flow.md` Core Rules 第 n 条 |
| `SCN-Rn` | `design/gdd/scene-flow.md` Core Rules 第 n 条 |
| `INP-Rn` | `design/gdd/input.md` Core Rules 第 n 条 |
| `ONB-Rn` | `design/gdd/first-run-onboarding.md` Core Rules 第 n 条 |
| `ADR-0002` | `docs/architecture/adr-0002-persistence-format.md` |
| `ADR-0005` | `docs/architecture/adr-0005-input-context-stack.md` |
| `PAT-nn` | `design/ux/interaction-patterns.md` |
| `ACC-xxx-nn` | `design/accessibility-requirements.md` |

---

## 1. 目的与玩家需求

`LoadReport` 的六种结果里，四种必须「安静」（`Fresh` / `Loaded` / `Migrated` / `Recovered` 的后者是一次性说明），只有两种会挡住或限制玩家：**`ReadOnly` 与 `Blocked`**。它们回答的是两个不同问题：

- **`ReadOnly`**：「你的存档没问题，但另一个实例正在写它。你可以看、可以玩，但这一会话不会保存。」玩家需要的是**知道限制、继续玩、或退出**，而不是被吓退。
- **`Blocked`**：「这份存档当前无法安全使用。你必须选一个出口，主菜单其它按钮都不能用。」玩家需要的是**原因、安全出口、绝不覆盖对方文件的保证**。

三条立场：

1. **不制造第二份判断**：界面只读 `LoadReport`，不根据文件时间、钱包、警告自行推断「应该能读」。
2. **锁优先于原因**：`lock_live = true` 时无论原因是更新版本、无法读取还是迁移失败，都只给「退出」「重试」（`SAV-R21`、`SAV-UI 4`）。
3. **绝不提供覆盖保存**：没有「覆盖」「忽略并新建」「强制读取」；`ReadOnly` 不提供「新建生涯」，`Blocked` 的「新建生涯」也要先隔离旧档。

---

## 2. 共享数据契约

| 字段 | 类型 | 本屏用途 | 不得做的事 |
|---|---|---|---|
| `outcome` | `Fresh / Loaded / Migrated / Recovered / ReadOnly / Blocked` | 唯一分派判据 | 不用 `writable` 反推 `outcome` |
| `reason` | 原因描述 | 选择 `Blocked` 文案与出口 | 不在 release 显示原始错误码/路径 |
| `writable` | bool | `false` 时所有写动作禁用 | 不假设「只读=不能启动游戏」 |
| `lock_live` | bool | `Blocked` 出口分支的最高优先级 | 不把 `lock_live` 与「另一个实例」混为一谈 |
| `source_version` | int | 仅 debug 详情显示 | 不在 release 显示版本号 |
| `warnings` | 列表 | 不进入本屏；由非模态通知/调试负责 | 不把警告塞进阻塞正文 |
| `first_run_eligible` | bool | 不直接显示；参与主菜单/开局 | 不在本屏重算 |

**分派规则**：`outcome = ReadOnly` → `ReadOnlyNotice`；`outcome = Blocked` → `BlockedModal`；其余不显示本屏。两者都在进入 `Menu` 后读取**一次**，同一 `LoadReport` 不重复弹（`UIF-R20`）。

---

## 3. ReadOnlyNotice（非模态说明）

### 3.1 进入与退出

| 项 | 契约 |
|---|---|
| 进入 | 进入 `Menu` 根屏后，`LoadReport.outcome = ReadOnly`，且本会话未展示过 |
| 位置 | `NoticeLayer` 底部横幅，位于主菜单按钮之下、页脚之上 |
| 初始焦点 | 「以只读继续」（`UIF-UI`） |
| 关闭 | 点击「以只读继续」、`ui_accept` 于该按钮、`ui_cancel`（Esc/Backspace/B），或任何屏幕变化 |
| 退出 | 「退出」→ 直接请求 `Quit`（`Menu` 不需确认，`SCN-R10`） |
| 重复 | 关闭后本会话不再出现；同一 `LoadReport` 不重复弹出 |

### 3.2 布局（1280×720）

```
┌──────────────────────────────────────────────────────────────┐
│  主菜单（档口名 / 入场费 / 钱包 / 开始营业 / 纪录 / 设置…）  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 只读模式                                               │  │
│  │ 另一个实例正在使用这份存档。本会话不会保存任何进度。   │  │
│  │                              [ 以只读继续 ]  [ 退出 ]  │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

- 横幅宽 `min(0.72 × W, 900)`，高按内容自适应；底边距安全区 32 px。
- 米白纸面、墨色文字；左侧 4 px 橄榄绿/中性色条（非文本），不使用警示红——这不是错误。
- 按钮高 ≥ 56 px；「以只读继续」在左、「退出」在右；初始焦点在左。
- 横幅**不覆盖**主菜单的可点击区域；`MOUSE_FILTER_STOP` 只作用于横幅自身矩形（非模态，不挡背后点击）。
- 150% 字号：正文换行，横幅可增高到两行按钮；不截断、不横向滚动。

### 3.3 状态与交互

| 状态 | 条件 | 行为 |
|---|---|---|
| `HIDDEN` | 非 `ReadOnly` 或已关闭 | 不占布局 |
| `SHOWN` | 首次进入 `Menu` | 焦点在「以只读继续」；主菜单仍可操作 |
| `CLOSING` | 关闭 | 80 ms 淡出；`reduce_motion` 时为 0 |
| `DISMISSED` | 已关闭 | 本会话不再显示 |

| 操作 | 键盘 | 手柄 | 鼠标 |
|---|---|---|---|
| 移动 | ← / →、A / D | 摇杆/十字键 | 悬停 |
| 以只读继续 | Enter / Space | A | 点击 |
| 退出 | 聚焦后确认 | A | 点击 |
| 关闭说明 | Esc / Backspace | B | 点击「以只读继续」 |

**`ui_cancel` 落点**：`Menu` 且 `dismissible_notice_active = true` → `DISMISS_NOTICE`（`back_target` 优先级 3）。关闭后焦点落到主菜单初始焦点「开始营业」；不得落到已隐藏的「升级」入口。

**非模态约束**：本说明**不**压 `Modal`，不形成焦点陷阱；鼠标可以直接点主菜单。玩家跳过说明不会失去「只读」这一事实——`writable = false` 仍生效。

### 3.4 只读期间的限制

| 区域 | 行为 | 依据 |
|---|---|---|
| 升级 / 购买 / 解锁入口 | **隐藏**（不是禁用） | `UIF-R20`、`SAV-UI 3` |
| 「新建生涯」 | 不提供 | 避免隔离对方文件（`SAV-UI 3`） |
| 开始营业 | 仍可开始（是否提示不保存见 OQ-5） | `UIF-R20` 未禁止 |
| 纪录 / 设置 | 仍可浏览；设置写机器级文件，不属于生涯档（`UIF-R24`） | 设置与只读无关 |
| 任何写盘动作 | 若入口因缺陷仍可见，必须禁用并显示「存档为只读」（`UIF-R22` 的 `NOT_WRITABLE`） | `UIF-R20` |
| 删除生涯 | 不可用（若从设置进入，删除入口禁用并给原因） | `SAV-R23` |

---

## 4. BlockedModal（强制模态）

### 4.1 进入与退出

| 项 | 契约 |
|---|---|
| 进入 | 进入 `Menu` 根屏后，`LoadReport.outcome = Blocked` |
| 上下文 | 界面流程 `push_context(Modal)`，获得焦点陷阱（`UIF-R20`） |
| `modal_cancel_enabled` | **false**（强制模态） |
| 初始焦点 | 「退出」（`UIF-UI`） |
| 关闭 | 只能通过「退出」「重试」「新建生涯（旧档保留）」之一 |
| `ui_cancel` | 无效；显式消费为空操作，不关闭、不穿透背景 |
| 窗口关闭 | 仍由场景流程直接退出，不是第三个按钮（`UIF-R20`、`SCN-R10`） |

### 4.2 出口矩阵（精确，锁优先）

| `lock_live` | 原因 | 按钮（左→右） | 初始焦点 | 重试 | 新建生涯 |
|---|---|---|---|---|---|
| `true` | 任意（更新版本 / 无法读取 / 迁移失败 / 持锁不可读） | 退出、重试 | 退出 | 是 | **否** |
| `false` | 存档属于更新版本 | 退出、新建生涯（旧档保留） | 退出 | **否** | 是（隔离旧档） |
| `false` | 存档与备份都无法读取 | 退出、新建生涯（旧档保留） | 退出 | **否** | 是（隔离旧档） |
| `false` | 迁移失败 | 退出、重试 | 退出 | 是 | **否**（旧档有效，只是读不了） |

> 判定顺序：先看 `lock_live`，再看 `reason`。`lock_live = true` 时表格后三行全部失效。`retry_load()` 只在「`lock_live = true`」或「迁移失败」时可用（`SAV-R` 接口表、`SAV-AC30`）。

### 4.3 布局（1280×720）

```
┌──────────────────────────────────────────────────────────────┐
│  主菜单（全部控件不可交互，压暗 55%）                         │
│                                                              │
│          ┌────────────────────────────────────────┐          │
│          │  存档无法使用                          │          │
│          │                                        │          │
│          │  这份存档来自更新版本的游戏。           │          │
│          │  当前版本无法读取它，文件不会被修改。   │          │
│          │                                        │          │
│          │        [ 退出 ]  [ 新建生涯（旧档保留）]│          │
│          └────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────────────┘
```

- 面板宽 `clamp(0.46 × W, 400, 600)`；标题 + 原因正文 + 两个按钮。
- 米白纸面、墨色文字；左侧 4 px 警示橙条（非文本 3:1），**不使用红色文字**（`ACC-VIS-04`）。
- 两按钮同排，高 ≥ 56 px；退出在左（默认焦点）；第二按钮是「重试」或「新建生涯（旧档保留）」，按矩阵决定。
- 正文只写原因与安全保证，不写错误码、不写文件路径（release）；debug 在按钮下方追加只读详情区。
- 150% 字号：正文换行、面板增高；按钮文字不换行；不截断、不横向滚动。

### 4.4 状态

| 状态 | 条件 | 可见 | 可交互 |
|---|---|---|---|
| `OPEN` | 已打开 | 标题/原因/两个按钮 | 是 |
| `RETRYING` | 已点「重试」 | 两按钮禁用 + 「正在重新读取…」 | 否 |
| `CREATING` | 已点「新建生涯」 | 两按钮禁用 + 「正在建立新生涯…」 | 否 |
| `RETRY_FAILED` | 重试仍失败 | 按新报告重建的按钮 + 「仍无法读取，请稍后再试」 | 是 |
| `CREATE_FAILED` | 新建失败 | 原按钮 + 失败原因 | 见 OQ-4 |
| `CLOSING` | 已成功或退出 | 淡出 | 否 |

**转移**

| 从 | 事件 | 到 | 副作用 |
|---|---|---|---|
| `OPEN` | `ui_accept` 于退出 | `CLOSING` | `request_transition(Quit)`；无需确认 |
| `OPEN` | `ui_accept` 于重试（仅允许分支） | `RETRYING` | `retry_load()` 恰好一次；重复激活忽略 |
| `RETRYING` | 返回 `LoadReport` | `CLOSING` / `RETRY_FAILED` | 成功则用新报告替换并关闭/转为 `ReadOnly` 提示；失败停留 |
| `OPEN` | `ui_accept` 于新建生涯 | `CREATING` | 建立新生涯；旧档隔离；重复激活忽略 |
| `CREATING` | 成功 | `CLOSING` | 关闭；刷新钱包/升级/纪录/主菜单 |
| `CREATING` | 失败 | `CREATE_FAILED` | 不关闭、不假装成功 |
| `OPEN` | `ui_cancel` / Esc / B | 不变 | 显式无操作；不关闭、不穿透 |
| 任意 | 窗口关闭 | 退出流程 | 不经过按钮集合 |

### 4.5 `ui_cancel` 落点

| 前置 | 按键 | 结果 | 依据 |
|---|---|---|---|
| `BlockedModal` 打开 | Esc / Backspace / B | 无动作，模态保持 | `back_target` 优先级 2（`modal_cancel_enabled = false`）→ `NONE` |
| `BlockedModal` 打开 | `ui_pause` | 无动作 | `Modal` 不含 `ui_pause`（`INP` States 表） |
| `RETRYING` / `CREATING` | 任意取消 | 无动作 | 提交中不可取消；防重复请求 |
| 任意 | 点击遮罩 | 无动作 | 强制模态 |

> **PAT-07 的「NONE 必须产生可见无动作」**：本屏在按下取消时让「退出」的焦点环重绘一次（不变粗、不位移、100 ms 内恢复；`reduce_motion` 时单帧高亮），不播放声音、不显示新文案。是否足够，见 OQ-1。

### 4.6 重试与新建生涯的业务契约

| 动作 | 调用 | 成功后 | 失败后 |
|---|---|---|---|
| 重试 | `存档.retry_load()` | 替换缓存 `LoadReport`；若变 `ReadOnly` 则关闭本模态并显示只读说明；若可写则关闭并恢复正常主菜单；若仍 `Blocked` 则按新报告的锁/原因重建出口并显示「仍无法读取」 | 停留本模态，保留原有出口，显示「仍无法读取」；不隔离任何文件 |
| 新建生涯 | 存档建立新生涯（旧档隔离） | 主菜单刷新钱包、升级、纪录；模态关闭；不显示只读说明 | 见 OQ-4；至少不覆盖旧档、不部分初始化 |
| 退出 | `request_transition(Quit)` | 直接退出 | — |

**禁止**：覆盖旧档、手动保存、把「新建生涯」做成第二次确认（GDD 未要求；它的代价已在按钮标签「旧档保留」中说明）、在 `RETRYING` 时并行发起第二次 `retry_load()`。

---

## 5. 模式应用

| 模式 | ReadOnlyNotice | BlockedModal |
|---|---|---|
| **PAT-08 非模态提示** | 是：底部横幅、不挡主菜单、可关闭、`ui_cancel` 关闭 | 否：强制模态，不是提示 |
| **PAT-07 强制模态与出口矩阵** | 否：有取消/关闭 | 是：`modal_cancel_enabled = false`，出口由矩阵决定，`ui_cancel` 无效 |
| **PAT-02 焦点陷阱与焦点归还** | 部分：初始焦点在说明上，但不困住焦点；关闭后回「开始营业」 | 是：焦点困在模态，背景主菜单不可达；关闭后按出口离开或刷新 |
| **PAT-03 带原因的禁用态** | 只读期间的购买/解锁**隐藏**；若可见则禁用并给「存档为只读」 | 背景所有写动作不可操作；「重试/新建」提交中禁用并给状态行原因 |
| **PAT-12 只读快照与口径所有权** | 是：只读 `LoadReport`，不重算可写性 | 是：只读 `LoadReport`，不自行判断锁/版本 |

---

## 6. 无障碍

| 要求 | 落点 |
|---|---|
| `ACC-VIS-02` | 正文/按钮 ≥ 20 px @720p；debug 详情 ≥ 16 px |
| `ACC-VIS-03` | 米白底 + 墨色文字 ≥ 4.5:1；警示条非文本 ≥ 3:1 |
| `ACC-VIS-05` / `07` | 焦点环 2 px + 2 px 间距；悬停浅底、焦点外框 |
| `ACC-VIS-08` | 灰度下能读出「只读/阻塞」、两个出口、焦点与禁用态 |
| `ACC-VIS-09` / `10` | 150% 下正文换行、按钮不换行、无截断/横向滚动 |
| `ACC-VIS-12` | 减少动态时无淡入、无焦点环过渡 |
| `ACC-MOT-01` / `06` | 纯键盘/手柄可完成关闭、退出、重试、新建；按钮 ≥ 56 px；焦点环不被裁切 |
| `ACC-COG-03` | 本屏与重试/新建均无超时；不自动退出 |
| `ACC-COG-08` | 原因文案来自本地化表；不显示原始错误码（debug 除外） |
| `ACC-AUD-02` / `06` | 无音频专属信息；静音下可完整操作 |

**安全出口**：`BlockedModal` 的默认焦点是「退出」，且 `ui_cancel` 不会误触发任何写动作；窗口关闭也安全退出。这满足「阻塞状态下不造成数据损失」的最低承诺。

---

## 7. 本地化

| Key | 中文（基准） | 约束 |
|---|---|---|
| `ui.readonly.title` | 只读模式 | ≤ 8 字 |
| `ui.readonly.body` | 另一个实例正在使用这份存档。本会话不会保存任何进度。 | ≤ 40 字；不写路径 |
| `ui.readonly.continue` | 以只读继续 | ≤ 8 字 |
| `ui.readonly.exit` | 退出 | ≤ 6 字 |
| `ui.blocked.title` | 存档无法使用 | ≤ 8 字 |
| `ui.blocked.reason.version_ahead` | 这份存档来自更新版本的游戏。当前版本无法读取它，文件不会被修改。 | — |
| `ui.blocked.reason.unreadable` | 存档与备份都无法读取。文件不会被修改。 | — |
| `ui.blocked.reason.migration_failed` | 存档需要迁移，但迁移失败。旧档仍然有效。 | — |
| `ui.blocked.reason.locked_unreadable` | 另一个实例正在使用这份存档，且文件暂时无法读取。 | — |
| `ui.blocked.retry` | 重试 | ≤ 6 字 |
| `ui.blocked.new_career` | 新建生涯（旧档保留） | ≤ 12 字；括号不可省 |
| `ui.blocked.retrying` | 正在重新读取… | — |
| `ui.blocked.creating` | 正在建立新生涯… | — |
| `ui.blocked.retry_failed` | 仍无法读取，请稍后再试。 | — |

- 原因句必须保持「文件不会被修改」这一安全保证；翻译不得弱化为「可能」。
- 「新建生涯（旧档保留）」不得简写成「新建」——旧档保留是玩家做决定的关键信息。

---

## 8. 验收标准

### ReadOnlyNotice

- [ ] **RO-AC01**（核心）GIVEN `outcome = ReadOnly`，WHEN 首次进入 `Menu`，THEN 底部显示一次说明，文案包含「不会保存」，焦点在「以只读继续」。
- [ ] **RO-AC02**（非模态）GIVEN 说明显示中，WHEN 用鼠标点击主菜单「开始营业」，THEN 可以点击并开始（说明不被当成模态阻挡）；说明随之关闭。
- [ ] **RO-AC03**（关闭）GIVEN 说明显示中，WHEN 按 `ui_cancel` 或激活「以只读继续」，THEN 说明关闭、本会话不再出现、焦点落到「开始营业」。
- [ ] **RO-AC04**（返回决策）GIVEN 说明显示且 `modal_depth = 0`，WHEN 按 `ui_cancel`，THEN 只关闭说明，不打开退出确认、不退出游戏。
- [ ] **RO-AC05**（退出）GIVEN 说明显示，WHEN 激活「退出」，THEN 直接请求 `Quit`，不弹确认（`Menu` 规则）。
- [ ] **RO-AC06**（隐藏入口）GIVEN `outcome = ReadOnly`，WHEN 检查主菜单，THEN 升级/购买/解锁入口不存在（不是禁用），且没有「新建生涯」。
- [ ] **RO-AC07**（不可写兜底）GIVEN 因缺陷仍显示某个购买入口，WHEN 尝试购买，THEN 按钮禁用并显示「存档为只读」，不创建临时文件。
- [ ] **RO-AC08**（不重复）GIVEN 已关闭说明，WHEN 在同一会话内切换屏幕再回主菜单，THEN 不再次弹出。
- [ ] **RO-AC09**（单账）GIVEN 说明显示，WHEN 检查正文，THEN 不出现钱包、纪录、版本号或文件路径。
- [ ] **RO-AC10**（无障碍）GIVEN 150% 字号与灰度截图，WHEN 检查，THEN 正文完整、两个出口可辨、焦点可见，不依赖颜色。

### BlockedModal

- [ ] **BLOCK-AC01**（强制模态）GIVEN `outcome = Blocked`，WHEN 进入 `Menu`，THEN 打开模态、`push_context(Modal)`、`modal_depth = 1`、背景主菜单不可交互。
- [ ] **BLOCK-AC02**（锁优先）GIVEN `lock_live = true` 且原因为更新版本/无法读取/迁移失败，WHEN 打开，THEN 只有「退出」「重试」，没有「新建生涯」。
- [ ] **BLOCK-AC03**（版本过高）GIVEN `lock_live = false` 且原因为更新版本，WHEN 打开，THEN 按钮为「退出」「新建生涯（旧档保留）」，无「重试」。
- [ ] **BLOCK-AC04**（无法读取）GIVEN `lock_live = false` 且主档与备份都不可读，WHEN 打开，THEN 按钮为「退出」「新建生涯（旧档保留）」。
- [ ] **BLOCK-AC05**（迁移失败）GIVEN `lock_live = false` 且迁移失败，WHEN 打开，THEN 按钮为「退出」「重试」，无「新建生涯」。
- [ ] **BLOCK-AC06**（默认焦点）GIVEN 任意分支，WHEN 打开，THEN 初始焦点为「退出」。
- [ ] **BLOCK-AC07**（不可取消）GIVEN 模态打开，WHEN 按 Esc / Backspace / 手柄 B，THEN 模态保持、不关闭、不穿透、不触发退出；窗口关闭仍可直接退出。
- [ ] **BLOCK-AC08**（背景不可操作）GIVEN 模态打开，WHEN 尝试用键盘/鼠标操作主菜单，THEN 全部无效；焦点不离开模态。
- [ ] **BLOCK-AC09**（重试）GIVEN 分支允许重试，WHEN 激活「重试」，THEN `retry_load()` 恰好一次、按钮禁用、显示「正在重新读取…」；再次激活不发起第二次。
- [ ] **BLOCK-AC10**（重试成功）GIVEN 重试返回 `LoadReport = ReadOnly`，WHEN 处理完成，THEN 关闭模态并显示只读说明；返回可写则关闭并恢复正常主菜单。
- [ ] **BLOCK-AC11**（重试失败）GIVEN 重试仍 `Blocked`，WHEN 返回，THEN 停留本模态、显示「仍无法读取」、按新报告的锁/原因重建出口、不隔离任何文件。
- [ ] **BLOCK-AC12**（新建生涯）GIVEN 分支允许新建，WHEN 确认「新建生涯（旧档保留）」后成功，THEN 旧档被隔离（改名而非删除）、新档建立、主菜单刷新钱包/升级/纪录，且没有发生覆盖写。
- [ ] **BLOCK-AC13**（无覆盖）GIVEN 任意分支，WHEN 遍历按钮与文案，THEN 不存在「覆盖保存」「忽略并读取」「强制新建」。
- [ ] **BLOCK-AC14**（窗口关闭）GIVEN 模态打开，WHEN 触发窗口关闭，THEN 直接进入退出流程，不要求先点按钮，也不把窗口关闭画成第三个按钮。
- [ ] **BLOCK-AC15**（无障碍）GIVEN 150% 字号、灰度、仅手柄，WHEN 完成打开→重试→失败→退出，THEN 全程可操作且原因/出口/焦点可辨。
- [ ] **BLOCK-AC16**（不重算）GIVEN 一个 `lock_live = false` 的迁移失败报告，WHEN 界面检查文件时间戳后「觉得能读」，THEN 仍按 `LoadReport` 给「退出/重试」，不得自行放行。

---

## 9. 开放问题

| # | 问题 | 双方依据 | 当前处理 |
|---|---|---|---|
| 1 | 强制模态的 `ui_cancel`：GDD 要求「显式消费为空操作」，PAT-07 又要求 `NONE` 必须产生**可见**的「无动作」。焦点环重绘是否足够？是否需要一行「请选择一个出口」？ | `UIF-R20` vs `PAT-07` / `UIF` `back_target` 注 | 本规格用焦点环重绘；若用户测试认为像卡死，再补静态提示（不新增按钮） |
| 2 | `Blocked` 下选择「新建生涯」后，新档是否获得首局引导资格（`first_run_eligible`）？GDD 只定义 `Fresh` / `Loaded` / `Migrated` 的判定。 | `SAV-AC31` vs `SAV-UI 4` + `ONB-R2` | 未定；本规格不显示、不推断，交由 `FirstRunGate` 裁定 |
| 3 | `retry_load()` 的时限与次数未定义；连续失败时玩家只能一直点「重试」或退出。是否需要退避/次数提示？ | `SAV-AC30`（允许再次重试）vs `ACC-COG-02` | 本规格允许无限重试、无超时；不新增计数文案，待实现评估 |
| 4 | 「新建生涯」失败（隔离改名失败、新档建立失败）时停在哪、显示什么、能否再试？GDD 未定义。 | `SAV-UI 4`（只写成功路径） | 本规格暂定停留 `CREATE_FAILED` 并显示原因，不关闭；需存档系统补契约 |
| 5 | `ReadOnly` 下是否允许「开始营业」？允许则整局无法入账，玩家可能白玩；禁止则「只读浏览」没有实际内容。是否在开始前加一次「本次会话不保存」确认？ | `UIF-R20`（未禁止开局）vs `UIF-R21` / `APPLIED_UNSAVED` 标记 | 本规格暂按允许，不额外确认；若加确认会与「不弹第二本账」冲突，需用户测试 |
| 6 | 只读期间「升级」入口是隐藏还是禁用？隐藏更干净，但玩家看不到「为什么不能升级」；禁用则需原因且可获焦。 | `UIF-R20`（隐藏）vs `PAT-03` / `ACC-VIS-15` | 本规格按 GDD 隐藏；若改为禁用，必须同时显示「存档为只读」 |
| 7 | 按钮文案不统一：本屏用「退出」，暂停层用「退出游戏」。是否统一？ | `UIF-UI`（两处分别写「退出」/「退出游戏」）vs `ACC-COG-08` 术语一致 | 本规格沿用各自 GDD 文案；建议统一为「退出游戏」，需回改 GDD |
| 8 | 新建生涯隔离文件的扩展名：存档 GDD 写 `corrupt-<UTC>.sav`，ADR-0002 的规范路径是 `.json`。 | `SAV-AC5` vs `ADR-0002` 路径表 | 本规格不显示文件名；实现前必须裁定，否则测试夹具会对不上 |
| 9 | 只读说明把初始焦点从「开始营业」抢到横幅，是否会让键盘玩家多按一次才能开局？ | `UIF-UI`（初始焦点「以只读继续」）vs `INP-R3`（主操作优先） | 本规格按 GDD；这是「必须告知」与「少一次按键」的取舍，待用户测试 |
| 10 | `Blocked` 的 debug 详情是否显示文件路径与记录 id？路径可能含用户名，截图/直播会泄露。 | `UIF-R18`（debug 显示失败文件与记录 id）vs 隐私 | debug 构建显示，release 不显示；是否对路径做家目录缩写，见 OQ-10 后续评估 |
