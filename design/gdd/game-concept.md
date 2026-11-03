# Game Concept: 回本 (Break Even)

*Created: 2026-11-03*
*Status: Draft*

---

## Elevator Pitch

> 这是一款回合制数值 roguelite：你走进自助餐厅，付一笔入场费，在有限的时间和胃容量里挑出回本最高的组合。每一局结束后，用赚到的钱永久升级胃容量和菜谱。

---

## Core Identity

| Aspect | Detail |
| ---- | ---- |
| **Genre** | 数值优化 roguelite + 回合制经营解谜 |
| **Platform** | PC（Steam / Epic） |
| **Target Audience** | 成就型 / 策略型玩家（详见 Target Player Profile） |
| **Player Count** | 单人（同屏合作推迟到后续版本） |
| **Session Length** | 30–120 分钟（3–6 局） |
| **Monetization** | 尚未决定（倾向买断） |
| **Estimated Scope** | Medium（约 3–6 个月，单人） |
| **Comparable Titles** | Brotato、Balatro、Luck be a Landlord |

---

## Core Fantasy

**我一眼看穿菜单。** 别人在自助餐里乱拿，你算得清每一口值多少钱——用最少的胃装下最贵的东西，然后在收银台前看着净利数字。

玩家在这里得到的是"精算者"的身份感：不是吃得最多，而是吃得最聪明。

---

## Unique Hook

它像 Brotato 的一局一局变强，**而且**每一口都是一道算术题：菜品的价格、饱腹值和腻度让"吃"变成有明确最优解的取舍，而不是拼手速。

这个钩子不是题材包装——它直接决定玩法：所有反馈都是数字，所有压力都来自"还能吃几口"。

---

## Player Experience Analysis (MDA Framework)

### Target Aesthetics (What the player FEELS)

| Aesthetic | Priority | How We Deliver It |
| ---- | ---- | ---- |
| **Sensation** (sensory pleasure) | 3 | 数字跳动、结算音效、饱腹条的即时反馈 |
| **Fantasy** (make-believe, role-playing) | 4 | "大胃王 / 精算食客"的身份，自助餐厅的场景 |
| **Narrative** (drama, story arc) | N/A | 不做剧情 |
| **Challenge** (obstacle course, mastery) | 1 | 每局的最优化决策、越来越紧的容量与时间 |
| **Fellowship** (social connection) | N/A | 首版不做；后续可补同屏合作 |
| **Discovery** (exploration, secrets) | 5 | 新档口、新菜谱、新腻度类型的解锁 |
| **Expression** (self-expression, creativity) | 3 | 吃法流派：纯高价流、解腻流、饮料流 |
| **Submission** (relaxation, comfort zone) | 4 | 回合制，无手速要求，可以慢慢算 |

### Key Dynamics (Emergent player behaviors)

- 玩家会比较"单位饱腹价值"，自发形成心算习惯
- 玩家会为了等下一波更贵的菜而跳过眼前的菜
- 玩家会为"解腻"预留容量，而不是单纯追求最高单价
- 玩家会发现连吃同类不划算，主动混搭
- 玩家之间会分享"最优菜单"和流派

### Core Mechanics (Systems we build)

1. **菜品数值系统**：每道菜有价格、饱腹值、口味标签和腻度曲线
2. **回合制营业时段**：每吃一道菜消耗一个时间格，时间归零即结账
3. **结算与入场费**：总吃进价值 − 入场费 = 本局净利
4. **局外升级**：胃容量档位、解腻能力、菜单信息可见度、新档口
5. **分批取餐窗口**：菜单分波刷新，后面的档口不可全见

---

## Player Motivation Profile

### Primary Psychological Needs Served

| Need | How This Game Satisfies It | Strength |
| ---- | ---- | ---- |
| **Autonomy** (freedom, meaningful choice) | 点菜顺序、流派构筑、何时收手都由玩家决定 | Core |
| **Competence** (mastery, skill growth) | 算术越来越熟、能预判腻度、升级肉眼可见 | Core |
| **Relatedness** (connection, belonging) | 首版单人，无社交系统 | Minimal |

### Player Type Appeal (Bartle Taxonomy)

- [x] **Achievers** — 通过净利纪录、档口解锁和升级树获得完成感
- [x] **Explorers** — 理解并穷尽数值系统，寻找最优组合
- [ ] **Socializers** — 首版不服务；后续同屏合作再补
- [ ] **Killers/Competitors** — 排行榜属于完整愿景，不进 MVP

### Flow State Design

- **Onboarding curve**：第一局只给 6 道菜，价格与饱腹直接可见；结算时展示单位价值
- **Difficulty scaling**：每解锁一个档口，入场费、菜品数量和腻度类型同步上升
- **Feedback clarity**：每吃一道菜立即更新净利；结算页给出明细与历史最佳
- **Recovery from failure**：一局 5–10 分钟，失败即重开，惩罚只落在本局收益

---

## Core Loop

### Moment-to-Moment (30 seconds)

看一道菜（价格 / 饱腹 / 腻度）→ 计算单位价值 → 决定吃、跳过还是等下一波 → 饱腹条与净利数字立即变化。

这一步本身必须有反馈价值：数字即时上涨、心算被验证。若去掉局外成长后这个动作不成立，整个游戏就不成立。

### Short-Term (5-15 minutes)

一局 = 一次自助餐：
- 付入场费进店，营业时段 = 有限回合数
- 每拿一盘消耗 1 个时间格和一部分胃容量
- 取餐窗口分批刷新，后续档口不可全见
- 结账：吃进总价值 − 入场费 = 本局净利

"再来一局"从"这局只差 3 块就回本，换个点法就行"开始。

### Session-Level (30-120 minutes)

3–6 局。每局结束看账、消费利润升级，然后在升级完成或资金不足时自然停下。局与局之间是明确停点。

### Long-Term Progression

局外永久成长：胃容量档位、解腻/消化能力、菜单信息可见度、新档口与新菜谱。长期目标是吃遍所有档口、达成里程碑；完成全部档口挑战即视为通关。

### Retention Hooks

- **Curiosity**：未解锁的档口和菜谱
- **Investment**：已投入的升级与纪录
- **Social**：首版没有；完整愿景中的排行榜补足
- **Mastery**：最高净利纪录、对数值系统的完整理解

---

## Game Pillars

### Pillar 1: 每口都是算术题

每个决定都应该能被玩家算清、并在结算时得到验证。

*Design test*：如果玩家无法在几个数字之间心算后做决定，这个设计就砍掉。

### Pillar 2: 没有白吃的菜

每个选择都必须挤掉别的选择，没有无成本的纯收益。

*Design test*：如果某个选项明显无脑最优，就削弱它或给它的成本加码。

### Pillar 3: 时间在走，手不用快

压力来自"还剩几个时间格"，不来自反应速度。

*Design test*：任何需要手快才能赢的方案，一律否决。

### Pillar 4: 数字看得见地涨

每次决定后立刻有明确、可读的反馈。

*Design test*：一个改动如果让反馈变模糊（动画过长、数值不可读），就否决。

### Anti-Pillars (What This Game Is NOT)

- **NOT 实时操作、闪避、微操**：会直接毁掉"精算"这一支柱
- **NOT 恐怖、丧尸、血腥题材**：与"自助餐"的轻松基调冲突，也会劝退目标玩家
- **NOT 需要查攻略才能理解的隐藏公式**："看得懂"是支柱的前提
- **NOT 首版同屏合作或联机**：几周的第一款游戏撑不起它，放进完整愿景

---

## Inspiration and References

| Reference | What We Take From It | What We Do Differently | Why It Matters |
| ---- | ---- | ---- | ---- |
| Brotato | 短局、局外成长的数值爽感 | 核心动作从"战斗"换成"点菜取舍" | 验证数值驱动的短局循环可行 |
| Balatro | 倍率与构筑带来的"再来一局" | 用价格/饱腹/腻度代替牌面倍率 | 验证纯数字决策的深度 |
| Luck be a Landlord | 槽位限制下的连锁优化 | 限制从槽位换成胃容量与时间格 | 验证"有限空间里做最优解" |
| 杯杯倒满 | 餐饮题材的亲近感与节奏 | 去掉实时操作，改为回合制 | 验证题材吸引相同玩家 |

**Non-game inspirations**：自助餐的价目板、餐厅收据、大胃王比赛节目。

---

## Target Player Profile

| Attribute | Detail |
| ---- | ---- |
| **Age range** | 16–35 |
| **Gaming experience** | Mid-core |
| **Time availability** | 工作日 30–60 分钟，周末更长 |
| **Platform preference** | PC |
| **Current games they play** | Brotato、Balatro、星露谷物语 |
| **What they're looking for** | 有明确目标、靠思考变强、不考验手速的短局游戏 |
| **What would turn them away** | 高操作密度、恐怖题材、需要长时间学习 |

---

## Technical Considerations

| Consideration | Assessment |
| ---- | ---- |
| **Recommended Engine** | Godot 4（用户已选） |
| **Key Technical Challenges** | 数值平衡与"被算穿"后的可玩性；菜单生成的多样性 |
| **Art Style** | 2D 平面色块 + 食物剪影 |
| **Art Pipeline Complexity** | Medium（自绘 2D 图标，数量可控） |
| **Audio Needs** | Minimal（点击、数字、结算音效） |
| **Networking** | None |
| **Content Volume** | 24–40 道菜、3–5 个档口、8–12 项升级、2–4 小时内容 |
| **Procedural Systems** | 菜单与档口的随机组合 |

---

## Risks and Open Questions

### Design Risks

- **最大风险**：数值一旦被算穿，游戏退化成查表。需要"腻度"和随机菜单持续制造取舍。
- 单局可能太短，玩家感受不到成长；需要靠局外升级节奏补偿。
- "时间压力"如果做成实时倒计时，会违背支柱 3。

### Technical Risks

- 低：回合制 2D UI，无物理、无网络，Godot 足以覆盖。

### Market Risks

- 中：同类数值 roguelike 竞争激烈（Balatro、Brotato 等）。
- 差异化依赖"自助餐"题材与一眼可懂的目标——这需要在商店页和首屏直接表达。

### Scope Risks

- 几周做第一款游戏，任何"多档口 + 图鉴 + 合作"的组合都会撑爆时间。
- 菜品数量直接决定美术工时，必须严格按 MVP 上限执行。

### Open Questions

- "腻度"是否足以制造持续取舍？→ 用 `/prototype` 做一天原型验证。
- 一局的理想回合数是多少？→ 原型中测试 8 / 10 / 12 回合。
- 局外升级的强度曲线该多陡？→ 垂直切片阶段再定。

---

## MVP Definition

**Core hypothesis**：玩家在"有限胃容量 + 有限时间"下做单位价值取舍，会感到精算的快感，并想再来一局。

**Required for MVP**：
1. 一个档口菜单池：12 道菜，带价格、饱腹值、腻度
2. 回合制营业时段（10 回合）+ 入场费结账
3. 局外 3 项升级：容量 +1、解腻、菜单信息
4. 重开按钮，一局可以完整重复

**Explicitly NOT in MVP**（defer to later）：
- 多档口与档口主题
- 菜品图鉴与里程碑
- 同屏合作与排行榜
- 每日挑战

### Scope Tiers (if budget/time shrinks)

| Tier | Content | Features | Timeline |
| ---- | ---- | ---- | ---- |
| **MVP** | 1 档口、12 道菜、3 项升级 | 核心循环 + 结账 + 重开 | 2–3 周 |
| **Vertical Slice** | 2 档口、24 道菜 | 完整升级树、结算与反馈打磨 | 4–6 周 |
| **Alpha** | 3–4 档口、全部升级 | 全部系统，占位美术 | 8–12 周 |
| **Full Vision** | 4 档口、40 道菜、图鉴 | 里程碑、每日挑战、排行榜 | 3–6 个月 |

---

## Visual Identity Anchor

**方向名**：食堂价目板

**视觉规则**：屏幕上的一切都像一块手写价目板——大数字、粗价格标签、克制的色块。

**支撑原则与设计测试**：
1. **数字优先**：价格与饱腹值的字号必须大于食物图标。测试：如果一眼看不清数字，就改布局。
2. **食物是剪影不是写实**：高饱和色块 + 简单形状。测试：如果需要画细节才能辨认，就退回剪影。
3. **界面像收据**：等宽数字、粗分隔线。测试：装饰元素若干扰读数，删掉。

**色彩哲学**：米白纸底（菜单/价目板）为主；"回本红"只用于正收益与关键数字；橄榄绿表示胃容量与安全区。避免大面积暖黄——会让界面显得廉价，也会削弱数字的对比度。

---

## Next Steps

- [ ] `/setup-engine`：确认 Godot 版本与引擎配置
- [ ] `/prototype 点菜取舍与腻度`：一天内验证核心假设
- [ ] PROCEED 后：`/art-bible`、`/map-systems`、`/design-system`（按 standard 流程）
- [ ] `/design-review design/gdd/game-concept.md`
- [ ] `/gate-check` 进入下一阶段
