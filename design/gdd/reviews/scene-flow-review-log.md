# 场景与流程管理（scene-flow.md）— 设计评审日志

## Review — 2026-11-03 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 3 | Recommended: 6
Summary: 状态机本体是扎实的：六个顶层状态的转移表闭合、first-wins 与幂等语义明确，与 数据配置 / 输入 / 界面流程 / 餐局流程 / 首局引导 / 音效反馈 六份 GDD 的接口逐条对得上，支柱 2/3/4 的落点也都站得住。三个 blocking 项都不是设计缺陷，而是 Formulas 的一处悬空数据来源、Cross-References 表的陈旧自相矛盾，以及公开接口上一个未定义的返回值——全部是局部文字修复，修完即可通过。无结构性返工。
Prior verdict resolved: First logged review。systems-index 记录本系统于 2026-11-03 通过评审（Approved），但该分支上没有对应的评审日志，无法比对先前 findings；本次按现状全文复审。

### 复审背景

本系统在 2026-11-03 通过评审后有两处改动：补写 Formulas 章节、在输入 GDD 中定名上下文切换接口（`request_context(ctx, clear_stack := true)`）。本次针对改动后的现状重新全文评审，而非增量评审。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。
Formulas 对本系统是**必需**的——它定义了单帧生效与 0.2 秒过渡上限两条数值规则，按 standard 层不能作为 advisory。
Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs。Game Feel 一节实质覆盖了 Player Fantasy 的内容，建议加一行指向而非重复。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 数据配置 | 上游（硬） | `design/gdd/data-config.md` | ✓ 存在，第 181 行回指本系统 |
| 输入 | 上游（硬） | `design/gdd/input.md` | ✓ 存在，第 171 行回指本系统 |
| 界面流程 | 下游（硬） | `design/gdd/ui-flow.md` | ✓ 存在，第 13/151 行回指本系统 |
| 餐局流程 | 下游（硬） | `design/gdd/run-flow.md` | ✓ 存在，第 13/23 行回指本系统 |
| 首局引导 | 下游（软） | `design/gdd/first-run-onboarding.md` | ✓ 存在，第 13/121 行回指本系统 |
| 音效反馈 | 下游（软） | `design/gdd/audio-feedback.md` | ✓ 存在，第 13/108 行回指本系统 |

双向一致性成立，无断链。存档被正确标记为「不属于依赖」而非遗漏。

### Registry Values

`design/registry/entities.yaml` 有 78 条命名条目，**没有任何条目由本系统拥有或引用本系统**（`source:` / `referenced_by:` 均无 `scene-flow.md`）。按 Phase 3 要求，用全部 78 个条目名（`_` 展开为 `[_ ]`、大小写不敏感）对本 GDD 全文扫描：**命中 0 行**，不存在数值复述冲突。

本 GDD 用到的 `Ready` / `Failed` 是 `数据配置` 的加载状态枚举（`data-config.md` 第 65–68 行），与 `save-system` 拥有的 `load_outcome`（`Fresh / Loaded / Migrated / Recovered / Blocked`）是两套不同的值域，未混淆。支柱编号经 `game-concept.md` 核对：支柱 2「没有白吃的菜」、支柱 3「时间在走，手不用快」、支柱 4「数字看得见地涨」——引用全部正确。

### Findings

- [BLOCKING] Formulas §状态生效延迟：`FRAME_TIME` 的 Source 标为 `project.yaml`，但 `project.yaml`（66 行）没有任何帧率键——已核对全部顶层键（schema_version / framework / project / modes / engine / specialists / naming / commands），不存在 `fps` / `max_fps` / `target_fps` / `frame_rate`。该公式的唯一外部输入无法从声明来源取值；若帧率不设上限，`1 / 目标帧率` 本身也无定义。需二选一：改为真实来源（`Engine.max_fps` 常量、`Engine.get_frames_per_second()`，或直接写死 60 Hz 常量并说明理由），或在 `project.yaml` 真正增加该键。Formulas 对本系统是必需章节，其两条公式之一目前无法求值，故阻塞。
- [BLOCKING] Cross-References 表：`界面流程`、`餐局流程`、`首局引导`、`音效反馈` 四行全部标为「未写」，而同一份文档的 Dependencies 表对这四个系统标为「GDD 已写（design/gdd/…）」，且四个文件均存在于磁盘。同一文档对同一事实给出两个相反陈述——这是评审后其它 GDD 陆续落地造成的陈旧内容，需按 Dependencies 表更正为实际路径。
- [BLOCKING] Detailed Design 规则 2 / Acceptance Criteria 29：幂等空操作（目标状态 == 当前状态且不带 payload）的**返回值未定义**。规则 2 规定 `request_transition` 返回 bool 且「调用方拿到 `false` 即知 payload 未被交付」，但空操作既不是「被接受」（未发生切换）也不是「被拒绝」（明确不记警告），而 AC 29 只覆盖被拒绝的情形。这不是纯理论缺口：同一帧内重复的 `Paused → Run`（暂停菜单「继续」被连点/连发）会落到这条路径，而 `ui-flow.md` 规则 8 规定「返回 `false` 时显示可见原因」。需明确规定空操作返回 `true` 或 `false`，并在 AC 中补一条。
- [RECOMMENDED] Open Questions：第 1 行（结算屏是否提升为独立顶层状态）已由 `ui-flow.md` 规则 3 落定——`Menu → MainMenu 或 ReceiptScreen`，不新增顶层状态；第 2 行（`RunConfig` / `RunResult` schema 归属）已由 `run-flow.md`「RunConfig 与 RunResult 的归属」一节明确回答（`RunConfig` 由界面流程组装，`RunResult` 由结算与入场费拥有）。两行应标记为已解决并给出链接，否则下游读到会以为仍未定。
- [RECOMMENDED] Formulas §过渡总时长（跨文档冲突）：本 GDD 两处声明 `FADE_OUT` / `FADE_IN` 的具体时长「由 `/ux-design` 的交互规格拥有」，而 `ui-flow.md` Open Question 2 写的是「由场景与流程管理拥有」。两份 GDD 对同一组值的所有者给出相反答案——需要定一个所有者，另一份同步。本 GDD 内部是一致的，冲突源在 `ui-flow.md`，故不阻塞本评审，但必须在实现前闭合。
- [RECOMMENDED] Edge Cases「同一帧内 `ui_pause` 与结算回传同时到达」：本条写「餐局流程依据 `request_transition` 返回 `false` 重新发起」，读起来像可以立即重试；但 `Paused` 下的 `Run → Menu` 并不在合法转移表内，立即重试会永远失败。`run-flow.md` 规则 14 已补足正确语义（「保留该 `RunResult`，在回到 `Run` 顶层状态后重发；暂停期间等待，不请求」），建议本条加一句状态约束或直接指向 run-flow 规则 14。
- [RECOMMENDED] Detailed Design 调用写法不统一：规则 2 定义的是 `request_transition(to, payload)`（只有目标状态，合法性按 `(current, to)` 校验），但正文与 Cross-References 多处写成 `request_transition(Menu → Run, RunConfig)`。建议统一为单一写法并在一处给出完整签名，避免实现者误以为需要传 `from`。
- [RECOMMENDED] Acceptance Criteria 32：一条判据内含两个独立断言（总时长 ≤ 0.20 s；关闭过渡时两者为 0），且前提「`/ux-design` 给出 FADE 值」依赖尚不存在的文档。建议拆成两条，并把前提改为「给定任意满足 0 ≤ FADE ≤ 0.10 的取值」使其今天就可测。
- [RECOMMENDED] Acceptance Criteria 9：「画面与暂停前一致」无法独立判定 pass/fail。建议改为可观测形式，例如「恢复后 HUD 节点实例未卸载/重建（实例 id 不变），且显示值与暂停帧相同」。
- [NICE-TO-HAVE] Tuning Knobs（standard 层 advisory）：`TRANSITION_CAP = 0.20` 是唯一常量型可调值，补一行安全区间与影响面即可满足 advisory 标准。
- [NICE-TO-HAVE] Formulas：`TRANSITION_CAP` 列在 Variables 表中却不出现在表达式里（`transition_total = FADE_OUT + FADE_IN` 已经由 0–0.10 的取值范围保证 ≤ 0.20）。属冗余但无害，可移到「约束」行下避免读者困惑。
- [NICE-TO-HAVE] Edge Cases 末两条（「当前已在 `Quit`」「下游试图直接改状态」）更接近不变量而非边界情形；放在 Edge Cases 可接受，无需改动。

### 已核对且无问题的跨文档接口（记录以免后续重复检查）

- `request_context(context, clear_stack := true) -> bool`、`context_changed(from, to)`：与 `input.md` 规则 12 及第 51/55 行完全一致，接口定名改动已正确落地。
- 五个默认输入上下文（Blocked / Menu / Menu / Run / Paused）：与 `input.md` 第 61–64 行的上下文能力表一致。
- `ui_pause` 由本系统消费、`ui_accept` 由界面流程消费：与 `ui-flow.md` 规则 15/27、`input.md` 规则 2 三方一致。
- `RunConfig` / `RunResult` / `init_failed` / `abandoned` 四个 payload：`ui-flow.md` 规则 8/9/16/19 与 `run-flow.md` 规则 4/13/14/16 均按本 GDD 的定义使用，无字段漂移。
- `Error → Boot`（仅 debug）、`Boot → Quit` 无条件合法：与 `ui-flow.md` 规则 17/18 一致。
- 首局引导在 `Run` 内 `push_context(Blocked)` 且不改顶层状态：与 `first-run-onboarding.md` 规则 4/16/17 一致。
- 顶层状态切换被 `audio-feedback.md` 用作 `flow.*` cue 的触发源：一致。

### Suggested systems-index status

**Needs Revision** —— 三个 blocking 项修复并复审通过前不应回到 Approved；修复完成、等待复审时改为 **In Review**。（本次评审按并行约束未改动 `design/gdd/systems-index.md`，状态由编排会话统一更新。）

### Verdict: NEEDS REVISION

---
Reviewed-Content-Hash: design/gdd/scene-flow.md ac1d04a52a78a3027e95b4c81cde898e1b5ead06
Reviewed-Content-Hash: design/registry/entities.yaml cdef3f7a3f4a270848207a53e3eef821342c6294
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 1e8e07619c6eb3c027dcb348bcf848c73f2b16b5
Reviewed-Content-Hash: design/gdd/input.md c96715b12512d1a093a10f6cf2e32030740be509
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 8d36e76a8f8024efa498c83cf83c804b856e77ba
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 778f0a836330a6410aee8812fa0801f9dd3cc4f3
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 1 | Recommended: 6
Summary: 上一轮三个 blocking 的字面修复全部落地且正确：`FRAME_TIME` 改为 60 Hz 设计常量并在 Source 列显式声明不读 `project.yaml`；Cross-References 四行改为真实 GDD 路径，与 Dependencies 表一致；幂等空操作的返回值已定义（空 payload 返回 `true`，带 payload 返回 `false`）并写进 AC 29。状态机本体仍无结构性问题。唯一 blocking 是这次修复留下的一处作用域冲突：幂等空操作规则未排除终态 `Quit` 与 `Boot` 待校验期，与「`Quit` 中拒绝任何请求」「`Boot` 待校验期除 `Quit` 外全拒绝」两条规则矛盾，AC 17 与 AC 19 会给出相反结果。
Prior verdict resolved: No —— 3 个 blocking 的字面问题已修复，但返回值的修复与其他状态规则产生了一处新的作用域冲突（见 blocking 1）。

### 上一轮 blocking 复核（逐条）

| 上轮 blocking | 现状 | 结论 |
|---|---|---|
| Formulas `FRAME_TIME` 来源悬空 | 第 101 行 Source 改为 `constant`，值 `0.0167（60 Hz）`，并注明「不读 `project.yaml`」；全文已无其它 `project.yaml` 依赖 | ✓ 已解决 |
| Cross-References 四行标「未写」 | 第 212–217 行全部改为实际路径 `design/gdd/*.md`，与 Dependencies 表一致；全文已无「未写」字样 | ✓ 已解决 |
| 幂等空操作返回值未定义 | 规则 2（第 28 行）与 AC 29（第 255 行）均已定义：`to == current` 且空 payload → `true`，状态不变、不发信号、不记日志；带 payload → `false` | ✓ 已解决（但引入作用域冲突，见下方 blocking） |

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统仍为必需（两条数值规则：单帧生效、过渡 ≤ 0.2 s）。
Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs（同上一轮，Game Feel 一节实质覆盖 Player Fantasy）。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 数据配置 | 上游（硬） | `design/gdd/data-config.md` | ✓ 存在，第 189 行回指本系统 |
| 输入 | 上游（硬） | `design/gdd/input.md` | ✓ 存在，第 51/73/171 行回指本系统 |
| 界面流程 | 下游（硬） | `design/gdd/ui-flow.md` | ✓ 存在，第 13/151 行回指本系统 |
| 餐局流程 | 下游（硬） | `design/gdd/run-flow.md` | ✓ 存在，第 13/23 行回指本系统 |
| 首局引导 | 下游（软） | `design/gdd/first-run-onboarding.md` | ✓ 存在，第 121/361 行回指本系统 |
| 音效反馈 | 下游（软） | `design/gdd/audio-feedback.md` | ✓ 存在，第 13/108 行回指本系统 |

双向一致性成立，无断链。存档仍被正确标记为「不属于依赖」而非遗漏。

### Registry Values

`design/registry/entities.yaml` 现为 78 条命名条目；用全部条目名（`_` 展开为 `[_ ]`、大小写不敏感）对本 GDD 全文扫描：**命中 0 行**。没有任何条目由本系统拥有或在 `referenced_by:` 中引用本系统。无数值复述冲突。

### Findings

- [BLOCKING] 规则 2（第 28 行）与 Edge Case 第 150 行 / AC 19（第 245 行）作用域冲突：幂等空操作规则写作「若 `to == current` 且 payload 为空，返回 `true`」，未限定状态；而第 150 行「当前已在 `Quit`：拒绝任何后续请求」与 AC 19「处于 `Quit`，收到任何切换请求 → 全部拒绝」都把同状态请求算作应拒绝。于 `Quit` 中调用 `request_transition(Quit)`（无 payload）时，规则 2 与 AC 17 要求返回 `true`，AC 19 要求返回 `false`——两条判据互斥；`Boot` 待校验期的同状态请求（第 66/138 行「除 `Quit` 外全部拒绝」）同理。需给幂等规则加作用域例外（推荐：终态 `Quit` 与 `Boot` 校验未返回期间不适用幂等空操作，任何请求一律返回 `false`），并把 AC 17 的前提限定为非 `Quit`、`Boot` 已就绪，使 AC 17 与 AC 19 不再重叠。
- [RECOMMENDED] Open Questions 第 1 行（结算屏是否提升为独立顶层状态）与第 2 行（`RunConfig` / `RunResult` schema 归属）仍未标记已解决；前者已由 `ui-flow.md` 规则 3（`Menu → MainMenu 或 ReceiptScreen`，不新增顶层状态）回答，后者已由 `run-flow.md`「RunConfig 与 RunResult 的归属」一节回答（`RunConfig` 由界面流程组装，`RunResult` 由结算与入场费拥有）。
- [RECOMMENDED] Formulas §过渡总时长（跨文档冲突，仍未闭合）：本 GDD 第 88/118/183 行声明 `FADE_OUT` / `FADE_IN` 由 `/ux-design` 交互规格拥有；`ui-flow.md` Open Question 2（第 617 行）写「过渡时长 FADE_OUT / FADE_IN 由场景与流程管理拥有」。本次 `ui-flow.md` 未变（receipt UNCHANGED），冲突原样存在；两份 GDD 必须定一个所有者。
- [RECOMMENDED] Edge Cases 第 134 行「餐局流程依据 `request_transition` 返回 `false` 重新发起」仍读作可立即重试，但 `Paused` 下的 `Run → Menu` 不在合法转移表内；`run-flow.md` 规则 14 已写明「保留该 `RunResult`，在回到 `Run` 顶层状态后重发；暂停期间等待，不请求」。建议本条直接指向该规则。
- [RECOMMENDED] 调用写法仍不统一：规则 2（第 28 行）定义 `request_transition(to, payload)`（按 `(current, to)` 校验），正文与 Cross-References 多处写作 `request_transition(Menu → Run, RunConfig)`（第 76/106 行等）。建议统一并在一处给出完整签名。
- [RECOMMENDED] AC 32（第 258 行）仍把两个独立断言（总时长 ≤ 0.20 s；关闭过渡时为 0）塞进一条，且前提「`/ux-design` 给出 FADE 值」依赖尚不存在的文档；建议拆条并将前提改为「给定任意满足 0 ≤ FADE ≤ 0.10 的取值」。
- [RECOMMENDED] AC 9（第 235 行）「画面与暂停前一致」仍不可独立判定 pass/fail；建议改为可观测形式，如「恢复后 HUD 根节点实例 id 不变，且显示值与暂停帧相同」。
- [NICE-TO-HAVE] Tuning Knobs 缺失（standard 层 advisory）：`TRANSITION_CAP = 0.20` 是唯一常量型可调值，补一行安全区间与影响面即可。
- [NICE-TO-HAVE] `TRANSITION_CAP` 仍列在 Variables 表中却不出现在表达式（`transition_total = FADE_OUT + FADE_IN`），可移到「约束」下避免读者困惑。

### 已核对且无问题的跨文档接口

- `request_context(context, clear_stack := true, allow := []) -> bool`、`context_changed(from, to)`：与 `input.md` 第 51/55/171 行一致；本 GDD 只调用 `request_context(default_ctx, clear_stack := true)`，未使用 `allow`，兼容。
- 五个默认输入上下文（Blocked / Menu / Menu / Run / Paused）：与 `input.md` 上下文能力表一致。
- `ui_pause` 由本系统消费、`ui_accept` 由界面流程消费：与 `ui-flow.md` 规则 15/27、`input.md` 规则 2 三方一致。
- `RunConfig` / `RunResult` / `init_failed` / `abandoned` 四个 payload：`ui-flow.md` 规则 5/8/9/16 与 `run-flow.md` 规则 4/13/14/16 均按本 GDD 的定义使用，无字段漂移。
- 首局引导在 `Run` 内 `push_context(Blocked)` 且不改顶层状态：与 `first-run-onboarding.md` 一致。
- 顶层切换事件作为 `audio-feedback.md` 的 `flow.*` cue 触发源：一致。
- 数据配置 `Ready` / `Failed` 与存档 `load_outcome`（`Fresh / Loaded / Migrated / Recovered / Blocked`）仍是两套值域，未混淆。

### Suggested systems-index status

**Needs Revision** —— 剩余 1 个 blocking 修复前不应回到 Approved；该 blocking 是两行规则的作用域限定，修完即可再次复审。（本次按并行约束未改动 `design/gdd/systems-index.md`，状态由编排会话统一更新。）

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/scene-flow.md c19ee4c2a48183476032f3ebbe2ceb9e43588324
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 001e4ec43326ff7ab46c54c4f626664bd9fc7e23
Reviewed-Content-Hash: design/gdd/input.md 776c41de0eb3edd497ae2ace8c78ba80192fc78f
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 2ef73066ae97ea1591a4ebaaeba6d4a97dcf4955
Reviewed-Content-Hash: design/gdd/audio-feedback.md 8ad91f894b044ae735d9c55d13606f8e17453900
Reviewed-Content-Hash: design/gdd/game-concept.md 37c487b5ca4f8898d9cef22c790b87625027b123

## Review — 2026-11-04 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 1 | Recommended: 6
Summary: 第三轮（终审）复审。上一轮唯一 blocking 的字面冲突已修复：规则 2 增加了作用域例外，AC 17 补了「非 `Quit` 且 `Boot` 已就绪」前提，AC 29 同步限定，`Quit` 中的同状态请求不再同时得到两种答案。但作用域例外的措辞写成了「这两段时间内任何请求一律返回 `false`」，把 `Boot` 待校验期唯一合法的 `Boot → Quit` 也一并判为拒绝，与规则 10、第 66 行的明文例外、AC 6、AC 13 直接矛盾；AC 17/29 的第二个 GIVEN 同样未限定为同状态请求。这是同一类返回值作用域问题的最后一处，修法是纯措辞限定，无结构性返工。
Prior verdict resolved: No —— 上一轮针对「`Quit` / `Boot` 待校验期的同状态请求」的冲突已解决，但作用域例外过度扩张，在 `Boot` 待校验期与 `Boot → Quit` 的合法性产生新的直接矛盾（见 blocking 1）。

### 上一轮 blocking 复核（逐条）

| 上轮 blocking | 现状 | 结论 |
|---|---|---|
| 幂等空操作未限定状态，与 `Quit` / `Boot` 待校验期的「一律拒绝」冲突 | 规则 2（第 28 行）已加作用域例外；AC 17（第 243 行）已加「当前状态不是 `Quit` 且 `Boot` 已完成校验」前提并补返回值；AC 29（第 255 行）已同步限定。`Quit` 中的同状态请求现在一致返回 `false` | ✓ 已解决（但作用域措辞过宽，见 blocking） |

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统仍为必需（两条数值规则：单帧生效、过渡 ≤ 0.2 s）。
Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs（Game Feel 一节实质覆盖 Player Fantasy）。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 数据配置 | 上游（硬） | `design/gdd/data-config.md` | ✓ 存在，第 190 行回指本系统 |
| 输入 | 上游（硬） | `design/gdd/input.md` | ✓ 存在，第 51/73/171 行回指本系统 |
| 界面流程 | 下游（硬） | `design/gdd/ui-flow.md` | ✓ 存在，第 13/151 行回指本系统 |
| 餐局流程 | 下游（硬） | `design/gdd/run-flow.md` | ✓ 存在，第 13/23 行回指本系统 |
| 首局引导 | 下游（软） | `design/gdd/first-run-onboarding.md` | ✓ 存在，第 121/361 行回指本系统 |
| 音效反馈 | 下游（软） | `design/gdd/audio-feedback.md` | ✓ 存在，第 13/108 行回指本系统 |

双向一致性成立，无断链。存档仍被正确标记为「不属于依赖」而非遗漏。

### Registry Values

`design/registry/entities.yaml` 共 78 条命名条目；用全部条目名（`_` 展开为 `[_ ]`、大小写不敏感）对本 GDD 全文扫描：**命中 0 行**。没有任何条目由本系统拥有或在 `referenced_by:` 中引用本系统。无数值复述冲突。

### 本轮同步复核的跨文档改动

本轮 `data-config.md` 与 `input.md` 在上次评审后有改动（回执为 `CHANGED`），逐项核对对本系统的影响：

- `data-config.md`：新增 `|D_bal| = 1` 校验与第 30 条 AC。`Ready` / `Failed` 两个状态名与第 190 行的回指未变，本系统读取的接口不受影响。
- `input.md`：`Blocked` 上下文改为「`ui_pause` 始终允许，`allow` 为追加而非替换」。本系统只调用 `request_context(default_ctx, clear_stack := true)`，不使用 `allow`；签名 `request_context(context, clear_stack := true, allow := []) -> bool` 与第 51/73 行一致，兼容。本系统在 `Boot` / `Menu` / `Error` / `Paused` 均不处理 `ui_pause`（规则 7 只在 `Run`），无冲突。

### Findings

- [BLOCKING] 规则 2 作用域（第 28 行）与 AC 17（第 243 行）/ AC 29（第 255 行）对 `Boot` 待校验期的覆盖过宽：作用域句写「这两段时间内**任何请求**一律返回 `false`」，AC 17 第二个 GIVEN 写「当前为 `Quit` 或 `Boot` 待校验期 → 返回 `false`」，AC 29 第二个 GIVEN 把「当前为 `Quit` / `Boot` 待校验期」与「同状态带 payload」并列为 `false`——三者都没有把「目标状态等于当前状态」作为 `Boot` 分支的条件。于是 `Boot` 待校验期的 `request_transition(Quit)` 按规则 2/AC 17/AC 29 返回 `false`（拒绝），但第 66 行明文规定「从 `Boot` 发起但校验结果尚未返回时的所有请求——**唯独 `Boot → Quit` 例外**，退出在启动阶段始终合法」，规则 10 要求该退出「由应用入口转成一次 `request_transition(Quit)`」，AC 6 写「退出请求仍直接进入 `Quit`」，AC 13 写 `Boot` 中退出「无确认直接进入 `Quit`」。`request_transition` 的 `true` / `false` 就是接受 / 拒绝（规则 2、AC 31），同一请求因此在 AC 13 下必须被接受、在规则 2/AC 17/AC 29 下必须被拒绝——两条判据互斥。另有两处同源口径未同步：第 68 行「同状态请求的判定」与 Edge Case 第 137 行仍把同状态无 payload 写成无条件的「幂等空操作」，未带 `Quit` / `Boot` 待校验期例外。**修法**（纯措辞）：① 规则 2 作用域改为「`Quit` 终态：任何请求返回 `false`；`Boot` 待校验期：同状态请求返回 `false`，其余请求按第 66 行处理——`Boot → Quit` 仍合法并返回 `true`」；② AC 17 第二个 GIVEN 补「请求目标等于当前状态」，并把 `Boot → Quit` 明确排除；③ AC 29 第二个 GIVEN 同样把 `Boot` 待校验期限定为同状态请求（`Quit` 可保留「任何请求」）；④ 第 68 行与 Edge Case 第 137 行引用该作用域例外。
- [RECOMMENDED] Open Questions 第 1 行（结算屏是否提升为独立顶层状态）与第 2 行（`RunConfig` / `RunResult` schema 归属）仍未标记已解决。前者已由 `ui-flow.md` 规则 3（`Menu → MainMenu 或 ReceiptScreen`，不新增顶层状态）回答，后者已由 `run-flow.md`「RunConfig 与 RunResult 的归属」一节回答（`RunConfig` 由界面流程组装，`RunResult` 由结算与入场费拥有）。
- [RECOMMENDED] Formulas §过渡总时长（跨文档冲突，仍未闭合）：本 GDD 第 88/118/183 行声明 `FADE_OUT` / `FADE_IN` 由 `/ux-design` 交互规格拥有；`ui-flow.md` Open Question 2（第 617 行）写「过渡时长 `FADE_OUT` / `FADE_IN` 由场景与流程管理拥有」。本轮 `ui-flow.md` 未变（回执 `UNCHANGED`），冲突原样存在；两份 GDD 必须定一个所有者。
- [RECOMMENDED] Edge Cases 第 134 行「餐局流程依据 `request_transition` 返回 `false` 重新发起」仍读作可立即重试，但 `Paused` 下的 `Run → Menu` 不在合法转移表内；`run-flow.md` 规则 14 已写明「保留该 `RunResult`，在回到 `Run` 顶层状态后重发；暂停期间等待，不请求」。建议本条直接指向该规则。
- [RECOMMENDED] 调用写法仍不统一：规则 2（第 28 行）定义 `request_transition(to, payload)`（按 `(current, to)` 校验），正文与 Cross-References 多处写作 `request_transition(Menu → Run, RunConfig)`（第 76/106 行等）。建议统一并在一处给出完整签名。
- [RECOMMENDED] AC 32（第 258 行）仍把两个独立断言（总时长 ≤ 0.20 s；关闭过渡时为 0）塞进一条，且前提「`/ux-design` 给出 FADE 值」依赖尚不存在的文档；建议拆条并将前提改为「给定任意满足 0 ≤ FADE ≤ 0.10 的取值」。
- [RECOMMENDED] AC 9（第 235 行）「画面与暂停前一致」仍不可独立判定 pass/fail；建议改为可观测形式，如「恢复后 HUD 根节点实例 id 不变，且显示值与暂停帧相同」。
- [NICE-TO-HAVE] Tuning Knobs 缺失（standard 层 advisory）：`TRANSITION_CAP = 0.20` 是唯一常量型可调值，补一行安全区间与影响面即可。
- [NICE-TO-HAVE] `TRANSITION_CAP` 仍列在 Variables 表中却不出现在表达式（`transition_total = FADE_OUT + FADE_IN`），可移到「约束」下避免读者困惑。
- [NICE-TO-HAVE] 规则 7 可补一句「`ui_pause` 仅在 `Run` / `Paused` 有意义」：`input.md` 本轮改为 `Blocked` 始终放行 `ui_pause`，`Boot`（Blocked）期间的暂停按键会到达本系统；规则 7 已限定「`Run` 中收到」，语义无冲突，但写明可免实现者误解。

### 已核对且无问题的跨文档接口

- `request_context(context, clear_stack := true, allow := []) -> bool`、`context_changed(from, to)`：与 `input.md` 第 51/55/171 行一致；本 GDD 只调用 `request_context(default_ctx, clear_stack := true)`，未使用 `allow`，兼容本轮 `allow` 追加语义的改动。
- 五个默认输入上下文（Blocked / Menu / Menu / Run / Paused）：与 `input.md` 上下文能力表一致。
- `ui_pause` 由本系统消费、`ui_accept` 由界面流程消费：与 `ui-flow.md` 规则 15/27、`input.md` 规则 2 三方一致。
- `RunConfig` / `RunResult` / `init_failed` / `abandoned` 四个 payload：`ui-flow.md` 与 `run-flow.md` 规则 4/13/14/16 均按本 GDD 的定义使用，无字段漂移。
- 首局引导在 `Run` 内 `push_context(Blocked)` 且不改顶层状态：与 `first-run-onboarding.md` 一致。
- 顶层切换事件作为 `audio-feedback.md` 的 `flow.*` cue 触发源：一致。
- 数据配置 `Ready` / `Failed` 与存档 `load_outcome`（`Fresh / Loaded / Migrated / Recovered / Blocked`）仍是两套值域，未混淆。

### Suggested systems-index status

**Needs Revision** —— 当前索引记为 **In Review**（`systems-index.md` 第 59 行）；剩余 1 个 blocking 是作用域措辞的限定，修完即可再次复审并回到 Approved。本次按并行约束未改动 `design/gdd/systems-index.md`，状态由编排会话统一更新。

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/scene-flow.md a2c3cc81155d51d9fae36ecc8ef27bfc1d689a59
Reviewed-Content-Hash: design/registry/entities.yaml d726500e94755343517565ffa65e786a7629c167
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 7dc918c23ab271495049b5fd33bddd08a5dc6c55
Reviewed-Content-Hash: design/gdd/input.md ee9ea6022f8e289d9e00ef8f9eabb181b0c5349f
Reviewed-Content-Hash: design/gdd/ui-flow.md 4eb7451abe5537272d446ec90150266f98e1588d
Reviewed-Content-Hash: design/gdd/run-flow.md 02d37ce06fa293e6b76cb22277a68b705e413448

## Review — 2026-11-04 — Verdict: APPROVED
Review mode: lean · Workflow: standard
Scope signal: M
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 0 | Recommended: 7
Summary: 第四轮复审。上一轮唯一 blocking 已按建议的最小改法修复，且没有过度修正：规则 2 的 `Boot` 分支限定为「同状态请求返回 `false`」，并明写「`Boot → Quit` 仍合法并返回 `true`」；AC 17 增加第三个 GIVEN、AC 29 的 `Boot` 分支同样限定为同状态；第 69 行与 Edge Case 第 138 行都回指规则 2。`Quit` 中同状态请求、`Boot` 待校验期同状态请求、`Boot → Quit` 三个场景的返回值在规则与 AC 之间现已一致，先前四处的直接矛盾全部闭合。本轮新增的规则 15 `state_changed(from, to)` 已与 `audio-feedback.md` 采用同一信号名与签名，未引入新的跨文档冲突。剩余 7 项 recommended 全部为此前遗留且不阻塞；其中最高优先是把规则 2 / AC 17 里指向「规则 6」的两处引用改正——`Boot → Quit` 例外实际写在「非法转移一律拒绝」段（第 67 行）与规则 10，但行为在第 67 行与 AC 6 已完整给出，故不判阻塞。
Prior verdict resolved: Yes —— 上一轮 blocking（作用域例外把 `Boot → Quit` 一并判为 `false`）已按建议的纯措辞限定修复，四处矛盾闭合。

### 上一轮 blocking 复核（逐条）

| 上轮 blocking | 现状 | 结论 |
|---|---|---|
| 作用域例外写成「这两段时间内任何请求一律 `false`」，把 `Boot → Quit` 也判为拒绝，与第 67 行、规则 10、AC 6、AC 13 矛盾；AC 17/29 未把 `Boot` 分支限定为同状态 | 规则 2（第 28 行）改为「`Quit` 终态：任何请求返回 `false`；`Boot` 尚未完成校验期间：同状态请求返回 `false`，其余请求按规则 6 处理——`Boot → Quit` 仍合法并返回 `true`」；AC 17（第 244 行）补第三个 GIVEN「`Boot` 待校验期请求 `Quit` → `true`」；AC 29（第 256 行）把 `Boot` 分支限定为「目标等于当前状态」；第 69 行与 Edge Case 第 138 行均回指规则 2 | ✓ 已解决 |

逐点核对四个返回值场景（规则 2 / AC 17 / AC 29 / 第 67 行 / AC 6 / AC 13 / AC 19 的口径）：

- `Quit` 中任何请求 → `false`：规则 2、AC 17、AC 19 一致 ✓
- `Boot` 待校验期同状态请求 → `false`：规则 2、AC 17、AC 29 一致 ✓
- `Boot` 待校验期非 `Quit` 的其他请求 → 拒绝：第 67 行、AC 6 一致 ✓（规则 2 的引用目标写错，见 findings 第 1 条；行为本身无歧义）
- `Boot` 待校验期请求 `Quit` → `true`：规则 2、AC 17、第 67 行、规则 10、AC 6、AC 13 一致 ✓

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 对本系统仍为必需（两条数值规则：单帧生效、过渡 ≤ 0.2 s）。
Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs（Game Feel 一节实质覆盖 Player Fantasy）。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 数据配置 | 上游（硬） | `design/gdd/data-config.md` | ✓ 存在，第 190 行回指本系统 |
| 输入 | 上游（硬） | `design/gdd/input.md` | ✓ 存在，第 51/56/74/172 行回指本系统 |
| 界面流程 | 下游（硬） | `design/gdd/ui-flow.md` | ✓ 存在，第 13/47/57 行回指本系统 |
| 餐局流程 | 下游（硬） | `design/gdd/run-flow.md` | ✓ 存在，第 13/23/42 行回指本系统 |
| 首局引导 | 下游（软） | `design/gdd/first-run-onboarding.md` | ✓ 存在，第 121/361 行回指本系统 |
| 音效反馈 | 下游（软） | `design/gdd/audio-feedback.md` | ✓ 存在，第 111/355 行回指本系统 |

双向一致性成立，无断链。存档仍被正确标记为「不属于依赖」而非遗漏。

### Registry Values

`design/registry/entities.yaml` 共 78 条命名条目；没有任何条目的 `source:` 或 `referenced_by:` 指向本系统或本 GDD（`grep scene-flow` 命中 0 行）。本系统只拥有状态名与转移语义，不含经济数值。上次评审之后 registry 的唯一改动是 `save_load_decision` 的 `output_range` 增加 `ReadOnly` 并细化迁移失败分支；本 GDD 不复述该值域，不构成冲突。

### 本轮同步复核的跨文档改动（2ec41d5..be69169）

- `scene-flow.md` 自身：规则 2 作用域限定、AC 17/29 限定、第 69 行与 Edge Case 138 回指、新增规则 15 `state_changed(from, to)`。
- `audio-feedback.md`：第 355 行的依赖行改为「顶级状态转移…与 `state_changed(from, to)` 信号」——与本 GDD 规则 15 同名同参，**新信号无漂移**。其 cue 表（第 66–69 行）是 `sfx.flow.pause` / `flow.resume` / `flow.menu` / `flow.quit`，与本 GDD 依赖行第 170 行的措辞不完全对应（见 findings 第 8 条）。
- `ui-flow.md`：第 47 行规则 3 确认 `Menu → MainMenu 或 ReceiptScreen`（不新增顶层状态）；第 93 行确认过渡「由 `场景与流程管理` 定义上限」；第 57 行规则 8 仍以 `request_transition(Menu → Run, RunConfig)` 发起。均与本 GDD 规则 1/5/12 一致。
- `run-flow.md`：规则 14（第 42 行）「保留该 `RunResult`，在回到 `Run` 顶层状态后重发」仍是 Edge Case 第 135 行所需的语义；规则 20 引用本 GDD 规则 12，正确。
- `first-run-onboarding.md`：第 79 行「`Paused` 清空临时栈 / 恢复 `Run` 时重新压入白名单」与本 GDD 规则 3 的 `clear_stack := true` 一致。
- `input.md`：`request_context(context, clear_stack := true, allow := []) -> bool` 签名未变；`Blocked` 仍为「`ui_pause` 始终允许」，本系统不使用 `allow`；新增的 `consume_action(action)` 属首局引导，本系统不涉及。
- `data-config.md`：`Ready` / `Failed` 与第 190 行回指未变；本轮改动只涉及「纪录与里程碑」的依赖行，与本系统无关。

### Findings

- [RECOMMENDED] **规则 2（第 28 行）与 AC 17（第 244 行）把 `Boot → Quit` 例外引用成「规则 6」**，但规则 6 是「一局的离开（`Run → Menu` 携带 `RunResult`）」；该例外实际写在「非法转移一律拒绝」段（第 67 行）与规则 10「退出」。同句「其余请求按规则 6 处理」对 `Boot` 待校验期的非 `Quit` 请求因此指向空处。行为本身在第 67 行与 AC 6 已完整给出（一律拒绝），六处口径不矛盾，故不判阻塞；建议两处改为「见第 67 行」，并把该分句写成「其余请求一律拒绝」。
- [RECOMMENDED] Open Questions 第 1 行（结算屏是否提升为独立顶层状态）与第 2 行（`RunConfig` / `RunResult` schema 归属）仍未标记已解决。前者已由 `ui-flow.md` 规则 3（第 47 行，`Menu → MainMenu 或 ReceiptScreen`，不新增顶层状态）回答；后者已由 `run-flow.md`「RunConfig 与 RunResult 的归属」一节（第 53 行）回答。建议移入「已解决」或删除。
- [RECOMMENDED] Formulas §过渡总时长（跨文档冲突，第 4 轮仍未闭合）：本 GDD 第 89/119/120/184 行声明 `FADE_OUT` / `FADE_IN` 的时长由 `/ux-design` 交互规格拥有；`ui-flow.md` Open Question 2（第 623 行）仍写「过渡时长 `FADE_OUT` / `FADE_IN` 由场景与流程管理拥有」。本轮 `ui-flow.md` 第 93 行已改口「由 `场景与流程管理` 定义上限」，与 `TRANSITION_CAP` 一致，冲突范围收窄为「时长归属」；仍需两份 GDD 指定唯一所有者。
- [RECOMMENDED] Edge Cases 第 135 行「餐局流程依据 `request_transition` 返回 `false` 重新发起」仍读作可立即重试，但 `Paused` 下的 `Run → Menu` 不在合法转移表内；`run-flow.md` 规则 14（第 42 行）已写明「保留该 `RunResult`，在回到 `Run` 顶层状态后重发；暂停期间等待，不请求」。建议本条直接指向该规则。
- [RECOMMENDED] 调用写法仍不统一：规则 2（第 28 行）定义 `request_transition(to, payload)`（按 `(current, to)` 校验），正文与 Formulas 示例写作 `request_transition(Menu → Run, RunConfig)`（第 77/107 行），Cross-References（第 215 行）又简写为「发起 `Menu → Run`」。建议统一并在一处给出完整签名。
- [RECOMMENDED] AC 32（第 259 行）仍把两个独立断言（总时长 ≤ 0.20 s；关闭过渡时为 0）塞进一条，且前提「`/ux-design` 给出 FADE 值」依赖尚不存在的文档；建议拆条并将前提改为「给定任意满足 0 ≤ FADE ≤ 0.10 的取值」。
- [RECOMMENDED] AC 9（第 236 行）「画面与暂停前一致」仍不可独立判定 pass/fail；建议改为可观测形式，如「恢复后 HUD 根节点实例 id 不变，且显示值与暂停帧相同」。
- [NICE-TO-HAVE] 新增规则 15（第 41 行）的 `state_changed(from, to)` 已与 `audio-feedback.md` 第 355 行对齐；但本 GDD 的 Dependencies 表第 170 行仍写「订阅状态切换事件（进入 `Run`、结算、退出；具体点 `Paused` / `Resumed` / `Quit`）」——未写信号名，且 `audio-feedback.md` 的 cue 表（第 66–69 行）没有「进入 `Run`」cue，反有进入 `Menu` 的 `sfx.flow.menu`。建议该行直接写 `state_changed(from, to)` 并指向该 cue 表。
- [NICE-TO-HAVE] Tuning Knobs 缺失（standard 层 advisory）：`TRANSITION_CAP = 0.20` 是唯一常量型可调值，补一行安全区间与影响面即可。
- [NICE-TO-HAVE] `TRANSITION_CAP` 仍列在 Variables 表（第 121 行）却不出现在表达式（第 113 行 `transition_total = FADE_OUT + FADE_IN`），可移到「约束」下避免读者困惑。
- [NICE-TO-HAVE] 规则 7 可补一句「`ui_pause` 仅在 `Run` / `Paused` 有意义」：`input.md` 第 2 条本轮改为 `Blocked` 始终放行 `ui_pause`，`Boot`（Blocked）期间按键会到达本系统；规则 7 已限定「`Run` 中收到」，语义无冲突。

### 已核对且无问题的跨文档接口

- `request_context(context, clear_stack := true, allow := []) -> bool`、`context_changed(from, to)`：与 `input.md` 第 51/56/172 行一致；本 GDD 只调用 `request_context(default_ctx, clear_stack := true)`，未使用 `allow`。
- `state_changed(from, to)`：`audio-feedback.md` 第 355 行已采用同名同参信号；`sfx.flow.pause` / `flow.resume` 与「由 `(from, to)` 推导」一致。
- 五个默认输入上下文（Blocked / Menu / Menu / Run / Paused）：与 `input.md` 上下文能力表一致。
- `ui_pause` 由本系统消费、`ui_accept` 由界面流程消费：与 `ui-flow.md` 规则 15/27、`input.md` 第 2 条三方一致。
- `RunConfig` / `RunResult` / `init_failed` / `abandoned` 四个 payload：`ui-flow.md` 规则 8、`run-flow.md` 规则 4/14/16 均按本 GDD 的定义使用，无字段漂移。
- 首局引导在 `Run` 内 `push_context(Blocked)` 且不改顶层状态：与 `first-run-onboarding.md` 第 79/121 行一致。
- 数据配置 `Ready` / `Failed` 与存档 `load_outcome`（`Fresh / Loaded / Migrated / Recovered / ReadOnly / Blocked`）仍是两套值域，未混淆。

### 跨文档待办（不属于本 GDD）

`audio-feedback.md` 第 386 行仍留着待办「`scene-flow.md` 的下游表把音效反馈记为 `GDD 未写`，应改为…」；本 GDD 第 170 行早已是「GDD 已写（`design/gdd/audio-feedback.md`）」。该待办已过期，应由音效反馈的会话清理，不计入本 GDD 的 findings。

### Suggested systems-index status

**Approved** —— 当前索引记为 **In Review**（`systems-index.md` 第 59 行）。0 个 blocking，剩余 7 项 recommended 与 4 项 nice-to-have 均不阻塞。本次按并行约束未改动 `design/gdd/systems-index.md`，状态由编排会话统一更新。

### Verdict: APPROVED

---

Reviewed-Content-Hash: design/gdd/scene-flow.md 4efc24d198112aa612551f5b3e5bdeeac28cd52e
Reviewed-Content-Hash: design/registry/entities.yaml f8361d463dc6172ce3c22a462bccce6d2037d35c
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: design/gdd/data-config.md 0be338a35d8e5fa9ded4c219b29801021835938a
Reviewed-Content-Hash: design/gdd/input.md 9432d857ed18dc216dc5a038f81608ae9e10d975
Reviewed-Content-Hash: design/gdd/ui-flow.md 21482b6d00538b95af113995e613b8df29985a8a
Reviewed-Content-Hash: design/gdd/run-flow.md ed8848ea7c41823db26e22fb7d5df5f998fc0234
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 3f99e25a3d35bd4922c0717adc033569a50abafb
Reviewed-Content-Hash: design/gdd/audio-feedback.md f6ac7fbd915341f63761529b664028eec6b216c0

## Review — 2026-11-06 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，未派发专员 agent）
Blocking items: 2 | Recommended: 8
Summary: 全文 lean 复审（非增量）。上一份 APPROVED 之后加入了 `fault.phase`（规则 6 与转移表第 61 行），语义本身与 ADR-0007 D7、`run-flow.md` 规则 18、`ui-flow.md` 规则 19 完全一致，`phase` 作判据、`init_failed` 只表示"payload 无 `RunResult`"三处口径对得上；规则 8 的放弃路径与 `ui-flow.md` 规则 16/AC 22、ADR-0007 D3 的提交边界也一致；规则 12 的淡入淡出与 ADR-0003 D8 的"遮盖—揭示 ≤0.2 s、状态先变"实现严格对偶；全文没有引入 `Settling` 顶层状态。两个 blocking 都是这次改动留下的半传播问题：转移表没有 RUN 相位那一行（其余接口行与 AC 27 仍只写 `init_failed`），以及规则 3 把 `Blocked` 的压栈方扩到了界面流程——`input.md` 第 70 行明写"无第三方"、ADR-0005 第 228 行把调用点限定在首局引导。两处都是局部措辞/表格补齐，无结构性返工。
Prior verdict resolved: No —— 2026-11-04 的 APPROVED 绑定的是旧字节，其后对规则 6 与转移表的实质改动使该回执失效；本轮按当前字节全文复审，发现 2 个新的 blocking。

### 回执状态（为何不能沿用旧判定）

`review-receipts.sh check` 对本 GDD 报 `CHANGED`，同时对 `design/registry/entities.yaml`、`data-config.md`、`input.md`、`ui-flow.md`、`run-flow.md`、`first-run-onboarding.md`、`audio-feedback.md` 全部报 `CHANGED`，仅 `SKILL.md` 与 `game-concept.md` 为 `UNCHANGED`。上一份 APPROVED（2026-11-04）绑定 `scene-flow.md 4efc24d1…`，当前字节为 `07753acc…`——回执失效，且跨文档事实全部需要重新核对，因此执行全文复审而非增量。头部的 `Status` 仍写"Formulas 章节补写、输入上下文接口定名（2026-11-03）"、`Last Updated: 2026-11-03`，未反映 11-04 之后加入 `fault.phase` 的那次改动；本次只记录，不修改头部。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Detailed Rules）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。
Formulas 对本系统是**必需**：规则 12 定义了单帧生效与 0.2 s 过渡上限两条数值规则，按 standard 层不能作为 advisory；该节两条公式都有变量表、取值范围、输出范围与示例，可求值。
Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs。Game Feel 一节实质覆盖 Player Fantasy 的内容。

### Dependency Graph

| 依赖 | 方向 | 文件 | 状态 |
|---|---|---|---|
| 数据配置 | 上游（硬） | `design/gdd/data-config.md` | ✓ 存在，第 201 行回指本系统 |
| 输入 | 上游（硬） | `design/gdd/input.md` | ✓ 存在，第 76/174 行回指本系统 |
| 界面流程 | 下游（硬） | `design/gdd/ui-flow.md` | ✓ 存在，第 155/429/526 行回指本系统 |
| 餐局流程 | 下游（硬） | `design/gdd/run-flow.md` | ✓ 存在，第 117/288 行回指本系统 |
| 首局引导 | 下游（软） | `design/gdd/first-run-onboarding.md` | ✓ 存在，第 150/412 行回指本系统 |
| 音效反馈 | 下游（软） | `design/gdd/audio-feedback.md` | ✓ 存在，第 130/381 行回指本系统 |

六条双向链路全部闭合，无断链。存档仍被正确标记为"不属于依赖"而非遗漏。

### Registry Values

`design/registry/entities.yaml` 现为 79 条命名条目；`grep scene-flow` 命中 0 行（无条目由本系统拥有，也无 `referenced_by:` 指向本系统）。按 Phase 3 用全部 79 个条目名（`_` 展开为 `[_ ]`、大小写不敏感）对本 GDD 全文扫描：**命中 0 行**，不存在数值复述冲突。本系统只拥有状态名与转移语义，本身不含经济数值。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | BLOCKING | `design/gdd/scene-flow.md:61`（对照 `:32`） | 转移表只有 `phase: INIT` 一行，规则 6 定义的 `phase: RUN` 那一行不存在；`init_failed` 在 `:77/:78/:142/:167/:168/:215/:216` 与 AC 27（`:254`）仍写作唯一 payload，全文再无一处提到 `fault` | `run-flow.md:46`（规则 18：`fault` 必须携带 `phase ∈ {INIT, RUN}`）；ADR-0007 D7（`adr-0007-run-state-settlement-transaction.md:256-279`：`fault.phase` 必须始终存在，INIT/RUN 各发一次 `run_faulted`）；`ui-flow.md:81`（规则 19 按 `phase` 分支文案） |
| 2 | BLOCKING | `design/gdd/scene-flow.md:29` | 规则 3 写"`Modal` 与 `Blocked` 可由界面流程或首局引导临时压栈"，把 `Blocked` 的压栈权授给了界面流程；界面流程只压 `Modal`，`Blocked` 只有首局引导（会话内）与本系统的 `Boot` 加载 | `input.md:70`（"`Blocked` 只由场景与流程管理（加载）和首局引导请求……除这两类请求方外，任何系统都不得请求 `Blocked`"）、`input.md:52`（`push_context` 调用方表：界面流程 `Modal` / 首局引导 `Blocked`）、`adr-0005-input-context-stack.md:228`（以 `&"Blocked"` 调 `push_context` 的调用点只允许 `first_run_onboarding.gd`） |
| 3 | RECOMMENDED | `design/gdd/scene-flow.md:41` | 规则 15 只写 `state_changed(from, to)`，而架构侧统一为三参（携带 payload） | `adr-0004-service-composition-signals.md:207/227`、`adr-0003-scene-screen-architecture.md:298`、`architecture.md:264`（`signal state_changed(from, to, payload)`） |
| 4 | RECOMMENDED | `design/gdd/scene-flow.md:28`、`:244` | `Boot → Quit` 例外被引用成"规则 6"，而规则 6 是"一局的离开"；该例外实际写在 `:67` 与规则 10 | 上一轮 APPROVED 的 findings 第 1 条（仍未修） |
| 5 | RECOMMENDED | `design/gdd/scene-flow.md:266-267` | Open Questions 第 1、2 行仍未标记已解决 | `ui-flow.md:47`（规则 3：`Menu → MainMenu 或 ReceiptScreen`，不新增顶层状态）；`run-flow.md:54`（"RunConfig 与 RunResult 的归属"） |
| 6 | RECOMMENDED | `design/gdd/scene-flow.md:88/119-120/183` | `FADE_OUT` / `FADE_IN` 的时长所有者与界面流程说法相反，第 4 轮仍未闭合 | `ui-flow.md:631`（Open Question 2："过渡时长由场景与流程管理拥有"） |
| 7 | RECOMMENDED | `design/gdd/scene-flow.md:135` | "依据 `request_transition` 返回 `false` 重新发起"读作可立即重试，但 `Paused` 下 `Run → Menu` 不在表内 | `run-flow.md:42`（规则 14：保留 `RunResult`，回到 `Run` 后重发，暂停期间等待） |
| 8 | RECOMMENDED | `design/gdd/scene-flow.md:76-77/106` | 调用写法不统一：签名为 `request_transition(to, payload)`，正文写作 `request_transition(Menu → Run, RunConfig)` | 本 GDD 规则 2（`:28`），一处完整签名即可 |
| 9 | RECOMMENDED | `design/gdd/scene-flow.md:259` | AC 32 把两个独立断言塞进一条，且前提依赖尚不存在的 `/ux-design` 文档 | 设计文档标准"AC 必须可独立判定" |
| 10 | RECOMMENDED | `design/gdd/scene-flow.md:236` | AC 9 "画面与暂停前一致"不可独立判定 pass/fail | 同上；`ui-flow.md:556`（AC 4 已用"实例 id 不变"的可观测写法，可照抄） |

**Blocking 1 的复核明细**——`fault.phase` 的语义本身是对的，缺陷在传播：

- 与本 GDD 一致的三处：ADR-0007 D7 的判据表（`run_started` 是否发出）、`run-flow.md:46` 的 `INIT` / `RUN` 定义、`ui-flow.md:81` 的文案分支（"不得对 `RUN` 说「本局未能开始」"）。规则 6 的三句话（进入失败不结算、开局后失败同带 `init_failed`、`phase` 才是判据）逐条对得上，无需改行为。
- 缺的是本系统的"闭合表"：按规则 2，转移表是合法性与 payload 的唯一出处（"请求先校验 `(current, to)` 是否在下面的转移表内"）。该表现在只有 `INIT` 一行，触发条件还窄写成"`RunConfig` 无法解析"，而 ADR-0007 D7 的 `INIT` 还包含 `station_id` 不存在、升级快照非法、子系统初始化失败、`run_id` 溢出；`RUN` 相位（`RunLog` 不变式失败、`settle` 返回结构化错误、重复 `run_id`）在表中完全没有落点。
- 因此一个只读本 GDD 的实现者会得出"开局后失败没有规定的出口"或"payload 只有 `init_failed`"两种错误结论；AC 27（`:254`）同样的写法使一条缺少 `fault` 的 payload 也能通过验收，新规则零覆盖。
- 最小改法：① 第 61 行拆/补为两行（`INIT` 与 `RUN`），或在该行触发条件写全两种相位；② `:78/:168/:216` 的接口行补 `fault.phase`；③ Edge Case `:142` 与 AC 27 补 `fault.phase = INIT`，并新增一条 RUN 相位的 AC（`Run` 中制造 `CAPACITY_MISMATCH` → 切 `Menu`、`init_failed = true` 且 `fault.phase = RUN`、无 `RunResult`、不写纪录）。

**Blocking 2 的复核明细**——`Modal` 与 `Blocked` 的压栈方不是同一组：

- `input.md:70` 是硬约束（"无第三方"），`input.md:52` 给出两个调用方，`ui-flow.md:51/:97` 只压 `Modal`（强调查找 `push_context(Modal)` / `pop_context()` 成对，且"不得调用 `request_context`"），`first-run-onboarding.md:44` 只压 `Blocked`。ADR-0005:228 进一步把 `&"Blocked"` 的调用点静态限定为 `first_run_onboarding.gd`——本 GDD 的这一句会直接让那条静态扫描失败。
- 本系统自己的 `Boot → Blocked` 走的是 `request_context`，与"临时压栈"是两条不同接口，规则 3 的这句也没有为它留位置。
- 最小改法：改写为"`Modal` 由界面流程、`Blocked` 由首局引导临时压栈；本系统自身只在 `Boot` 通过 `request_context` 进入 `Blocked`"。

### 已核对且无问题的跨文档接口（记录以免后续重复检查）

- **放弃路径**（规则 8）：`ui-flow.md:75`（规则 16）与 AC 22（`ui-flow.md:583`）逐字对应——确认后只请求一次 `Paused → Menu, abandoned = true`，无收据、无纪录、不入账；ADR-0007 D3（`:140-180`）的 `commit` 只挂在 `Run → Menu` 被接受上，放弃路径从不触发事务，二者一致。
- **淡入淡出**（规则 12 / Formulas）：ADR-0003 D8（`:242`）实现为"遮盖—揭示"，时长 = `FADE_OUT + FADE_IN` ≤ 0.2 s，在状态写入与界面重建之后启动、从不 `await`；与 `transition_total` 的求和形状、AC 31/33 的"过渡不计入 `state_latency_frames`"完全对偶。
- **顶层状态集合**：`Boot / Error / Menu / Run / Paused / Quit` 六态，与 ADR-0003 D7（`:228` 的 `Settling` 是 `RunFlow` 的内部阶段，不是顶层状态）、`audio-feedback.md:130/381`、`architecture.md:78` 一致；本 GDD 全文无 `Settling` 顶层态或隐含等待态（规则 12、Formulas 末句都显式否定）。
- **逐帧校验与所有权**：规则 2 的"帧初状态 + 首个合法者生效"与 ADR-0003:459、ADR-0004:201（"处理器可以请求切换，但请求按帧初状态校验，派发深度 ≤ 1"）一致。
- **切换顺序**：规则 3/15 的"写入状态 → 切上下文 → 发出 `state_changed`"与 ADR-0004 D8（`:207/227`）的扇出前置约定一致；ADR-0007 D3 第 5 步写 `SceneFlow` 写入 `Menu` 后发信号，亦一致。
- **输入上下文**：五个默认上下文（Blocked / Menu / Menu / Run / Paused）与 `input.md:61-64` 的能力表一致；`Error → Menu` 是本系统对 `input.md` 五上下文的合法复用（`input.md` 不设 Error 上下文，ADR-0005:123 亦允许 `Error` 内存在 `Modal`），不构成新增上下文。
- **请求形态**：`request_context(default_ctx, clear_stack := true)` 与 `input.md:51/174`、ADR-0005 D1 的签名一致；本系统不使用 `allow`，不涉及首局引导的 `consume_action`。
- **`ui_pause` / `ui_accept` 归属**：本系统只消费 `ui_pause`（`input.md:78`、ADR-0005:235），`ui_accept` 由界面流程消费（`ui-flow.md:73`），三方一致。
- **数据配置返回值**：`Ready` / `Failed` 与 `data-config.md:68-70` 的状态表一致，未与存档 `load_outcome` 混淆。
- **TR 覆盖**：`requirements-traceability.md:65-69` 的五条 TR 均有 ADR 归属且标 `Covered`；新规则 6 的 `fault.phase` 没有独立 TR（现有 TR-scene-flow-002 只写"payload handoff"），可在补齐 blocking 1 时顺带补一条。

### 跨文档待办（不属于本 GDD，仅记录）

- ADR-0007 D3 的验收条（`adr-0007-run-state-settlement-transaction.md:570`）用"`Handoff` 中请求 `Paused → Menu (abandoned)`"作为放弃夹具，而 `ui-flow.md:73`（规则 15）在 `Ending` / `Settling` / `Handoff` 隐藏「放弃本局」——该路径在界面层不可达，应由 ADR 或界面流程的所有者裁定，不计入本 GDD 的 findings。
- `audio-feedback.md` 早前的"`scene-flow.md` 下游表记为 GDD 未写"待办已过期（本 GDD 第 170 行早已是"GDD 已写"），仍应由音效反馈的会话清理。

### Suggested systems-index status

**Needs Revision** —— 2 个 blocking（转移表缺 RUN 相位那行、规则 3 把 `Blocked` 的压栈方扩到界面流程）修复并复审通过前不应为 Approved。注意 `design/gdd/systems-index.md:60` 当前记为 **Approved**，而本 GDD 头部记为 `In Review`——索引跑在了证据前面，需由编排会话按本次判定改正为 **Needs Revision**（本次按并行约束未改动索引与头部）。

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/scene-flow.md 07753accfc002be7257cbcdd17323c73c2b6845a
Reviewed-Content-Hash: design/registry/entities.yaml f7b5aded130f371160d84bdaf678f8530ddc0807
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/run-flow.md 56f6f4866e1b5ab7c8a5a8f04d22078e7d05e1cb
Reviewed-Content-Hash: design/gdd/input.md 27c9d885eb2ba1f8d4b3df72ff6640fed0a5dd36
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/data-config.md 9ecc2a7c18aba33c64bc3391594748e1063851d2
Reviewed-Content-Hash: design/gdd/audio-feedback.md 2c87b5f93a9b3af1937aafc1b4c5c5d7bedb56f2
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a
Reviewed-Content-Hash: docs/architecture/adr-0003-scene-screen-architecture.md 3847588766a746042167ef131a23bb79765ec74f
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/architecture.md 963f89279f980b90e9d1d1d588b1d630b8fd867e
Reviewed-Content-Hash: docs/architecture/requirements-traceability.md efc28c613de2b73657961d3829f460307d5dbe92

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean 模式，Phase 3b 未执行）
Blocking items: 1 | Recommended: 6
Summary: 全文 lean 复审（非增量），不是增量比对。上一轮 2 个 blocking 中，blocking 2（规则 3 把 `Blocked` 压栈权授给界面流程）已修好：`:29` 现在写"`Modal` 只由界面流程压栈，`Blocked` 只由首局引导（引导段）与本系统（`Boot` 加载）请求——界面流程**不得**压 `Blocked`"，与 `input.md:70`、ADR-0005 D9 一致。blocking 1 只修了一半：转移表新增 `:62` 的 RUN 相位行、`fault` 的语义与 ADR-0007 D7 逐条对齐、依赖表 `:169`/`:170` 与交叉引用 `:217`/`:218` 都已补 `fault.phase`、新增 AC 27a 覆盖 RUN 相位——但上一轮点名的第三个落点 `:78`（Interactions 的界面流程行）没改，仍只列 `RunResult` / `abandoned` / `init_failed`，紧邻的 `:79`（餐局流程行）也只写 `init_failed`。界面流程的实现者按规则 2 只能读 Interactions 表与转移表，前者仍给出"`init_failed` 就是全部 payload"的读法，ADR-0007 D7 约束 1 的"任何界面不得只读它"在本行仍被违反，且它恰是本轮新增 AC 27a 的被测系统。其余为措辞与自洽性问题，无结构性返工。
Prior verdict resolved: Partial —— 2026-11-06 的 blocking 2 已修好；blocking 1 的转移表行、依赖表、交叉引用、AC 均已补齐，唯 `:78`/`:79` 两行漏改，故同一 blocking 以残余形态延续（不是新问题）。上轮 8 条 recommended 中 `:41` 的 `state_changed` 两参写法与 Open Questions 第 1/2 行仍未处理。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/scene-flow.md`、`design/registry/entities.yaml`、`data-config.md`、`input.md`、`run-flow.md`；`UNCHANGED`：`SKILL.md`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`design-docs.md`、`systems-index.md`、`adr-0003`、`adr-0004`、`adr-0005`、`adr-0007`、`architecture.md`、`requirements-traceability.md`。上一份 NEEDS REVISION（2026-11-06）绑定 `4efc24d1…`/`07753acc…` 序列，当前字节为 `4f0e5074…`；目标文档与四份跨文档事实（输入、餐局流程、数据配置、注册表）均已变动，故按全文复审。头部 `Status` 仍停在 `In Review 2026-11-06`，只记录不改。

### 完整性（standard 层：5 节必需 + 有数值规则时的 Formulas）

必需 6/6 齐全：Overview、Detailed Design（= Core Rules / States and Transitions）、Formulas、Edge Cases、Dependencies、Acceptance Criteria。Formulas 两条（单帧生效、`transition_total ≤ 0.2 s`）都有变量表与值域，可求值。Advisory 缺口（不阻塞）：Player Fantasy、Tuning Knobs；Game Feel 覆盖前者实质。

### Dependency Graph

六条双向链路仍全部闭合（数据配置、输入、界面流程、餐局流程、首局引导、音效反馈），`data-config.md:201`、`input.md`、`ui-flow.md`、`run-flow.md`、`first-run-onboarding.md`、`audio-feedback.md` 均有回指。存档仍正确标为"无直接调用"，不是遗漏。

### Registry Values

`design/registry/entities.yaml` 1355 行、79 条命名条目；`grep scene-flow` 命中 0 行，无条目由本系统拥有或 `referenced_by:` 指向本系统。本系统只拥有状态名与转移语义，无经济数值可复述，无冲突。

### Findings

| # | 级别 | 位置（file:line） | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/scene-flow.md:78`、`:79` | Interactions 的界面流程行仍写"订阅 `RunResult`、`abandoned`、`init_failed` 与当前顶层状态"，餐局流程行仍写"启动失败时发起 `Run → Menu, init_failed`"——两处都没有 `fault.phase`，而上轮的最小改法明确点名 `:78/:168/:216`，本轮只改了 `:168`/`:216`。`:62` 的 RUN 行与 AC 27a 都要求界面按 `phase` 分支，`:78` 却是界面流程唯一被告知的 payload 清单 | ADR-0007 D7 约束 1（`adr-0007-run-state-settlement-transaction.md:275`："任何界面不得只读它。`fault.phase` 必须始终存在"）；`ui-flow.md:81`（规则 19 按 `phase` 分支）；本文件 `:32`、`:62`、`:169`、`:217`、AC 27a（`:257`） |
| 2 | RECOMMENDED | `design/gdd/scene-flow.md:61`、`:62` | payload 写成字面量 `fault = {phase: INIT}` / `{phase: RUN}`，把 `fault` 描述成只有 `phase` 一个字段。ADR-0007 D7 的界面文案是"「本局未能开始」+ **原因**"，`run_id` 溢出还要带 `detail.code`；本系统自称"只转交 payload"，但字面量写法会让实现者只转发 `phase`、丢掉原因 | `adr-0007-run-state-settlement-transaction.md:270-276`、`:285`（`detail.code = RUN_ID_OVERFLOW`）；`ui-flow.md:81`（"+ 原因"） |
| 3 | RECOMMENDED | `design/gdd/scene-flow.md:29` | 修好 blocking 2 的同一句把两条不同接口混成"请求方"："`Blocked` 只由首局引导（引导段）与本系统（`Boot` 加载）请求"。本系统 `Boot` 走的是 `request_context`（清栈置默认），不是 `push_context`；ADR-0005 D9 把 `&"Blocked"` 的 `push_context` 调用点静态限定为（会话内的）首局引导，本系统的名字出现在"压栈方"语句里会让那条扫描产生歧义 | `adr-0005-input-context-stack.md:228`（D9 调用点限定）、`:118`（`request_context` 语义）；`input.md:52`（`push_context` 调用方表） |
| 4 | RECOMMENDED | `design/gdd/scene-flow.md:41` | 规则 15 仍写 `state_changed(from, to)` 两参，架构侧统一为三参（携带 payload），且 `AppRoot` 的固定六步扇出依赖 payload（`init_failed` 分支） | `adr-0004-service-composition-signals.md:207`、`:227`（`state_changed(from, to, payload)`）；上轮 recommended 第 3 条，未修 |
| 5 | RECOMMENDED | `design/gdd/scene-flow.md:257`（AC 27a） | "本系统转交 payload 时不改写、不丢弃 `phase`，**两者**都不得停在 `Run`"——"两者"指 `init_failed` 与 `fault.phase`？还是两种故障路径？GIVEN 只给了一条（`settle` 结构化错误），却在 THEN 里断言两个对象；可独立判定的写法应写成"状态切为 `Menu` 且 payload 中的 `fault.phase` 仍为 `RUN`" | design-docs「验收标准必须可独立判定」 |
| 6 | RECOMMENDED | `design/gdd/scene-flow.md:228` | 集成判据只点名第 9、10、14、22、23 条依赖其他系统，新增的 AC 27 / 27a 实际需要餐局流程制造故障、界面流程消费 payload 才能验证，未列入；AC 27a 恰是本轮新增，最容易被当成"只需本系统即可测" | 本文件 AC 27/27a（`:256-257`）；同上 |

### 已核对且无问题（记录以免重复检查）

- **`fault.phase` 语义**：`:32` 规则 6 的三句（进入失败不是结算、开局后失败同带 `init_failed`、`phase` 才是判据）与 ADR-0007 D7 的判据表（`run_started` 是否发出）逐条同义；`:62` 触发条件（`RunLog` 不变式失败 / `settle` 结构化错误）与 D7 的 RUN 列一致；`:61` 的 INIT 触发条件已从"`RunConfig` 无法解析"扩到"任一子系统初始化失败 / `run_id` 溢出"，与 D7 的 INIT 列一致。
- **INIT/RUN 两行的分流**：`:62` 备注"界面按 `phase` 说「本局结算失败，收益不计」"与 `ui-flow.md:81` 的"不得对 `RUN` 使用「本局未能开始」"一致；`:144` Edge 的"本系统不区分两条故障路径，只转交 payload"与 D7 的"`SceneFlow` 不判定相位"分工一致。
- **规则 3 的新措辞**：`:29` 的"界面流程**不得**压 `Blocked`"是硬约束，与 `input.md:70`、`:52`、ADR-0005 D9（`:228`）一致；`Boot → Blocked` 用 `request_context` 的说法与规则 2 的签名（`default_ctx, clear_stack := true`）一致。
- **放弃路径**：`:34` 与 `ui-flow.md` 规则 16 / AC 22 仍一致（只请求一次 `Paused → Menu, abandoned = true`，无收据、不触发结算事务）；ADR-0007 D3 的 `commit` 只挂在 `Run → Menu` 被接受上。
- **顶层状态集合**：仍是六态，`Settling` 是 `RunFlow` 内部阶段（`run-flow.md`），本文件无隐含等待态；规则 12 与 Formulas 末句显式否定"过渡算状态"。
- **REGISTRY / 数值**：无条目，无冲突（见上）。

### Suggested systems-index status

**Needs Revision** —— 1 个 blocking（`:78`/`:79` 的 payload 清单漏 `fault.phase`）修好并复审通过前不应为 Approved。`design/gdd/systems-index.md:60` 当前记为 **Approved**，仍跑在证据前面；本次按并行约束不改索引与头部。

### Verdict: NEEDS REVISION

---

Reviewed-Content-Hash: design/gdd/scene-flow.md 4f0e50743f8b72012ff85540a58dfa1481cd676f
Reviewed-Content-Hash: design/registry/entities.yaml 1a9dceea86fa114cbbbe92eeaab97ff540737277
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/input.md 18815e7d6735d9a0995b8cd79a762b516a07be1a
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/run-flow.md 346526135c3bfbf7829d9a9bc6955d9088722b80
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0013-action-pipeline-end-judge.md 3e709cc7b5901dc11ba7ae9a7b7f73d19fc616e8

---

## Review — 2026-11-17 — Verdict: NEEDS REVISION
Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 1 | Recommended: 6
Summary: 全文 lean 复审（非增量）。上轮唯一的 blocking 已修好：交互表 `:78`（界面流程）与 `:79`（餐局流程）现在都携带 `fault.phase`（`INIT` / `RUN`），与转移表 `:61`/`:62`、规则 6（`:32`）、依赖表 `:169`/`:170`、交叉引用 `:217`/`:218`、AC 27/27a 以及 ADR-0007 D7 约束 1（"任何界面不得只读它，`fault.phase` 必须始终存在"）一致；`ui-flow.md:81` 的 `phase` 分支现在能拿到数据。新发现 1 个 blocking：同一张交互表的"输入"行把场景与流程管理写成 `context_changed` 的订阅者，与 ADR-0005 D9、ADR-0004 D9 冲突。其余为旧 recommended 未修，以及状态表 `:49` 的入口条件仍按 INIT 命名。
Prior verdict resolved: Yes —— 上轮的 1 个 blocking（`:78`/`:79` 缺 `fault.phase`）已修好，不是以残余形态延续。上轮 6 条 recommended 全部未修（4 条以原文复发，2 条见下表 #3、#6）。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/scene-flow.md`、`design/registry/entities.yaml`、`data-config.md`、`input.md`、`run-flow.md`；`UNCHANGED`：`SKILL.md`、`design-docs.md`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`systems-index.md`、`adr-0003`、`adr-0004`、`adr-0005`、`adr-0007`、`architecture.md`、`requirements-traceability.md`。上一份 NEEDS REVISION（2026-11-17）绑定 `4f0e5074…`，当前字节为 `1a984f82…`；目标文档与四份跨文档事实均变动，按全文复审。头部 `Status` 仍停在 `In Review — 2026-11-17 修订`，只记录不改。

### 上轮 blocking 的修复核对（含要求的三项对照）

- **交互表两行（`:78`、`:79`）**：界面流程行现为"订阅 `RunResult`、`abandoned`、`init_failed` + `fault.phase`（`INIT` / `RUN`）与当前顶层状态"；餐局流程行现为"启动失败或运行期故障时发起 `Run → Menu, init_failed + fault`（`fault.phase ∈ {INIT, RUN}`）"。界面流程不再可能把 `init_failed` 读成全部 payload。
- **对照转移表**：`:61`（`RunConfig` 无法解析 / 任一子系统初始化失败 / `run_id` 溢出 → `INIT`）与 `:62`（`RunLog` 不变式失败 / `settle` 结构化错误 → `RUN`）两条触发条件的划分与交互表一致；两者的"不是结算、不产生 `RunResult`、不写纪录 / `run_started` 从未发出"也与 D7 判据表同义。
- **对照 AC 27 / 27a**：AC 27（`:256`）断言 `Menu` + `init_failed = true` + `fault.phase = INIT`、不产生 `RunResult`、不写纪录；AC 27a（`:257`）断言 `fault.phase = RUN` 且"本系统转交 payload 时不改写、不丢弃 `phase`"。两条与交互表、依赖表一致，且是当前唯一被测的两条故障路径。
- **对照 ADR-0007 D7**：D7 的三条实现约束（`init_failed` 只是兼容位、`settlement_aborted` 不是故障、`run_faulted` / `run_discarded` 互斥）在本文件中没有相反表述；`:144` 的"本系统不区分两条故障路径，只转交 payload"与 D7 的分工一致（`detail.code` 完整性见下表 #4）。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | **BLOCKING** | `design/gdd/scene-flow.md:77`、`:216` | 交互表"输入"行与交叉引用都写"订阅 `context_changed` 与 `ui_pause` 触发 `Run ⇄ Paused`"。ADR-0005 D9 明文"**`SceneFlow` 不订阅 `context_changed`**……让它再订阅会形成「自己触发、自己响应」的回环"，ADR-0004 D9 的 `context_changed` 订阅者只有 `UIRoot`（深度断言）与 `AppRoot`（日志）。本行把一条不存在的入边写成契约；按它实现就是 ADR 禁止的重入。只保留 `ui_pause` 订阅即正确 | ADR-0005 D9（`adr-0005:234`）；ADR-0004 D9（`adr-0004:228`） |
| 2 | RECOMMENDED | `:49` | 状态表 `Menu` 的"进入条件"仍写"启动成功 / 一局结算 / **进入失败返回** / 放弃一局"——只命名 INIT 路径。按 `:62`，开局后故障同样回到 `Menu`；该列应写"进入失败或开局后故障返回"（与本轮修好的 `:78`/`:79` 同类残余）（上轮 blocking 的同类实例，本轮新增记录） | 本文件 `:61`、`:62`、AC 27a（`:257`）；ADR-0007 D7 |
| 3 | RECOMMENDED | `:29` | 规则 3 仍把两条不同接口混成"请求方"："`Blocked` 只由首局引导（引导段）与本系统（`Boot` 加载）请求"。本系统 `Boot` 走 `request_context`（清栈置默认），ADR-0005 D9 把 `&"Blocked"` 的 `push_context` 调用点静态限定为（会话内的）首局引导（上轮第 3 条，未修） | ADR-0005 D9（`adr-0005:228`）、D1（`:118`）；`input.md:52` |
| 4 | RECOMMENDED | `:61`、`:62` | payload 仍是字面量 `fault = {phase: INIT}` / `{phase: RUN}`，把 `fault` 写成只有 `phase` 一个字段。ADR-0007 D7 的界面文案是"「本局未能开始」+ **原因**"，`run_id` 溢出还要带 `detail.code = RUN_ID_OVERFLOW`；本系统自称"只转交 payload"，字面量写法会让实现者只转发 `phase`（上轮第 2 条，未修） | ADR-0007 D7（`adr-0007:270-276`、`:285`）；`ui-flow.md:81` |
| 5 | RECOMMENDED | `:41` | 规则 15 仍写 `state_changed(from, to)` 两参，架构侧统一为 `state_changed(from, to, payload)`，且 `AppRoot` 的固定六步扇出第 2 步 `_ui.reset_for_state(from, to, payload)` 依赖 payload（`init_failed` 分支）（上轮第 4 条，未修） | ADR-0004 D8/D9（`adr-0004:207`、`:227`） |
| 6 | RECOMMENDED | `:257`（AC 27a） | THEN 仍写"本系统转交 payload 时不改写、不丢弃 `phase`，**两者**都不得停在 `Run`"——"两者"无先行词（`init_failed` 与 `fault.phase`？还是两条故障路径？），GIVEN 只给一条。应写"状态切为 `Menu`，payload 中 `fault.phase` 仍为 `RUN`" | design-docs「验收标准必须可独立判定」 |
| 7 | RECOMMENDED | `:228` | 集成判据仍只点名第 9、10、14、22、23 条；AC 27 / 27a 需要餐局流程制造故障、界面流程消费 payload 才能验证，未列入（上轮第 6 条，未修） | 本文件 AC 27/27a（`:256-257`） |

（上表 #2–#7 计 6 条 recommended。）

### 已核对且无问题（记录以免重复检查）

- **`abandoned` payload**：`:65`、`:34` 的 `Paused → Menu, abandoned = true` 与 `ui-flow.md:75`、`:141`、`:155` 一致；不产生 `RunResult`、不写纪录与 ADR-0007（`:570`）的 Handoff 放弃断言一致。
- **`RunResult` payload**：`:60`、`:240`（AC 11）与 ADR-0007 Architecture Diagram 的 `request_transition(Run -> Menu, result)` 一致，同帧恰好一次交付不落盘不缓存。
- **`init_failed` 两故障共用**：`:32`、`:61`、`:62`、`:143`、`:144` 五处口径统一（`init_failed` 只表示"payload 中没有 `RunResult`"，`phase` 是文案判据）。
- **Blocked 归属**：规则 3 的"界面流程**不得**压 `Blocked`"与 `input.md:70`、`:52`、ADR-0005 D9 一致；`Boot → Blocked` 走 `request_context`，`Blocked` 只在 `Run` 由首局引导压栈，与 `input.md:56` 的"深度 2"论证一致（接口措辞见 #3）。
- **其余条目无 INIT-only 假设**：`:32`、`:143`、`:144`、`:169`、`:170`、`:217`、`:218`、AC 27/27a 均两相位齐备；`Menu` 目标状态在两个故障路径下相同，无需新增状态。
- **完整性（standard 层）**：Overview、Detailed Design、Formulas、Edge Cases、Dependencies、Acceptance Criteria 6/6 齐全；Formulas 两条（`state_latency_frames`、`transition_total`）有变量表、值域与约束，可求值。Player Fantasy、Tuning Knobs 为 advisory 缺口（Game Feel 覆盖前者实质）。
- **Dependency Graph**：上游 `data-config.md`（`:203` 回指）、`input.md`（`:163`、`:216` 回指）；下游 `ui-flow.md`（`:155`、`:429`）、`run-flow.md`（`:117`）、`first-run-onboarding.md`（`:148`、`:413`）、`audio-feedback.md`（`:130`、`:381`）全部存在并回指。存档仍正确标为"无直接调用"。
- **Registry Values**：`entities.yaml` 无 `source: design/gdd/scene-flow.md` 的条目，也无 `referenced_by:` 指向本系统——本系统只拥有状态名与转移语义，无数值可复述，无冲突。

### Verdict: NEEDS REVISION

**NEEDS REVISION**（1 blocking · 6 recommended）。上轮的 payload 阻塞确认修好，`fault.phase` 现在在交互表、转移表、依赖表、交叉引用与 AC 27/27a 五处闭环。唯一 blocking 是交互表"输入"行多写了一条 `context_changed` 订阅：ADR-0005 D9 明确禁止、ADR-0004 D9 也把订阅者限定为 `UIRoot` + `AppRoot`，删掉这四个字即与两张 ADR 对齐。

- 建议 `systems-index` 状态：**Needs Revision**（第 3 行当前 `Approved`；本次未改索引与文件头，由编排会话统一处理）。
- 本 GDD 未被修改，只追加本日志条目。

Reviewed-Content-Hash: design/gdd/scene-flow.md 1a984f827d11dd07627e2c82f3b97d4ed58cd93b
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: design/gdd/input.md 1ef730801dde50820c7d8e3f99ecd31ac476d2c6
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/run-flow.md 34881a6a257179f401908799c00b8f396ad279ed
Reviewed-Content-Hash: design/gdd/data-config.md 36fd1ebaca36d1edee0c93b674f21d0418798713
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a

---

## Review — 2026-11-17 — Verdict: APPROVED（第 4 轮：`context_changed` 订阅修复后的全文复审）

Review mode: lean · Workflow: standard
Scope signal: S
Specialists: none（lean — Phase 3b 未运行）
Blocking items: 0 | Recommended: 8
Summary: 全文 lean 复审（非增量），按当前字节重跑 Phase 0–4。上轮唯一 blocking 是**完整**修复，不是局部补丁：三处（交互表 `:77`、依赖表 `:163`、交叉引用 `:216`）现在都写「调用 `request_context(default_ctx, clear_stack := true)` 并读返回值（**不订阅 `context_changed`**，ADR-0005 D9）」，全文再无一处把本系统写成该信号的订阅者；要求的 `fault.phase ∈ {INIT, RUN}` 在交互表的界面流程行 `:78` 与餐局流程行 `:79` 两处都在，与转移表 `:61`/`:62`、规则 6（`:32`）、依赖表 `:169`/`:170`、交叉引用 `:217`/`:218`、AC 27/27a 与 ADR-0007 D7 一致。新发现 1 个 recommended（文档末仍称 `docs/architecture/` 只有 `tr-registry.yaml`、并要求一份 critical ADR，而目录已有 20 份 ADR，其中 ADR-0003/0004/0005/0007 已回答所列全部问题）与 1 个规则内自相矛盾的括注（规则 2 的「同一帧重复的 `Paused → Run` 落到幂等路径」与同段 first-wins、AC 16 冲突）。两者都不改变合法转移集合或信号形状，判 APPROVED。
Prior verdict resolved: Yes —— 上轮 blocking（交互表「输入」行与交叉引用写成 `context_changed` 订阅者）已修好且三处落点全覆盖；上轮 6 条 recommended 全部原文复发（下表 #2–#7），另新增 2 条（#1、#8）。

### 回执状态

`review-receipts.sh check` 报 `CHANGED`：`design/gdd/scene-flow.md`、`data-config.md`、`input.md`、`run-flow.md`、`adr-0013`；`UNCHANGED`：`entities.yaml`、`SKILL.md`、`design-docs.md`、`ui-flow.md`、`first-run-onboarding.md`、`audio-feedback.md`、`game-concept.md`、`systems-index.md`、`adr-0003`、`adr-0004`、`adr-0005`、`adr-0007`、`architecture.md`、`requirements-traceability.md`。上一份 NEEDS REVISION（第三轮）绑定 `1a984f82…`，当前字节为 `101d7a9f…`，故按全文复审。头部 `Status` 为「In Review — 2026-11-17 修订：处理第三轮 lean 复审的 1 条 blocking（本系统不订阅 `context_changed`，ADR-0005 D9）；待第四轮 lean 复审」，与第三轮日志相符，只记录不改。

### 上轮 blocking 的修复核对（三项指定对照）

- **① 订阅归属（`:77`、`:163`、`:216`）已修好，无残余。** 全文 `grep context_changed` 仅 3 行，全部是否定式并附 ADR-0005 D9；交互表「输入」行现在是「输出：切换时调用 `request_context(default_ctx, clear_stack := true)` 并读返回值……输入：轮询 `ui_pause` 触发 `Run ⇄ Paused`；`ui_accept` 由界面流程消费，本系统不监听」，与 ADR-0005 D9（`adr-0005:234`：`SceneFlow` 不订阅、回执即返回值；`:230`：`is_action_just_pressed(&"ui_pause")` 唯一调用方为 `SceneFlow`）及 ADR-0004 D9（`adr-0004:228`：`context_changed` 订阅者仅 `UIRoot` / `AppRoot`）一致。
- **② `fault.phase ∈ {INIT, RUN}` 两行齐备（`:78`、`:79`）。** 界面流程行：「订阅 `RunResult`、`abandoned`、`init_failed` + `fault.phase`（`INIT` / `RUN`）与当前顶层状态」；餐局流程行：「启动失败或运行期故障时发起 `Run → Menu, init_failed + fault`（`fault.phase ∈ {INIT, RUN}`）」。与 ADR-0007 D7「`init_failed` 只是兼容位、任何界面不得只读它、`fault.phase` 必须始终存在」（`adr-0007:275`）与 `ui-flow.md:81` 规则 19 的 `INIT` / `RUN` 双文案一致；界面流程不再可能把 `init_failed` 读成全部 payload。`:169`/`:170`、`:217`/`:218` 同步携带，AC 27（`:256`）与 27a（`:257`）是被测判据。
- **③ `adr-0005` 文件名核实。** 任务给出的 `docs/architecture/adr-0005-ui-input-context-model.md` **不存在**；目录中实际文件为 `docs/architecture/adr-0005-input-context-stack.md`（本文件正文引用的 `ADR-0005 D9` 即指它，哈希已按真实文件名生成）。`adr-0004-*` 唯一匹配 `adr-0004-service-composition-signals.md`，`adr-0007-*` 唯一匹配 `adr-0007-run-state-settlement-transaction.md`，均存在。

### Findings

| # | 严重度 | 位置 | 发现 | 违反的权威 |
|---|---|---|---|---|
| 1 | RECOMMENDED | `design/gdd/scene-flow.md:276`、`:274` | 末段仍写「`docs/architecture/` 目前只有 `tr-registry.yaml`」并要求「→ 需要一份 critical ADR」。该陈述已失真：目录现有 20 份 ADR（0001–0020）加 `architecture.md`、`control-manifest.md`，而所列四个待决问题（通知机制、`Quit` 是否独立状态、`close_requested` 拦截、`reduce_motion` 来源）已分别由 ADR-0004 D8/D9/ADR-0003/ADR-0002/ADR-0008 回答。保留该句会让下游以为 critical ADR 缺失并重复立项，也会让 `/gate-check technical-setup` 的判据看起来悬空 | `docs/architecture/` 实际目录；ADR-0004 D8/D9（`adr-0004:203-228`）；ADR-0003、ADR-0005、ADR-0007 |
| 2 | RECOMMENDED | `:49` | 状态表 `Menu` 的「进入条件」仍写「启动成功 / 一局结算 / 进入失败返回 / 放弃一局」，只命名 INIT 路径。按 `:62`，开局后故障同样回到 `Menu`；该列应写「进入失败或开局后故障返回」（上轮第 2 条，未修） | 本文件 `:61`、`:62`、AC 27a（`:257`）；ADR-0007 D7 |
| 3 | RECOMMENDED | `:29` | 规则 3 仍把两条不同接口混成「请求方」：「`Blocked` 只由首局引导（引导段）与本系统（`Boot` 加载）请求」。本系统 `Boot` 走 `request_context`（清栈置默认），ADR-0005 D9 把 `&"Blocked"` 的 `push_context` 调用点静态限定为（会话内的）首局引导（`adr-0005:228`）；本系统名出现在「压栈方」语句里会让那条扫描产生歧义（上轮第 3 条，未修） | ADR-0005 D9（`adr-0005:228`）、D1（`:118`）；`input.md:52` |
| 4 | RECOMMENDED | `:61`、`:62` | payload 仍是字面量 `fault = {phase: INIT}` / `{phase: RUN}`，把 `fault` 写成只有 `phase` 一个字段。ADR-0007 D7 的界面文案是「本局未能开始」+ **原因**，`run_id` 溢出还要带 `detail.code = RUN_ID_OVERFLOW`（`adr-0007:285`）；本系统自称「只转交 payload」，字面量写法会让实现者只转发 `phase`（上轮第 4 条，未修） | ADR-0007 D7（`adr-0007:270-276`、`:285`）；`ui-flow.md:81` |
| 5 | RECOMMENDED | `:41` | 规则 15 仍写 `state_changed(from, to)` 两参，架构侧统一为 `state_changed(from, to, payload)`，`AppRoot` 六步扇出的第 2 步 `_ui.reset_for_state(from, to, payload)` 依赖 payload 的 `init_failed` 分支（上轮第 5 条，未修）。本规则末句已把「信号还是直接调用、顺序与重入」交给 ADR，因此属标注陈旧而非契约冲突，但两参写法与 ADR 的签名不一致，建议改为三参或显式注明「签名以 ADR-0004 D8/D9 为准」 | ADR-0004 D8/D9（`adr-0004:207`、`:227`） |
| 6 | RECOMMENDED | `:257`（AC 27a） | THEN 仍写「本系统转交 payload 时不改写、不丢弃 `phase`，**两者**都不得停在 `Run`」——「两者」无先行词（`init_failed` 与 `fault.phase`？还是两条故障路径？），GIVEN 只给一条。应写「状态切为 `Menu`，payload 中 `fault.phase` 仍为 `RUN`」（上轮第 6 条，未修） | design-docs「验收标准必须可独立判定」 |
| 7 | RECOMMENDED | `:228` | 集成判据仍只点名第 9、10、14、22、23 条；AC 27 / 27a 需要餐局流程制造故障、界面流程消费 payload 才能验证，未列入（上轮第 7 条，未修） | 本文件 AC 27/27a（`:256-257`） |
| 8 | RECOMMENDED | `:28` | 规则 2 的幂等括注「（同一帧重复的 `Paused → Run` 落到此路径）」与同段第一句「**一帧内的多个请求全部针对帧初状态校验**……其余一律拒绝并各记日志」及 AC 16 相反：帧初为 `Paused` 时第二个 `Paused → Run` 按帧初状态仍合法，但按 first-wins 应被拒绝并记 WARNING，不会落到「返回 `true`、不记日志」。这决定了同一场景下返回值与日志两处可观测结果，需删掉括注或改为「上一帧已切到 `Run` 后再次请求」 | 本文件规则 2（`:28`）、AC 16（`:245`）、`:70` 的同状态判定 |
| — | （沿用记录） | `:145`、`:229` | Edge Case（同一帧 `ui_pause` 与结算回传）与会话语义已按 `run-flow.md` 规则 14 区分「等待回到 `Run` 后重发」与「立即重试」，本文件未写死错误重试路径，无冲突 | — |

### 已核对且无问题（记录以免重复检查）

- **信号归属**：`state_changed(from, to, payload)` 只有 `AppRoot` 一个订阅者、由它扇出六步（ADR-0004 D8，`adr-0004:203-217`）。本文件 `:41`、`:78`、`:169` 用「订阅 / 接收 payload」描述**行为**而非连接点，且规则 15 显式把机制交给 ADR，故与 ADR-0004 D8 不构成订阅者冲突（签名见 #5）。
- **`fault.phase` 语义**：`:32` 三句（进入失败不是结算、开局后失败同带 `init_failed`、`phase` 才是判据）与 ADR-0007 D7 判据表（`run_started` 是否发出）逐条同义；`:61`/`:62` 触发条件与 D7 的 INIT / RUN 列一致；`:143`/`:144` 的两条 Edge 与「本系统不区分两条故障路径」的分工一致。
- **Blocked 归属**：规则 3 的「界面流程**不得**压 `Blocked`」与 `input.md:70`、`:52`、ADR-0005 D9（`:228`）一致；`Boot → Blocked` 走 `request_context`，首局引导只在 `Run` 压 1 层，与 `input.md:56` 的「深度 2」论证一致（接口措辞见 #3）。
- **`ui_pause` / `ui_accept` / `ui_cancel` 分工**：本系统轮询 `ui_pause`、不监听 `ui_accept`（`adr-0005:230`、D4 `:163`）；`Paused` 中 `ui_cancel` 由界面流程经 `UIRoot` 消费后请求退出确认（`:78`），与 `ui-flow.md:55` 规则 7 ⑧、`:140` 的 `PauseOverlay → Modal1` 一致；`Run` 中的 `ui_cancel` 归 HUD（`in-game-hud.md:59`），本系统不涉及。
- **`RunResult` / `abandoned` / `RunConfig`**：`:60`、`:240`（AC 11）与 ADR-0007 Architecture Diagram 的 `request_transition(Run -> Menu, result)` 一致，同帧恰好一次、不落盘不缓存；`:34`、`:65` 的放弃路径与 `ui-flow.md:75`、`:141`、`:155` 一致。
- **单帧生效与过渡上限**：`:94-106` 的 `state_latency_frames = ceil(t_apply / FRAME_TIME)` 在 `t_apply = 0` 下恒为 0，`FRAME_TIME` 已改为 60 Hz 设计常量并声明不读 `project.yaml`；`:110-128` 的 `transition_total ≤ 0.20 s` 与 `ui-flow.md` 规则 26（`:86`）「只播、不延迟」一致；AC 31–33 三条与 Formulas 一一对应，可求值、可判定。
- **完整性（standard 层）**：`gdd-structure-check` 报 6/6 必需节齐全且非空；Player Fantasy、Tuning Knobs 为 advisory 缺口（Game Feel 覆盖前者实质）。
- **Dependency Graph**：上游 `data-config.md`（`:203` 回指）、`input.md`（`:163`、`:216` 回指）；下游 `ui-flow.md`（`:13`、`:156`）、`run-flow.md`、`first-run-onboarding.md`（`:150`、`:412`）、`audio-feedback.md` 均存在并回指；存档仍正确标为「不属于依赖」。`:178` 对索引第 3 行「数据配置, 输入」的自述与 `systems-index.md:60` 相符。
- **Registry Values**：`entities.yaml` 无 `source: design/gdd/scene-flow.md` 的条目，也无 `referenced_by:` 指向本系统——本系统只拥有状态名与转移语义，无数值可复述，无冲突。
- **AC 编号**：1–33 连续，`27a` 为 27 的相位补充，无重号；`:226` 的「第 31–33 条覆盖两条数值规则」与 Formulas 相符。

### 判定说明

上轮 blocking 的修法是干净的三处否定式替换，没有留下第二读法；`fault.phase` 的完整链路（规则 6 → 转移表两行 → 交互表两行 → 依赖表两行 → 交叉引用两行 → AC 27/27a）也保持完好，本轮确认没有回归。上轮 6 条 recommended 是措辞与陈旧引用，不影响实现面：其中影响最大的 #1 是对仓库事实的错误陈述（声称无 ADR），属陈旧文献问题而非契约冲突；#8 是规则内的一处错误括注，规范句（first-wins）与 AC 16 已给出唯一可实现的读法，无函数返回值依赖该括注。按「是否直接改变实现/信号形状、是否使 AC 或规则不可通过」的判据，本轮无 blocking。

### Scope Signal

Rough scope signal: S —— 单系统状态机，两个数值量均为常量级上限（无 balance 曲线），六个接口全部已由 ADR-0003/0004/0005/0007 定案。

### Verdict: APPROVED

- 建议 `systems-index` 状态：**Approved**（第 3 行当前仍记 `Approved`；本次未改索引、未改 GDD 与文件头，按并行约束只追加本日志）。
- 本 GDD 未被本会话修改，只追加本日志条目。
Reviewed-Content-Hash: design/gdd/scene-flow.md 101d7a9fb773613fed51e508997410204e823e15
Reviewed-Content-Hash: design/registry/entities.yaml 6fd2fcaa1a1417219f92a38d2bc3c767d46b25d5
Reviewed-Content-Hash: .claude/skills/design-review/SKILL.md a63d8fba9b7142eeb43791f061e2e1533c429b43
Reviewed-Content-Hash: .claude/rules/design-docs.md 638ca578625c5a3d24a5bafe0524777b351ab2f8
Reviewed-Content-Hash: docs/architecture/adr-0005-input-context-stack.md 8fc4f89bb89cfb7adf4498c243b74d105754a343
Reviewed-Content-Hash: docs/architecture/adr-0004-service-composition-signals.md f35769ae953b312ef0d07124ac2f6293b95e220a
Reviewed-Content-Hash: docs/architecture/adr-0007-run-state-settlement-transaction.md 27029b18c6265825a3ffb4def6849166fb5736aa
Reviewed-Content-Hash: design/gdd/ui-flow.md 48e965a219eaf9c4c8cf34c4de2977ee6662a9e1
Reviewed-Content-Hash: design/gdd/in-game-hud.md b3bf25a6c9aae77d037a0a13a142d7cc62f3cf03
Reviewed-Content-Hash: design/gdd/run-flow.md f584ae1193d3a9e039b38c02338cafc163efa92a
Reviewed-Content-Hash: design/gdd/first-run-onboarding.md 46ba1e41bba159828384590641c147dfc579e252
Reviewed-Content-Hash: design/gdd/systems-index.md 3f1fa74ca6507898756185f9530eaf4ccfe9a27a

## Re-stamp — 2026-11-17
Status 头行由 `In Review` 改为 `Approved (2026-11-17)`；仅元数据，未改规则、公式、AC、依赖、契约或平衡数值（`.claude/rules/design-docs.md` 的 advisory 情形）。以下为当前字节的盖章：

Reviewed-Content-Hash: design/gdd/scene-flow.md 2439af499a5bdd48e2623ed7abe998d20c8b397c
