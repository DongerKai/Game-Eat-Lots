# ADR-0006: 确定性随机、种子派生与波次健康校验预算

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。其他 agent 与任何 skill 都不能自行设置。**

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + agents（technical-director 授权；godot-specialist 引擎校验；编排会话合并登记）

## Summary

《回本》的核心承诺之一是「同一档口、同一种子，在任何平台、任何帧序下得到同一份菜单」。本 ADR 把这句话落成三个可测试的契约：**用 `(station_id, seed, attempt)` 经 32 位分道 FNV-1a 派生出唯一的 int64 种子**；**用「id 升序规范序 + 反向 Fisher–Yates + 拒绝采样」在纯整数域内洗牌并按 `wave_size` 切片**；**用逐波重置的背包 DP 在 ≤ 10⁴ 个节点内算出 `wave_reachable_opt`，以整数交叉相乘形式与 `0.9 × opt_value` 比较，最多重掷 20 次，超预算或超次数都不阻断开局**。所有随机性与浮点都被排除在划分与健康判定之外：`RandomNumberGenerator` 只是被显式播种的字节流来源，且其输出有金标准夹具兜底。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | Core |
| **Layer** | Feature |
| **Knowledge Risk** | HIGH — `RandomNumberGenerator` 的 PCG 实现细节不在 LLM 训练截止（2025-05）之内可验证的范围内；`VERSION.md` 把 4.5/4.6/4.7 全部标为 HIGH，且本机没有编辑器可实测 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`、`breaking-changes.md`、`deprecated-apis.md`；`design/gdd/menu-waves.md`、`dish-values.md`、`data-config.md`、`run-flow.md`；`docs/architecture/architecture.md`（原则 5、`MenuWaves` 边界、TR 表）；Godot 4.7 API 文档 `RandomNumberGenerator`、`PackedStringArray`、`String.to_utf8_buffer` |
| **Post-Cutoff APIs Used** | 无新增 API 依赖。仅用 `RandomNumberGenerator.seed` / `randi()`（4.0 起稳定）、`String.to_utf8_buffer()`、`PackedStringArray.sort()`。刻意**不**使用 `Array.shuffle()`（内部使用全局 RNG）、`randi_range()`（内部取模，偏置不可控）、`String.hash()`（跨版本不保证稳定） |
| **Verification Required** | ① `RandomNumberGenerator.seed` 赋值后 `randi()` 的前 8 个输出在 4.7.x 上等于本 ADR 记录的金标准；② 同一 `seed` 在 Windows / macOS / Linux 三平台输出逐位相同（有符号 int64 赋值不改变位模式）；③ `randi()` 的返回值域确为 `[0, 2³²−1]`（若为有符号 32 位，`& 0xFFFFFFFF` 后仍等价，但需实测确认）；④ `StringName` → `String` → UTF-8 字节对全部合法 id 稳定；⑤ `PackedStringArray.sort()` 对 ASCII id 为码点升序（不受区域设置影响） |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。本 ADR 的每条引擎断言都必须先按 `Verification Required` 逐条实测；未通过前不得写依赖该断言的 story。金标准夹具（①）是唯一能在引擎悄悄更改 RNG 算法时把问题变成红灯的手段。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0001（内容数据格式）— `dish_pool`、`wave_size`、`price`、`satiety` 都从 `DataConfig` 只读取得，且 `Ready` 是 `initialize` 的前置条件。ADR-0004（服务组合）— 组合根负责把 `DataConfig` 注入 `MenuWaves`，本 ADR 不规定构造顺序 |
| **Enables** | 菜单波次（#10）的全部实现 story；局内 HUD 的波次条、`?` 占位与预览（HUD 只读本 ADR 的 `visible_dishes` / `preview_dishes` / `hidden_counts`）；餐局流程的开局编排（`TR-run-flow-002`）；局外升级对 `MENU_VISION` 的价值估算 |
| **Blocks** | 任何依赖「同一种子复现同一菜单」的验收（AC 1–3、18）；任何需要重掷语义的平衡校验；教学档口的 `pool_override` 流程 |
| **Ordering Note** | 本局种子的**来源**（谁生成、是否展示、是否持久化）不归本 ADR：MVP 不保存种子（`save-system.md` OQ7 已回答），种子由「餐局流程」在开局时生成并一次性传入。本 ADR 只规定「拿到种子之后的一切必须是纯函数」 |

## Context

### Problem Statement

`menu-waves.md` 把「确定性」写成一句要求（规则 10、Edge Case「同一 `(station_id, seed)` 在不同平台生成」），却没有说明确定性从哪一层开始。如果只写「用种子的 `RandomNumberGenerator` 洗牌」，仍有三处会破坏可复现性：**种子怎么从字符串派生**（`String.hash()` 不稳定）、**洗牌的具体算法**（正序/逆序 Fisher–Yates 结果不同，`Array.shuffle()` 用的是全局 RNG 且不暴露中间状态）、**健康校验的浮点阈值**（`0.9 × 104` 在 float 下可能得到 `93.60000000000001`）。这三处任一漂移，跨平台基准夹具（AC 3）就会变成一条永远修不好的红灯。

同时，健康校验的**成本**必须被固定：`wave_reachable_opt` 是逐波重置 DP，状态数随 `(C0+1)×(T+1)` 增长；没有节点预算，一个后续档口的参数就能让开局卡住几百毫秒，而 GDD 要求「检查函数不抛异常、不阻断开局」（AC 16、23）。所以本 ADR 必须同时回答「算什么」和「算不完怎么办」。

### Current State

没有代码，没有既有实现。`architecture.md` 已固定 `MenuWaves` 的对外方法（`initialize` / `visible_dishes` / `preview_dishes` / `consume` / `advance`）与 `initialize` 的参数表；`menu-waves.md` 已固定公式 `wave_count = ceil(|M| / wave_size)`、`W_i = shuffled(M, seed)[i·wave_size : (i+1)·wave_size]`、`preview_dishes = min(menu_vision, wave_size−1, |W_{i+1}|)`、`advance_cost ∈ {0,1}`，以及 `wave_reachable_opt` 的五条转移。缺的是这些公式在 GDScript 里的**唯一实现**：哈希常量、洗牌循环方向、DP 的遍历顺序、节点如何计数、阈值如何避免浮点。

### Constraints

- **纯整数**：`architecture.md` 原则 5 要求「整数运算、种子 RNG、逻辑中不读墙钟」。划分与健康判定中不得出现任何 `float` 运算或 `float` 比较。
- **一次划分、永不改变**：`menu-waves.md` 规则 2/11。划分在 `initialize` 内完成并冻结，局内任何操作都不重排。
- **不依赖容器顺序**：不得依赖 `Dictionary` 遍历、`res://` 目录扫描顺序或资源加载顺序（规则 10 的 Edge Case）。
- **节点预算**：单次可达检查 ≤ 10⁴ 节点；波界状态 ≤ `(wave_count+1)×(C0+1)×(T+1)`（MVP `4×21×11 = 924`）。
- **不阻断**：健康校验失败只重掷，重掷用尽就采用最好的一次并写 warning；任何路径都不得抛异常、不得拒绝开局。
- **只读上游**：本系统不改任何菜的价格/饱腹，不读也不写腻度状态（规则 1、14）。
- **单人单线程**：MVP 无后台线程，校验在开局那一帧同步完成。

### Requirements

- 给定 `(station_id, seed)`，候选序列（含每一次重掷）必须完全由这两个值决定，不能依赖尝试顺序之外的任何状态。
- 给定 `(station_id, seed, attempt)`，划分必须逐项相等——同一构建内两次开局、以及不同平台之间。
- 洗牌必须是**真正的均匀排列**（每个排列等概率），不是「按哈希排序」这类近似。
- 健康判定必须给出三值 `HEALTHY` / `DEGRADED` / `UNHEALTHY`，且 `DEGRADED` 的触发条件是「可达检查未完成」，不是「结果不好」。
- 节点预算耗尽必须**可观测**（warning + `DEGRADED`），不得静默降级。
- `pool_override` 必须同时作用于划分与健康校验，且其基准是本局实际菜池。
- 结果类型必须有封闭错误词汇，失败不得用 `null` / 空数组冒充。

## Decision

### 1. 种子派生：32 位分道 FNV-1a，输入为 `(station_id, seed, attempt)`

**规范输入字节串**（顺序固定，缺一不可）：

```
B = UTF8(String(station_id)) ‖ 0x00 ‖ LE64(seed) ‖ LE16(attempt)
```

- `UTF8(...)`：id 语法是 `^[a-z][a-z0-9_]*$`（`data-config.md`），全部 ASCII，因此 UTF-8 与字节等价；仍按 UTF-8 取字节，以便未来放宽 id 规则时不改变实现。
- `0x00` 是域分隔符，防止 `("a", …)` 与 `("a\u0000…")` 碰撞（当前语法不会产生，但成本为零）。
- `LE64(seed)`：int64 的二进制补码小端 8 字节。
- `LE16(attempt)`：候选序号，`0` 为首次，`1..20` 为第 1..20 次重掷。

**哈希**：对 `B` 做两条独立车道的 32 位 FNV-1a，再折成一个有符号 int64。

```gdscript
class_name SeedHash extends RefCounted

const FNV_OFFSET: int = 0x811C9DC5
const FNV_PRIME: int = 0x01000193
const LANE_B_SALT: int = 0x9E3779B9
const MASK32: int = 0xFFFFFFFF
const SIGN32: int = 0x80000000
const TWO32: int = 0x100000000

static func _fnv1a32(bytes: PackedByteArray, offset: int) -> int:
    var h: int = offset & MASK32
    for b in bytes:
        h = ((h ^ int(b)) * FNV_PRIME) & MASK32   # 乘积 < 2^56，int64 内精确
    return h

static func _le64(v: int) -> PackedByteArray:
    var out := PackedByteArray()
    out.resize(8)
    var u: int = v
    for i in 8:
        out[i] = u & 0xFF
        u = u >> 8            # 负数为算术右移，低字节仍是补码字节
    return out

static func _le16(v: int) -> PackedByteArray:
    var out := PackedByteArray()
    out.resize(2)
    out[0] = v & 0xFF
    out[1] = (v >> 8) & 0xFF
    return out

static func candidate_seed(station_id: StringName, run_seed: int, attempt: int) -> int:
    var b := String(station_id).to_utf8_buffer()
    b.append(0x00)
    b.append_array(_le64(run_seed))
    b.append_array(_le16(attempt))
    var lo: int = _fnv1a32(b, FNV_OFFSET)
    var hi: int = _fnv1a32(b, FNV_OFFSET ^ LANE_B_SALT)
    return _fold_u64(hi, lo)

static func _fold_u64(hi: int, lo: int) -> int:
    # 把 (hi, lo) 解释为无符号 64 位，再无损映射到有符号 int64。
    if hi >= SIGN32:
        return (hi - TWO32) * TWO32 + lo     # 负数侧，量级 < 2^63
    return hi * TWO32 + lo                   # 正数侧，量级 < 2^63
```

**为什么不用 `String.hash()`**：它返回的是引擎内部的字符串哈希，没有跨版本稳定性承诺；一旦升级引擎，所有历史基准夹具失效，而玩家感知是「同一个种子换了菜单」。`_fold_u64` 把两条 32 位车道拼成 64 位，逐位确定，且完全避开「有符号 64 位乘法溢出是否 wrap」这个 GDScript 未明确定义的灰区（两条车道都 < 2³²，乘法 < 2⁵⁶）。

**尝试序号参与哈希**是确定性的关键：重掷不推进任何 RNG 状态，第 `k` 个候选只由 `(station_id, seed, k)` 决定。因此「20 次重掷之后选了哪一个」也是 `(station_id, seed)` 的纯函数，可以整段作为基准夹具。

### 2. 规范序、稳定洗牌与均匀性

**规范序**：进入洗牌前，把菜池 id 按**码点升序**排序（`PackedStringArray.sort()`，ASCII 子集即字节序）。理由：

1. `dish_pool` 是作者写的 `Array[StringName]`，其顺序不是契约的一部分；把它固定成规范序，使划分成为「集合 + 种子」的纯函数，而不是「数组书写顺序 + 种子」的函数。
2. `pool_override` 常由运行时构造（教学流程、测试夹具）。若调用方用 `Dictionary.keys()` 构造，顺序就随哈希表实现漂移；规范序直接消灭这条依赖。
3. 与 `dish-values` 的「id 升序」排错顺序一致，日志与夹具可直接比对。

> 这条不改变 `menu-waves.md` 的语义：GDD 说「`M` 的顺序即洗牌前的输入」，本 ADR 只是把该顺序**定义为**规范序。`wave_count`、切片、预览、可达最优都与输入顺序无关（可达最优只依赖集合），因此唯一受影响的是「哪道菜进哪一波」，而那本来就是随机结果。

**洗牌**：反向 Fisher–Yates，索引从 `n−1` 递减到 `1`。

```gdscript
static func shuffled(ids: PackedStringArray, stream: SeededStream) -> PackedStringArray:
    var out := ids.duplicate()
    for i in range(out.size() - 1, 0, -1):
        var j: int = stream.below(i + 1)
        if j != i:
            var t := out[i]
            out[i] = out[j]
            out[j] = t
    return out
```

方向固定为**反向**（从尾到头）。正向与反向都能产生均匀排列，但结果不同；把它写死在 ADR 里，任何重写都不得改成正向。

**拒绝采样**，不用取模：

```gdscript
class_name SeededStream extends RefCounted

var _rng: RandomNumberGenerator
var _draws: int = 0

func _init(seed64: int) -> void:
    _rng = RandomNumberGenerator.new()
    _rng.seed = seed64          # 只在这里播种；绝不使用全局 randi()

func draw_count() -> int:
    return _draws

func next_u32() -> int:
    _draws += 1
    return _rng.randi() & 0xFFFFFFFF

func below(n: int) -> int:
    # 均匀返回 [0, n)。n 必须 > 0；用拒绝采样消除取模偏置。
    var limit: int = (SeedHash.TWO32 / n) * n
    while true:
        var r: int = next_u32()
        if r < limit:
            return r % n
```

`limit = floor(2³² / n) × n` 是 ≤ 2³² 的最大 `n` 的倍数；丢掉 `[limit, 2³²)` 后余数严格均匀。拒绝循环的期望次数 < 2，且完全由字节流决定，因此仍是确定性的。`n ≤ 8`（`wave_size` 上限），`limit` 与 `TWO32` 都在 int64 内精确。

**为什么不用 `Array.shuffle()`**：它内部走全局 RNG（`randomize()` 之后不可复现），且不返回消耗了多少次抽样；基准夹具无法定位漂移。**为什么不用 `randi_range()`**：其内部实现可能取模，偏置随引擎版本变化，且属于未文档化的实现细节。**为什么不用「按 `hash(dish, seed)` 排序」**：排序只能产生「按哈希值的排列」，不是均匀分布，且比较器需要全序（哈希冲突时必须再引入 id 兜底），复杂度与不可解释性都更高。

### 3. 划分与可见度：一次性冻结

```gdscript
func _build_partition(pool_sorted: PackedStringArray, wave_size: int, seed64: int) -> Array[PackedStringArray]:
    var stream := SeededStream.new(seed64)
    var order := SeedHash.shuffled(pool_sorted, stream)
    var waves: Array[PackedStringArray] = []
    var i := 0
    while i < order.size():
        var end: int = mini(i + wave_size, order.size())
        waves.append(order.slice(i, end))
        i = end
    return waves
```

- `effective_wave_size = clamp(station.wave_size, 1, |M|)`；`|M| < wave_size` 时退化为 1 波（`menu-waves.md` Edge Case），并发 `WAVE_SIZE_CLAMPED` warning。`wave_size < 1` 由数据配置在加载期拒绝；本系统再防御一次，返回 `INVALID_WAVE_SIZE`。
- `wave_count = ceil(|M| / effective_wave_size)`，末波只含余数，不补齐、不重复。
- `visible_dishes()` 返回当前波中**尚未被吃**的 `DishData`，顺序即波内顺序；`preview_dishes()` 返回 `W_{i+1}` 的前 `min(menu_vision, wave_size−1, |W_{i+1}|)` 道；最后一波返回空数组。
- `hidden_counts()` 返回 `[|W_{i+1}| − preview, …]`，供 HUD 画 `?` 槽；它只是 `|W_j|` 的算术，不泄露内容。
- `menu_vision < 0` 钳制为 0 并发 `MENU_VISION_CLAMPED`。`menu_vision` 在 `initialize` 时冻结，局中传入的新值一律忽略（规则 11）。

### 4. 健康基准：`opt_value` 纯函数，按菜池指纹记忆化

`wave_healthy` 的下界是 `0.9 × opt_value(本局实际菜池, C0, T)`。本 ADR 的实现方式：

```gdscript
# 与 dish-values.md 的 V[i][c][t] 完全同一条递推，只在本系统内部使用。
# 结果不离开 health()，不进存档、不上 HUD、不参与结算。
static func opt_value(pool: Array[DishData], c0: int, t: int) -> int:
    var w: int = c0 + 1
    var v := PackedInt64Array()
    v.resize(w * (t + 1))
    v.fill(0)
    for d in pool:                      # pool 已按 id 升序，顺序固定
        var s: int = d.satiety
        for c in range(c0, s - 1, -1):
            for k in range(t, 0, -1):
                var cand: int = d.price + v[(c - s) * (t + 1) + (k - 1)]
                if cand > v[c * (t + 1) + k]:
                    v[c * (t + 1) + k] = cand
    return v[c0 * (t + 1) + t]
```

- **记忆化键**：`pool_fingerprint = SeedHash.fingerprint_ids(pool_sorted)`——把规范序 id 用 `0x1F` 连接成字节串，走同一组 32 位车道（域分隔符用 `&"pool"`），得到 64 位指纹；再与 `(C0, T)` 组成 `Dictionary` 键。每个进程内每个菜池最多算一次；20 次重掷共用同一基准。
- **预算**：`|pool| × (C0+1) × (T+1)` 个格子。超过 `BASELINE_CELL_BUDGET = 65536` 时不计算，基准记为 `−1`（不可用），健康判定退化为「非空 + 覆盖」并返回 `DEGRADED` + `BASELINE_BUDGET_EXCEEDED`。MVP：`12 × 21 × 11 = 2772`（全池 104 的模型是 104 道菜的 Full Vision，仍只有 24 024）。
- **与 GDD「构建期预计算表」的关系**：GDD 规则 9 的意图是「不要每次开局重算全池最优」。本 ADR 用「进程内一次性记忆化 + 同一纯函数」达成同一目的，而不引入一张需要导出依赖的生成物表（`pool_override` 是任意子集，构建期无法穷举）。构建期脚本调用**同一个** `opt_value` 生成 `design/balance/opt_value.csv` 供平衡报告与 CI 断言；两处结果必须逐值相等，这本身就是一条测试。**语义没有改变**：基准仍是 `opt_value(实际菜池, C0, T)`。
- **不暴露**：`opt_value` 是 `MenuWaves` 的 `static` 内部函数，不进入公开 API，不返回给 HUD/结算/存档。`dish-values` 的 `TR-dish-values-003`（运行期不做最优求解、不暴露 `opt_value`）指的是**不提供玩法查询接口**；本 ADR 的健康门是唯一例外，且结果不外流。该例外需在 `dish-values.md` 下次修订时在其「接口所有权」段落补一句，不改本 ADR。

### 5. 逐波重置 DP：状态、转移与节点预算

**状态**：`best[i][c][t]` = 处理完前 `i` 波、已用容量 `c`、已用回合 `t` 时的最大已吃价值；`−1` 表示不可达。`i ∈ [0, wave_count]`、`c ∈ [0, C0]`、`t ∈ [0, T]`。

```
idx(i, c, t) = (i × (C0+1) + c) × (T+1) + t
best[idx(0, 0, 0)] = 0
```

**每波预处理**（不计入节点）：对该波 `W_i`（`n = |W_i| ≤ 8`）枚举 `2ⁿ` 个子集，预存 `sat[mask]`、`cnt[mask]`、`val[mask]`。用最低位递推，避免 `2ⁿ × n` 的重复求和：

```
low  = mask & (-mask)
bit  = ctz(low)                     # 预存 bit_index[low]
prev = mask ^ low
sat[mask] = sat[prev] + satiety[bit]
cnt[mask] = cnt[prev] + 1
val[mask] = val[prev] + price[bit]
```

**主循环**（`answer` 为终止值，初值 `0`）：

```
for i in 0 .. wave_count-1:
    for each (c, t) with best[i][c][t] >= 0:
        for mask in 0 .. 2^n-1:
            nodes += 1
            if nodes > NODE_BUDGET: return TRUNCATED
            if c + sat[mask] > C0: continue
            if t + cnt[mask] > T: continue
            var score = best[i][c][t] + val[mask]
            if i == wave_count-1:
                answer = max(answer, score)                 # 末波结束，不扣跳过
            elif cnt[mask] == n:
                relax(i+1, c + sat[mask], t + cnt[mask], score)   # 吃完免费推进
            elif t + cnt[mask] < T:
                relax(i+1, c + sat[mask], t + cnt[mask] + 1, score)  # 付 1 回合跳过
            else:
                answer = max(answer, score)                 # 非末波中途回合耗尽，免费终止
```

这五条与 `menu-waves.md` Formulas 的五条转移**一一对应**，不得合并或重排。`wave_reachable_opt = answer`。

**节点计数口径**：一个节点 = 一次「可达 carry 状态 × 一个子集」的评估。子集预处理不计入；`relax` 不计入；空波提前返回不计入。因此 MVP 的节点数有明确上界：首波 1 个 carry 状态 × 16 个子集 = 16 节点；次波 ≤ 16 个可达状态 × 16 = 256；末波 ≤ 256 × 16 = 4096；合计 ≈ 4368 < 10⁴。而「每波都满 256 子集」的病态上界会被预算截断而不是拖死开局。

**硬预算**：

- `NODE_BUDGET = 10000`（GDD 上限）。
- `STATE_CELL_BUDGET = 262144`：`(wave_count+1)×(C0+1)×(T+1)` 超过即**不分配**，直接 `TRUNCATED`。防止后续档口的 `T = 999` 把 `PackedInt64Array` 撑到几十 MB。
- `TRUNCATED` 的三个来源（节点超限、格子超限、波数 × 子集提前可判超限）统一返回同一个哨兵，调用方不区分。

### 6. 健康判定与重掷：整数阈值，绝不浮点

**阈值**用交叉相乘，不用 `0.9`：

```
reachable × 10 < baseline × 9   →  低于下界（UNHEALTHY）
```

等价于 `reachable ≥ ceil(9 × baseline / 10)`。MVP：`baseline = 104` → 需要 `reachable ≥ 94`；`95×10 = 950 ≥ 936` 通过，`93×10 = 930 < 936` 不通过，与 AC 15/16 完全一致。**不得**写成 `reachable < 0.9 * baseline`：float 在 `93.6` 附近的行为依赖舍入，且违反原则 5。

**判定顺序**（严格按此顺序，先命中先返回）：

```gdscript
enum Health { HEALTHY, DEGRADED, UNHEALTHY }

static func judge(partition, pool: Array[DishData], base: int, reach: ReachEval) -> Health:
    if has_empty_wave(partition) or not covers_exactly_once(partition, pool):
        return Health.UNHEALTHY                 # 生成有 bug，必须重掷
    if reach.truncated or base < 0:
        return Health.DEGRADED                  # 算不完 ≠ 算出来不好
    if reach.value * 10 < base * 9:
        return Health.UNHEALTHY
    return Health.HEALTHY
```

- `covers_exactly_once` 用计数数组与规范序逐一比对，不遍历 `Dictionary`。
- **`DEGRADED` 不重掷**（AC 23：注入超预算生成器 → 不重掷、不抛异常、本局正常开始）。重掷的语义是「换一个候选」，而超预算是「这台机器上算不出」，换候选只会再超一次。

**重掷循环**：

```gdscript
const MAX_ATTEMPTS: int = 21          # 首次 + 20 次重掷

var best: HealthEval = null
var best_attempt: int = -1
for attempt in MAX_ATTEMPTS:
    var s64 := SeedHash.candidate_seed(station_id, run_seed, attempt)
    var part := _build_partition(pool_sorted, ws, s64)
    var ev := _evaluate(part, pool, c0, t)
    if ev.health == Health.HEALTHY:
        return _ok(ev, attempt)
    if ev.health == Health.DEGRADED:
        return _ok(ev, attempt)          # 不重掷
    if best == null or ev.reachable_opt > best.reachable_opt:
        best = ev; best_attempt = attempt          # 严格大于：并列取更早的候选
    # UNHEALTHY 继续
return _ok(best, best_attempt, WarningCode.HEALTH_BEST_EFFORT_AFTER_REROLLS)
```

- **并列取更早**：`>` 而不是 `>=`，保证结果与候选顺序绑定，可复现。
- **用尽 20 次**：采用得分最高的一次，写**恰好一条** warning（AC 16），`attempts = 21`，本局照常开始。
- **永不抛异常**：所有失败路径都是返回值；`assert` 不参与输入校验（导出构建会剥离）。
- `attempts` 是诊断字段：`1` 表示一次通过（AC 15），`21` 表示重掷用尽。

### 7. 所有权、状态机与扣除回合的边界

| 关注点 | 所有者 | 本系统的义务 |
|---|---|---|
| 本局种子、`turn_budget`、`menu_vision` 的来源 | 餐局流程 | 只消费，校验 `T ≥ 1`，开局后冻结 |
| `dish_pool`、`wave_size`、`price`、`satiety` | 数据配置 / 菜品数值 | 只读；不改值、不写腻度 |
| 划分、波索引、本波剩余、可见/预览 | **菜单波次（本 ADR）** | 唯一写者；HUD 只读 |
| 回合计数与扣除 | 时间与回合 | **不持有**计数器；`advance()` 不扣回合 |
| 健康基准与可达 DP 的算法 | **菜单波次（本 ADR）** | 纯函数；结果不外流 |
| 「无可吃菜」信号与结束原因 | 餐局流程 | `consume` 返回 `NO_EATABLE` / `end_pickup` 返回 `ENDED`，由餐局流程结束本局 |

**动作语义**（`advance()` 的扣费顺序是硬约束）：

- `consume(dish_id)`：从当前波移除；若本波变空 → 非末波返回 `ADVANCED`（免费推进），末波返回 `NO_EATABLE`。本波仍有菜 → `NONE`。
- `advance()`：**只用于非末波的付费跳波**。它丢弃本波剩余菜、`wave_index += 1`，**不扣回合**。扣回合由餐局流程在调用前完成：`if Time.can_spend_turn(): Time.spend_turn(); MenuWaves.advance()`。两者之间没有可失败的步骤，因此不需要回滚。末波调用返回 `FINAL_WAVE_USE_END_PICKUP`，不扣回合、不推进。
  GDD 规则 5 说「本系统只发起扣除」，指的是**计数器不归本系统**；「发起」体现在动作链路上——`advance()` 是一个有回合代价的动作，由餐局流程在处理 `skip_wave` 意图时按固定顺序触发扣费与推进。若让 `advance()` 自己调用 `TimeTurns.spend_turn()`，本系统就获得了第二个写入点，且「扣了回合但推进失败」需要回滚，与 `TR-run-flow-003` 的原子提交相冲突。
- `end_pickup()`：仅末波合法，丢弃剩余菜并返回 `ENDED`；非末波返回 `NOT_FINAL_WAVE`。餐局流程由此结束本局并发出一次「无可吃菜」。
- `health()`：返回 `initialize` 冻结下来的 `HealthEval`（含 `attempts`、`reachable_opt`、`baseline_opt`、`nodes`、`truncated`），不重新计算。

**`initialize` 恰好一次**：第二次调用返回 `ALREADY_INITIALIZED`，不修改已有划分（`TR-run-flow-002` 的「一帧内按序初始化、失败不发 `RunResult`」）。

## Architecture Diagram

```
RunFlow.initialize(RunConfig)
  │  seed / turn_budget / menu_vision
  ▼
MenuWaves.initialize(station_id, seed, T, menu_vision, pool_override)
  │
  ├─ DataConfig.get_station(station_id) ──► dish_pool / wave_size      （只读）
  ├─ canonicalize(M)  ──►  id 码点升序
  │
  ├─ for attempt in 0..20:
  │     seed64 = SeedHash.candidate_seed(station_id, seed, attempt)    ← 纯函数
  │     partition = SeededStream(seed64) → Fisher–Yates → slice
  │     HealthGate.judge(
  │         baseline = opt_value(pool, C0, T)        ← 进程内记忆化，不外流
  │         reach    = wave_reset_dp(partition, C0, T, nodes ≤ 1e4)
  │     ) → HEALTHY | DEGRADED | UNHEALTHY
  │
  └─ WaveInitResult { error, partition, health, attempts, warnings }

局内
  HUD ── visible_dishes() / preview_dishes() / hidden_counts()  （只读投影）
  RunFlow ── consume() / advance() / end_pickup()               （唯一写入口）
  TimeTurns ── spend_turn()                                     （回合唯一所有者）
```

## Key Interfaces

> `architecture.md` 的 `Result[T, E]` 是设计级记法；实现用下列 `RefCounted` 结果类型承载同一语义。每个 `class_name` 各占一个 `.gd`；此处合并展示只为阅读。

```gdscript
# Feature — src/feature/menu_waves/

class_name MenuWaves extends RefCounted

const NODE_BUDGET: int = 10000
const STATE_CELL_BUDGET: int = 262144
const BASELINE_CELL_BUDGET: int = 65536
const MAX_ATTEMPTS: int = 21          # 首次 + 20 次重掷
const LOWER_BOUND_NUM: int = 9
const LOWER_BOUND_DEN: int = 10

enum Health { HEALTHY, DEGRADED, UNHEALTHY }

enum WaveError {
    OK,
    DATA_NOT_READY,
    UNKNOWN_STATION,
    EMPTY_POOL,
    DUPLICATE_POOL_ID,
    INVALID_WAVE_SIZE,
    INVALID_TURN_BUDGET,
    OVERRIDE_UNKNOWN_DISH,
    OVERRIDE_DUPLICATE_DISH,
    ALREADY_INITIALIZED,
    NOT_INITIALIZED
}

enum WarningCode {
    HEALTH_BEST_EFFORT_AFTER_REROLLS,
    HEALTH_NODE_BUDGET_EXCEEDED,
    HEALTH_STATE_CELLS_EXCEEDED,
    HEALTH_BASELINE_UNAVAILABLE,
    WAVE_SIZE_CLAMPED,
    MENU_VISION_CLAMPED,
    PREVIEW_TRUNCATED
}

enum ConsumeStatus { ADVANCED, NO_EATABLE, NONE, NOT_ACTIVE, UNKNOWN_DISH, DISH_NOT_IN_WAVE }
enum AdvanceStatus { ADVANCED, NOTHING_TO_SKIP, FINAL_WAVE_USE_END_PICKUP, NOT_ACTIVE }
enum EndPickupStatus { ENDED, NOT_ACTIVE, NOT_FINAL_WAVE }

func initialize(station_id: StringName, seed: int, turn_budget: int,
                menu_vision: int, pool_override: Array[StringName] = []) -> WaveInitResult
func visible_dishes() -> Array[DishData]
func preview_dishes() -> Array[DishData]
func hidden_counts() -> Array[int]
func get_run_menu() -> Array[StringName]
func wave_index() -> int
func wave_count() -> int
func remaining_in_wave() -> int
func consume(dish_id: StringName) -> ConsumeResult
func advance() -> AdvanceResult
func end_pickup() -> EndPickupResult
func health() -> HealthEval


class_name WaveInitResult extends RefCounted
var error: MenuWaves.WaveError = MenuWaves.WaveError.OK
var partition: Array[PackedStringArray] = []
var health_eval: HealthEval = null
var warnings: Array[WaveWarning] = []
func is_ok() -> bool


class_name WaveWarning extends RefCounted
var code: MenuWaves.WarningCode
var attempt: int = -1
var detail: String = ""


class_name HealthEval extends RefCounted
var health: MenuWaves.Health = MenuWaves.Health.HEALTHY
var attempts: int = 0
var reachable_opt: int = -1        # -1 = 未完成
var baseline_opt: int = -1         # -1 = 不可用
var nodes: int = 0
var truncated: bool = false


class_name ReachEval extends RefCounted          # 逐波 DP 的原始输出，供 judge 使用
var value: int = 0
var nodes: int = 0
var truncated: bool = false


class_name ConsumeResult extends RefCounted
var status: MenuWaves.ConsumeStatus
var wave_index: int = 0
var remaining: int = 0


class_name AdvanceResult extends RefCounted
var status: MenuWaves.AdvanceStatus


class_name EndPickupResult extends RefCounted
var status: MenuWaves.EndPickupStatus


# Foundation — src/foundation/rng/
class_name SeedHash extends RefCounted
static func candidate_seed(station_id: StringName, run_seed: int, attempt: int) -> int
static func fingerprint_ids(ids: PackedStringArray) -> int    # 规范序 id 的 64 位指纹（域 = "pool"）

class_name SeededStream extends RefCounted
func _init(seed64: int) -> void
func next_u32() -> int
func below(n: int) -> int
func draw_count() -> int
```

## Implementation Guidelines

**必须**

1. 划分与健康判定只使用整数。任何 `float` 变量、常量或比较出现在这条路径上都视为违反原则 5；`0.9` 必须写成 `9/10` 交叉相乘。
2. 所有随机抽样必须来自 `SeededStream`，且 `SeededStream` 只由 `candidate_seed` 构造；`RandomNumberGenerator` 实例必须是局部新建的，不得 `randomize()`、不得复用、不得读全局 `randi()`。
3. 菜池在洗牌前必须按 id 码点升序规范化；`pool_override` 走同一条规范化路径。
4. 洗牌必须是反向 Fisher–Yates（`i` 从 `n−1` 到 `1`，`j = below(i+1)`），且 `below()` 必须是拒绝采样。
5. 候选序号必须进入哈希输入（`LE16(attempt)`）；重掷不得依赖前一个候选的 RNG 状态。
6. 健康判定顺序必须是「空波/覆盖 → 预算/基准 → 下界」，`DEGRADED` 一旦命中立即返回，不重掷。
7. 节点计数必须在「评估子集」处自增，且 `nodes > NODE_BUDGET` 时立刻返回 `TRUNCATED`，不得跑完再判断。
8. DP 必须逐波重置（状态只携带 `i / used_capacity / used_turns`），不得展开成跨波的全子集背包。
9. 并列候选必须取更早的 `attempt`（`>` 比较），保证结果与候选顺序一一对应。
10. 所有失败必须走封闭枚举返回值；`initialize` 失败时 `partition` 为空、`health_eval` 为 `null`。
11. 健康检查的基准、可达值、节点数只写进 `HealthEval` 与日志；不得进入 `RunResult`、存档、HUD 或任何玩家可见文本。
12. `pool_override` 非空时必须同时替换划分输入与基准菜池；id 必须存在于目录，重复 id 视为错误。
13. `advance()` 不得扣回合；调用顺序固定为「餐局流程 `can_spend_turn()` → `spend_turn()` → `MenuWaves.advance()`」。

**不得**

1. 不得在划分或健康路径使用 `Array.shuffle()`、`randi()`、`randi_range()`、`randf()`、`RandomNumberGenerator.randomize()`。
2. 不得用 `String.hash()`、`hash()` 或任何引擎内部哈希作为种子或排序键。
3. 不得依赖 `Dictionary` 遍历顺序、`res://` 扫描顺序或资源加载顺序构造菜池。
4. 不得在运行时修改 `dish_pool`、`wave_size`、`menu_vision` 或任何菜的数值。
5. 不得把 `wave_reachable_opt` 与 `opt_value` 暴露为公开 API，也不得让任何下游系统调用它们。
6. 不得让健康校验抛出异常、`assert` 失败或阻塞开局；导出构建会剥离 `assert`，所有输入校验必须是显式分支。
7. 不得在末波调用 `advance()` 扣回合；末波只能用 `end_pickup()`，且免费。
8. 不得为「让测试通过」而修改 12 道菜的真实数值（`menu-waves.md` 测试夹具约定）。

## Alternatives Considered

### Alternative 1: `Array.shuffle()` + 全局 `randi()`

**Description**：`randomize()` 之后直接 `pool.shuffle()`，种子只用于 `seed()` 全局 RNG。

**Pros**：一行代码；引擎已实现均匀洗牌。

**Cons**：全局 RNG 是进程级共享状态，任何其他系统抽一次随机数就改变本局划分；无法在同一进程内并排跑两个候选（重掷必须先恢复状态）；跨平台/跨版本无金标准。

**Estimated Effort**：0.5 天。

**Rejection Reason**：直接违反规则 10 与 AC 1–3。确定性要求「划分只依赖 `(station_id, seed)`」，全局状态天然做不到。

### Alternative 2: 按 `hash(dish_id, seed)` 排序代替洗牌

**Description**：给每道菜算一个 64 位哈希，按哈希升序排列后切片。

**Pros**：天然「无状态」，实现比 Fisher–Yates 短；每道菜的位置可单独复算。

**Cons**：排列分布不均匀（哈希值有序 ≠ 随机排列），且哈希冲突时必须引入 id 兜底，兜底规则本身又成为新的确定性契约；统计上「每道菜进第 1 波的概率」不再相等，与「均匀洗牌」的 GDD 表述不符。

**Estimated Effort**：0.5 天。

**Rejection Reason**：便宜但错。均匀性是可测试的性质（卡方检验），而重写洗牌的代价很低，没有理由牺牲正确性。

### Alternative 3: 用 `String.hash()` 直接从 `station_id` 派生种子

**Description**：`seed64 = hash(station_id) ^ run_seed`。

**Pros**：不用自己写哈希；一行。

**Cons**：`String.hash()` 的算法与返回值宽度没有跨版本稳定性承诺；升级引擎即改变全部历史划分，基准夹具失效且玩家感知为「种子被改了」。

**Estimated Effort**：0.1 天。

**Rejection Reason**：把确定性外包给一个不承诺稳定的私有实现，是这类 bug 的经典来源。

### Alternative 4: 用完整背包（不按波重置）求可达最优

**Description**：把「哪一波」编码进物品维度，做一次三维 DP 求全局最优。

**Pros**：状态定义更「标准」；不需要逐波子集枚举。

**Cons**：状态数变成 `|M| × (C0+1) × (T+1)`，MVP 就有 `12×21×11 = 2772`，后续档口线性膨胀；更重要的是它无法表达「非末波中途回合耗尽可免费终止」与「吃完免费推进 / 跳过着 1 回合」这两条波次特有规则，会算出一个**高于真实可达值**的上界，让健康门形同虚设。

**Estimated Effort**：2 天。

**Rejection Reason**：模型错误。GDD 的五条转移就是逐波重置 DP 的定义，换算法等于换语义。

### Alternative 5: 重掷用 `seed + 1` 递增

**Description**：`seed64 = base_seed + attempt`。

**Pros**：极简；相邻种子在 PCG 下输出差异很大。

**Cons**：`(station_id, seed)` 与 `(station_id, seed+1)` 会共享大部分候选序列（第 k 次重掷重合），跨档口的种子空间会重叠；不同 `station_id` 的候选序列也可能碰撞。基准夹具只能锁定「某个种子的前 21 个候选」，无法解释。

**Estimated Effort**：0.2 天。

**Rejection Reason**：廉价但劣。把 `attempt` 与 `station_id` 一起喂进哈希，成本相同、隔离性更好。

## Consequences

### Positive

- 「同一种子同一菜单」变成一条可在 CI 里对三个平台跑的字节级断言，而不是一句设计口号。
- 划分与健康判定完全无浮点、无全局状态，可以在同一进程内并排跑任意多个候选，重掷与基准夹具都变成纯函数测试。
- 节点预算与状态格子预算把「后续档口参数爆炸」挡在开局之前：最坏情况是 `DEGRADED` + 一条 warning，而不是掉帧或卡死。
- `HealthEval` 把「为什么这局是 `DEGRADED`」变成可观测字段（`nodes`、`truncated`、`baseline_opt`），平衡报告与 bug 单可以直接引用。
- 规范序让 `pool_override` 与 `dish_pool` 走同一条路径，教学档口不需要第二套划分代码。

### Negative

- 自己实现哈希与拒绝采样，比 `Array.shuffle()` 多约 60 行代码和一组金标准夹具；`RandomNumberGenerator` 的算法若在 4.8 改变，夹具会红灯，需要一次显式的「重新基线」决策。
- 规范序使「作者调整 `dish_pool` 书写顺序」不再影响划分——这对作者是行为变化，需要在内容文档里说明「顺序无意义」。
- 健康基准在运行期计算（记忆化），与 GDD 字面的「构建期预计算表」不同；虽然结果相同，但需要在 `menu-waves.md` / `dish-values.md` 下次修订时同步措辞。
- `DEGRADED` 分支的健康门只检查非空与覆盖，低于下界的划分在超预算机器上会被放行；这是「宁可不阻断」的自觉取舍。

### Neutral

- `MAX_ATTEMPTS = 21` 把「20 次重掷」写成「首次 + 20」，与 AC 16 的措辞一致；`attempts` 字段让两种读法都能被测试区分。
- 节点计数是实现口径，不是玩法口径；调整 `NODE_BUDGET` 只影响 `DEGRADED` 的出现频率，不影响任何数值。
- `opt_value` 的记忆化缓存只增不减（每个菜池一条），MVP 最多两条（档口池 + 教学池），可忽略。

## Risks

- **引擎 RNG 算法变更**：4.8 若更换 PCG 常量，所有历史划分改变。缓解：`Verification Required` ①记录 8 个金标准输出，CI 在引擎升级 PR 上必须先跑通；未通过则显式重新基线并接受一次「种子语义变更」的公告。
- **`randi()` 的返回值域与文档不符**：若返回有符号 32 位，`& 0xFFFFFFFF` 后位模式仍一致，但 `below()` 的 `limit` 比较需要实测确认。缓解：②③两条验证 + 一个覆盖 `n = 2..8` 的分布夹具。
- **有人用 64 位乘法实现哈希**：GDScript 有符号溢出行为未在文档中承诺 wrap，逐平台可能不同。缓解：本 ADR 只允许 32 位车道（乘积 < 2⁵⁶），并在代码评审清单里禁止 `*` 两个「可能 > 2³¹」的数。
- **节点预算导致 `DEGRADED` 常态化**：若后续档口 `wave_size` 升到 8 且 `T` 很大，可达状态数增长会让 10⁴ 频繁触发，健康门实际上只剩「非空 + 覆盖」。缓解：`nodes` 与 `truncated` 进平衡报告；若 `DEGRADED` 占比 > 5%，在 ADR 修订中提高预算或改用「每波只枚举 ≤ 4 道最优子集」的剪枝（需重新证明下界仍成立）。
- **0.9 阈值过宽**：当前参数下重掷几乎不触发（两次跳过仍有 95），健康门接近死代码。缓解：这是 `menu-waves.md` OQ4 的已知问题，本 ADR 只保证阈值可配置为整数分子/分母（`LOWER_BOUND_NUM/DEN`），收紧到 0.95 不需要改结构。
- **`pool_override` 由 `Dictionary` 构造**：规范序消除了顺序依赖，但重复 id 仍可能混入。缓解：`OVERRIDE_DUPLICATE_DISH` 显式失败，不做静默去重。
- **本机无编辑器**：所有引擎断言未经实测。缓解：把 `Verification Required` 的五条作为第一个波次 story 的前置 spike，未通过前不写依赖它的 story。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `menu-waves.md` | TR-menu-waves-001：确定性播种划分；同一 `(station_id, seed)` 跨平台一致 | 决策 §1 固定规范字节串与 32 位分道 FNV-1a；决策 §2 固定反向 Fisher–Yates + 拒绝采样；`Verification Required` ①② 提供跨平台金标准 |
| `menu-waves.md` | TR-menu-waves-002：`visible_dishes` / `preview_dishes`，预览上限 `wave_size−1` | 决策 §3：可见集为当前波未吃菜；预览取 `min(menu_vision, wave_size−1, \|W_{i+1}\|)`；末波为空；`hidden_counts()` 供 `?` 槽 |
| `menu-waves.md` | TR-menu-waves-003：`advance_cost` 空波/末波为 0，其余为 1；`end_pickup` 免费 | 决策 §7：`consume` 吃空自动免费推进；`advance()` 只承载非末波付费跳波且**不扣回合**；`end_pickup()` 仅末波、免费、返回 `ENDED` |
| `menu-waves.md` | TR-menu-waves-004：逐波重置 DP，924 carry 状态，吃菜计回合，不计腻度，≤10⁴ 节点，`DEGRADED` 回退 | 决策 §5：状态 `(i, c, t)`、五条转移逐条对应、节点计数口径、`NODE_BUDGET`/`STATE_CELL_BUDGET` 双预算、`TRUNCATED → DEGRADED`；静态 `satiety` 明确不计腻度 |
| `menu-waves.md` | TR-menu-waves-005：`wave_healthy` 下界 `0.9 × opt_value`；最多 20 次重掷后取最好一次 + warning | 决策 §4 基准纯函数与记忆化；决策 §6 整数交叉相乘（`10 × reachable < 9 × baseline`）、判定顺序、`MAX_ATTEMPTS = 21`、并列取更早、`HEALTH_BEST_EFFORT_AFTER_REROLLS` 恰好一条 |
| `menu-waves.md` | TR-menu-waves-006：`initialize` 接受 `pool_override`，健康校验亦用覆盖池 | 决策 §2 规范序；决策 §3 划分输入替换；决策 §4 基准菜池替换；决策 §7 未知/重复 id 显式失败 |
| `menu-waves.md` | AC 3 跨平台基准夹具；AC 15/16 阈值行为；AC 18 固定 seed 的波内归属；AC 22 覆盖池基准；AC 23 超预算注入 | 决策 §1 的 `attempt` 入哈希使整段候选序列可做基准；§6 的整数阈值与 `judge` 顺序直接对应 15/16/22/23；§2 的规范序让 AC 18 的 seed 在任一平台可复现 |
| `dish-values.md` | `opt_value(M, C, T)` 递推、int64、只用于 CI/设计；运行期不暴露 | 决策 §4 用**同一条** `V[i][c][t]` 递推实现内部基准，结果只存在 `HealthEval`；不新增公开 API、不进存档/HUD/结算 |
| `data-config.md` | `dish_pool`、`wave_size`（int ≥ 1，默认 4）、`price`、`satiety` 只读 | 决策 §3 的 `wave_size` 钳制与 `INVALID_WAVE_SIZE`；决策 §7 的所有权表：本系统不改任何上游字段 |
| `run-flow.md` | TR-run-flow-002：一帧内按序初始化，失败不发 `RunResult` | 决策 §7：`initialize` 恰好一次（`ALREADY_INITIALIZED`），所有失败走返回值，不抛异常 |
| `architecture.md` | 原则 5：确定性纯核心、整数运算、种子 RNG、逻辑不读墙钟 | 决策 §1/§2/§5/§6 全整数；`Implementation Guidelines` 必须 1 与不得 1 把「禁止浮点/全局 RNG」写成硬约束 |

## Performance Implications

- **CPU**：一次候选 = 规范序排序 `O(n log n)`（n ≤ 104）+ 洗牌 `n` 次拒绝采样 + DP ≤ 10⁴ 节点。MVP 实测预期 < 0.3 ms/候选；20 次重掷全跑满 < 4 ms。健康基准每菜池只算一次（2772 格），摊到整局可忽略。
- **Memory**：`best` 数组 = `(wave_count+1)×21×11` 个 int64 ≈ 7.4 KiB（MVP）；子集表每波 ≤ 256 × 3 个数组 ≈ 5 KiB。基准 DP 的 `PackedInt64Array` = 2772 × 8 B ≈ 22 KiB，记忆化后常驻。全部远低于单帧预算，且不分配在每帧路径上。
- **Load Time**：开局 `initialize` 的 p95 目标 ≤ 4 ms（含最坏 21 次候选）；典型 1 次候选 < 0.5 ms。不涉及文件 IO。
- **Network**：无。

## Migration Plan

绿地实现，无既有格式。落地顺序（每步都可独立测试）：

1. `SeedHash`（`_fnv1a32` / `_le64` / `_le16` / `_fold_u64` / `candidate_seed`）+ 金标准夹具（固定 `(station_id, seed, attempt)` → 固定 int64）。
2. `SeededStream` + `below()` 的分布夹具（`n = 2..8`，百万次抽样，卡方通过）+ `RandomNumberGenerator` 前 8 个输出的金标准。
3. `shuffled()` + 排列均匀性夹具（`n = 4`，全部 24 种排列各出现、频次接近）。
4. `_build_partition` + AC 1/2/3 夹具（含跨平台基准文件）。
5. `opt_value` + 与 `design/balance/opt_value.csv` 的逐值比对（104 @ C0=20/T=10）。
6. `wave_reset_dp` + 五条转移的单点夹具（手算一个 2 波小例）。
7. `judge` + 阈值边界夹具（93 / 94 / 95 / 104，`10×` 交叉相乘）。
8. 重掷循环 + AC 15/16/22/23 夹具（含「必然不达标」与「必然超预算」的可注入生成器）。
9. 接入 `RunFlow.initialize` 的编排，跑通一局的划分与推进。

`STATE_CELL_BUDGET` / `BASELINE_CELL_BUDGET` / `MAX_ATTEMPTS` 首版即固定为上述值，调参必须改 ADR。

**Rollback plan**：本 ADR 在写代码前确定，回滚即改文档。若 `RandomNumberGenerator` 的金标准无法在目标平台对齐，退路是**用 `SeededStream` 自己实现 PCG32**（同样 64 位状态、32 位输出，常量写进本 ADR），把引擎 RNG 从关键路径上彻底移除——接口（`next_u32` / `below`）不变，上层零改动。若规范序被推翻，只需删除 `canonicalize` 一步，洗牌与 DP 不受影响。

## Validation Criteria

| # | 验证 | 证据 |
|---|------|------|
| 1 | AC 1：12 道菜 / `wave_size = 4` → 3 波 × 4 道，并集等于全池、无重复、每波非空 | 单元测试断言波数、每波长度、id 计数 |
| 2 | AC 2：同一构建连续两次 `initialize` 的波次归属、波内顺序、价格、饱腹逐项相同 | 两次结果深比较 |
| 3 | AC 3：`(S01, seed = 12345)` 的基准文件在第二个平台逐项相等 | 提交的基准 JSON + 跨平台 CI 任务 |
| 4 | 候选序列确定性：`(station_id, seed)` 的前 21 个 `candidate_seed` 与基准文件逐值相等 | 金标准夹具 |
| 5 | `RandomNumberGenerator` 在固定 `seed` 下的前 8 个 `randi()` 输出等于本 ADR 记录值 | `Verification Required` ① 的 spike |
| 6 | 排列均匀性：`n = 4` 的 24 种排列在 10⁶ 次抽样中频次落在卡方 99% 区间内 | 统计夹具（只在 CI 手动任务跑） |
| 7 | AC 15 / 16：`reachable = 95` → `HEALTHY`、`attempts = 1`；`93` → 重掷；21 次全不达标 → 采用最高分 + 恰好一条 warning + 开局成功 | 注入可达值的生成器 |
| 8 | AC 22：`pool_override` 的基准按覆盖池计算（覆盖池 104 → 阈值 94），划分与校验都只含覆盖池 id | 双菜池夹具 |
| 9 | AC 23：注入超节点预算的生成器 → `DEGRADED`、一条 warning、`attempts = 1`、不重掷、不抛异常 | 注入 `nodes = 10001` 的桩 |
| 10 | 末波 `advance()` 返回 `FINAL_WAVE_USE_END_PICKUP` 且回合不变；`end_pickup()` 返回 `ENDED` 且回合不变 | 状态机夹具 |
| 11 | 吃空当前波 → `ADVANCED` 且回合不变；非末波 `advance()` → 回合由 `TimeTurns` 扣 1，`MenuWaves` 内无回合字段 | 调用记录器 F3 式夹具 |
| 12 | 静态扫描：`src/feature/menu_waves/` 与其 RNG 依赖中不存在 `float` / `randf` / `randomize` / `shuffle()` / `hash()` | CI grep 规则 |
| 13 | `opt_value` 与 `design/balance/opt_value.csv` 逐值相等（含 104 @ C0=20/T=10） | 数字比对夹具 |
| 14 | `\|M\| < wave_size` → 1 波 + `WAVE_SIZE_CLAMPED`；`\|M\| = 0` → `EMPTY_POOL` | 边界夹具 |

## Related

- `docs/architecture/architecture.md` — Required ADR #6、API Boundaries（`MenuWaves`）、原则 5、`TR-menu-waves-001..006`
- `design/gdd/menu-waves.md` — 规则 2 / 5 / 7 / 9 / 10 / 11、Formulas（`wave_partition` / `preview_dishes` / `advance_cost` / `wave_reachable_opt` / `wave_healthy`）、AC 1–3 / 15 / 16 / 22 / 23
- `design/gdd/dish-values.md` — `opt_value` 递推与「仅 CI/设计使用」的约束
- `design/gdd/data-config.md` — `dish_pool`、`wave_size`、`price`、`satiety` 的只读契约
- `design/gdd/run-flow.md` — 开局初始化顺序（`TR-run-flow-002`）与「无可吃菜」的结束路径
- ADR-0001（内容数据格式）— `DataConfig` 的 `Ready` 前提与 id 稳定性
- ADR-0004（服务组合）— 组合根注入 `DataConfig`，并决定 `initialize` 的调用点
- ADR-0007（局状态与结算事务边界）— 本局的 `seed` 与划分只活在一局内，不入存档（MVP）

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | `architecture.md` 的 `MenuWaves` 边界只列了 `visible_dishes` / `preview_dishes` / `consume` / `advance`，缺 `end_pickup()` 与 `health()`——后者是 `menu-waves.md` 规则 5 与 AC 7/19 的硬需求。本 ADR 按 GDD 实现这两个方法；`architecture.md` 的接口清单需在下次修订时补齐（本 ADR 不代为修改）。 | technical-director | 下一个架构修订 |
| 2 | `RandomNumberGenerator` 的 PCG 常量是否在 4.7.x 与未来 4.8 保持稳定？若 4.8 变更，是按「重新基线 + 公告」处理，还是提前改用自实现 PCG32（Rollback plan）？ | godot-specialist | 引擎升级到 4.8 之前 |
| 3 | 健康阈值 0.9 在 MVP 参数下几乎不触发（`menu-waves.md` OQ4）。是否在首个档口就用 `LOWER_BOUND_NUM/DEN = 19/20`（0.95）？本 ADR 只把阈值做成整数比例，不改默认值。 | systems-designer | 原型后 |
| 4 | `wave_size = 8` 且 `T > 20` 的后续档口会让 10⁴ 节点预算频繁触发。届时应提高预算，还是改为「每波只枚举容量占优的前 k 个子集」的剪枝？剪枝需要重新证明下界仍成立。 | lead-programmer + systems-designer | 第二个档口设计时 |
| 5 | `pool_override` 是否允许包含档口菜池之外的同目录菜品（教学档口跨档口借菜）？本 ADR 允许，只要求 id 存在于目录；若设计上要求「必须是子集」，需在 `menu-waves.md` 明确。 | game-designer | 首局引导实现前 |
