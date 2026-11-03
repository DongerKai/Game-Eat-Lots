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
