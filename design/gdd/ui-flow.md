# 界面流程 (UI Flow)

> **Status**: Designed
> **Author**: user + game-designer（自动设计模式；`systems-designer` 与 `qa-lead` 评审）
> **Last Updated**: 2026-11-03
> **Last Verified**: 2026-11-03
> **Implements Pillar**: 支柱 4「数字看得见地涨」；约束来自支柱 3「时间在走，手不用快」

## Summary

界面流程是玩家在餐局之外与餐局之间操作的全部界面：主菜单、结算收据、升级页、纪录页、设置、暂停菜单、错误屏，以及把它们串起来的屏幕栈、焦点与返回行为。它把顶层状态翻译成屏幕，把 `RunResult` 变成一张可复盘的收据，把钱包、纪录与存档状态呈现给玩家；它自己不做任何数值计算。没有它，玩家无法开始一局、无法读懂一局，也无法把赚到的钱花出去。

> **Quick reference** — Layer: `Presentation` · Priority: `MVP` · Key deps: `场景与流程管理`、`输入`、`餐局流程`、`结算与入场费`、`局外升级`、`纪录与里程碑`、`存档`、`音效反馈`、`数据配置`

## Overview

界面流程是玩家在「吃」与「算账」之间看到的一切：启动等待、错误屏、主菜单、结算收据、升级页、纪录页、设置、暂停菜单，以及把它们串起来的屏幕栈、焦点与返回行为。它把「场景与流程管理」的六个顶层状态翻译成具体的界面，把「餐局流程」交出的 `RunResult` 变成一张可以复盘的收据，把「局外升级」「纪录与里程碑」「存档」的结果呈现给玩家；它自己不做任何数值计算。

玩家与它的交互是持续的、低操作的：用方向键或手柄在控件间移动，用 `ui_accept` 确认，用 `ui_cancel` 返回。它不要求反应速度——没有倒计时确认、没有限时选择、没有必须靠鼠标完成的动作（支柱 3）。它存在的理由是支柱 4：数字必须被看见、被读懂、被比较，否则「回本」只是一句规则，而不是玩家能指着收据说出口的事实。

本系统只绘制、只路由、只转发。屏幕上的每个数字都来自拥有它的系统；界面流程拥有的是屏幕层级、焦点归属、返回栈、结算屏的打开时机，以及一组可以被 QA 独立验证的导航规则。它不拥有局内 HUD（`局内 HUD`）、顶层状态机（`场景与流程管理`）、输入映射（`输入`）或任何一局的数值（`结算与入场费` / `局外升级` / `纪录与里程碑`）。

## Detailed Design

### Core Rules

1. **职责边界** — 本系统拥有：顶层状态到界面的映射、屏幕栈与返回行为、键盘/手柄焦点归属、模态的呈现与焦点陷阱、结算收据的打开时机与内容编排、设置屏与机器级设置模型、以及界面语义事件（`ui_button_activated`、收据数字滚动、收据关闭）。它**不拥有**：顶层状态与合法转移（`场景与流程管理`）、局内 HUD（`局内 HUD`）、动作映射与输入上下文状态机（`输入`）、任何一局的数值与 `RunResult` 字段（`结算与入场费`）、钱包与升级价格（`局外升级`）、纪录比较与里程碑判定（`纪录与里程碑`）、存档事务与载入结果（`存档`）、音频播放（`音效反馈`）、菜品/档口/里程碑记录本身（`数据配置`）。

2. **屏幕清单固定** — MVP 的界面集合固定为下表。新增屏幕必须改本 GDD，不得由实现临时增加。`局内 HUD` 的一屏不在本清单内，它是同级系统，不是本系统的屏幕。

   | 屏幕 / 层 | 顶层状态 | 形态 | 拥有者 | 说明 |
   |---|---|---|---|---|
   | `BootWaiting` | `Boot` | 全屏 | 界面流程 | 只显示等待文案，无进度条、无输入 |
   | `ErrorScreen` | `Error` | 全屏 | 界面流程 | 只提供退出；debug 额外提供重载数据 |
   | `MainMenu` | `Menu` | 全屏根屏 | 界面流程 | 开始营业、升级、纪录、设置、退出；钱包与入场费常驻 |
   | `UpgradeScreen` | `Menu` | 子屏 | 界面流程 | 升级目录、钱包、购买确认与失败态 |
   | `RecordsScreen` | `Menu` | 子屏 | 界面流程 | 纪录页、里程碑列表、完成状态 |
   | `ReceiptScreen` | `Menu` | 全屏根屏 | 界面流程 | 结算收据；打开时取代主菜单成为根屏 |
   | `PauseOverlay` | `Paused` | 覆盖层 | 界面流程 | 覆盖在 HUD 之上的暂停菜单，不卸载 HUD |
   | `SettingsModal` | 任意有界面的状态 | 模态 | 界面流程 | 音频、无障碍、HUD 显示、删除生涯 |
   | `ConfirmModal` | 任意有界面的状态 | 模态 | 界面流程 | 退出、放弃、购买、删除生涯的确认 |
   | `NoticeLayer` | 任意有界面的状态 | 非模态提示 | 界面流程 | 保存指示、恢复说明、进入失败、完成横幅 |
   | 局内一屏 | `Run` | 全屏 | `局内 HUD` | 不归本系统绘制、不进入本系统的屏幕栈 |

3. **一个顶层状态恰好一个界面根** — 除 `Quit` 外，每个顶层状态都有明确归属的界面根；界面流程每次渲染前以 `场景与流程管理` 的当前顶层状态为准，内容不得与状态矛盾（`场景与流程管理` UI Requirements 1、5）。映射固定为：`Boot → BootWaiting`、`Error → ErrorScreen`、`Menu → MainMenu 或 ReceiptScreen`、`Run → 局内 HUD`、`Paused → PauseOverlay + 局内 HUD`。顶层状态切换生效时，界面流程丢弃旧状态的屏幕栈与模态栈，按新状态重建，不保留任何跨状态界面。

4. **屏幕栈最多两层** — 菜单内的全屏屏幕组成一个栈：`MainMenu` 是根（深度 1），`UpgradeScreen` 与 `RecordsScreen` 是子屏（深度 2）。只允许从根推入一个子屏，不允许子屏再推子屏；`MAX_UI_STACK_DEPTH = 2`。`ReceiptScreen` 是根屏，不叠在主菜单之上：`Run → Menu` 时它以**替换根**的方式出现，`返回菜单` 时替换回 `MainMenu`。`PauseOverlay` 与 `BootWaiting` / `ErrorScreen` 各自是所在状态的唯一根。设置与确认不是栈屏幕，它们是模态（规则 5）。`ui_stack_depth` 见 Formulas。

5. **模态与屏幕栈分离，最多两层** — 设置与确认框是模态，不进入屏幕栈。打开模态时调用 `输入.push_context(Modal)`，关闭时调用 `pop_context()`；模态期间的导航、`ui_accept`、`ui_cancel` 只作用于模态。`MAX_MODAL_DEPTH = 2`，只允许两种组合：①任意根屏或子屏打开一个设置/确认模态（深度 1）；②在 `SettingsModal` 内打开删除生涯的 `ConfirmModal`（深度 2）。第二层仍以 `push_context(Modal)` 进入；`输入` 必须允许栈顶已为 `Modal` 时再次压栈（见 Open Question 14）。不允许第三层，不允许从确认框再开设置。深度已满时新的模态请求被拒绝并记 WARNING，**不排队**。模态必须有一个确定的关闭路径，任何离开都先清空模态栈，不得把 `Modal` 上下文泄漏到下一个界面。

6. **焦点唯一、可预测、可恢复** — 每个屏幕与模态都定义一张**按屏幕位置排序**的焦点顺序表（上→下、左→右；不按节点树顺序，`输入` UI Requirements 2）。屏幕打开时，焦点落在顺序表中第一个**启用且可见**的控件（`initial_focus`，见 Formulas）；没有可聚焦控件是屏幕缺陷，不得以“无焦点”上线。禁用控件不可获焦。模态打开时焦点移入模态并形成焦点陷阱；关闭后焦点回到打开它的控件（`focus_restore`，见 Formulas）。鼠标悬停与键盘/手柄焦点分开记录、分开绘制（`输入` 规则 8、UI Requirements 4）；鼠标移动不抢键盘焦点，键盘导航不清除悬停。**不可逆或破坏性确认的默认焦点是「取消」**（退出、放弃本局、购买、删除生涯）；非破坏性模态的默认焦点是主操作。

7. **返回是确定的决策，不是默认退出** — `ui_cancel` 的行为由 `back_target`（Formulas）唯一决定，优先级固定为：①有可取消模态 → 关闭模态（模态捕获 `ui_cancel`）；②有强制模态 → 无动作；③收据数字滚动中 → 跳过滚动；④`Menu` 子屏 → 弹回主菜单；⑤`Menu` 根屏且为收据 → 返回主菜单；⑥`Menu` 根屏且为主菜单 → 打开退出确认；⑦`Paused` → 打开退出确认；⑧`Error` / `Boot` / `Run` / `Quit` → 无动作。`ui_cancel` 在任何状态下都**不得直接退出游戏**，退出必须经过确认（`Run` / `Paused`）或明确的「退出」控件（`Menu` / `Error`）。强制模态（如存档 `Blocked` 说明）忽略 `ui_cancel`，只接受其列出的出口。

8. **开始一局** — 主菜单的「开始营业」组装 `RunConfig`：`station_id` 取 MVP 唯一档口 `station_canteen`；`upgrade_snapshot = 局外升级.resolve_upgrade_snapshot()` 的**副本**（开局冻结，`局外升级` 规则 13）；`seed = null`；`is_first_run = 纪录与里程碑.snapshot().career.runs_settled == 0`；`pool_override = null`。组装完成后调用 `场景与流程管理.request_transition(Menu → Run, RunConfig)`。返回 `false` 时留在主菜单并显示可见原因，不进入加载态、不重试、不伪造成功。主菜单必须在开始按钮旁显示本档口的入场费（`结算与入场费` UI Requirements）与当前钱包（可为负）；两者只读，来自 `数据配置.get_station()` 与 `局外升级.get_wallet()`。

9. **收据在 `RunResult` 交付的同一帧打开** — `场景与流程管理` 在 `Run → Menu` 生效的同一帧把 `RunResult` 恰好一次交给界面流程。界面流程在该帧内完成入账编排（规则 10）并把 `ReceiptScreen` 设为根屏；`receipt_open_latency_frames = 0`（Formulas）。`RunResult` 按引用保留，只用于本张收据；不重算、不缓存到收据关闭之后、不为同一 `run_id` 打开第二张收据。切换不插入加载屏或「结算中」等待态；局内最终画面保留到收据覆盖它的那一帧（`餐局流程` UI Requirements 5、6）。

10. **结算入账编排：先校验纪录，再动钱包，最后一次提交** — 界面流程是 MVP 的**结算事务编排者**（回答 `纪录与里程碑` Open Question 1）。收据绘制前必须按固定顺序完成一次编排：①`纪录与里程碑.apply_run(result)`；若返回 `INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY`，整笔放弃——不调用 `局外升级`、不提交任何命名空间，收据仍显示 `RunResult` 的原始值但标注「结算数据异常，未入账、未记录」；②`局外升级.apply_run_result(result)` 恰好一次，处理 `APPLIED` / `APPLIED_UNSAVED` / `WALLET_OVERFLOW`；③把 `progression`、`records`（以及未来存在的 `unlocks`）合并为**一次** `存档.commit`。写盘失败时按 `存档` 规则 18 与 `局外升级` 规则 6 保留内存值、标记 dirty、在下个写入时点重试，收据带「未保存」标记。同一 `run_id` 的重复 `RunResult` 不重复入账。**接口约束**：为满足 `存档` 规则 80 的单事务要求，`apply_run_result` 需要可分离的「prepare（内存应用，不提交）」与「commit」两段；若架构阶段维持其内部提交的现状，MVP 退化为两次串行提交，且必须在纪录致命错误时回滚已入账钱包——见 Open Questions 1。

11. **收据只陈述 `RunResult`，不重算、不展示最优解** — `ReceiptScreen` 的每一行、每个合计都直接读 `RunResult`：行顺序严格按 `rows`（吃下顺序），行字段为菜名、价格、基础饱腹、腻度追加（0 显示 `—`）、实际饱腹、静态单位价值、有效单位价值；合计区为总值、入场费（负项）、净利（带符号）、回本判定（「回本」/「未回本」）、回本率（`122.3%`）、已用/总容量、浪费容量、已用/总回合、腻度额外占用合计。不排序、不合并、不补算、不显示 `opt_value` 或“本可以多赚 X”（`结算与入场费` 规则 9–12）。空局显示「这一局什么都没吃」的空状态；`entry_fee = 0` 显示「免单」，不出现 `∞` / `NaN`。`record_broken` 到达时并排显示旧最佳，「新纪录」印章**只在新最佳 `net_profit > 0`** 时出现；没有破纪录时不显示任何安慰性文案（`纪录与里程碑` UI Requirements 5）。`record_broken` 在 `apply_run` 期间同步到达时，界面流程把 `previous_best` 暂存给本张收据，不另开提示。新达成里程碑显示为收据底部的一条非模态「新达成」条；完成横幅按规则 23 处理。按钮由界面流程拥有，固定为「再来一局」「返回菜单」，位置不随净利正负变化；初始焦点为「再来一局」。

12. **再来一局是全新一局** — 收据上的「再来一局」重新组装一份**全新** `RunConfig`：同一 `station_id`、重新调用 `resolve_upgrade_snapshot()` 得到的新快照、`seed = null`、`is_first_run = false`；不复用上一局的种子、快照、`run_id` 或局内状态（`餐局流程` UI Requirements 4）。随后请求 `Menu → Run`；请求被拒绝时留在收据并显示原因。收据关闭时通知音效取消待播的结算结果 cue。

13. **返回菜单不重复入账** — 收据上的「返回菜单」把根屏替换回 `MainMenu`；钱包、纪录、里程碑已在规则 10 中处理完毕，不再调用 `apply_run_result` 或 `apply_run`。主菜单刷新钱包、纪录入口与升级入口的可见状态，焦点回到「开始营业」。返回菜单同样通知音效取消待播结果 cue。

14. **收据数字滚动是表现，不是第二次求和** — 收据的总值、净利（以及显示的百分比）可以从 0 滚动到 `RunResult` 的最终值，但**终点必须逐字段等于 `RunResult`**，滚动过程不产生任何新的数值事实。滚动时长 `RECEIPT_ROLL_SECONDS` 是数据文件值（0.3–1.5 秒，默认 0.8；`reduce_motion` 开启时为 0）。滚动开始时发 `number_roll_started(field, from, to, duration_ms)`，结束时发 `number_roll_finished(field, skipped: bool)`；玩家在滚动中第一次按 `ui_accept` 或 `ui_cancel` 只跳过滚动（`skipped = true`），**不激活**当前焦点控件；需要再按一次才会激活。滚动结束或被跳过后，焦点仍在初始控件上。

15. **暂停菜单是覆盖层，不是局内状态** — `Run` 中 `ui_pause` 由 `场景与流程管理` 切到 `Paused`；界面流程在 HUD 之上绘制 `PauseOverlay`，**不卸载、不重建**局内界面（`场景与流程管理` UI Requirements 3），恢复后画面与暂停前一致。菜单项固定为「继续」「设置」「放弃本局」「退出游戏」；初始焦点为「继续」。`ui_accept` 作用于「继续」时请求 `Paused → Run`（`场景与流程管理` 规则 7）；`设置` 打开 `SettingsModal`；`放弃本局` 打开确认；`退出游戏` 打开确认。**进入 `Ending` / `Settling` / `Handoff` 后，暂停菜单只保留「继续」「退出游戏」，不提供「放弃本局」**（`餐局流程` UI Requirements 9）。界面流程只读 `餐局流程` 的只读快照 `run_state` 判断当前阶段，不自行推算、不缓存第二份阶段状态。暂停期间不推进回合、不结算、不改变任何局内数字。

16. **放弃与退出都不是结算** — 「放弃本局」确认后请求 `Paused → Menu`，携带 `abandoned = true`；不产生 `RunResult`、不打开收据、不写纪录、不入账（`场景与流程管理` 规则 8、`餐局流程` 规则 16）。「退出游戏」在 `Run` / `Paused` 需确认，在 `Boot` / `Menu` / `Error` 直接执行；窗口关闭请求走同一规则（`场景与流程管理` 规则 10）。退出前调用 `存档.flush()`；flush 失败时显示「未保存，仍要退出？」并再确认一次，不静默丢弃。所有退出最终请求 `Quit`。

17. **启动等待没有进度条，也没有最短展示时长** — `Boot` 期间只显示 `BootWaiting` 的等待文案与静态标识；不做进度条、不做转场动画、不设最短展示时长。输入上下文为 `Blocked`，不接受任何界面动作。数据校验结果一到，立即按 `Boot → Menu` 或 `Boot → Error` 替换根屏（`场景与流程管理` 规则 4、12）。

18. **错误屏是受限态，不是主菜单的变体** — `ErrorScreen` 显示可读的错误说明；唯一的退出控件是「退出」，debug 构建额外显示「重新加载数据」并展示失败文件与记录 id。**不存在任何进入 `Menu` 或 `Run` 的控件**；release 构建中「重新加载数据」不是存在后被拒绝，而是根本不存在（`场景与流程管理` 规则 9、UI Requirements 2）。错误信息不能只靠颜色传达；`ui_cancel` 不产生隐藏动作。

19. **进入失败、故障与放弃只给提示，不给收据** — `Menu` 根屏因 `init_failed = true` 或 `fault` 明细进入时，显示非模态提示「本局未能开始」+ 原因（debug 显示故障码与字段），不打开收据、不入账、不写纪录；焦点回到「开始营业」，提示在确认、取消或下一次屏幕变化时消失，同一事件不重复弹出。`abandoned = true` 进入时显示「本局已放弃，收益不计」的非模态提示，同样不打开收据。提示层不阻塞输入，不改变主菜单的可用控件。

20. **存档载入结果：成功的安静，异常的必须说清楚** — 进入 `Menu` 后读取一次 `LoadReport`。`Fresh` / `Loaded` / `Migrated` 不弹提示（`Migrated` 的迁移写回由保存指示体现）。`Recovered` 显示一次非模态说明，写明来源是备份与可能丢失的范围，玩家可关闭，**不重复弹出**。`Blocked` 显示**强制模态**：只提供「退出」与「新建生涯（旧档保留）」两个出口，默认焦点为「退出」，`ui_cancel` 无效，**不得提供覆盖保存**（`存档` UI Requirements 3）。强制模态同样经 `push_context(Modal)` 获得焦点陷阱，并显式消费 `ui_cancel` 为空操作；窗口关闭请求仍按 `场景与流程管理` 规则 10 直接退出（`Menu` 中无需确认），不算第三个按钮。`Blocked` 未解决前，主菜单的其他控件不可操作；选择「新建生涯」后由存档建立新生涯，界面流程刷新钱包、升级、纪录与主菜单；选择「退出」则请求 `Quit`。`LoadReport.writable = false` 时，所有写盘依赖的动作（购买、解锁、任何写入入口）禁用并显示原因（`局外升级` 规则 19）。

21. **保存指示安静、不阻塞、不发声** — 每次成功写入生涯档后显示「已保存」，约 `SAVE_INDICATOR_SECONDS = 1.5` 秒后淡出；不阻塞输入、不移动布局、不发出声音。首次建立生涯与迁移写回同样显示（`存档` UI Requirements 1）。失败提示按场景区分：购买失败显示「未保存，购买未生效」并恢复按钮；纪录写入失败显示「纪录可能未保存」，不打断收据阅读；钱包溢出显示「入账失败」；`APPLIED_UNSAVED` 在钱包旁显示「本次会话不保存」标记（`存档` UI Requirements 2、`局外升级` UI Requirements）。**设置写入不显示生涯保存指示**——设置不属于生涯档（`存档` 规则 20）。

22. **升级页：目录只读，购买先确认后落盘** — `UpgradeScreen` 读取 `局外升级.get_catalog()`、`get_wallet()`、`can_purchase(id)`。每条显示名称、等级 `n / max`、当前效果、下一级效果、下一级价格、可购买性与不可购买原因（钱包不足 / 已满级 / 本档口已看满 / 存档不可写 / 前置未解锁）。钱包常驻同屏显示，负数带负号（`局外升级` UI Requirements）。数据未 `Ready` 时显示错误，不显示占位价格、不用默认值兜底（`局外升级` 规则 20）。点击可购买项打开 `ConfirmModal`，显示价格与购买后余额，默认焦点为「取消」；确认后才调用 `purchase(id)`，`purchase` 仍完整校验。购买进行中再次确认被忽略，不发起第二笔；`BUSY` 显示「正在处理上一笔」并保持按钮禁用直到第一笔返回。`COMMIT_FAILED` 显示「未保存，购买未生效」并恢复按钮；`MAX_LEVEL` 显示「已满级」；`NO_EFFECT` 显示「本档口已看满（最多 Z 道）」；`NOT_WRITABLE` 禁用按钮并显示原因。不自动购买；首局引导可以高亮「推荐」但不能替玩家确认（`局外升级` UI Requirements）。`progression_changed` 到达时刷新列表与钱包。

23. **纪录页只读快照；完成横幅一次性** — `RecordsScreen` 读取 `纪录与里程碑.snapshot()`、`milestone_progress(id)`，以及 `数据配置` 的里程碑目录（名称、条件文案、阈值、`required_for_completion`）。每档口一行：档口名、最佳净利（带符号）、回本率（免单显示「免单」）、总值、达成日期；未打过显示明确空状态；里程碑按「未达成 / 已达成」分组，显示进度 `当前值 / 阈值 单位`；未通关显示「距离通关还差 N 项」并列出必需项。**不重算比较、不重算回本率、不自行判定完成**：`completed` 只取 `snapshot().completed_at != null`，锁存后即使目录变化也不撤销；「还差 N 项」只是对目录与已达成集合的展示计数。完成横幅在 `career_completed()` 到达时显示一次；若收据正开着，等数字滚动结束后再显示；若玩家已离开收据，则等下一次 `MainMenu` 根屏显示——**不在一张新的收据上显示**，以免与「新达成」条冲突；可关闭、不阻塞、不重复（`纪录与里程碑` UI Requirements 3–5）。

24. **设置是机器级模型，不属于生涯档** — 界面流程拥有设置屏与机器级设置模型（`存档` 规则 20、`音效反馈` UI Requirements）。MVP 设置项与范围：`master_volume_db`（−80…0，`−80` 视为静音）、`sfx_volume_db`（−40…0）、`muted`（开/关）、`reduce_repetitive_cues`（开/关，默认关）、`text_scale_percent`（100 / 125 / 150）、`reduce_motion`（开/关）、`hud_opacity_percent`（60–100）。修改**立即生效**：音频值由 `音效反馈` 消费，文本缩放与 HUD 不透明度由 `局内 HUD` 与所有界面消费；`reduce_motion` 同时把界面过渡与收据滚动时长置 0。设置在设置模态关闭或退出游戏时写入机器级设置文件；写入失败显示「设置可能未保存」并保留内存值。设置不随生涯删除、不进入生涯备份。**删除生涯**放在设置屏的次级分区，需要两次确认（第二次默认「取消」），确认文案写明旧档会被隔离保留；删除后主菜单刷新为新生涯状态，设置不变（`存档` UI Requirements 5、规则 23）。不提供虚拟摇杆或触屏控件（`输入` Open Question 5）。

25. **界面语义事件：只在提交时发声** — 界面流程发 `ui_button_activated(control_id)`，**每次已提交的控件激活恰好一次**；焦点移动、悬停、点击空白、禁用控件、按下但未提交都不发（`音效反馈` 前置契约）。收据滚动发 `number_roll_started` / `number_roll_finished`，跳过时 `skipped = true`。收据关闭、返回菜单、开始新一局时发 `receipt_closed(reason: restart | menu | quit)`，让音效取消待播的结算结果 cue（`音效反馈` Edge Cases）。界面流程不播放音频、不持有 cue、不订阅原始输入。

26. **过渡只播，不延迟状态；没有加载屏** — 顶层切换的淡入淡出由 `场景与流程管理` 定义上限（总时长 ≤ 0.2 秒），界面流程只负责播放，**不得推迟状态生效**；关闭视觉过渡时 `FADE_OUT = FADE_IN = 0`，切换仍立即生效（`场景与流程管理` 规则 12、Formulas）。`Menu ↔ Run` 之间不插入加载屏、过场或「结算中」等待态（`餐局流程` UI Requirements 5）。没有任何装饰性动画可以推迟数字出现。

27. **输入上下文只经请求改变，且必须成对** — 界面流程只通过 `输入.push_context(Modal)` / `pop_context()` 管理模态上下文；**不得调用 `request_context`、不得直接赋值上下文**。顶层状态切换会清空临时栈（`输入` 规则 12），因此界面流程必须在切换生效时丢弃自己的模态栈与屏幕栈，并在新状态内重建。每次 `push_context(Modal)` 都必须有配对的 `pop_context()`；任何离开路径（确认、取消、顶层切换、退出）都不得留下未弹出的 `Modal`。`ui_pause` 由 `场景与流程管理` 消费，界面流程不监听；`ui_accept` / `ui_cancel` / 导航由界面流程在自己的上下文内消费。

28. **界面状态是输入的纯函数** — 给定同一（顶层状态、payload、屏幕栈、模态栈、设置），界面流程必须得到同一屏幕、同一焦点、同一可操作集合；除保存指示的 1.5 秒显示与收据滚动外，不存在隐藏计时器。保存指示与滚动只影响呈现，不影响任何数据事实，也不得成为确认或退出的前置条件（`输入` Acceptance Criteria 16：不存在接受倒计时）。

### States and Transitions

界面流程没有自己的顶层状态机——它的状态是四个已有事实的组合：`场景与流程管理` 的顶层状态、屏幕栈、模态栈、以及当前是否有一张打开的收据。下表把这个组合命名为 UI 状态，便于测试与实现，但它不是第五个状态机。

| UI 状态 | 顶层状态 | 屏幕栈 | 模态深度 | 输入上下文 | 允许动作 |
|---|---|---|---|---|---|
| `BootWaiting` | `Boot` | `[BootWaiting]` | 0 | Blocked | 无 |
| `ErrorRoot` | `Error` | `[ErrorScreen]` | 0 | Menu | 退出；debug 重载数据 |
| `MenuRoot` | `Menu` | `[MainMenu]` | 0 | Menu | 导航、`ui_accept`、`ui_cancel`（→退出确认） |
| `MenuSub` | `Menu` | `[MainMenu, Upgrade \| Records]` | 0 | Menu | 导航、`ui_accept`、`ui_cancel`（→弹回） |
| `Receipt` | `Menu` | `[ReceiptScreen]` | 0 | Menu | 导航、`ui_accept`（先跳过滚动）、`ui_cancel`（先跳过滚动，再返回菜单） |
| `PauseOverlay` | `Paused` | `[PauseOverlay]` | 0 | Paused | 导航、`ui_accept`（继续）、`ui_cancel`（→退出确认） |
| `Modal1` | 任意有界面的状态 | 基底屏幕栈 | 1 | Modal | 导航、`ui_accept`、`ui_cancel`（强制模态除外） |
| `Modal2` | 任意有界面的状态 | 基底 + 设置 | 2 | Modal | 只允许删除生涯确认 |
| `RunNoUI` | `Run` | `[]` | 0 | Run | 无（局内 HUD 拥有画面） |
| `Quit` | `Quit` | `[]` | 0 | — | 无 |

**转移表**

| 从 | 触发 | 到 | 副作用 |
|---|---|---|---|
| `BootWaiting` | `Boot → Menu` | `MenuRoot` | 读一次 `LoadReport`；按规则 20 显示恢复/阻塞提示 |
| `BootWaiting` | `Boot → Error` | `ErrorRoot` | 无 |
| `ErrorRoot` | 退出控件 | `Quit` | 直接退出，无需确认（`场景与流程管理` 规则 10） |
| `MenuRoot` | 「开始营业」 | `RunNoUI` | 组装 `RunConfig`；请求 `Menu → Run`；成功才离开 |
| `MenuRoot` | 打开升级 / 纪录 | `MenuSub` | 推入一个子屏；初始焦点为子屏主控件 |
| `MenuSub` | `ui_cancel` / 返回控件 | `MenuRoot` | 弹回主菜单；焦点回到打开它的入口 |
| `MenuRoot` | `ui_cancel` | `Modal1` | 打开退出确认；默认焦点「取消」 |
| `MenuRoot` | 设置入口 | `Modal1` | `push_context(Modal)`；打开设置 |
| `MenuSub`（升级） | 可购买项 | `Modal1` | 打开购买确认；默认焦点「取消」 |
| `Modal1`（设置） | 删除生涯 | `Modal2` | 第二层确认；默认焦点「取消」 |
| `Modal2` | 确认 / 取消 | `Modal1` | 关闭第二层；确认时执行删除并刷新界面 |
| `Modal1` | 确认 / 取消 / 强制模态的出口 | 基底 UI 状态 | `pop_context()`；恢复打开前的焦点 |
| `Menu` 任意 | 收到 `RunResult` | `Receipt` | 同帧入账编排（规则 10）；收据替换根屏 |
| `Receipt` | 「再来一局」 | `RunNoUI` | 全新 `RunConfig`；通知音效 `restart` |
| `Receipt` | 「返回菜单」/ `ui_cancel` | `MenuRoot` | 替换根屏；不重复入账；通知音效 `menu` |
| `RunNoUI` | `ui_pause`（由场景与流程管理消费） | `PauseOverlay` | HUD 保留，覆盖暂停菜单 |
| `PauseOverlay` | 「继续」/ `ui_pause` | `RunNoUI` | 请求 `Paused → Run`；画面与暂停前一致 |
| `PauseOverlay` | 「设置」 | `Modal1` | 打开设置；关闭后回到暂停菜单 |
| `PauseOverlay` | 「放弃本局」/ 「退出游戏」/ `ui_cancel` | `Modal1` | 打开对应确认；默认焦点「取消」 |
| `Modal1`（放弃确认） | 确认 | `MenuRoot` | 请求 `Paused → Menu, abandoned = true`；提示「本局已放弃」 |
| `Modal1`（退出确认） | 确认 | `Quit` | `flush()`；失败时二次确认；通知音效 `quit` |
| 任意 | 顶层状态变化 | 按新状态重建 | 丢弃屏幕栈与模态栈；不得保留旧状态界面 |
| `MenuRoot` | `init_failed` / `fault` / `abandoned` | `MenuRoot` | 显示非模态提示；不打开收据 |
| `MenuRoot` | `Recovered` | `MenuRoot` | 一次非模态说明；可关闭、不重复 |
| `MenuRoot` | `Blocked` | `Modal1` | 强制模态；只提供「退出」「新建生涯」 |
| 任意 `Menu` 界面 | 存档写入成功 | 不变 | 显示 1.5 秒「已保存」；不改变状态 |

**同帧冲突**：同一帧收到 `RunResult` 与 `ui_cancel` 时，先处理 `RunResult`（收据成为根屏），再在收据上下文解释 `ui_cancel`（跳过滚动或返回菜单），不得把 `ui_cancel` 泄漏给主菜单的退出确认。同一帧收到 `ui_accept` 与 `ui_cancel` 时，按 `输入` 规则 2 与当前上下文只消费一个动作。顶层切换与模态关闭同帧发生时，先由 `场景与流程管理` 完成切换，界面流程随后清空模态并重建，不尝试把模态恢复到新状态。

### Interactions with Other Systems

| 系统 | 方向 | 数据 / 接口 | 硬 / 软 |
|---|---|---|---|
| 场景与流程管理 | 上游（权威） | 读：当前顶层状态；订阅 `Menu` / `Run` / `Paused` / `Error` 切换与交付的 payload（`RunResult`、`abandoned`、`init_failed`）。写：`request_transition(Menu → Run, RunConfig)`、`request_transition(Paused → Run)`、`request_transition(Paused → Menu, abandoned)`、`request_transition(Quit)` | 硬 |
| 输入 | 上游（权威） | 读：`ui_up/down/left/right`、`ui_accept`、`ui_cancel`、焦点与悬停状态。写：`push_context(Modal)` / `pop_context()`。**不调用 `request_context`、不监听原始事件** | 硬 |
| 餐局流程 | 上游（数据） | 读：`RunConfig` 的校验结果（经切换 payload）；`RunResult` 的交付时机；`Ending` / `Settling` / `Handoff` 期间暂停菜单的限制。写：无（界面流程只组装 `RunConfig` 并请求切换） | 硬 |
| 结算与入场费 | 上游（数据） | 读：`RunResult` 的 `rows`、`total_value`、`entry_fee`、`net_profit`、`return_ratio` / `return_percent_x10`、容量/回合/腻度明细；`broke_even` 派生谓词。**不重算、不排序、不展示最优解** | 硬 |
| 局外升级 | 上游（数据） | 读：`get_wallet()`、`get_catalog()`、`get_level(id)`、`next_price(id)`、`can_purchase(id)`、`resolve_upgrade_snapshot()`、`progression_changed`。写：`apply_run_result(result)` 恰好一次、`purchase(id)` | 硬 |
| 纪录与里程碑 | 上游（数据） | 读：`snapshot()`、`milestone_progress(id)`、`record_broken` / `milestone_earned` / `career_completed` 信号。写：`apply_run(result)` 恰好一次（由界面流程作为结算事务编排者调用） | 硬 |
| 存档 | 上游（数据） | 读：`LoadReport`（`outcome`、`writable`、`warnings`）、`SaveStatus` / `save_status_changed`。写：`commit({progression, records[, unlocks]})`、`flush()`、`delete_career(confirm_token)` | 硬 |
| 音效反馈 | 同级协调（双向） | 界面流程拥有主音量 / 音效音量 / 静音 / 减少重复音效四项设置值；音效反馈消费它们。界面流程发 `ui_button_activated(control_id)`、收据 `number_roll_started/finished`、`receipt_closed(reason)`；音效不订阅原始输入、不读 `RunResult` | 硬（事件与设置契约）；播放本身为软 |
| 数据配置 | 上游（显示只读） | 读：`get_station(station_id)` 的 `display_name`、`entry_fee`（主菜单与收据的显示）；里程碑目录（名称、条件文案、阈值、`required_for_completion`）。**只用于显示，不参与任何玩法判定** | 硬（显示）；MVP 仅一个档口 |
| 局内 HUD | 同级协调 | 无数据依赖。屏幕归属：HUD 画局内一屏；界面流程画暂停菜单、收据、菜单、升级、设置、错误屏。`Paused` 时暂停菜单覆盖 HUD 但不卸载；`Settling` 时收据覆盖 HUD；HUD 保留到收据出现的那一帧 | 协调 |
| 首局引导（#18，未设计） | 下游（软） | 可在本系统控件上叠加讲解与高亮，可请求 `Blocked` 上下文；界面流程只提供控件锚点，不隐藏任何数字、不改变屏幕流程 | 软 |
| 解锁与档口（#13，未设计） | 下游（软） | 将来在主菜单提供档口选择；MVP 只有一个档口，主菜单不显示选择器。接口见 Open Questions 5、6 | 软 |

**接口所有权**：界面流程拥有屏幕集合、屏幕栈、焦点顺序、模态呈现、收据的打开时机与呈现编排、设置模型与界面语义事件。上游拥有各自的数据、判定与状态；任何系统不得直接写界面栈、焦点或模态栈，也不得替界面流程决定收据内容。界面流程不得替上游计算任何玩法数值，也不得把显示值回写成数据。

**事件与信号（过去式命名）**：`screen_changed(screen_id, ui_stack_depth)`、`modal_opened(modal_id, modal_depth)`、`modal_closed(modal_id, restored_focus_id)`、`receipt_opened(run_id)`、`receipt_closed(run_id, reason)`、`ui_button_activated(control_id)`、`number_roll_started(field, from, to, duration_ms)`、`number_roll_finished(field, skipped)`。这些事件供音效反馈与 QA 观察；它们不改变上游状态。

## Formulas

本节定义的是**呈现层可测规则**：屏幕栈深度、模态深度、返回决策、初始焦点、收据打开延迟、激活事件、收据行数与滚动进度。它们不是玩法数值，也不进入平衡表；但它们是可验收的硬约束，因此按公式写。所有布局与主题值的权威存放点是界面主题资源（Godot `Theme` / `UIConfig`），代码里不写字面量。

### 屏幕栈深度 `ui_stack_depth`

The `ui_stack_depth` formula is defined as:

`ui_stack_depth = |UI_STACK|`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `UI_STACK` | `ScreenId[]` | 0–2 项 | calculated | 当前屏幕栈；根屏在索引 1；`ReceiptScreen` 以替换根的方式出现 |
| `MAX_UI_STACK_DEPTH` | int | 2 | constant | 屏幕栈深度上限 |
| `screen_id` | `ScreenId` | 见 Core Rules 2 | calculated | 栈顶屏幕；深度不能区分 `MainMenu` 与 `ReceiptScreen`，消费者必须读它 |

**Output Range:** `0 | 1 | 2`。`0` 只允许出现在 `Run`（HUD 拥有画面）与 `Quit`；`1` 是根屏；`2` 是根屏 + 一个子屏。任何 `> 2` 或 `[ReceiptScreen, 子屏]` 都是缺陷。

**Example:** `MainMenu` → 1；推入 `UpgradeScreen` → 2；`ui_cancel` 弹回 → 1；`RunResult` 到达后 `[ReceiptScreen]` → 1。

### 模态深度 `modal_depth`

The `modal_depth` formula is defined as:

`modal_depth = |MODAL_STACK|`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `MODAL_STACK` | `ModalId[]` | 0–2 项 | calculated | 模态栈；模态不进入屏幕栈 |
| `MAX_MODAL_DEPTH` | int | 2 | constant | 模态深度上限 |
| `input_context` | enum | `Blocked / Menu / Run / Paused / Modal` | `输入` | 当前输入上下文；必须与 `modal_depth` 配对 |

**Output Range:** `0 | 1 | 2`。合法组合只有：`0`（无模态）；`1`（设置或任一确认）；`2`（设置 + 删除生涯确认）。深度已满时新请求返回 `false`，不排队、不替换。

**Example:** 主菜单打开设置 → 1；设置内打开删除生涯确认 → 2；确认或取消 → 1；关闭设置 → 0。

**配对断言（不是公式）**：`input_context == Modal ⇔ modal_depth > 0`；每次顶层状态切换后 `modal_depth = 0`（`输入` 规则 12 已清空临时栈，界面流程不得再弹栈）。

### 返回决策 `back_target`

The `back_target` formula is defined as:

`back_target(state, screen_id, modal_depth, modal_cancel_enabled, roll_active) = 第一条命中的规则`

| 优先级 | 条件 | 结果 |
|---|---|---|
| 1 | `modal_depth > 0 ∧ modal_cancel_enabled` | `CLOSE_MODAL` |
| 2 | `modal_depth > 0 ∧ ¬modal_cancel_enabled` | `NONE` |
| 3 | `roll_active` | `SKIP_ROLL` |
| 4 | `state = Menu ∧ screen_id = ReceiptScreen` | `REPLACE_ROOT(MAIN_MENU)` |
| 5 | `state = Menu ∧ screen_id ∈ {UpgradeScreen, RecordsScreen}` | `POP_SCREEN` |
| 6 | `state = Menu ∧ screen_id = MainMenu` | `OPEN_QUIT_CONFIRM` |
| 7 | `state = Paused` | `OPEN_QUIT_CONFIRM` |
| 8 | 其余 | `NONE` |

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `state` | enum | `Boot / Error / Menu / Run / Paused / Quit` | `场景与流程管理` | 当前顶层状态 |
| `screen_id` | enum | 见 Core Rules 2 | calculated | 栈顶屏幕；它已编码栈顶，`back_target` 不再单独接收 `ui_stack_depth` |
| `modal_depth` | int | 0–2 | `modal_depth` | 有模态时模态捕获 `ui_cancel` |
| `modal_cancel_enabled` | bool | true / false | 模态规格 | 强制模态（如 `Blocked`）为 `false` |
| `roll_active` | bool | true / false | 收据滚动控制器 | 是否正在滚动数字 |
| `back_target` | enum | `CLOSE_MODAL / SKIP_ROLL / REPLACE_ROOT / POP_SCREEN / OPEN_QUIT_CONFIRM / NONE` | calculated | 唯一返回结果 |

**Output Range:** 上表枚举值。`OPEN_QUIT_CONFIRM` 只在 `modal_depth = 0` 时可达；`REPLACE_ROOT` 只用于 `ReceiptScreen → MainMenu`；`SKIP_ROLL` 只在无模态时可达——模态打开时 `ui_cancel` 先归模态。`NONE` 必须产生可见的“无动作”而不是静默吞掉按键。

**Example:** `Menu` + `UpgradeScreen` + `modal_depth = 0` + `roll_active = false` → `POP_SCREEN`；`Paused` + `PauseOverlay` + `modal_depth = 0` → `OPEN_QUIT_CONFIRM`；`Menu` + `ReceiptScreen` + 滚动中 → `SKIP_ROLL`；`Blocked` 强制模态 → `NONE`。

### 初始焦点 `initial_focus`

The `initial_focus` formula is defined as:

`initial_focus(screen) = first(c ∈ screen.focus_order | c.enabled ∧ c.visible)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `focus_order` | `ControlId[]` | ≥ 1 项 | 屏幕规格 / `/ux-design` | 按屏幕位置排序的焦点顺序（上→下、左→右） |
| `enabled` | bool | true / false | 控件状态 | 禁用控件不可获焦 |
| `visible` | bool | true / false | 控件状态 | 隐藏控件不可获焦 |
| `initial_focus` | `ControlId \| NONE` | — | calculated | 屏幕打开时聚焦的控件；`NONE` 是缺陷 |

**Output Range:** 一个可用控件 id。若全表没有可用控件，返回 `NONE` 并记 ERROR——屏幕不得以“无焦点”上线。破坏性/不可逆确认的 `focus_order` 以「取消」开头；这是构建顺序表的规则，不是本函数的参数。

**Example:** `ReceiptScreen.focus_order = [再来一局, 返回菜单]` → `initial_focus = 再来一局`；`ConfirmModal`（退出/放弃/购买/删除生涯）的 `focus_order = [取消, 确认]` → `取消`。

**焦点恢复（不是公式）**：`focus_restore = focus_before_open`；若该控件在模态关闭时已禁用或移除，退回打开者所在屏幕 `focus_order` 的第一个可用控件，并记 WARNING。

### 收据打开延迟 `receipt_open_latency_frames`

The `receipt_open_latency_frames` formula is defined as:

`receipt_open_latency_frames = frame(receipt_opened) − frame(run_result_delivered)`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `frame(receipt_opened)` | int | ≥ 0 | 事件日志 | 收据根屏建立的帧号 |
| `frame(run_result_delivered)` | int | ≥ 0 | 事件日志 | `RunResult` 交付给界面流程的帧号 |
| `receipt_open_latency_frames` | int | 必须为 0 | calculated | 两者相差的帧数 |

**Output Range:** 必须为 `0`。任何非 0 值都是 `餐局流程` UI Requirements 5 的违反，不是性能问题。不得用 `ceil(t / FRAME_TIME)` 之类的连续时间换算——验收以事件日志的帧号为准。

**Example:** 帧 N 交付 `RunResult`，帧 N 建立收据 → `0`。

### 激活事件计数 `activation_event_count`

The `activation_event_count` formula is defined as:

`activation_event_count = 1 if (focused_enabled ∧ action_consumed ∧ ¬skip_consumed) else 0`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `focused_enabled` | bool | true / false | 输入 / 控件状态 | 被激活的控件当前可用 |
| `action_consumed` | bool | true / false | 界面流程 | 动作被该控件消费并提交 |
| `skip_consumed` | bool | true / false | 收据滚动控制器 | 该次 `ui_accept` / `ui_cancel` 只用于跳过滚动 |
| `activation_event_count` | int | 0 / 1 | calculated | 本次输入发出的 `ui_button_activated` 数 |

**Output Range:** `0` 或 `1`；同一帧至多一次。焦点移动、悬停、点击空白、禁用控件、滚动跳过都是 `0`。

**Example:** 购买确认「确认」→ 1；收据滚动中按 `ui_accept` → 0（`skip_consumed = true`）；点击空白 → 0。

### 收据行数 `receipt_rows_rendered`

The `receipt_rows_rendered` formula is defined as:

`receipt_rows_rendered = |RunResult.rows|`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `RunResult.rows` | `EatenRow[]` | 0–N | `结算与入场费` | 按吃下顺序排列的收据行 |
| `receipt_rows_rendered` | int | 0–N | calculated | 收据创建的**行模型**数；可滚动时可见行可以更少 |

**Output Range:** `0–N`，且必须等于 `|rows|`。不排序、不合并、不截断、不分页；`0` 显示空状态，不显示空表。滚动只影响可见行，不影响行模型数。

**Example:** `rows` 有 7 条 → 创建 7 个行模型，顺序与 `rows` 相同；`rows` 为空 → 空状态。

### 收据滚动 `receipt_roll_duration` / `roll_progress`

The `receipt_roll_duration` formula is defined as:

`receipt_roll_duration = 0 if reduce_motion else clamp(RECEIPT_ROLL_SECONDS, 0.3, 1.5)`

The `roll_progress` formula is defined as:

`roll_progress(t) = 1 if receipt_roll_duration = 0 else clamp(t / receipt_roll_duration, 0, 1)`

The `displayed_value` formula is defined as:

`displayed_value(t) = to if roll_progress(t) = 1 else round(from + (to − from) × roll_progress(t))`

**Variables:**

| Variable | Type | Range | Source | Description |
|---|---|---|---|---|
| `RECEIPT_ROLL_SECONDS` | float (s) | 0.3–1.5，默认 0.8 | data file | 收据数字滚动时长 |
| `reduce_motion` | bool | true / false | 设置模型 | 开启时滚动时长置 0 |
| `t` | float (s) | 0–`receipt_roll_duration` | calculated | 从滚动开始起的时长；`D` 在滚动开始时冻结 |
| `from` | int | 0 | constant | 滚动起点 |
| `to` | int64 | `RunResult` 字段值 | `结算与入场费` | 滚动终点，必须逐字段等于 `RunResult` |
| `roll_progress` | float | 0–1 | calculated | 滚动进度 |
| `displayed_value` | int64 | `from`–`to` | calculated | 当前帧显示的数字；`t ≥ D` 时精确等于 `to` |

**Output Range:** `roll_progress` 为 `0–1`；`displayed_value` 在 `from` 与 `to` 之间，`t ≥ D` 时必须**精确等于** `to`，不是“接近”。所有字段共用同一个 `D`；`D` 在滚动开始时冻结，滚动中切换 `reduce_motion` 只把当前滚动立即结束到最终值，不改变已经用过的分母。跳过时每个字段各发一次 `number_roll_finished(field, skipped = true)`。

**Example:** `D = 0.8`、`t = 0.4` → `roll_progress = 0.5`；`displayed_value = round(0 + (to − 0) × 0.5)`；`t ≥ 0.8` → `displayed_value = to`。`reduce_motion = true` → `D = 0`，立即显示 `to`，发 `started(duration_ms = 0)` + `finished(skipped = false)`。

### 断言（不是公式）

以下规则没有可计算的表达式，但必须作为运行时断言或测试夹具检查：

- `ui_stack_depth ≤ MAX_UI_STACK_DEPTH`；深度 2 的栈不得是 `[ReceiptScreen, 子屏]`。
- `input_context == Modal ⇔ modal_depth > 0`；顶层切换后 `modal_depth = 0`。
- 每个屏幕/模态至少有一个 `enabled ∧ visible` 的控件；否则记 ERROR。
- 收据滚动结束后，每个显示字段与 `RunResult` 对应字段逐字段相等；`broke_even` 只读派生谓词，不重新比较。
- 同一帧至多一次 `ui_button_activated`。
- 任何离开路径后不存在未弹出的 `Modal` 上下文。

## Edge Cases

**状态与同帧冲突**

- **If `RunResult` 与 `ui_cancel` 在同一帧到达**：先建立收据根屏，再在收据上下文解释 `ui_cancel`——滚动中先跳过滚动，否则返回主菜单；不得把该 `ui_cancel` 泄漏给主菜单的退出确认。
- **If 同一 `run_id` 的 `RunResult` 被交付两次**：第二份被忽略并记 WARNING；不重复入账、不打开第二张收据、不重复发 `receipt_opened`。
- **If `RunResult` 在非 `Menu` 顶层状态被交付**：`场景与流程管理` 已拒绝并记 ERROR；若仍到达界面流程，界面流程拒绝处理并记 ERROR，不改变当前界面。
- **If 顶层切换生效时仍有打开的模态**：`输入` 的 `request_context(clear_stack := true)` 已清空上下文；界面流程直接丢弃模态栈与屏幕栈，**不再调用 `pop_context()`**，按新状态重建，避免二次弹栈。
- **If `ui_accept` 与 `ui_cancel` 在同一帧按下**：按 `输入` 规则只消费一个动作；界面流程不自行合并、不执行两个动作。

**收据与结算**

- **If 收据正在滚动数字，玩家第一次按 `ui_accept` 或 `ui_cancel`**：只跳过滚动（`number_roll_finished(field, skipped = true)`），不激活「再来一局」或「返回菜单」；第二次按下才执行对应控件。
- **If `reduce_motion` 开启**：发 `number_roll_started(..., duration_ms = 0)` 后立即发 `number_roll_finished(..., skipped = false)`；最终值立刻显示，不产生 tick；下一次 `ui_accept` 直接激活控件。
- **If 玩家在滚动中把 `reduce_motion` 打开**：当前滚动立即结束到最终值，发 `number_roll_finished(..., skipped = false)`；没有残留 tick，焦点不变。
- **If 本局一道菜都没吃（`rows` 为空）**：收据显示「这一局什么都没吃」的空状态，不显示一张全 0 的表；总值 0、净利为 `−entry_fee`、回本判定「未回本」（除非 `entry_fee = 0`）；两个按钮照常可用。
- **If `entry_fee = 0`**：回本判定恒为「回本」，回本率位置显示「免单」，不出现 `∞`、`NaN` 或 `100%`；新纪录印章仍按 `net_profit > 0` 判断。
- **If `record_broken` 的 `previous_best = null`（首次纪录）**：只显示新最佳；「新纪录」印章仅在 `new_best.net_profit > 0` 时出现，不显示“旧最佳”。
- **If 新最佳 `net_profit ≤ 0`**：不显示「新纪录」印章、不显示安慰文案；若存在旧最佳，并排显示旧值，否则只显示本局。
- **If 同一帧达成多条里程碑**：收据底部「新达成」条按 id 升序列出全部；`career_completed()` 同帧到达时，完成横幅仍只显示一次，并排在里程碑条之后。
- **If `career_completed()` 在收据滚动中到达**：横幅等滚动结束（或跳过）后显示；若玩家在滚动结束前离开收据，横幅改为在下一次 `MainMenu` 根屏显示，且只显示一次。
- **If `纪录与里程碑.apply_run` 返回致命错误**（`INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY`）：不调用 `局外升级.apply_run_result`、不提交任何命名空间；收据仍显示 `RunResult` 的原始值，但顶部标注「结算数据异常，未入账、未记录」；不显示新纪录印章。
- **If 钱包入账溢出（`WALLET_OVERFLOW`）**：不提交本次结算事务，纪录更新一并放弃；收据显示原始值并标注「入账失败」；不显示新纪录印章。
- **If 结算事务写盘失败**：钱包与纪录保留内存新值、标记 dirty、由存档在下个时点重试；收据显示「未保存」标记；不把收据内容回滚成旧值，也不重复入账。
- **If 收据未关闭时又发生 `Run → Menu`**：不可能——一局只交付一次 `RunResult`；若发生，按同一 `run_id` 重复交付处理（忽略并记 WARNING）。

**开始、重开与退出**

- **If `request_transition(Menu → Run, RunConfig)` 返回 `false`**：留在主菜单，显示可见原因，焦点回到「开始营业」；不进入加载态、不重试、不假装开始。
- **If 收据「再来一局」的切换请求被拒绝**：留在收据，显示原因，焦点回到「再来一局」；不组装第二份 `RunConfig`、不关闭收据。
- **If `RunConfig` 组装失败**（档口无法解析）：主菜单的「开始营业」禁用并显示原因；不发起切换。
- **If `init_failed` 与 `abandoned` 同时出现**：`init_failed` 优先，显示「本局未能开始」；`abandoned` 提示不显示（本局从未开始）。
- **If 退出确认的 `flush()` 失败**：显示「未保存，仍要退出？」并再确认一次；取消回到打开确认前的界面，确认则进入 `Quit`。
- **If 窗口关闭请求到达时已有普通模态打开**：在 `Menu` 中直接请求 `Quit`（`场景与流程管理` 规则 10 对 `Menu` 不要求确认），并丢弃模态；在 `Paused` 中关闭当前模态、打开退出确认；任何情况下不叠加第二层模态。
- **If 窗口关闭请求到达时 `Blocked` 强制模态打开**：映射为 `Blocked` 的「退出」出口，直接请求 `Quit`；不打开第二个模态，不覆盖存档。
- **If 在 `Error` 状态按 `ui_cancel`**：无动作；只有「退出」（debug 另有「重新加载数据」）能离开。

**存档与设置**

- **If `LoadReport.outcome = Blocked`**：显示强制模态，只提供「退出」「新建生涯（旧档保留）」；`ui_cancel` 无效；默认焦点「退出」；任何其他主菜单控件不可操作。选择「新建生涯」后刷新钱包、升级、纪录与主菜单，选择「退出」则请求 `Quit`。
- **If `LoadReport.outcome = Recovered`**：显示一次非模态说明（来源为备份、可能丢失最近一次事务）；玩家可关闭，不重复弹出；主菜单照常可用。
- **If `LoadReport.outcome = Migrated`**：不弹提示；迁移写回由 1.5 秒「已保存」指示体现；若写回失败，按保存失败提示处理。
- **If 生涯在设置屏被删除**：关闭两层模态，刷新主菜单为新生涯（钱包 0、纪录空、升级归零）；设置保持原值；不重新载入设置文件。
- **If 设置写入失败**：保留内存值并显示「设置可能未保存」；不显示生涯「已保存」指示；下次修改或退出时重试。
- **If 购买尚未返回时再次确认**：忽略第二次确认，不发起第二笔；若上游返回 `BUSY`，显示「正在处理上一笔」并保持按钮禁用，直到第一笔返回。
- **If 文本缩放切到 150%**：当前屏幕立即重排；收据行不得重叠、金额列不得错位；焦点环仍可见；当前焦点控件保持。若某屏在 150% 下放不下，允许滚动，不允许截断关键信息。
- **If 音频被静音**：界面照常工作；不出现只靠声音传达的状态；静音解除后不补播此前的声音。

**输入与焦点**

- **If 鼠标悬停在控件 B 上而键盘焦点在控件 A**：两套焦点样式同时绘制；移动鼠标不改变键盘焦点；点击 B 触发一次 `ui_button_activated(B)`，不得同时触发 A。
- **If 手柄在模态打开时断开**：模态不自动关闭；焦点保持在原控件；键盘仍可完成模态的全部操作。
- **If 方向键按住跨上下文切换**：按 `输入` 规则不产生导航，必须释放后重按；界面流程不自行重复触发。
- **If 模态关闭**：焦点回到打开它的控件；若该控件已被禁用或移除，焦点落到当前屏幕顺序表中第一个可用控件，并记 WARNING。
- **If 一个屏幕没有任何可用焦点控件**：视为屏幕缺陷；界面流程记 ERROR，保留上一个焦点或聚焦屏幕根节点，不得让 `ui_accept` 无处可去。
- **If 模态深度已满（2）仍有新模态请求**：拒绝并记 WARNING，不排队、不替换当前模态；调用方必须等当前模态关闭后再请求。
- **If 强制模态（`Blocked`）收到 `ui_cancel`**：忽略，不关闭；只接受其两个出口。

## Dependencies

**上游依赖：**

| 系统 | 硬 / 软 | 接口 | 状态 |
|---|---|---|---|
| 场景与流程管理 | 硬 | 读顶层状态与切换 payload（`RunResult`、`abandoned`、`init_failed`）；写 `request_transition(Menu → Run, RunConfig)`、`request_transition(Paused → Run)`、`request_transition(Paused → Menu, abandoned)`、`request_transition(Quit)` | GDD 已写（`design/gdd/scene-flow.md`，In Review） |
| 输入 | 硬 | 读 `ui_up/down/left/right`、`ui_accept`、`ui_cancel`、焦点与悬停；写 `push_context(Modal)` / `pop_context()`；不调用 `request_context` | GDD 已写（`design/gdd/input.md`） |
| 餐局流程 | 硬 | 本系统组装 `RunConfig` 并交给 `场景与流程管理`，餐局流程消费它；接收 `RunResult` 的交付时机；只读 `run_state` 快照判断 `Ending` / `Settling` / `Handoff` 期间暂停菜单的限制 | GDD 已写（`design/gdd/run-flow.md`） |
| 结算与入场费 | 硬 | `RunResult` 的 `rows`、`total_value`、`entry_fee`、`net_profit`、`return_ratio` / `return_percent_x10`、容量/回合/腻度明细；`broke_even` 派生谓词 | GDD 已写（`design/gdd/settlement-and-entry-fee.md`） |
| 局外升级 | 硬 | `get_wallet()`、`get_catalog()`、`can_purchase(id)`、`resolve_upgrade_snapshot()`、`apply_run_result(result)`、`purchase(id)`、`progression_changed` | GDD 已写（`design/gdd/meta-upgrades.md`） |
| 纪录与里程碑 | 硬 | `apply_run(result)`、`snapshot()`、`milestone_progress(id)`、`record_broken` / `milestone_earned` / `career_completed` | GDD 已写（`design/gdd/records-and-milestones.md`） |
| 存档 | 硬 | `LoadReport`（`outcome`、`writable`、`warnings`）、`SaveStatus` / `save_status_changed`、`commit`、`flush()`、`delete_career(confirm_token)` | GDD 已写（`design/gdd/save-system.md`） |
| 音效反馈 | 硬（设置与事件契约） | 设置值：主音量、音效音量、静音、减少重复音效；界面事件：`ui_button_activated`、收据滚动、`receipt_closed` | GDD 已写（`design/gdd/audio-feedback.md`） |
| 数据配置 | 硬（显示只读） | `get_station(station_id)` 的 `display_name` / `entry_fee`；里程碑目录（名称、条件文案、阈值、`required_for_completion`）。只用于显示，不参与玩法判定 | GDD 已写（`design/gdd/data-config.md`） |

**下游依赖方：**

| 系统 | 硬 / 软 | 接口 | 状态 |
|---|---|---|---|
| 音效反馈 | 硬（事件契约）/ 软（播放） | 消费 `ui_button_activated`、`number_roll_started/finished`、`receipt_closed`；消费 UI 设置值 | GDD 已写（双向：本系统也消费其设置契约） |
| 首局引导（#18） | 软 | 在本系统控件上叠加讲解与高亮，可请求 `Blocked` 上下文；不得隐藏数字、不得改变屏幕流程 | GDD 未写 |
| 解锁与档口（#13） | 软 | 将来在主菜单提供档口选择；MVP 单档口，无选择器 | GDD 未写 |

**不属于依赖的关系：**

- **局内 HUD**：同级协调，不是上下游。HUD 画局内一屏，界面流程画暂停菜单、收据、菜单、升级、设置、错误屏；`Paused` 时暂停菜单覆盖 HUD 但不卸载，`Settling` 时收据覆盖 HUD。两者无数据依赖（`局内 HUD` Interactions）。
- **菜品数值 / 腻度与口味 / 菜单波次 / 胃容量 / 时间与回合**：界面流程不直接读取它们。它们的值经 `RunResult` 到达收据，经 HUD 到达局内画面；界面流程不保留第二份真相。
- **存档与设置**：设置不进入生涯档，删除生涯不重置设置（`存档` 规则 20）。界面流程拥有设置模型，但不把设置写进生涯档。
- **游戏概念**：来源文档，不是系统依赖。

**双向一致（待编排会话处理，本文件不修改）**：`场景与流程管理`、`餐局流程`、`结算与入场费`、`局外升级`、`纪录与里程碑`、`存档`、`音效反馈`、`输入` 的下游表中均已有「界面流程」行，状态仍是「GDD 未写」，需改为「GDD 已写（`design/gdd/ui-flow.md`）」。`数据配置` 的下游表目前**没有**「界面流程」行，需要新增一行，注明「显示只读：`StationData.display_name` / `entry_fee`、里程碑目录」。`局内 HUD` 的协调行状态需从「GDD 未写」改为已写。`systems-index.md` 第 16 行需要更新 Design Doc、Status 与 `Depends On`（补 `纪录与里程碑`、`音效反馈`、`数据配置`）。

## Visual/Audio Requirements

**视觉语言：食堂价目板。** 所有屏幕沿用游戏概念的视觉锚点——平整的价目板、文字优先、无装饰。没有插画、没有渐变光效、没有装饰性边框；层级靠字号、字重、留白与一条分隔线建立。数字是主角：金额、回合、容量、百分比使用等宽数字字体，标签比数字小一档、颜色更淡、基线对齐，字号放大时整行不跳动。

**每屏的视觉层级**：

- `MainMenu`：档口名与入场费是最大的一组数字；「开始营业」是唯一的主按钮；钱包、升级、纪录、设置、退出按固定顺序排列。入场费必须在开始按钮旁，玩家不能在不知道价格的情况下走进餐厅（`结算与入场费` UI Requirements）。
- `ReceiptScreen`：收据行是主体，合计区在右下或底部；`总值 / 入场费 / 净利` 三级字号；回本判定用文字 + 符号，不只靠颜色；空局显示一句空状态，不是一张全 0 表。
- `UpgradeScreen`：每项一张卡或一行，`当前效果 → 下一效果` 左右并排，价格与钱包同屏；不可购买的原因紧贴按钮，不用 tooltip 藏起来。
- `RecordsScreen`：每档口一行，最佳净利与回本率右对齐；里程碑分组用标题分隔；完成状态与「还差 N 项」在列表上方。
- `PauseOverlay`：覆盖层降饱和 HUD 但不隐藏数字；按钮竖排，焦点环清晰。
- `ErrorScreen`：平静、文字优先，不使用红色警报式闪烁；退出是唯一按钮。

**焦点、悬停与禁用**：键盘/手柄焦点与鼠标悬停是两套可同时存在的样式（`输入` 规则 8）。焦点环至少 2 px，并用形状或外扩轮廓表达，不只换颜色；悬停样式更淡，不能与焦点混淆。禁用控件降低对比度但保持文字可读，并在旁边给出原因，不能只变灰。

**动效**：只允许三类——顶层切换的淡入淡出（总时长 ≤ 0.2 秒，`场景与流程管理` 规则 12）、收据数字滚动（`RECEIPT_ROLL_SECONDS`，可跳过）、保存指示的淡出（1.5 秒后）。没有入场动画、没有按钮弹跳、没有装饰性粒子；任何动画都不得推迟数字出现或状态生效。`reduce_motion` 开启时，过渡与滚动时长都置 0。

**颜色与对比度**：所有文本与背景组合对比度 ≥ 4.5:1（与 `局内 HUD` 同一门槛）；正负、可购买/不可购买、回本/未回本都必须同时有符号与文字，不能只靠颜色。`design/accessibility-requirements.md` 尚未创建；本 GDD 只固定上述下限，完整标准由该文件拥有。

**音效**：界面流程不拥有任何 cue、不播放音频。它只发语义事件（`ui_button_activated`、收据滚动、`receipt_closed`），由 `音效反馈` 决定音色与时机。**没有焦点移动音**——焦点移动用视觉表达，避免奖励快速扫读（`音效反馈` 决策 5）。每一条声音信息都必须有视觉对应物，游戏可以完全静音通关。

> **📌 UX Flag — 界面流程**：`design/ux/` 尚未存在。Phase 4（Pre-Production）写 epic 之前，必须用 `/ux-design` 为 `main-menu`、`settlement`（收据）、`upgrades`、`records`、`settings`、`pause`、`error` 七个屏幕产出 UX 规格；相关 story 引用 `design/ux/[screen].md`，不直接引用本 GDD。本 GDD 只记录屏幕流程、焦点规则与信息清单，不创建 UX 文件。

## Game Feel

**目标手感是「翻页」，不是「过场」。** 玩家按下「开始营业」到看见第一道菜之间，不存在可被察觉的等待；打开升级、纪录、设置像翻一页账本，不像加载一个关卡。屏幕切换在 1 帧内完成，淡入淡出是唯一允许的视觉延迟，且不推迟状态与数字出现。

- **焦点要「贴手」**：方向键按下即移动焦点，不加速、不惯性、不等动画；模态打开时焦点立即进入模态，关闭时立即回到原控件。焦点环不闪烁、不脉动。
- **确认要「稳」**：退出、放弃、购买、删除生涯的默认焦点都在「取消」；玩家不会因为连按确认而误删生涯或误买升级。确认框出现时不抢走已发生的输入——第一次 `ui_accept` 只用于确认，不会穿透到下一屏。
- **收据要「读得完」**：收据不是动画表演。数字滚动只是让净利有一个“落下来”的瞬间，最多 0.8 秒，可一键跳过；行与合计在滚动前后位置不变，不因为数字变长而抖动。
- **暂停要「原样」**：暂停覆盖层不卸载 HUD、不重置焦点、不改变任何数字；恢复后画面与暂停前逐像素一致。
- **失败要「平静」**：错误屏、`Blocked` 说明、进入失败提示都用价目板式排版，说明发生了什么与唯一出路；不闪烁、不倒数、不制造紧迫感（支柱 3）。
- **不做**：加载进度条、转场动画、按钮弹跳、焦点音、限时确认、任何让玩家等一等才看到数字的东西。

## UI Requirements

本系统就是 UI 系统；下表是 MVP 的屏幕契约。信息清单是**必须包含**的下限，不是版式规范——版式、间距、字体与动效细节由 `/ux-design` 的屏幕规格拥有。

| 屏幕 / 模态 | 进入 | 必须包含 | 初始焦点 | 返回 |
|---|---|---|---|---|
| `BootWaiting` | `Boot` | 等待文案、静态标识；无进度条、无最短展示时长 | 无 | 无 |
| `ErrorScreen` | `Error` | 可读的错误标题与说明；唯一的「退出」；debug 额外「重新加载数据」、失败文件、记录 id | 「退出」 | `ui_cancel` 无效 |
| `MainMenu` | `Menu` 根屏 | 档口名、入场费、钱包（带符号）；「开始营业」「升级」「纪录」「设置」「退出」；保存指示与非模态提示层 | 「开始营业」 | `ui_cancel` → 退出确认 |
| `ReceiptScreen` | `RunResult` 同帧 | 行：菜名、价格、基础饱腹、腻度追加（0 显示 `—`）、实际饱腹、静态单位价值、有效单位价值；合计：总值、入场费（负项）、净利（带符号）、回本判定、回本率、已用/总容量、浪费容量、已用/总回合、腻度额外占用合计；空局空状态；免单；新纪录印章 + 旧最佳；「新达成」条；钱包；「再来一局」「返回菜单」 | 「再来一局」 | `ui_cancel` → 先跳过滚动，再返回主菜单 |
| `UpgradeScreen` | `Menu` 子屏 | 每条：名称、等级 `n / max`、当前效果、下一效果、下一价格、可购买性与原因；钱包常驻；购买确认；`COMMIT_FAILED` / `MAX_LEVEL` / `NO_EFFECT` / `NOT_WRITABLE`；推荐高亮 | 第一条可购买项的购买按钮；若无可购买项则为「返回」 | `ui_cancel` → 弹回主菜单 |
| `RecordsScreen` | `Menu` 子屏 | 每档口一行：档口名、最佳净利（带符号）、回本率（免单显示「免单」）、总值、达成日期、空状态；里程碑按「未达成 / 已达成」分组，含名称、条件、进度 `当前值 / 阈值 单位`、达成日期；完成状态与「距离通关还差 N 项」；完成横幅 | 第一条纪录行；空则为「返回」 | `ui_cancel` → 弹回主菜单 |
| `PauseOverlay` | `Paused` | 「继续」「设置」「放弃本局」「退出游戏」；`Ending` / `Settling` / `Handoff` 时隐藏「放弃本局」 | 「继续」 | `ui_cancel` → 退出确认 |
| `SettingsModal` | 任意有界面的状态 | 音频：主音量、音效音量、静音、减少重复音效；无障碍：文本缩放、减少动态；显示：HUD 不透明度；次级：删除生涯（两次确认）；「关闭」 | 主音量滑杆 | `ui_cancel` → 关闭并恢复打开前焦点 |
| `ConfirmModal` | 任一确认动作 | 标题、后果文案、「取消」、「确认」；不可逆/破坏性动作默认「取消」 | 「取消」 | `ui_cancel` → 取消 |
| `BlockedModal` | `LoadReport = Blocked` | 三种原因之一；「退出」「新建生涯（旧档保留）」；无「覆盖保存」 | 「退出」 | `ui_cancel` 无效（强制模态） |
| `NoticeLayer` | 各种事件 | 保存指示、恢复说明、进入失败/放弃提示、完成横幅 | 无（非模态） | 可关闭项用 `ui_accept` / `ui_cancel` 关闭 |

**焦点与顺序**：每个屏幕的焦点顺序按屏幕位置（上→下、左→右），不按节点树；初始焦点以上表为准。模态打开时焦点移入模态并形成陷阱，关闭后回到打开它的控件；若该控件已不可用，退回父屏幕顺序表的第一个可用控件。禁用控件不可获焦，且必须在控件旁给出原因。鼠标悬停与键盘/手柄焦点分别绘制（`输入` 规则 8）；鼠标点击只激活悬停控件，一次点击只发一次 `ui_button_activated`。

**键盘/手柄完整可玩**：每个屏幕与模态都必须能只用键盘或手柄完成全部操作，包括滑杆（左右调整）、列表浏览、确认与返回；鼠标是可选加速器。不提供虚拟摇杆或触屏控件（`输入` Open Question 5）。

**数字与文案格式**：正数带 `+`、负数带 `−`；回本率保留一位小数并带 `%`（`122.3%`）；免单档口显示「免单」；腻度追加为 0 显示 `—`；空状态用完整句子（「这一局什么都没吃」「还没有账本」），不显示全 0 表格。所有玩家可见字符串使用 `tr()`，不得硬编码；正式文案与字符上限由 `/ux-design` 的文案表拥有。

**无障碍**：所有文本/背景组合对比度 ≥ 4.5:1；正负、回本/未回本、可购买/不可购买同时用符号与文字表达；字号 100 / 125 / 150% 三档，150% 时收据行不换行错位、金额列不重叠、焦点环不被裁切、关键信息不截断（允许滚动，不允许消失）；`reduce_motion` 关闭过渡与数字滚动；不存在限时确认；`Error` 与 `Blocked` 的信息不能只靠颜色传达。完整标准见未来的 `design/accessibility-requirements.md`（尚未创建）。

> **📌 UX Flag — 界面流程**：本系统有 UI 需求且 `design/ux/` 尚未存在。Phase 4（Pre-Production）写 epic 之前，必须用 `/ux-design` 覆盖 `main-menu`、`settlement`、`upgrades`、`records`、`settings`、`pause`、`error` 七个屏幕；相关 story 引用 `design/ux/[screen].md`，不直接引用本 GDD。本文件不创建 UX 规格。

## Cross-References

| 文档 | 关系 | 本系统引用 / 依据 |
|---|---|---|
| `design/gdd/game-concept.md` | 来源 | 支柱 4「数字看得见地涨」、支柱 3「时间在走，手不用快」、支柱 2「没有白吃的菜」、视觉锚点「食堂价目板」、核心循环「总价值 − 入场费 = 净利」 |
| `design/gdd/scene-flow.md` | 上游（权威） | 六个顶层状态、`request_transition`、`RunResult` / `abandoned` / `init_failed` 交付、暂停路由、过渡上限 |
| `design/gdd/input.md` | 上游（权威） | 动作集、导航重复、双焦点、`push_context(Modal)` / `pop_context()`、初始焦点与键盘可玩要求 |
| `design/gdd/run-flow.md` | 上游（数据） / 边界 | `RunConfig` 组装、`RunResult` 同帧交付、`Ending` / `Settling` / `Handoff` 的暂停限制、`run_state` 只读快照 |
| `design/gdd/settlement-and-entry-fee.md` | 上游（数据） | `RunResult` 字段、收据顺序与合计、空局与免单、不展示最优解、`broke_even` 派生谓词 |
| `design/gdd/meta-upgrades.md` | 上游（数据） | `get_wallet` / `get_catalog` / `can_purchase` / `resolve_upgrade_snapshot` / `apply_run_result` / `purchase` / `progression_changed` |
| `design/gdd/records-and-milestones.md` | 上游（数据） | `apply_run` / `snapshot` / `milestone_progress`、`record_broken` / `milestone_earned` / `career_completed`、新纪录印章规则、完成横幅 |
| `design/gdd/save-system.md` | 上游（数据） | `LoadReport`、`SaveStatus`、保存指示、`Blocked` / `Recovered`、`commit` / `flush` / `delete_career`、设置不属于生涯档 |
| `design/gdd/audio-feedback.md` | 同级（接口） | 音频设置四项、`ui_button_activated`、收据滚动事件、`receipt_closed`、无焦点音 |
| `design/gdd/data-config.md` | 上游（显示只读） | `StationData.display_name` / `entry_fee`、里程碑目录（名称、条件、阈值、`required_for_completion`）；不参与玩法判定 |
| `design/gdd/in-game-hud.md` | 同级（协调） | 屏幕归属、暂停覆盖层不卸载 HUD、收据覆盖 HUD 的时机、HUD 不透明度与文本缩放设置 |
| `design/gdd/systems-index.md` | 索引 | 优先级 MVP、层 Presentation、设计顺序 #13、依赖行需补 `纪录与里程碑` / `音效反馈` / `数据配置` |
| `design/accessibility-requirements.md` | 待创建 | 对比度、字号缩放、减少动态、颜色之外的第二通道；本 GDD 只固定下限 |
| `design/ux/[screen].md` | 待创建 | 七屏 UX 规格；story 引用 UX 规格，不直接引用本 GDD |

**双向回指（待编排会话处理）**：`场景与流程管理`、`输入`、`餐局流程`、`结算与入场费`、`局外升级`、`纪录与里程碑`、`存档`、`音效反馈`、`局内 HUD` 的下游/协调表中已有「界面流程」行，状态需改为已写；`数据配置` 需新增「界面流程（显示只读）」行；`systems-index.md` 第 16 行需更新。详见 Dependencies 末段。

## Acceptance Criteria

> **测试夹具（一次定义，下文引用）**：F1 帧索引事件日志（`frame`、事件名、payload）；F2 可注入时钟（保存指示、收据滚动）；F3 上游调用记录器（`load` / `apply_run` / `apply_run_result` / `commit` / `purchase` / `flush` / `delete_career` 的调用次数、参数与顺序）；F4 `run_state` 快照探针；F5 `record_broken` 夹具（含 `previous_best = null` 与负纪录）；F6 逐屏焦点清单（每屏可用控件 id、顺序、禁用项）；F7 设置模型注入器；F8 截图夹具（1280×720、150% 字号、焦点/悬停、空局、免单、`Blocked`）。

> **公式覆盖**：`ui_stack_depth` 由第 1、3、5 条；`modal_depth` 由第 39 条；`back_target` 由第 38 条；`initial_focus` 由第 37 条；`receipt_open_latency_frames` 由第 11 条；`activation_event_count` 由第 43 条；`receipt_rows_rendered` 由第 13、14 条；`receipt_roll_duration` / `roll_progress` / `displayed_value` 由第 19、20 条。其余判据来自 Detailed Design 与 Edge Cases。

> **集成判据**：第 1、2、5、11、18、21–24、37–39、41–42 条需要 `场景与流程管理` / `输入` 的测试替身；第 6–10、12–17、28–32 条需要 `餐局流程` / `结算与入场费` / `局外升级` / `纪录与里程碑` 的测试替身；第 25–28、33–34 条需要 `存档` 的载入/写入失败注入点。只测本系统的判据使用 F1 + F6 + F7 即可。

**屏幕与状态映射**

1. **GIVEN** `Boot`，**WHEN** `Boot → Menu` 生效，**THEN** 屏幕栈恰为 `[MainMenu]`、`ui_stack_depth = 1`、焦点为「开始营业」、`load()` 恰好调用一次，且没有任何加载帧。
2. **GIVEN** `Boot`，**WHEN** 数据校验失败进入 `Error`，**THEN** 屏幕栈为 `[ErrorScreen]`，只有「退出」（debug 另有「重新加载数据」）；不存在进入 `Menu` / `Run` 的控件；`ui_cancel` 不产生任何动作；release 构建中「重新加载数据」不存在。
3. **GIVEN** `Run`，**WHEN** 检查界面流程，**THEN** 它没有屏幕、`ui_stack_depth = 0`，局内画面由 `局内 HUD` 拥有；界面流程不绘制、不覆盖 HUD。
4. **GIVEN** `Paused`，**WHEN** 打开暂停菜单，**THEN** HUD 实例 id 与暂停前相同、未被卸载；菜单项为「继续」「设置」「放弃本局」「退出游戏」，焦点为「继续」；**GIVEN** `run_state ∈ {Ending, Settling, Handoff}`，**THEN** 「放弃本局」不存在。
5. **GIVEN** 任一模态或子屏打开，**WHEN** 顶层状态切换生效，**THEN** 屏幕栈与模态栈被丢弃、`modal_depth = 0`、不调用 `pop_context()`，并按新状态重建界面。

**开始与重开**

6. **GIVEN** `MenuRoot`，**WHEN** 按「开始营业」，**THEN** 只组装一份 `RunConfig`：`station_id = station_canteen`、`seed = null`、`pool_override = null`、`is_first_run = (career.runs_settled == 0)`、`upgrade_snapshot` 是一次 `resolve_upgrade_snapshot()` 的副本；只调用一次 `request_transition(Menu → Run, RunConfig)`；不插入加载屏。
7. **GIVEN** `request_transition(Menu → Run)` 返回 `false`，**WHEN** 开始营业，**THEN** 留在主菜单、显示可见原因、焦点回到「开始营业」；不重试、不进入加载态、不假装成功。
8. **GIVEN** 档口无法解析或 `RunConfig` 组装失败，**WHEN** 检查主菜单，**THEN** 「开始营业」禁用并显示原因；不发起切换。
9. **GIVEN** 收据，**WHEN** 按「再来一局」，**THEN** 组装全新 `RunConfig`（第二次 `resolve_upgrade_snapshot()`，对象不同；`is_first_run = false`；`seed = null`）、请求一次 `Menu → Run`、发 `receipt_closed(reason = restart)`；新局获得新的 `run_id`。
10. **GIVEN** 收据，**WHEN** 「再来一局」的切换请求被拒绝，**THEN** 留在收据、显示原因、焦点回到「再来一局」；不组装第二份 `RunConfig`、不关闭收据。

**收据**

11. **GIVEN** `Run → Menu` 在帧 N 交付 `run_id = R` 的 `RunResult`，**WHEN** 该帧渲染，**THEN** `ReceiptScreen` 在帧 N 成为根屏，`receipt_open_latency_frames = 0`；`apply_run` → `apply_run_result` → `commit` 各恰好一次且顺序固定；同一 `R` 再次交付不改变任何调用计数。
12. **GIVEN** `records.apply_run` 返回 `INCONSISTENT_RESULT` / `OVERFLOW` / `DATA_NOT_READY`，**WHEN** 处理 `RunResult`，**THEN** 不调用 `apply_run_result`、不提交任何命名空间；收据显示原始 `RunResult` 值并标注「结算数据异常，未入账、未记录」；不显示新纪录印章。
13. **GIVEN** `rows = [A, B, C]` 且其中一行 `cloy_extra = 0`，**WHEN** 打开收据，**THEN** `receipt_rows_rendered = 3`，顺序与 `rows` 相同；该行腻度追加显示 `—`；总值、入场费、净利、回本判定、容量/回合/腻度合计与 `RunResult` 逐字段相等。
14. **GIVEN** `rows = []`，**WHEN** 打开收据，**THEN** 显示「这一局什么都没吃」的空状态，不显示全 0 表；总值为 0、净利为 `−entry_fee`、回本判定为「未回本」（`entry_fee = 0` 时除外）。
15. **GIVEN** `entry_fee = 0`，**WHEN** 打开收据，**THEN** 回本判定为「回本」、回本率位置显示「免单」；不出现 `∞`、`NaN` 或 `100%`。
16. **GIVEN** `record_broken` 且新最佳 `net_profit > 0`，**WHEN** 收据打开，**THEN** 显示「新纪录」印章与旧最佳（首次纪录 `previous_best = null` 时不显示旧最佳）；**GIVEN** 新最佳 `net_profit ≤ 0`，**THEN** 不显示印章、不显示安慰文案。
17. **GIVEN** 任意 `RunResult`，**WHEN** 检查收据，**THEN** 不排序、不合并、不截断行；净利、总值、回本判定直接取自 `RunResult`；不显示 `opt_value` 或“本可以多赚 X”。
18. **GIVEN** 收据，**WHEN** 按「返回菜单」，**THEN** 根屏替换为 `MainMenu`、`apply_run` / `apply_run_result` 计数仍为 1、焦点为「开始营业」、发 `receipt_closed(reason = menu)`。
19. **GIVEN** 收据滚动中，**WHEN** 第一次按 `ui_accept` 或 `ui_cancel`，**THEN** 只跳过滚动（每个字段一次 `number_roll_finished(..., skipped = true)`），不激活按钮；第二次按下才执行控件。**GIVEN** `reduce_motion = true`，**THEN** 发 `number_roll_started(..., duration_ms = 0)` + `number_roll_finished(..., skipped = false)`，最终值立即显示。
20. **GIVEN** 滚动结束或跳过，**WHEN** 检查显示值，**THEN** 每个字段与 `RunResult` 精确相等；`receipt_roll_duration` 在滚动开始时冻结，滚动中切换 `reduce_motion` 只把当前滚动立即结束到最终值。

**暂停、放弃与退出**

21. **GIVEN** `Paused`，**WHEN** 按「继续」，**THEN** 只请求一次 `Paused → Run`，HUD 画面与暂停前一致；**WHEN** 打开设置后按 `ui_cancel`，**THEN** 只关闭设置、回到暂停菜单，不恢复 `Run`。
22. **GIVEN** `Paused`，**WHEN** 按「放弃本局」，**THEN** 打开确认框、默认焦点「取消」；确认后只请求一次 `Paused → Menu, abandoned = true`，不打开收据、不写纪录、不入账；取消则留在暂停菜单。
23. **GIVEN** `Run` / `Paused`，**WHEN** 请求退出或关闭窗口，**THEN** 先打开确认框（默认「取消」）；确认后调用 `flush()` 并进入 `Quit`；**GIVEN** `flush()` 失败，**THEN** 显示「未保存，仍要退出？」并再确认一次。
24. **GIVEN** `Boot` / `Menu` / `Error`，**WHEN** 请求退出或关闭窗口，**THEN** 无需确认直接进入 `Quit`。

**存档**

25. **GIVEN** `LoadReport = Fresh / Loaded / Migrated`，**WHEN** 进入主菜单，**THEN** 不弹载入提示；**GIVEN** `Recovered`，**THEN** 显示一次非模态说明，可关闭且不重复。
26. **GIVEN** `LoadReport = Blocked`，**WHEN** 进入主菜单，**THEN** 模态深度为 1、只有「退出」「新建生涯（旧档保留）」两个按钮、默认焦点「退出」、`ui_cancel` 无效、无「覆盖保存」；主菜单其他控件不可操作；「新建生涯」隔离旧档并刷新钱包/升级/纪录；窗口关闭按 `Menu` 规则直接 `Quit`。
27. **GIVEN** 一次成功写入，**WHEN** 保存完成，**THEN** 显示「已保存」约 1.5 秒后淡出；不阻塞输入、不移动布局、不发声；**GIVEN** 购买写盘失败，**THEN** 显示「未保存，购买未生效」并恢复按钮；**GIVEN** 设置写入失败，**THEN** 只显示「设置可能未保存」，不显示生涯保存指示。
28. **GIVEN** `LoadReport.writable = false`，**WHEN** 打开升级页，**THEN** 所有购买按钮禁用并显示原因；`purchase` 不被调用。

**升级与纪录**

29. **GIVEN** `UpgradeScreen`，**WHEN** 打开，**THEN** 每条显示名称、等级 `n / max`、当前效果、下一效果、下一价格、可购买性与原因；钱包常驻且负数带负号；初始焦点为第一条可购买项的购买按钮，若无可购买项则为「返回」。
30. **GIVEN** 可购买项，**WHEN** 点击购买，**THEN** 打开确认框（默认「取消」，显示价格与购买后余额）；确认后只调用一次 `purchase(id)`；**GIVEN** `COMMIT_FAILED`，**THEN** 显示「未保存，购买未生效」并恢复按钮；`MAX_LEVEL` / `NO_EFFECT` / `NOT_WRITABLE` / `DATA_NOT_READY` / `BUSY` 显示对应原因；不存在自动购买。
31. **GIVEN** `RecordsScreen`，**WHEN** 打开，**THEN** 每档口一行（档口名、最佳净利带符号、回本率、总值、达成日期），空生涯显示「还没有账本」；里程碑按「未达成 / 已达成」分组并显示 `当前值 / 阈值 单位`；`completed` 只取 `completed_at != null`，不由界面重算。
32. **GIVEN** `career_completed()` 到达，**WHEN** 收据正开着，**THEN** 横幅等滚动结束后显示一次；**GIVEN** 玩家已离开收据，**THEN** 横幅在下一次 `MainMenu` 显示，不在一张新收据上显示；横幅可关闭、不阻塞、不重复。

**设置与无障碍**

33. **GIVEN** 设置屏，**WHEN** 修改主音量、音效音量、静音、减少重复音效、文本缩放、减少动态、HUD 不透明度，**THEN** 立即生效；设置写入机器级文件，不进入生涯档；删除生涯后设置保持原值。
34. **GIVEN** 设置屏的删除生涯，**WHEN** 选择删除，**THEN** 需要两次确认（第二次默认「取消」）；确认后主菜单刷新为新生涯（钱包 0、纪录空、升级归零），设置不变。
35. **GIVEN** 文本缩放 150%，**WHEN** 打开主菜单、收据、升级、纪录、设置、暂停、错误屏，**THEN** 无文字溢出、无相邻元素重叠、无关键信息截断；金额列不重叠；焦点环可见；允许滚动，不允许消失。
36. **GIVEN** 每一组文本/背景组合，**WHEN** 用对比度工具实测，**THEN** 全部 ≥ 4.5:1；正负、回本/未回本、可购买/不可购买同时有符号与文字，不只靠颜色。

**焦点、返回与模态**

37. **GIVEN** F6 的逐屏焦点清单，**WHEN** 每个屏幕/模态首次显示，**THEN** `initial_focus` 等于清单写死的控件；若清单为空则记 ERROR。
38. **GIVEN** F1，**WHEN** 逐项触发 `ui_cancel`：主菜单根屏 → 打开退出确认；升级/纪录子屏 → 弹回；收据 → 返回主菜单；暂停 → 打开退出确认；`Error` → 无动作；可取消模态 → 关闭；强制 `Blocked` 模态 → 无动作；**THEN** 结果与 `back_target` 表逐项一致。
39. **GIVEN** 设置内打开删除生涯确认，**WHEN** 检查模态深度，**THEN** `modal_depth = 2`；**GIVEN** 已为 2 时请求第三个模态，**THEN** 返回 `false`、不排队、不替换；**THEN** `input_context == Modal ⇔ modal_depth > 0`，顶层切换后 `modal_depth = 0`。
40. **GIVEN** 模态关闭，**WHEN** 打开它的控件已禁用或移除，**THEN** 焦点退回父屏幕顺序表第一个可用控件并记 WARNING；否则回到原控件。
41. **GIVEN** 鼠标悬停在 B、键盘焦点在 A，**WHEN** 移动鼠标与按方向键，**THEN** 两套焦点样式同时绘制；鼠标移动不改变键盘焦点；点击 B 只发一次 `ui_button_activated(B)`。
42. **GIVEN** 只有键盘或只有手柄，**WHEN** 从主菜单走完一局到收据、再返回主菜单，**THEN** 全程无需鼠标；所有滑杆、列表、确认与返回均可操作。

**事件与证据**

43. **GIVEN** F1，**WHEN** 焦点移动、悬停、点击空白、按禁用控件、滚动跳过，**THEN** `activation_event_count = 0`；**WHEN** 一次已提交激活，**THEN** 恰好 1 次；同一帧至多 1 次。
44. **GIVEN** 收据关闭 / 返回菜单 / 开始新一局，**WHEN** 检查事件，**THEN** 各发一次 `receipt_closed(reason)`；音效的待播结算结果 cue 被取消。
45. **GIVEN** 实现完成，**WHEN** 关闭验收，**THEN** `production/qa/evidence/` 中存在以下保留截图：主菜单（1280×720）、收据（含行与合计）、收据空局、免单、新纪录印章、升级页、纪录页、设置屏、暂停覆盖层、`Error` 屏、`Blocked` 模态、150% 字号下的收据与设置屏、键盘焦点与鼠标悬停同时可见。

## Open Questions

| # | 问题 | 负责人 | 何时解决 |
|---|---|---|---|
| 1 | **结算事务的单事务边界如何落地？** 本 GDD 指定界面流程为编排者，并要求 `apply_run_result` 可拆成「prepare（内存应用，不提交）」与「commit」两段，以便与 `records` 合并为一次 `save.commit`（`存档` 规则 80）。当前 `局外升级` 规则 6 描述的是「内存应用 → commit」的内部提交。若维持现状，MVP 只能两次串行提交，且纪录致命错误时无法回滚已入账钱包。需要架构阶段决定接口形状。 | lead-programmer + technical-director | 架构阶段 |
| 2 | **`reduce_motion` 如何影响 `场景与流程管理` 的过渡？** 过渡时长 `FADE_OUT` / `FADE_IN` 由场景与流程管理拥有，设置值由界面流程拥有。是场景与流程管理读取设置模型，还是界面流程在切换前把设置传给它？本 GDD 只规定「关闭时两者为 0」，不规定传输方式。 | lead-programmer + ux-designer | 架构阶段 |
| 3 | **设置文件的格式、路径与写入时机**（如 `user://settings.cfg`；每次修改即写还是关闭时写；损坏时回退默认值）。设置不属于生涯档，需与存档的目录/文件命名一起确定。 | lead-programmer + ux-designer | 架构阶段 |
| 4 | **收据屏是否需要提升为独立顶层状态？** 当前归入 `Menu` 根屏（`场景与流程管理` Open Question 1）。若收据需要阻止一切返回主菜单的操作，或需要独立的暂停语义，则要新增顶层状态；本 GDD 的 `back_target` 与 `ui_stack_depth` 需同步改。 | game-designer | 垂直切片前 |
| 5 | **主菜单的档口展示由谁汇总？** 本 GDD 让界面流程直接读 `数据配置.get_station()` 的 `display_name` / `entry_fee`，这是显示只读依赖。若未来 `解锁与档口`（#13）需要“已解锁档口列表 + 当前选择”，是继续直读数据配置，还是由一个菜单视图模型汇总？ | systems-designer + game-designer | 设计 #13 时 |
| 6 | **多档口时的开始流程**：主菜单需要档口选择器、每个档口的入场费与纪录入口；MVP 单档口不显示选择器。选择器是主菜单内联，还是独立子屏？ | ux-designer | 设计 #13 时 |
| 7 | **收据滚动的时长、可跳过提示与「减少动态」的呈现**：0.8 秒是默认值还是上限？跳过提示放在哪里？`reduce_motion` 开启时是否仍显示“滚动”状态文案？ | ux-designer | Phase 4（写 epic 前） |
| 8 | **`Blocked` 三种原因的正式文案与隔离文件路径**：三条原因（更新版本 / 无法读取 / 另一实例）的玩家可读文案、是否显示文件路径、删除/隔离后的找回说明。 | ux-designer + lead-programmer | Phase 4 |
| 9 | **完成横幅的精确时机与位置**：在收据滚动结束后、主菜单顶部，还是纪录页入口？与「新达成」条的视觉关系？ | ux-designer | Phase 4 |
| 10 | **退出时 flush 失败的二次确认文案与强制退出策略**：允许“仍要退出”还是必须留在游戏内重试？本 GDD 暂定允许二次确认后退出。 | ux-designer + qa-lead | Phase 4 |
| 11 | **首局引导（#18）如何锚定界面控件**：引导需要主菜单「开始营业」、收据单位价值列、升级页推荐项的锚点；锚点由界面流程提供还是由引导按控件 id 查找？ | ux-designer | 设计 #18 时 |
| 12 | **文案与本地化**：所有玩家可见字符串的正式中文、字符上限、折行规则与 `tr()` key 命名。 | writer + localization-lead | Phase 4 |
| 13 | **结算致命错误时收据是否仍显示原始值**：本 GDD 暂定“显示原始 `RunResult` + 标注未入账未记录”，因为界面流程不能重算、也不能伪造零分。需要确认这是否会让玩家把一份已知不一致的收据当成事实。 | game-designer + qa-lead | 架构阶段 |
| 14 | **`push_context(Modal)` 在栈顶已为 `Modal` 时的语义**：`输入` 规则 12 说“空操作（同值覆盖）不发信号”，但没有明确“压栈一个与栈顶同值的上下文”算不算空操作。删除生涯确认需要连续两次压入 `Modal` 并两次弹出；若第二次压栈被当作空操作，`pop_context()` 会把设置也一起弹出。需要 `输入` GDD 明确：压栈按栈深度变化处理，即使值相同也压栈并发一次 `context_changed(Modal, Modal)`。 | systems-designer + lead-programmer | 架构阶段 |

**GDD 定义行为，ADR 记录实现路径。** 以下实现问题需要在 ADR 中落地：屏幕控制器与场景树结构（单根控制器 vs 每屏一个场景）、屏幕栈与模态栈的实现方式、焦点邻居的手工配置 vs 自动排序、过渡动画的播放节点、设置文件的序列化、事件日志的落点。屏幕层级、返回决策、焦点归属与收据打开时机的**行为**已在正文定义，ADR 只决定实现手段。
