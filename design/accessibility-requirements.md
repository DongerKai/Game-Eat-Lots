# Accessibility Requirements（无障碍要求）

> **Status**: Committed — Tier 承诺已定；逐项实现审计未开始。**3 项 Standard 要求目前在 GDD 里没有载体**（输入重绑、色盲调色板开关、教学回看），见 Known Intentional Limitations 与 Open Questions 第 1 条；在 producer 裁定「补进 GDD」或「正式降级」之前，本文件不是 Certified。
> **Author**: user + ux-designer（autonomous 模式；Lean 模式不派生无障碍子代理）
> **Last Updated**: 2026-11-04
> **Game**: 回本 (Break Even)
> **Accessibility Tier Target**: Standard
> **Platform(s)**: PC（Windows / Linux / macOS）；键盘 + 鼠标为主、手柄部分支持、无触屏
> **External Standards Targeted**:
> - WCAG 2.1 Level **AA**（仅适用条款：1.4.3 文字对比度、1.4.4 文本缩放、1.4.11 非文本对比度、2.1.1 键盘、2.4.7 焦点可见。2.4.3 焦点顺序与 3.3.x 表单条款按游戏语境改写，不作逐条符合声明）
> - AbleGamers CVAA Guidelines — **不适用**：本作无语音聊天、无消息、无用户生成内容，不属 CVAA 覆盖的通信功能
> - Xbox Accessibility Guidelines (XAG) — **No**（无 Xbox 发行计划）
> - PlayStation Accessibility (Sony Guidelines) — **No**（无 PS 发行计划）
> - Apple / Google Accessibility Guidelines — **N/A**（无移动版）
> **Accessibility Consultant**: None engaged
> **Linked Documents**: `design/ux/hud.md`（局内 HUD 规格）、`design/gdd/systems-index.md`（系统清单与 UX note）、`design/gdd/input.md`、`ui-flow.md`、`in-game-hud.md`、`audio-feedback.md`、`first-run-onboarding.md`、`design/art/art-bible.md`
> **Template**: Accessibility Requirements

> **为什么这份文件存在**：逐屏的无障碍标注属于 UX 规格（`design/ux/hud.md` § 10 是局内那一屏）。本文件记录**项目级承诺**：选定的档位、跨系统要求编号、逐系统无障碍矩阵、测试计划与审计历史。当某个功能与本文件冲突时，**本文件优先**——改功能，不改承诺，除非 producer 批准正式修订。
>
> **更新时机**：每次 `/gate-check` 通过后、每次无障碍审计后、以及 `systems-index.md` 新增系统时（新增系统必须在 Per-Feature Accessibility Matrix 补一行）。

## 引用代号

| 代号 | 含义 |
|---|---|
| `ACC-VIS-nn` / `ACC-MOT-nn` / `ACC-COG-nn` / `ACC-AUD-nn` / `ACC-PLT-nn` | 本文件的要求编号；被 `design/ux/*.md`、stories 与测试计划引用 |
| `HUD-Rn` / `HUD-ACn` / `HUD-OQn` | `design/gdd/in-game-hud.md` 规则 / 验收 / 开放问题 |
| `INP-Rn` / `INP-ACn` / `INP-OQn` | `design/gdd/input.md` |
| `UIF-Rn` / `UIF-OQn` | `design/gdd/ui-flow.md` |
| `ONB-Rn` / `ONB-OQn` | `design/gdd/first-run-onboarding.md` |
| `AUD` | `design/gdd/audio-feedback.md`（cue 事件目录） |
| `ART-§n` | `design/art/art-bible.md` 第 n 节 |
| `HUD UX §n` | `design/ux/hud.md` 第 n 节 |

**一句话结论**：本作的无障碍瓶颈**不在手速，在眼睛和注意力**——所以承诺集中在「对比度、字号可放大、颜色永不单独承载语义、没有计时器」。三项缺口全部落在**设置入口**上（重绑、色盲调色板、教学回看），而这三项都可以在不改玩法规则的前提下补上。

---

## Accessibility Tier Definition

### Tier Definitions

| Tier | Core Commitment | Typical Effort |
|------|----------------|----------------|
| **Basic** | Critical player-facing text is readable at standard resolution. No feature requires color discrimination alone. Volume controls exist for music, SFX, and voice independently. The game is completable without photosensitivity risk. | Low — primarily design constraints |
| **Standard** | All of Basic, plus: full input remapping on all platforms, subtitle support with speaker identification, adjustable text size, at least one colorblind mode, and no timed input that cannot be extended or toggled. | Medium — requires dedicated implementation work |
| **Comprehensive** | All of Standard, plus: screen reader support for menus, mono audio option, difficulty assist modes, HUD element repositioning, reduced motion mode, and visual indicators for all gameplay-critical audio. | High — requires platform API integration and significant UI architecture |
| **Exemplary** | All of Comprehensive, plus: full subtitle customization (font, size, color, background, position), high contrast mode, cognitive load assist tools, tactile/haptic alternatives for all audio-only cues, and external third-party accessibility audit. | Very High — requires dedicated accessibility budget and specialist consultation |

### This Project's Commitment

**Target Tier**: **Standard**

**Rationale**：本作是回合制算术游戏，**没有实时输入、没有 QTE、没有连打、没有镜头与准星**（`INP-R5`、`HUD-R17`、`INP-AC16`），所以动作类游戏最常见的运动障碍在这里几乎不存在——Basic 与 Standard 在“能不能玩完”上的差距，主要落在**看得清**与**按键能改**这两件事上。反过来，它的信息密度是玩法本身：一屏要同时读回合、容量、净利、差额、四张卡（`HUD` 信息层级 ①–⑤），这对低视力、色觉缺陷、阅读障碍与注意力障碍玩家的负担远高于普通回合制。色觉缺陷约影响 8% 男性，而本作的涨/亏、可吃/不可吃、已揭晓/未揭晓都用颜色做第一通道（`ART-§4`），**这一条不解决，Standard 以下没有意义**，所以文本缩放、颜色冗余与灰度可读被提到 ship-blocking。Comprehensive 不选的实理由：屏幕阅读器要穿透一层**自绘 HUD**（不是 Godot `Control` 树，见 Platform 一节）、HUD 重排会破坏“三块读数永不遮挡”的硬判据（`HUD UX § 9`）、单声道音频在无定位音的游戏中无收益——三项都要显著工程预算，而它们覆盖的玩家子集小于上面两条。代价说明：降到 Basic 会失去字号放大与颜色冗余，按 AbleGamers 的分档口径，等于排除需要 >20 px 文本的低视力玩家与依赖非色相编码的色觉缺陷玩家。

**Features explicitly in scope (beyond tier baseline)**：
- **三档文本缩放覆盖全部屏幕**（不只 HUD）：菜单、收据、升级、纪录、设置、引导气泡、错误屏同样吃 `text_scale_percent`（`ACC-VIS-09`）。模板只要求“可调字号”，这里升级为全屏一致。
- **灰度可读作为 ship-blocking 验收**（`ACC-VIS-08`）：比“至少一个色盲模式”更强——不做“切了某个模式才可用”的设计，默认状态就必须在灰度下成立。
- **无计时器承诺写成可测条款**（`ACC-COG-02`/`03`）：包括确认面板 60 秒不自动关闭、`ui_accept` 等待 10 秒结果不变（`INP-AC16`）。
- **HUD 文字 α 锁定 1.0**（`ACC-VIS-14`）：不透明度可调，但文字不跟着变透明，避免“调低不透明度后读不清”。

**Features explicitly out of scope**（逐条对应 Known Intentional Limitations）：
- 输入重绑（Standard 项，`INP-R10` 明确 MVP 不做）——**缺口**
- 色盲调色板**开关**（Standard 项，`UIF-R24` 的设置清单里没有这一项）——**缺口**；结构编码本身在范围内
- 教学回看 / Help 屏（Standard 项，`ONB-R2` 禁止重看，`UIF-R2` 屏幕清单无 Help）——**缺口**
- 屏幕阅读器、HUD 重排、单声道音频、高对比模式（均为 Comprehensive）
- 触屏与虚拟摇杆（`INP-OQ5`）
- 字幕与说话人标识（**N/A**：无配音，见 Auditory）

---

## Visual Accessibility

字号基准为 1280×720。**数字 28 px / 正文 20 px / 注释 16 px** 是 HUD 的下限（`HUD UX § 7`）；菜单类屏幕正文 20 px、按钮 20 px、收据表头 16 px。

| Feature | Target Tier | Scope | Status | Implementation Notes |
|---------|-------------|-------|--------|---------------------|
| `ACC-VIS-01` 最小字号 — HUD | Standard | 局内一屏全部文本 | 未实现 | 数字 ≥ 28 px、正文 ≥ 20 px、注释 ≥ 16 px @720p；**注释字号只用于非决策信息**（如“共 3 组”）。电视观看距离按 720p 基准 ×1.5 生效（`HUD UX § 9`）。 |
| `ACC-VIS-02` 最小字号 — 菜单 / 收据 / 升级 / 纪录 / 设置 / 错误 | Standard | 界面流程拥有的全部屏幕 | 未实现 | 正文 ≥ 20 px、按钮 ≥ 20 px、表头与注释 ≥ 16 px @720p。收据行即使密集也不得降到 16 px 以下；行高随字号增长，宁可滚动（`UIF-R11` 的滚动行为）。 |
| `ACC-VIS-03` 文字对比度 ≥ 4.5:1 | Standard | 全部屏幕的全部文字，含 ≥24 px 大字号 | 未实现 | 用 `hud_contrast_ratio`（sRGB / WCAG 相对亮度）实测，**大字号不放宽到 3:1**（艺术总监的价目板底纹会让“看起来够大”产生误判）。实测表见下。 |
| `ACC-VIS-04` 语义色 token 的使用边界 | Standard | 全部 UI | 已定（待回改 GDD） | 橄榄绿 `#6E7C3C`（4.0:1）与警示橙 `#B5651D`（3.8:1）**永不承载文字**；回本红 `#C63A2F` 只许落在米白底（`/深面板 = 2.9:1`）；冷灰 `#8C918E` **在米白底上 2.8:1，禁止承载文字**（只可做禁用填充与非文本装饰）。“比数字淡一档的标签”改用深灰 `#4A4E4A`（7.5:1）。冲突见 Open Questions 第 7 条。 |
| `ACC-VIS-05` 非文本对比度 ≥ 3:1 | Standard | 焦点环、容量条填充/边界、分隔线、开关与选中态 | 未实现 | 交互蓝 `#2F6F8F` / 米白 = 4.9:1，通过；**交互蓝 / 深面板 = 2.7:1，不通过**——焦点环若落在深底控件上，必须在外侧加 2 px 米白描边把蓝环与深底隔开。容量条填充（橄榄绿 4.0:1）与空缺（米白）自身即对比，加粗边界保证灰度下也能看清“装到哪”。 |
| `ACC-VIS-06` 色彩之外的结构编码 | Standard | 全部颜色语义 | 已定 | 正收益 = `+` 号 + 红；负值 = `−`（U+2212）+ 墨黑；容量 = 填充/空缺 + `已用 12 / 20 格`；不可吃 = 40% 不透明 + 2 px 左线 + 原因句；未揭晓 = `?` + 菜数；解腻 = 「解」字方标；焦点/悬停 = 描边环 vs 浅底（形状不同）。完整表见 `HUD UX § 10.1` 与下方 Audit。 |
| `ACC-VIS-07` 双焦点可区分且非颜色 | Standard | HUD 与全部列表 | 未实现 | 键盘/手柄焦点 = 2 px 实线外框 + 2 px 间距；鼠标悬停 = 浅色底、无外框。两者形状不同，可同屏并存（`INP-R8`、`HUD-AC38`）。**只靠色相区分视为缺陷**。 |
| **`ACC-VIS-08` 灰度可读（Grayscale）** | Standard | 全部屏幕的全部语义状态 | 未实现 | **一份去掉色相的截图必须能读出**：净利正负、是否回本、可吃/不可吃、跳过代价、容量余量、波次进度与未揭晓数、腻度计数与解腻、焦点位置、结束/故障原因、按钮可用/禁用、钱包正负、回本判定、纪录/里程碑、「未保存」标记（`HUD-AC29`、`HUD-AC42`）。验收方式是逐行跑下面的 Audit 表，不靠肉眼判断。 |
| `ACC-VIS-09` 文本缩放 100 / 125 / 150% | Standard | 全部屏幕 | 未实现 | 设置项 `text_scale_percent`（`UIF-R24`），**只有三档，不提供 75%**。缩放只放大文字与行高，不放大图标与安全边距；布局按规则重排。三档都必须能完成一局并读完收据。 |
| `ACC-VIS-10` 缩放后的重排规则 | Standard | 全部屏幕 | 未实现 | 任何字号下：不截断（名称 2 行截断后完整名进详情行）、不重叠、不消失、不出现横向滚动。150% 时 HUD 卡片降为 2 列（`HUD-AC11`、`HUD-AC30`），菜单与收据行改为折行并保持可滚动。 |
| `ACC-VIS-11` 信息不依赖悬停 | Standard | HUD、升级、纪录、收据 | 已定 | 不存在“只有悬停才出现”的信息（`HUD UX § 10.6`）：卡片全部字段同屏（`HUD-R8`），焦点详情行由**焦点或悬停任一**触发。纯鼠标用户不需要移动鼠标来确认信息是否存在。 |
| `ACC-VIS-12` 减少动态 `reduce_motion` | Standard | 全部 UI 过渡、数字动效、菜单切换 | 未实现 | 开启后：数字位移/放大、焦点描边过渡、拒绝抖动、确认面板淡入、`Handoff` 淡出、界面过渡、收据数字滚动**全部置 0**，替换为瞬时状态切换（`HUD-AC31`、`UIF-R14`、`UIF-R24`）。保留的“动效”只有语义必须的状态差异（描边变粗、面板出现）。 |
| `ACC-VIS-13` 无闪光 / 频闪 | Basic | 全局 | 已定 | 本作**不产生**粒子、闪光、屏幕震动、镜头位移（`HUD` Visual：无 VFX、无粒子、无屏幕震动）。因此不需要 Harding FPA 逐条审计，也不需要光敏警告屏；**若未来加入任何全屏闪光，本节必须重写并加入启动前警告**。 |
| `ACC-VIS-14` HUD 不透明度 60–100% | Standard | 局内 HUD | 未实现 | `hud_opacity_percent` 只对**面板与线**乘 alpha；**文字 α 恒为 1.0**，面板底色单独钳制 ≥ 0.70，因此最低档仍 ≥ 4.5:1（`HUD-AC32`）。**不提供 0%（全关 HUD）**——那会让本局无法进行（`HUD-OQ6`）。 |
| `ACC-VIS-15` “不可吃”与“已禁用”必须肉眼可分 | Standard | 菜品卡与全部按钮 | 已定 | 两者都变灰是允许的，但：不可吃 = 有原因句、**仍可获焦、仍可读全部字段**（`HUD-R10`）；流程禁用（`Ending` / `Frozen` / `Faulted`）= 无原因句、**不可获焦、不画焦点环**。灰度下靠“有没有原因句、有没有焦点环”区分，不靠灰色深浅。 |

### Color-as-Only-Indicator Audit

> 每一行都必须有一个在**任意色板**与**灰度**下都成立的非颜色通道。状态：已定 = 设计已确定；未实现 = 设计已定但代码/资产未做。

| Location | Color Signal | What It Communicates | Non-Color Backup | Status |
|----------|-------------|---------------------|-----------------|--------|
| 顶栏净利 | 红 = 正收益，墨黑 = 负值 | 这一局到目前为止赚/亏 | `+` / `−`（U+2212）符号；金额列右对齐；位置固定 | 已定 |
| 顶栏回本差额 | 红 = 已回本附近 | 还差多少 / 已回本 | 文案「还差 46 元」/「已回本」；不显示 `0`（`HUD-R18`） | 已定 |
| 容量条 | 橄榄绿 = 已用 | 胃里装了多少 | 填充/空缺形状 + 加粗边界 + `已用 12 / 20 格` | 已定 |
| 菜品卡可吃性 | 40% 不透明 + 左线 | 这道菜放不放得下 | 「需要 6 格，剩 4 格」原因句（100% 不透明）+ 仍可获焦 | 已定（`HUD-AC18`） |
| 卡片解腻标记 | 橄榄绿描边 | 这道菜会解掉一格腻度 | 「解」字方标 + 详情行「将解掉：肉 −1」 | 已定 |
| 波次条 | 冷灰 = 未揭晓 | 还有几波没看到 | `?` 字符 + 菜数（`?4`）+ 实心/空心槽 | 已定 |
| 焦点 vs 悬停 | 蓝色描边 vs 浅底 | 键盘焦点在哪、鼠标在哪 | 形状不同（描边环 / 无环填充），可同屏并存 | 已定 |
| 按钮可用性 | 冷灰 = 禁用 | 这个按钮现在不能按 | 不可获焦 + 无焦点环 + 旁边一句原因（如「没有回合了」） | 已定 |
| 主菜单钱包 | 红 / 墨黑 | 钱包是否为正 | 正负号 + 数字对齐 | 未实现 |
| 收据回本判定 | 红 = 回本 | 这一局赢没赢 | 文案「回本」/「未回本」+ 回本率百分比 + 净利符号 | 未实现 |
| 收据「新纪录」 | 红印章 | 破了个人最佳 | 「新纪录」文字 + 旧最佳并排显示（`UIF-R11`） | 未实现 |
| 升级页可购买性 | 颜色 + 灰 | 买得起/买不起 | 价格与余额同时显示 + 不可购买原因句 | 未实现 |
| 纪录/里程碑 | 颜色分组 | 已达成/未达成 | 「已达成 / 未达成」分组标题 + `当前值 / 阈值` 进度 | 未实现 |
| 「未保存」标记 | 颜色 | 本次会话不写盘 | 文字「本次会话不保存」/「未保存」+ 保存指示 | 未实现 |
| 错误屏 / 故障横幅 | 冷灰去饱和 + 45° 斜线纹 | 这张单打不出来 | 明确文案 + 斜线纹 + 错误码（debug） | 已定 |
| 汉堡式“负值报警” | 橙 | 时间/容量告急 | **本作不存在**：容量满不报警（`HUD` Visual），回合耗尽不弹窗（`HUD-R5`） | 不适用 |

### 实测对比度表（WCAG 相对亮度，非估计）

| 组合 | 实测 | 允许用途 |
|---|---|---|
| 墨黑 `#1F1F1C` / 米白 `#F5F1E6` | **14.6:1** | 全部正文与数字 |
| 米白 `#F5F1E6` / 深面板 `#1F2A24` | **13.2:1** | 深底上的文字（引导气泡正文） |
| 次级墨 `#4A4E4A` / 米白 | **7.5:1** | 标签、单位、注释（“比数字淡一档”的唯一合法做法） |
| 交互蓝 `#2F6F8F` / 米白 | **4.9:1** | 焦点环、悬停底、选中下划线（非文本，≥3:1） |
| 回本红 `#C63A2F` / 米白 | **4.6:1** | 正收益数字、2 px 关键线；**仅米白底** |
| 橄榄绿 `#6E7C3C` / 米白 | **4.0:1** | 容量条填充（非文本）；**禁止文字** |
| 警示橙 `#B5651D` / 米白 | **3.8:1** | 2 px 线（非文本）；**禁止文字**；MVP 不使用 |
| 冷灰 `#8C918E` / 米白 | **2.8:1** | **禁止文字**；仅禁用填充、装饰、非文本边界 |
| 冷灰 `#8C918E` / 深面板 | **4.6:1** | 深底次级文字可用（同一 token 换个底就合法） |
| 回本红 `#C63A2F` / 深面板 | **2.9:1** | **禁止**——“数字必须落在米白纸面”的实测依据 |
| 交互蓝 `#2F6F8F` / 深面板 | **2.7:1** | **禁止**直接作焦点环；需 2 px 米白隔边 |

### Text Scaling（100% / 125% / 150%）

| 屏幕 / 元素 | 100% | 125% | 150% | 溢出行为 |
|---|---|---|---|---|
| HUD 顶栏 | 72 px 高 | 72–84 | ≤ 96 | 波次条只保留一行汇总文字 |
| HUD 卡片网格 | 4 列 × 216×176 | 3 列 | 2 列 × 448×224 | 名称 2 行截断；价格/饱腹不折行 |
| HUD 详情行 | 1 行 / 44 px | 56 px | 2 行 / 72 px | 超 2 行视为文案缺陷，回改文案表 |
| HUD 右栏（腻度 / 预览 / 按钮） | 280 px | 280 | 280 | 组名与计数同行或折行，不截断计数 |
| 收据行 | 单价、饱腹、追加、实际饱腹、两种单位价值 | 折为 2 行 | 折为 2–3 行 | 行高增加并纵向滚动，**不缩字号、不横向滚动** |
| 收据合计区 | 单列 | 单列 | 单列 | 净利与回本判定不得被折到屏外；滚动可用键盘/手柄 |
| 主菜单 / 升级 / 纪录 | 列表 | 列表 | 列表 | 按钮文字不折；列表项折行 |
| 设置模态 | 单项一行 | 单项一行 | 标签与数值可折 2 行 | 滑块与开关命中区不缩小（≥ 44×44） |
| 引导气泡 | ≤ 3 行 | ≤ 3 行 | ≤ 3 行 | 超出即文案缺陷（`ONB` 文案上限 48 字短版） |

**判定线**：三档 × 四种目标分辨率（1280×720、1280×720@150%、1920×1080、2560×1080）的 12 组组合都必须“无溢出、无重叠、无消失”。已知最紧的一处是 **150% 下 HUD 网格与详情行只剩 16 px 余量**（`HUD UX § 10.2`），必须实机复核。

### Grayscale Requirement（`ACC-VIS-08`）

做法：对每一组目标截图（`Active` 720p / 1080p / 150%、`Frozen`、`Ending`、确认面板、不可吃卡、`Faulted`、收据、升级、设置）**去掉色相**后逐条回答：

1. 净利是正是负？（看符号与墨色深浅，不看红）
2. 回本了没有？（看「还差 46 元」/「已回本」）
3. 哪张卡不能吃、为什么？（看原因句与左线）
4. 跳过要付什么？（看「放弃 3 道 · 花 1 回合」整句）
5. 容量还剩几格？（看填充形状与整数）
6. 还有几波没揭晓？（看 `?` 与数字）
7. 哪些组已经腻了、哪道菜能解？（看组名 ×N 与「解」标）
8. 焦点在哪？（看 2 px 外框，不看蓝）
9. 为什么结束 / 出了什么错？（看通栏文字）
10. 这个按钮现在能不能按？（看焦点环与原因句）

十问全部答得出才算通过；任何一问答不出，就回到上表补一个非颜色通道，**不许靠“加一个色盲模式”绕过**。

---

## Motor Accessibility

本作没有实时输入：每个动作在按下沿触发一次，无长按、无连打、无同时按键、无计时窗口（`INP-R1` 的动作集只有 8 个，`INP-R5` 规定按下沿一次触发）。因此本节的多数模板项是**不适用**，而不是“以后做”。

| Feature | Target Tier | Scope | Status | Implementation Notes |
|---------|-------------|-------|--------|---------------------|
| `ACC-MOT-01` 键盘/手柄完整可玩 | Standard | 全部屏幕与局内操作 | 未实现 | 只用键盘或只用手柄都能从开局走到结算并读完收据（`INP-R4`、`INP-AC4`）。鼠标是可选加速器。HUD 内只有 **5 个焦点位**（4 卡 + 1 按钮），不存在“必须用鼠标”的动作（`game_skip_wave` 有 S / X 绑定，右键不是唯一通道）。 |
| `ACC-MOT-02` 完整输入重绑 | Standard | 全部平台 | **未满足（MVP 缺口）** | `INP-R10` 明确「无重绑（MVP）」，`INP-OQ1` 把重绑推到上线前。缓解：动作只有 8 个、方向键与 WASD 双绑定、无组合键、无长按；Steam Input 可在系统层重映射。**这是 Standard 的硬缺口，见 Known Intentional Limitations 第 1 条。** |
| `ACC-MOT-03` 无按住 / 连打 / 计时输入 | Standard | 全部交互 | 已定 | 不存在 hold 动作（因此不需要 toggle 替代）；`ui_accept` / `game_skip_wave` 长按只触发一次（`INP-R5`、`INP-AC5`）；无 QTE、无节奏输入、无“几秒内决定”（`INP-AC16`：等待 10 秒结果不变）。模板的 “hold 替代 / rapid input 替代 / 计时乘数” 三行**不适用**。 |
| `ACC-MOT-04` 输入方式随时切换、焦点保持 | Standard | PC | 未实现 | 键鼠与手柄可随时切换，不需要重启或重开菜单；切换后焦点停在原控件（`HUD-AC39`）。HUD 与按钮**不画键位图标**（`HUD UX § 9`，`INP-OQ2` 未决），只写语义动作名，避免切设备后提示说谎。 |
| `ACC-MOT-05` 单手可达 | Standard | 手柄 | 部分满足 | 无同时输入；手柄上 D-pad 导航 + A 确认 + B 取消 + X 跳过可由单手握持完成（拇指在正面）。键盘单手**不承诺**：确认键（Enter/Space）与方向键相距较远，且无重绑。缓解见第 1 条。**不提供“单手模式”开关**。 |
| `ACC-MOT-06` 命中区与焦点可见 | Standard | 全部可交互元素 | 未实现 | 卡片 ≥ 200×152 px、按钮 ≥ 56 px 高；焦点环 2 px 实线 + 2 px 间距，**永不被裁切**、永不被遮挡（`HUD-AC41`）。焦点环是非文本对比度条款（`ACC-VIS-05`）。 |
| `ACC-MOT-07` 导航重复与回滞 | Standard | 键盘 / 摇杆 | 已定 | 首次重复延迟 `D = 0.35 s`、间隔 `I = 0.12 s`（`INP` 调参起点），摇杆进入 0.60 / 释放 0.40 的回滞防漂移。焦点移动一步一格、不加速（`INP` Game Feel）。误导航**零代价**：移动焦点不消耗回合或容量。 |
| `ACC-MOT-08` 无精度要求 | Standard | 全局 | 已定 | 没有瞄准、没有拖拽、没有双击、没有右键作为唯一入口；点击落空不产生任何后果（`INP` Edge Cases）。模板的 “Aim assist / Auto-sprint / Platforming assist” **不适用**。 |
| `ACC-MOT-09` HUD 元素重排 | Comprehensive | 局内 HUD | **不做** | 固定九宫格；重排会破坏“回合/容量/净利永不遮挡”的硬判据（`HUD UX § 9`）。缓解：150% 缩放 + 60% 不透明度 + 焦点位固定为 5 个。见 Known Intentional Limitations 第 4 条。 |

---

## Cognitive Accessibility

| Feature | Target Tier | Scope | Status | Implementation Notes |
|---------|-------------|-------|--------|---------------------|
| `ACC-COG-01` 随时暂停 | Basic | 全部状态 | 已定 | `ui_pause` 在 `Blocked` 中也始终放行（`INP` 规则 12），因此加载、首局引导、结算动画都可以暂停；`Run` 中暂停不推进回合、不改任何数字（`UIF-R15`）。暂停时 HUD 保留、只降饱和，玩家不会丢失位置。 |
| `ACC-COG-02` 无任何时间压力 | Standard | 全部界面 | 已定 | 没有倒计时、没有秒表、没有“思考超时”、没有自动推进的引导（`HUD-R17`、`ONB` Game Feel）。唯一的“时间”是回合数这个资源，它不会自己减少。 |
| `ACC-COG-03` 确认面板不会自动关闭 | Standard | HUD 确认面板、确认模态 | 已定 | 等待 60 秒不操作，面板既不关闭也不自动确认（`HUD-AC33`）；收据数字滚动可被跳过但不会自行跳过关键判定（`UIF-R14`）。 |
| `ACC-COG-04` 可操作提示不早于 5 秒消失 | Standard | 全部提示 | 已定 | 本作没有自动消失的**可操作**提示：确认面板要输入、错误与结束横幅保留到离开、引导气泡要确认。唯一的短暂提示是“动作被拒”的强调态（≤ 150 ms）——但**原因句同时常驻在卡片与详情行**，所以短暂强调消失不构成信息丢失。保存指示（1.5 s）与读操作无关。 |
| `ACC-COG-05` 教学可回看 | Standard | 首局引导 | **未满足（MVP 缺口）** | `ONB-R2` 规定「没有中途开关、没有设置项、没有『再看一次』入口」，`UIF-R2` 的屏幕清单里也没有 Help 屏。缓解：教学只讲 HUD 上已有的字段（价格、饱腹、容量、回合、波次、解腻），这些字段**在整局中常驻可读**；首局未结算就放弃时下一局仍是首局（`ONB-R22`），教学会重来。见 Known Intentional Limitations 第 3 条。 |
| `ACC-COG-06` 目标清晰 | Standard | 局内 | 已定 | 任何时刻玩家只需要回答一个问题：「这一口吃哪道 / 跳波 / 结束」。三个选项都常驻可见（菜品卡 + C3 按钮），当前波与剩余回合同屏（`HUD` 信息层级 ①）。**不需要推断**“该干什么”，也没有跨屏任务。 |
| `ACC-COG-07` 认知负荷上限 | Comprehensive（本作按 Standard 自愿执行） | 逐系统 | 已定 | 见 Per-Feature Accessibility Matrix 的「同时追踪项」。判据：任一系统要求玩家同时记住 **> 4 项**即触发复核。HUD 恰好 4 个读数（回合、容量、净利、差额）+ 4 张卡；卡片是**逐张比较**，不同时要求记住。腻度组计数可达 6 组 → 面板逐行列出、不要求记忆。 |
| `ACC-COG-08` 文案与数字一致 | Standard | 全部文本 | 已定 | 所有玩家可见字符串来自文本表（`HUD-R20`、`ONB-R20`），数字来自读数、不写死在句子里；术语固定（回合 / 容量 / 格 / 净利 / 回本 / 腻度），不出现“最优解”“建议先吃”。金额整数元、无小数、无千分位（`HUD-R18`）。 |
| `ACC-COG-09` 设置即时生效并持久化 | Standard | 设置模态 | 未实现 | 七项设置（主音量、音效音量、静音、减少重复音效、文本缩放、减少动态、HUD 不透明度）修改立即生效（`UIF-R24`），关模态或退出时写机器级设置文件；写失败显示「设置可能未保存」并保留内存值。 |
| `ACC-COG-10` 难度与辅助 | Standard | 全局 | **N/A** | 本作没有难度参数、没有失败性实时挑战、没有资源惩罚到不可恢复的机制；“回本/未回本”是成绩，不是失败墙。因此不提供难度滑块，也不提供“跳关”。若未来加入计时或惩罚机制，本节必须重写。 |

---

## Auditory Accessibility

本作 **MVP 与 Vertical Slice 无背景音乐、无配音、无对话、无旁白**（`AUD` 规则 18，`game-concept` 无叙事系统）。因此字幕、说话人标识、字幕定制、声音描述字幕**全部 N/A**，且不需要在首启时询问字幕偏好。

| Feature | Target Tier | Scope | Status | Implementation Notes |
|---------|-------------|-------|--------|---------------------|
| `ACC-AUD-01` 字幕 | Basic | — | **N/A** | 无语音内容。若未来加入配音，必须补 100% 字幕、说话人标识与字幕区布局（且不得占用 HUD 卡片面积），并重写本节。 |
| `ACC-AUD-02` 每个游戏关键音效有视觉等价 | Standard | `AUD` cue 目录全部 | 已定（逐条见下表） | 总则：**没有任何一条音频携带唯一信息**。设计顺序是先有视觉事实、再有声音——声音是对已发生事实的强调，不是通知。 |
| `ACC-AUD-03` 独立音量 | Basic | Master / SFX | 未实现 | 设置提供主音量（−80…0 dB）、音效音量（−40…0 dB）、静音（`UIF-R24`）。模板要求音乐/SFX/语音/UI 四路独立——本作无音乐与语音，UI 音随 SFX 总线，因此**两路 + 静音**是完整覆盖，不是删减。`reduce_repetitive_cues` 另提供“减少重复音效”开关。 |
| `ACC-AUD-04` 单声道音频 | Comprehensive | 全局 | **不做** | 本作无定位音、无立体声分离信息、无“听声辨位”；单声道折叠不改变任何可玩信息。属无收益项，不作为缺口计入。 |
| `ACC-AUD-05` 助听器 / 高频 | Standard | 全部 cue | 已定 | 任何 cue 都不得只靠高频（> 4 kHz）传达信息；所有 cue 都有视觉等价（`ACC-AUD-02`），因此高频被滤掉不损失任何决策。 |
| `ACC-AUD-06` 静音 / 无设备是合法状态 | Standard | 全局 | 已定 | 静音与设备缺失只丢弃请求、不报错、不阻塞、不改状态（`AUD` 规则 15/17）；**整局可以在完全静音下完成且不丢失任何信息**——这正是本节的验收方式。 |
| `ACC-AUD-07` 减少重复音效 | Standard | 全局 | 未实现 | `reduce_repetitive_cues`（默认关）降低连续数字 tick 与重复点击音；开启后不得改变任何视觉事实或节奏。 |

### Gameplay-Critical SFX Audit

| Sound Effect | What It Communicates | Visual Backup | Caption Required | Status |
|-------------|---------------------|--------------|-----------------|--------|
| `sfx.run.start` | 进店、开局 | 首帧 HUD 全部读数出现（净利 `−85`、`剩余 10 / 10`） | 否 — 视觉充分 | 已定 |
| `sfx.eat.commit` | 这一口落定 | 回合、容量、净利、卡片在同一帧变化（`HUD-R4`、`HUD-AC15`） | 否 | 已定 |
| `sfx.eat.reject` | 放不下 / 没有回合了 | 卡片原因句 + 详情行文字 + 2 px 左线（`HUD-R11`） | 否 | 已定 |
| `sfx.wave.skip` | 翻到下一波 | 波次条索引变化 + 新卡片出现 + 回合数减一 | 否 | 已定 |
| `sfx.meal.end` | 主动收手 | `Ending` 横幅 + 结束原因文字，控件禁用 | 否 | 已定 |
| `sfx.run.end` | 本局结束判定 | `Ending` 横幅 + 数值冻结 | 否 | 已定 |
| `sfx.run.fault` | 数据/结算故障 | `Faulted` 横幅 + 受影响数字 `—` + 错误码（debug） | 否 | 已定 |
| `sfx.run.discard` | 本局被丢弃 | 回到主菜单 + 「本局已放弃，收益不计」提示（`UIF-R19`） | 否 | 已定 |
| `sfx.settle.print` / `broke_even` / `loss` | 收据打开、回本与否 | 收据屏本身：净利、回本判定「回本」/「未回本」、回本率 | 否 | 已定 |
| `sfx.num.tick` / `sfx.num.land` | 数字上涨 / 停住 | 数字**先同帧到位**（`HUD-R4`）；本规格默认 `duration_ms = 0`，tick 为 0 次（`HUD-OQ3`） | 否 | 已定 |
| `sfx.ui.click` | 控件被激活 | 控件状态变化（进入下一屏、开关翻转、焦点仍在） | 否 | 已定 |
| `sfx.flow.pause` / `resume` / `menu` | 冻结 / 恢复 / 回菜单 | 整屏降饱和 + 暂停菜单出现 / 消失 / 主菜单出现 | 否 | 已定 |
| `sfx.record.broken` / `sfx.milestone` / `sfx.career.complete` | 新纪录 / 新里程碑 / 通关 | 收据上的「新纪录」印章、底部「新达成」条、完成横幅（`UIF-R23`） | 否 | 已定 |
| 焦点移动、悬停、按下未提交 | 无（`AUD` 规则 19/20：刻意不做） | 焦点环 / 悬停底 | 否 | 不适用 |

---

## Platform Accessibility API Integration

| Platform | API / Standard | Features Planned | Status | Notes |
|----------|---------------|-----------------|--------|-------|
| PC — Steam | Steam Input / Steam Accessibility | 系统层按键重映射（覆盖 `ACC-MOT-02` 的缓解路径）、手柄型号图标由 Steam 提供（本作不画图标） | 未实现 | Steam Input 可在系统层改键，但**不能替代游戏内重绑**：它不覆盖键盘、不写进本作设置文件，也不解决“不知道能改”的发现性问题。 |
| PC — Windows（OS 级） | 鼠标键、粘滞键、筛选键、显示缩放 | 不干扰系统级辅助（不独占输入、不锁光标、不忽略 OS 缩放） | 未实现 | 窗口可缩放、不强制全屏；OS 显示缩放下按窗口尺寸重排，不做二次 DPI 缩放（`HUD UX § 9`）。 |
| PC — 屏幕阅读器（NVDA / JAWS / Windows Narrator） | Godot `AccessKit`（4.5+，覆盖 `Control` 节点的名称/角色/状态） | 菜单与设置的元素名、角色、选中态 | **不承诺（Comprehensive）** | 两个障碍：①局内 HUD 是**自绘层**（`_draw`），不是 `Control` 树，AccessKit 读不到；②菜单虽有 `Control` 节点，但本作未定义 accessible name/role 数据。降级方案见 Known Intentional Limitations 第 5 条。 |
| Xbox (GDK) | XAG / Game Core Accessibility | — | **N/A** | 无 Xbox 发行计划，不承诺 XAG 合规。 |
| PlayStation 5 | Sony AccessibilityNode | — | **N/A** | 无 PS 发行计划。 |
| Nintendo Switch | 平台无障碍指南 | — | **N/A** | 无 Switch 发行计划。 |
| iOS / Android | UIAccessibility / TalkBack | — | **N/A** | 无移动版；触屏与虚拟摇杆也不做（`INP-OQ5`）。 |
| Steam Deck（硬件形态） | 手柄 + 触屏 | 手柄路径完整可玩 | **未验证** | 1280×800（16:10）的布局档未定（`HUD-OQ9`）；触屏不提供虚拟摇杆。 |

---

## Per-Feature Accessibility Matrix

> `systems-index.md` 的 20 个系统全部列出；无玩家可见界面的系统标“不适用”并说明原因。新增系统必须补行。「同时追踪项」是 `ACC-COG-07` 的输入：> 4 触发复核。

| System | Visual Concerns | Motor Concerns | Cognitive Concerns | Auditory Concerns | Addressed | Notes |
|--------|----------------|---------------|-------------------|------------------|-----------|-------|
| 数据配置 | 无界面；错误只经错误屏 | 不适用 | 不适用 | 无 | 不适用 | 数据校验失败经 `ErrorScreen` 给出可读文案（`UIF-R18`），不靠颜色。 |
| 输入 | 焦点环与悬停必须非颜色可辨 | **重绑缺失是 Standard 缺口**；无 hold/连打 | 8 个固定动作，记忆面小 | 不适用（焦点无声） | 部分 | 见 `ACC-MOT-02`、`ACC-VIS-07`。 |
| 场景与流程管理 | 状态切换无闪光；降饱和为静态 | 不适用 | 暂停与恢复不丢进度 | 不适用 | 已定 | 过渡 ≤ 0.2 s，`reduce_motion` 时置 0。 |
| 存档 | `Blocked` / `ReadOnly` / `Recovered` 必须文字说明；「未保存」不能只靠色 | 不适用 | 强制模态有确定出口；默认焦点在「退出」 | 不适用 | 已定 | 不提供覆盖保存；删除生涯两次确认（`UIF-R20`、`UIF-R24`）。 |
| 餐局流程 | HUD 流程态（`Active`/`Frozen`/`Ending`/`Faulted`）视觉可辨 | 不适用 | **同时追踪：4 个读数** | 不适用 | 已定 | 结束原因必须可归因（`RUN-UI 8`）。 |
| 时间与回合 | `剩余 6 / 10` + 槽位，颜色不承载语义 | 不适用 | **同时追踪：1 个**（回合数） | 不适用 | 已定 | 无秒数、无递减动画（`HUD-R12`）。 |
| 胃容量 | 容量条填充/空缺 + 整数 + 加粗边界 | 不适用 | **同时追踪：1 个**（剩余格） | 不适用 | 已定 | 满了不变红、不报警（`HUD` Visual）。 |
| 菜品数值 | 价格/饱腹用等宽数字，单位写清（元 / 格） | 不适用 | **同时追踪：2 个**（价格、饱腹） | 不适用 | 已定 | 不显示 `unit_value`，不替玩家算比值（`HUD-R9`）。 |
| 腻度与口味 | 「解」字标 + 组名 ×N；橄榄绿不承载文字 | 不适用 | **同时追踪：最多 6 组**（触发复核） | 不适用 | 部分 | 组数多时要靠面板逐行读；详情行给出“将解掉：肉 −1”避免心算。 |
| 菜单波次 | 未揭晓用 `?` + 菜数；跳过代价整句 | 不适用（但右键跳过有误触风险，`HUD-OQ5`） | **同时追踪：波次索引 + 剩余回合** | 不适用 | 部分 | 跳过不可撤销且无确认 → 误触代价高，见 Open Questions 第 9 条。 |
| 结算与入场费 | 收据为纯文字 + 数字；回本判定有文字 | 不适用 | **同时追踪：总值、入场费、净利、回本率**（4，上限） | 不适用 | 未实现 | 收据不排序、不显示最优解（`UIF-R11`）；150% 时长行折行并可滚动。 |
| 局外升级 | 价格与余额同时显示；不可购买有原因句 | 不适用 | **同时追踪：价格、余额、当前/下一级效果** | 不适用 | 未实现 | 购买确认默认焦点在「取消」。 |
| 解锁与档口 | 锁定态不能只靠灰；要写价格与差额 | 不适用 | 锁定原因与解锁进度需并列 | 不适用 | 未实现 | 数据未就绪时不显示占位价格（与 `UIF-R22` 同款原则）。 |
| 纪录与里程碑 | 达成/未达成有文字分组与进度 | 不适用 | 里程碑列表可能很长 → 分组折叠 | 不适用 | 未实现 | 只读快照；「还差 N 项」为展示计数。 |
| 局内 HUD | 全部语义非颜色可辨；灰度 ship-blocking | 5 个焦点位；不用鼠标可玩 | **4 个读数 + 4 张卡逐张比较** | 无音频（只发事件） | 部分 | 详见 `design/ux/hud.md` § 10 与 `ACC-VIS-01..15`。 |
| 界面流程 | 全屏均可缩放；焦点环可见；错误不靠颜色 | 模态焦点陷阱；默认焦点可预测 | 屏幕栈 ≤ 2，返回行为确定 | 无音频（只发事件） | 部分 | 屏幕清单固定 12 项（`UIF-R2`）；无 Help 屏（`ACC-COG-05` 缺口）。 |
| 音效反馈 | 不适用（不绘制） | 不适用 | 无（静音可完整游玩） | **全部 cue 有视觉等价** | 已定 | 见 `ACC-AUD-02` 与 SFX Audit。 |
| 首局引导 | 气泡对比度 ≥ 4.5:1、150% 三行上限；锚点高亮用文字标签而非仅颜色 | `Blocked` 期间仍可暂停；跳过用 `ui_cancel` | **一次一条、只讲一件事**；无计时 | 不适用 | 部分 | 不可回看是 Standard 缺口（`ACC-COG-05`）。 |
| 在线挑战（Full Vision） | 排行榜需文字名次与并列说明 | 不适用 | 每日挑战的时间限制会成为新认知负担 | 不适用 | 未开始 | 设计时若引入“每日限时”，必须重写 `ACC-COG-02`。 |
| 同屏合作（Full Vision，延后） | 双人读数分区 | 双手/双输入会推翻 `ACC-MOT-03` | 双人同时决策会超 4 项上限 | 不适用 | 未开始 | 延后系统；实现前必须重跑本矩阵。 |

---

## Accessibility Test Plan

证据统一放 `production/qa/evidence/a11y/`，文件名前缀为要求编号（如 `ACC-VIS-03_hud-100pct.png`）。测试方法分三类：自动化（脚本/截图分析）、内部手动（模拟障碍）、用户测试。

| Feature | Test Method | Test Cases | Pass Criteria | Responsible | Status |
|---------|------------|------------|--------------|-------------|--------|
| `ACC-VIS-03` / `05` 对比度 | 自动化 — 对最终色值跑 WCAG 相对亮度计算，并对真实截图取像素复核 | 上表 11 组组合 × 100% / 60% 不透明度 × 深/浅世界底色 | 全部文字组合 ≥ 4.5:1；非文本 ≥ 3:1；无“看起来够大就放宽” | ux-designer | Not Started |
| `ACC-VIS-08` 灰度 | 手动 — 全部目标截图去色相 + Coblis 三型模拟 | `Active` 720p / 1080p / 150%、`Frozen`、`Ending`、确认面板、不可吃卡、`Faulted`、收据、升级、设置 | Grayscale 十问全部答得出；三型模拟下无信息丢失 | ux-designer | Not Started |
| `ACC-VIS-09` / `10` 文本缩放 | 手动 + 截图 — 三档 × 四种目标分辨率 | 每屏检查溢出、重叠、消失、横向滚动；完成一局并读完收据 | 12 组组合全部“无溢出、无重叠、无消失”；150% 下 HUD 2 列且网格与详情行不重叠 | ux-designer | Not Started |
| `ACC-VIS-12` 减少动态 | 手动 — 开启后走完一局并翻遍菜单 | 数字变化、焦点移动、拒绝、确认面板、`Handoff`、界面过渡、收据滚动 | 全部为瞬时切换；无位移、抖动、淡入；没有任何过渡延长数字出现时间 | qa-tester | Not Started |
| `ACC-VIS-14` HUD 不透明度 | 手动 + 截图 — 60% 与 100% 各一套 | `Active`、`Frozen`、`Ending`、确认面板 | 文字 α 恒 1.0；最低档对比度仍 ≥ 4.5:1；面板 ≥ 0.70 | ux-designer | Not Started |
| `ACC-MOT-01` 键盘 / 手柄完整可玩 | 手动 — 拔掉鼠标，只用键盘；再只用柄 | 开局 → 吃菜 → 跳过 → 结束取餐 → 收据 → 升级 → 设置 → 退出 | 全程可达；焦点永不丢失；无“只能用鼠标”的步骤 | qa-tester | Not Started |
| `ACC-MOT-02` 重绑（缺口跟踪） | 手动 — 尝试把 8 个动作改到非默认键 | 键盘与手柄各一次 | 当前预期**失败**（无入口）；本行用于跟踪缺口是否关闭 | producer | Blocked |
| `ACC-MOT-06` 焦点可见 | 截图 + 视觉 gate | 4 卡 + 按钮 + 模态 + 确认面板；三档字号 | 2 px 环完整、不裁切、与悬停可区分、非文本对比 ≥ 3:1 | ux-designer | Not Started |
| `ACC-COG-02` / `03` 无时间压力 | 手动 — 计时 | 确认面板停 60 s；`ui_accept` 前停 10 s；暂停 5 min 后恢复 | 不自动关闭、不自动确认、不超时、结果与立即操作相同 | qa-tester | Not Started |
| `ACC-COG-04` 提示可读 | 手动 — 逐条触发提示并计时 | 拒绝、结束、故障、保存指示、引导气泡、收据异常 | 可操作提示不早于 5 s 消失（或常驻）；短暂强调消失后原因仍可读 | qa-tester | Not Started |
| `ACC-COG-09` 设置持久化 | 手动 — 改七项设置后重启与崩溃退出各一次 | 文本缩放、减少动态、不透明度、音量、静音、减少重复音效 | 全部立即生效；重启后保持；写失败有「设置可能未保存」且保留内存值 | qa-tester | Not Started |
| `ACC-AUD-02` 静音可玩 | 手动 — 全局静音，完成一局并读完收据 | 吃、拒绝、跳波、结束、故障、结算 | 不靠声音也能做出每个正确决定；无“听了才知道”的时刻 | qa-tester | Not Started |
| `ACC-AUD-03` 音量 | 手动 — 两路音量到 0、静音切换 | 结算结果 cue、UI 点击、数字 tick | 独立生效；静音后不补播；恢复后下一条正常 | qa-tester | Not Started |
| 平台 — 系统级辅助 | 手动 — 开启 OS 显示缩放、粘滞键、鼠标键 | 1280×720 / 1920×1080、全屏与窗口 | 不卡死、不忽略 OS 缩放、不锁定光标 | qa-tester | Not Started |
| 用户测试 — 色觉缺陷 | 用户测试（AbleGamers Player Panel 或等效） | 完整一局 + 收据 + 升级 | 参与者无需请求颜色澄清即可完成并说明盈亏 | producer | Not Started |
| 用户测试 — 低视力 / 阅读障碍 | 用户测试 | 150% 字号完整一局 | 无需放大镜/缩放工具；能读出净利、差额、可吃性 | producer | Not Started |

---

## Known Intentional Limitations

> 每一条都是风险。前三条是 **Standard 档的正式缺口**，不是“以后再说”——它们必须在写 epic 前由 producer 裁定：补进 GDD，或正式把承诺降为 Basic+。

| Feature | Tier Required | Why Not Included | Risk / Impact | Mitigation |
|---------|--------------|-----------------|--------------|------------|
| 输入重绑（键盘 / 手柄） | Standard | `INP-R10` 规定 MVP 无重绑，`INP-OQ1` 推到上线前；重绑入口会触及“固定 8 动作”的 InputMap 契约与设置清单 | 无法使用单手键位、左手键位、自适应控制器或习惯特定布局的玩家会被门槛挡在门外；这是 Standard 最常被引用的项 | 8 个动作、方向键与 WASD 双绑定、无组合键、无长按、无计时；Steam Input 可系统层改键；`ACC-MOT-03`/`07`/`08` 已把运动需求压到极低 |
| 色盲调色板开关 | Standard | `UIF-R24` 的 MVP 设置清单只有六项，没有色盲模式；增加设置项需要改 GDD | 严格按档位定义，本项“缺失”；对已习惯“开一个模式就好”的玩家，发现性差 | **默认状态即色盲安全**：全部语义有非颜色通道（`ACC-VIS-06`）、灰度 ship-blocking（`ACC-VIS-08`）、结构编码优先于调色板（`ART-§4`）。即“模式”没有开关，但效果对所有人默认生效 |
| 教学回看 / Help 屏 | Standard | `ONB-R2` 明确“没有『再看一次』入口”，`UIF-R2` 的屏幕清单固定且无 Help；加屏需要同时改两份 GDD | 首次弹出就跳过、或隔天再玩的玩家无法复读“价格与饱腹要一起看”；对阅读障碍/注意力障碍玩家影响更大 | 教学只讲 HUD 常驻字段，任何时候都能在屏幕上重新读到同样的数；未结算就放弃时下一局仍是首局（`ONB-R22`），教学会重来；文本表可复用，补 Help 屏的成本低 |
| HUD 元素重排 | Comprehensive | 固定九宫格；重排会破坏“回合/容量/净利三块永不遮挡”的硬判据（`HUD UX § 9`） | 使用眼动/头控设备、周边视野受限的玩家无法把读数移到舒适区 | 150% 缩放 + 60% 不透明度 + 焦点位仅 5 个；三块核心读数固定在顶栏，位置永远一致（可预测性替代可移动性） |
| 菜单与 HUD 的屏幕阅读器支持 | Comprehensive | 局内 HUD 是自绘层（`_draw`），不在 `Control` 树内，Godot `AccessKit` 读不到；菜单元素也未定义 accessible name/role | 盲人玩家无法通过 OS 读屏独立完成一局；这是本作最大的无障碍边界 | 键盘/手柄全链路可玩 + 固定初始焦点 + 状态变化都有文字；不声称“可被读屏使用”。若未来投入，优先给菜单补 name/role，再评估 HUD 的朗读摘要 |
| 单声道音频 | Comprehensive | 无定位音、无立体声分离信息；折叠无收益 | 单侧失聪玩家不受影响（无信息在声道差里） | 不需要缓解；`ACC-AUD-06` 保证静音可完整游玩 |
| 高对比模式 | Comprehensive | 基础对比已达 14.6:1（墨黑/米白），且面板不透明度有 0.70 下限 | 极低视力玩家可能仍希望纯黑白 + 全不透明 | 60–100% 不透明度 + 150% 字号已覆盖大部分需求；不为“模式”而模式 |
| 亮度 / 伽马调节 | Basic | 设置清单（`UIF-R24`）无此项；PC 平台通常由 OS/显示器承担 | 在严重偏亮/偏暗的电视或投影上，纸色底可能发灰，红色数字对比下降 | 语义不依赖红色（`ACC-VIS-06`）；墨黑/米白 14.6:1 有巨大余量；不使用中间调承载信息 |
| UI 缩放到 75% 以下 | Basic | 只提供 100/125/150 三档文本缩放，不提供整体 UI 缩放 | 希望更紧凑、或在小屏上想看到更多卡的玩家无法调整 | 100% 已是紧凑档（常驻面积 26.5%）；卡片列数由 `hud_card_columns` 自动决定，不靠玩家手调 |
| 触屏 / 虚拟摇杆 | Basic | `INP-OQ5` 明确首版不做；本作无触屏目标 | Steam Deck 触屏用户必须使用手柄或外接键鼠 | 手柄路径完整可玩；不做半套触控 |
| 16:10 / Steam Deck 布局档 | — | `HUD-OQ9` 未决，未验证 | 1280×800 下安全边距按高度算、纵向多 80 px，可能出现未预期的挤压 | 按公式 `s = 1.0` 渲染并在实机复核；未承诺前不写入验收 |

---

## Audit History

| Date | Auditor | Type | Scope | Findings Summary | Status |
|------|---------|------|-------|-----------------|--------|
| 2026-11-04 | Internal — ux-designer | Desk audit（仅文档，无构建） | 档位承诺 vs 现有 GDD 的设置与屏幕清单 | 发现 3 项 Standard 缺口（重绑、色盲模式开关、教学回看）；发现 1 处颜色 token 冲突（冷灰作次级文字实测 2.8:1）；确认无闪光/无计时/无 hold 输入，Basic 的光敏与音量条款成立 | Open — 待 producer 裁定 |
| — | — | 待首次内部实机审计 | 全量 Test Plan | — | Not Started |

---

## External Resources

| Resource | URL | Relevance |
|----------|-----|-----------|
| WCAG 2.1 | https://www.w3.org/TR/WCAG21/ | 对比度公式（1.4.3 / 1.4.11）、文本缩放（1.4.4）、键盘（2.1.1）、焦点可见（2.4.7）——本文件实测表的计算依据 |
| Game Accessibility Guidelines | https://gameaccessibilityguidelines.com | 按成本分档的游戏专项清单；本文件的“Basic 项先做、Comprehensive 项后置”排序参照它 |
| AbleGamers Player Panel | https://ablegamers.org/player-panel/ | 用户测试渠道（`ACC-VIS-08`、`ACC-MOT-01` 的最终验收） |
| Colour Blindness Simulator (Coblis) | https://www.color-blindness.com/coblis-color-blindness-simulator/ | 三型色觉模拟；用于灰度之外的第二道复核 |
| Xbox Accessibility Guidelines (XAG) | https://docs.microsoft.com/gaming/accessibility/guidelines | 本作不承诺 XAG，但其“颜色以外通道”“可读字号”条目作为设计参考 |
| Accessible Games Database | https://accessible.games | 同类回合制/卡牌游戏的已发布无障碍决策参考 |
| Godot Accessibility (AccessKit) 文档 | https://docs.godotengine.org/en/stable/tutorials/ui/accessibility.html | 菜单读屏的技术边界；需对照 `engine-reference/godot/` 确认 4.7 的行为 |

---

## Open Questions

| Question | Owner | Deadline | Resolution |
|----------|-------|----------|-----------|
| 1. 本文件承诺 **Standard**，但重绑、色盲模式开关、教学回看三项在 GDD 里没有载体。是补进 GDD（改 `input.md` / `ui-flow.md` / `first-run-onboarding.md`），还是正式把承诺降为 **Basic+**（并改写档位段）？ | producer + ux-designer | 写 epic 前（Pre-Production gate） | **未决**——本文件按“保留 Standard 承诺 + 明示缺口”撰写，不替 producer 决定 |
| 2. 是否在 `UIF-R24` 的设置清单里加 `colorblind_palette`（off / deutan-protan / tritan）？若不加，如何向玩家说明“本作默认就色盲安全”？ | ux-designer + game-designer | 写 epic 前 | 未决——当前缓解是结构编码 + 灰度验收 |
| 3. 输入重绑是否提前到 MVP（`INP-OQ1`）？重绑 UI 放 `SettingsModal` 的哪个分区？ | game-designer + producer | 上线前（若提前则写 epic 前） | 未决 |
| 4. 教学回看：新增 Help 屏（改 `UIF-R2` 屏幕清单），还是在 `SettingsModal` 放一个“重看首局讲解”的只读页？ | ux-designer + game-designer | 写 epic 前 | 未决 |
| 5. 亮度/伽马归谁：进 `UIF-R24` 设置清单（需改 GDD），还是明确声明“由 OS/显示器负责”？ | ui-programmer + ux-designer | 垂直切片前 | 未决 |
| 6. Godot 4.7 的 AccessKit 是否覆盖本项目用到的 `Control` 类型（按钮、滑块、开关、列表）？自绘 HUD 是否有官方朗读回退？ | ui-programmer | 技术设计阶段 | 未决——需查 `engine-reference/godot/`，不得凭印象声称支持 |
| 7. 冷灰 `#8C918E` 在米白底实测 **2.8:1**，但 `ART-§4` 把它定为“次级文字”。是加深该 token（如 `#4A4E4A`，7.5:1），还是加深纸底？需艺术总监定 | art-director | 垂直切片前 | 未决——本文件已禁止冷灰承载文字 |
| 8. HUD 不透明度下限 60% 是否够低（明亮世界下是否仍刺眼）？是否要给 0%（全关 HUD）并接受“本局不可进行”的风险？ | ux-designer + accessibility-specialist | 垂直切片前 | 未决（与 `HUD-OQ6` 同一问题） |
| 9. 右键一次误触 = 丢整波 + 1 回合且不确认（`HUD-OQ5`、`WAVE-OQ8`）。从无障碍角度，是否需要长按 ≥ 400 ms 或双击？ | game-designer + ux-designer | 首个可玩原型后 | 未决 |
| 10. 导航重复（0.35 s / 0.12 s）是否暴露给玩家调节？ tremor 玩家可能希望更长延迟 | game-designer + ux-designer | 垂直切片 | 未决（与 `INP-OQ3` 同一问题） |
| 11. 150% 字号 + 60% 不透明度 + 1280×720 三条件叠加时，HUD 网格与详情行只剩 16 px 余量（`HUD UX § 10.2`）。是否需要为这一组合单开一档？ | ui-programmer | 垂直切片前 | 未决 |
| 12. 若未来加入配音或背景音乐，本文件的字幕（N/A）与两路音量（`ACC-AUD-03`）必须重写；谁负责在加语音前先改本文件？ | producer | 加语音/音乐的设计评审时 | 未决——当前无语音无音乐，条款成立 |
