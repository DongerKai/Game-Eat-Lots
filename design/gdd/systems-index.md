# Systems Index: 回本 (Break Even)

> **Status**: Approved
> **Created**: 2026-11-03
> **Last Updated**: 2026-11-03
> **Source Concept**: design/gdd/game-concept.md

---

## Overview

This game needs the systems for a single, self-contained loop: one run of a
buffet, decided in turns, scored against the entry fee, then upgraded between
runs. There is no combat, no AI, no narrative, and no networking in the MVP — the
mechanical surface is narrow and deep, and the pillars (每口都是算术题 · 没有白吃的菜 ·
时间在走手不用快 · 数字看得见地涨) put the weight on numbers and feedback rather than
on dexterity or content volume.

**Foundation** — 数据配置 holds every dish, upgrade and station definition so no
gameplay value is hardcoded; 输入 maps keyboard/mouse (and partial gamepad) to
the turn-based UI; 场景与流程管理 is the app-level state machine that opens the
game, enters a run and returns to the menu.

**Core** — 存档 persists upgrades, unlocks and records across sessions; 餐局流程 is
one run from entry fee to settlement to restart; 时间与回合 spends a time slot per
dish and ends the run when it runs out; 胃容量 caps how much can be eaten and
decides when the player is stuffed; 菜品数值 computes price, satiety and unit
value (all inferred except the dish values themselves, which the concept names).

**Feature** — 腻度与口味 discounts repeated dishes and creates the ongoing tradeoff
that keeps the numbers from being solved once; 菜单波次 refreshes the buffet in
waves so the player cannot see everything up front; 结算与入场费 turns eaten value
into net profit; 局外升级 spends that profit on capacity, clout and menu vision;
解锁与档口 opens new stations and dishes; 纪录与里程碑 tracks best runs and the
completion condition.

**Presentation** — 局内 HUD shows satiety, remaining time, current net profit and the
dish cards; 界面流程 covers menu, receipt-style settlement, upgrade screen and
settings; 音效反馈 punctuates clicks, rising numbers and settlement.

**Polish** — 首局引导 teaches unit value with a six-dish first run; 在线挑战 adds
daily challenges and a leaderboard; 同屏合作 is the deferred co-op system.

**Dependency notes** — 数据配置 and 菜品数值 are the widest bottlenecks: a change to
the dish schema reaches 腻度, 菜单波次, 结算 and HUD. 结算与入场费 feeds every
progression system, and 餐局流程 is the skeleton that 时间与回合, 胃容量, HUD and
首局引导 hang from. 场景与流程管理 is the app shell: 数据配置 and 输入 feed it, and
界面流程 and 餐局流程 both hang from its Menu/Run states. No circular dependencies
were found.

---

## Systems Enumeration

| # | System Name | Category | Priority | Status | Design Doc | Depends On |
|---|-------------|----------|----------|--------|------------|------------|
| 1 | 数据配置 | Core | MVP | Designed | design/gdd/data-config.md | — |
| 2 | 输入 | Core | MVP | Designed | design/gdd/input.md | — |
| 3 | 场景与流程管理 | Core | MVP | In Review | design/gdd/scene-flow.md | 数据配置, 输入 |
| 4 | 存档 | Persistence | MVP | Designed | design/gdd/save-system.md | 数据配置 |
| 5 | 餐局流程 | Core | MVP | Designed | design/gdd/run-flow.md | 场景与流程管理, 数据配置, 输入 |
| 6 | 时间与回合 | Core | MVP | Not Started | — | 餐局流程 |
| 7 | 胃容量 | Gameplay | MVP | Not Started | — | 餐局流程, 数据配置 |
| 8 | 菜品数值 | Gameplay | MVP | Approved | design/gdd/dish-values.md | 数据配置 |
| 9 | 腻度与口味 | Gameplay | MVP | Designed | design/gdd/cloying-and-taste.md | 菜品数值, 餐局流程 |
| 10 | 菜单波次 | Gameplay | Vertical Slice | Designed | design/gdd/menu-waves.md | 菜品数值, 数据配置, 餐局流程 |
| 11 | 结算与入场费 | Economy | MVP | Designed | design/gdd/settlement-and-entry-fee.md | 菜品数值, 腻度与口味, 胃容量, 时间与回合, 餐局流程 |
| 12 | 局外升级 | Progression | MVP | Not Started | — | 结算与入场费, 存档, 数据配置 |
| 13 | 解锁与档口 | Progression | Vertical Slice | Not Started | — | 局外升级, 存档 |
| 14 | 纪录与里程碑 | Progression | Vertical Slice | Not Started | — | 结算与入场费, 存档 |
| 15 | 局内 HUD | UI | MVP | Not Started | — | 餐局流程, 时间与回合, 胃容量, 菜品数值, 输入, 结算与入场费 |
| 16 | 界面流程 | UI | MVP | Not Started | — | 餐局流程, 结算与入场费, 局外升级, 场景与流程管理, 输入, 存档 |
| 17 | 音效反馈 | Audio | Vertical Slice | Not Started | — | 餐局流程, 结算与入场费, 场景与流程管理 |
| 18 | 首局引导 | Meta | Vertical Slice | Not Started | — | 餐局流程, 局内 HUD, 输入, 场景与流程管理, 菜品数值 |
| 19 | 在线挑战 | Meta | Full Vision | Not Started | — | 结算与入场费, 存档 |
| 20 | 同屏合作 | Meta | Full Vision | Not Started | — | 几乎全部系统（延后） |

> **UX note**: 场景与流程管理 imposes UI requirements on screens owned by 界面流程
> (Error screen, Pause overlay, Quit-confirm modal). Run `/ux-design` for them
> before writing epics; stories should cite the UX spec, not the GDD directly.
>
> 腻度与口味 also imposes UI requirements on 局内 HUD（菜品卡的实际饱腹与解腻预览、
> 腻度面板）。`/ux-design` 需覆盖这两块，相关 story 引用 `design/ux/hud.md`。
>
> 菜单波次 also imposes UI requirements on 局内 HUD（波次条与 `?` 占位、下一波预览、
> 跳过/结束取餐的确认框）。`/ux-design` 需一并覆盖。
>
> 结算与入场费 imposes UI requirements on a receipt-style settlement screen and on
> the run HUD's live net profit / break-even shortfall. `/ux-design` must cover both;
> stories should cite `design/ux/settlement.md` and `design/ux/hud.md`.
>
> 存档 imposes UI requirements on 界面流程（保存指示、恢复说明、`Blocked` 模态、
> 删除生涯确认）。`/ux-design` must cover them; stories should cite the matching
> `design/ux/` spec.
>
> 餐局流程 constrains 局内 HUD flow states（活动/冻结/结束/交接）and the receipt
> screen's opening timing. `/ux-design` must cover `design/ux/hud.md` and
> `design/ux/settlement.md`.

---

## Categories

| Category | Description | Typical Systems |
|----------|-------------|-----------------|
| **Core** | Foundation systems everything depends on | Input, scene management, run state machine |
| **Gameplay** | The systems that make the game fun | Dish values, cloying, capacity, menu waves |
| **Progression** | How the player grows over time | Meta upgrades, unlocks, records |
| **Economy** | Resource creation and consumption | Settlement, entry fee, net profit |
| **Persistence** | Save state and continuity | Save/load of upgrades, unlocks, records |
| **UI** | Player-facing information displays | HUD, menu/upgrade/settlement screens |
| **Audio** | Sound and music systems | Click, number and settlement SFX |
| **Meta** | Systems outside the core game loop | Onboarding, daily challenges, leaderboard, co-op |

---

## Priority Tiers

| Tier | Definition | Target Milestone | Design Urgency |
|------|------------|------------------|----------------|
| **MVP** | Required for the core loop to function. Without these, you can't test "is this fun?" | First playable prototype | Design FIRST |
| **Vertical Slice** | Required for one complete, polished area. Demonstrates the full experience. | Vertical slice / demo | Design SECOND |
| **Alpha** | All features present in rough form. Complete mechanical scope, placeholder content OK. | Alpha milestone | Design THIRD |
| **Full Vision** | Polish, edge cases, nice-to-haves, and content-complete features. | Beta / Release | Design as needed |

No system is currently assigned to Alpha: content-bearing systems land in the
Vertical Slice tier, and the rest are Full Vision.

---

## Dependency Map

### Foundation Layer

1. **数据配置** — every dish, upgrade and station value lives here; nothing else can be tuned without it. No dependencies.
2. **输入** — keyboard/mouse and partial gamepad; no gameplay dependency.
3. **场景与流程管理** — the app-level state machine. Depends only on 数据配置（校验结果）and 输入（上下文切换）, so it stays in this layer; both edges are same-layer and do not change the design order.

### Core Layer (depends on foundation)

1. **存档** — depends on: 数据配置（字段 schema）
2. **餐局流程** — depends on: 场景与流程管理, 数据配置, 输入
3. **时间与回合** — depends on: 餐局流程
4. **胃容量** — depends on: 餐局流程, 数据配置
5. **菜品数值** — depends on: 数据配置

### Feature Layer (depends on core)

1. **腻度与口味** — depends on: 菜品数值
2. **菜单波次** — depends on: 菜品数值, 数据配置
3. **结算与入场费** — depends on: 菜品数值, 腻度与口味, 胃容量, 时间与回合
4. **局外升级** — depends on: 结算与入场费, 存档, 数据配置
5. **解锁与档口** — depends on: 局外升级, 存档
6. **纪录与里程碑** — depends on: 结算与入场费, 存档

### Presentation Layer (depends on features)

1. **局内 HUD** — depends on: 餐局流程, 时间与回合, 胃容量, 菜品数值, 输入, 结算与入场费
2. **界面流程** — depends on: 餐局流程, 结算与入场费, 局外升级, 场景与流程管理, 输入
3. **音效反馈** — depends on: 餐局流程, 结算与入场费

### Polish Layer (depends on everything)

1. **首局引导** — depends on: 餐局流程, 局内 HUD, 输入
2. **在线挑战** — depends on: 结算与入场费, 存档
3. **同屏合作** — depends on: 几乎全部系统（延后）

---

## Recommended Design Order

| Order | System | Priority | Layer | Agent(s) | Est. Effort |
|-------|--------|----------|-------|----------|-------------|
| 1 | 数据配置 | MVP | Foundation | game-designer | S |
| 2 | 输入 | MVP | Foundation | game-designer | S |
| 3 | 场景与流程管理 | MVP | Foundation | game-designer | S |
| 4 | 存档 | MVP | Core | game-designer | S |
| 5 | 餐局流程 | MVP | Core | game-designer | M |
| 6 | 时间与回合 | MVP | Core | game-designer | S |
| 7 | 胃容量 | MVP | Core | game-designer | S |
| 8 | 菜品数值 | MVP | Core | game-designer | M |
| 9 | 腻度与口味 | MVP | Feature | game-designer | M |
| 10 | 结算与入场费 | MVP | Feature | economy-designer | S |
| 11 | 局外升级 | MVP | Feature | systems-designer | M |
| 12 | 局内 HUD | MVP | Presentation | ux-designer | M |
| 13 | 界面流程 | MVP | Presentation | ux-designer | M |
| 14 | 菜单波次 | Vertical Slice | Feature | game-designer | S |
| 15 | 解锁与档口 | Vertical Slice | Feature | systems-designer | M |
| 16 | 纪录与里程碑 | Vertical Slice | Feature | game-designer | S |
| 17 | 音效反馈 | Vertical Slice | Presentation | sound-designer | S |
| 18 | 首局引导 | Vertical Slice | Polish | ux-designer | S |
| 19 | 在线挑战 | Full Vision | Polish | network-programmer | L |
| 20 | 同屏合作 | Full Vision | Polish | network-programmer | L |

---

## Circular Dependencies

- None found.

---

## High-Risk Systems

| System | Risk Type | Risk Description | Mitigation |
|--------|-----------|-----------------|------------|
| 腻度与口味 | Design | May not create enough ongoing tradeoff; the game degrades into a lookup table once solved | Validate in a one-day `/prototype`; keep the cloying coefficient as a tuning knob |
| 菜品数值 | Design | The value space may be too simple — one obvious optimal dish | Prototype with a wider price/satiety distribution and more dishes |
| 结算与入场费 | Design | "回本" may not read clearly; players cannot tell whether they are winning | HUD shows unit value and running net profit directly |
| 菜单波次 | Design | Random menus may destroy the planning fantasy | Tune wave size and information visibility during the Vertical Slice |
| 同屏合作 | Scope | Would blow the scope of a first game | Stays in Full Vision; excluded from MVP and Vertical Slice |

---

## Progress Tracker

| Metric | Count |
|--------|-------|
| Total systems identified | 20 |
| Design docs started | 9 |
| Design docs reviewed | 2 |
| Design docs approved | 1 |

> 场景与流程管理于 2026-11-03 通过设计评审（Approved），随后补写了 Formulas 章节并在
> 输入 GDD 中定名了上下文切换接口，两处都是评审后才出现的改动，因此状态回到 **In Review**
> 等待复审。复审通过后改回 Approved。
| MVP systems designed | 8/13 |
| Vertical Slice systems designed | 1/5 |

---

## Next Steps

- [x] Review and approve this systems enumeration
- [ ] Design MVP-tier systems first (use `/design-system [system-name]`)
- [ ] Run `/design-review` on each completed GDD
- [ ] Run `/gate-check technical-setup` when MVP systems are designed
- [ ] Validate the highest-risk systems with `/vertical-slice` before committing to Production
