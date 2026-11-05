# ADR-0002: 持久化格式、原子写入与跨平台写锁

## Status

Accepted

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + agents（technical-director 授权；godot-specialist 引擎校验；编排会话合并登记）

## Summary

《回本》的生涯档需要一个能在崩溃、改版与双开下都不丢数据的落盘方案，同时必须让 `wallet` 这类 64 位整数无损往返、并让同一内存状态两次写出逐字节相同的文件。本 ADR 决定：用**规范 JSON 文本**（所有整数以十进制字符串传输，文件内不出现任何 JSON number）作为唯一格式，用**临时文件 → 备份 → 主档**的固定改名顺序保证原子性，用**原子创建的锁目录 + 心跳时间戳**实现跨平台单写者（Godot 的 `OS.is_process_running` 只看得见子进程，不能用作存活探测），并为版本 0 夹具提供一个可注入的迁移链骨架。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（latest patch 4.7.2；本机未安装编辑器，见 VERSION.md `Installed at pin time: NOT DETERMINED`） |
| **Domain** | Core |
| **Layer** | Foundation |
| **Knowledge Risk** | MEDIUM — `docs/architecture/architecture.md` 的 Engine Knowledge Gap Summary 把「Core / persistence」列为 MEDIUM（`FileAccess.store_*` 自 4.4 起返回 `bool`）；`VERSION.md` 自身只按版本标 HIGH |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`breaking-changes.md`、`current-best-practices.md`、`deprecated-apis.md`、Godot 4.7 API 文档 `JSON` / `DirAccess` / `FileAccess` / `OS` / `Time`，以及 4.7 分支源码 `core/io/json.cpp`、`core/io/dir_access.cpp`、`drivers/unix/dir_access_unix.cpp`、`drivers/windows/dir_access_windows.cpp`、`drivers/unix/os_unix.cpp`、`drivers/windows/os_windows.cpp` |
| **Post-Cutoff APIs Used** | `FileAccess.store_buffer` / `store_string` 的 `bool` 返回值（4.4 起）；`JSON.stringify(data, indent, sort_keys, full_precision)` 的 `sort_keys` 参数（4.0 起，默认 `true`）。无 4.5–4.7 新增 API 依赖 |
| **Verification Required** | ① `DirAccess.make_dir` 在目标已存在时返回 `ERR_ALREADY_EXISTS`——**已由 4.7 源码确认**（两个平台驱动都有该分支），仍需在本机三平台冒烟确认；② `JSON.stringify(..., sort_keys = true)` 的递归排序与两次调用逐字节相同——**已由源码确认递归**，仍需实测嵌套样例；③ Windows 上 `DirAccess.rename` 会**先 `remove` 目标再 `MoveFile`**（源码确认），因此不是原子替换——需实测崩溃窗口行为（见 Risks）；④ 心跳在暂停（`Paused`）与失焦下继续推进——需在真机验证驱动者的 `PROCESS_MODE_ALWAYS`；⑤ `FileAccess.flush()` 的实际持久化边界（见 Risks） |

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | None — 本 ADR 只依赖引擎与 OS 持久化 API，不依赖其他 ADR |
| **Enables** | ADR-0007（Run state ownership and settlement transaction boundary，Required ADR #7）——结算事务边界建立在 `Save.commit` 的原子性之上；ADR-0008（Settings persistence，Required ADR #8）——其机器级设置文件可复用本 ADR 的原子写入原语 |
| **Blocks** | 存档（#4）的全部实现 story；局外升级、解锁与档口、纪录与里程碑的写盘路径；界面流程 `Blocked` / `ReadOnly` 分支的验收 |
| **Ordering Note** | 「谁在启动序列里调用 `Save.load()`」不在本 ADR 决定，归 ADR-0004（Service composition）。本 ADR 只固定前置条件与调用次数 |

## Context

### Problem Statement

存档 GDD 把九个实现细节留给了架构阶段（Open Question 2、3、9 与 `architecture.md` 的 QQ-02、QQ-03）：序列化格式、int64 编码、目录与文件名、写锁机制、原子性顺序的落地方式与迁移骨架。这些问题互相耦合——格式决定 int64 怎么编码，编码决定解码器是否需要 schema，锁决定 `ReadOnly` / `Blocked` 的判定时机——必须一次性定下来，否则存档、局外升级、纪录三条写盘路径会各自发明格式。

### Current State

没有任何存档代码，也没有任何 ADR。`architecture.md` 已固定 `SaveSystem` 的对外 API（`load` / `retry_load` / `get_namespace` / `commit` / `flush` / `delete_career`）、六个载入结果、`LoadReport` 字段与 `save_bytes` 上界；存档 GDD 已固定 25 条核心规则与 30 条验收标准。缺失的是这些契约在磁盘上和 OS 上的具体形态。

### Constraints

- **格式必须让同一内存状态两次写出逐字节相同**（存档 GDD 规则 24），且明确要求「map 按键排序、整数按十进制、无随机 id、无平台相关换行」。「整数按十进制」排除了任何二进制编码。
- **int64 必须无损**：`9007199254740993` 与 `-9223372036854775808` 往返后逐位相等（GDD 规则 16、AC 19）。
- **不引入第三方 addon**：Godot 自带 `JSON` / `FileAccess` / `DirAccess` 即可完成。
- **单写者**：同一份生涯档同时只允许一个实例写入；判定不了就按「存活」处理（GDD 规则 21）。
- **写入预算**：Full Vision 满载约 8.5 KiB，写入 p95 ≤ 50 ms、载入 p95 ≤ 100 ms（AC 27、28）。
- **单线程主线程**：MVP 不做后台保存线程（`architecture.md` 数据流 §5）。
- **不做防篡改**：无加密、无校验和（GDD 规则 25）。

### Requirements

- 序列化格式在「逐字节可复现」与「int64 无损」两条约束下同时成立。
- 磁盘布局固定，且隔离文件（corrupt / deleted）的命名在 Windows 上合法。
- 锁的获取必须是真正的原子 test-and-set，不能是「先查再写」。
- 锁的回收有确定的判定表，覆盖「存活信号不可用」的每一种情况。
- 迁移链是纯函数序列，可从版本 0 夹具测试。
- 存档子系统拥有信封、结构校验、原子性、锁与迁移骨架；各命名空间的语义归其所有者。

## Decision

### 1. 序列化格式：规范 JSON 文本，数值一律十进制字符串

生涯档是 **UTF-8 编码、无 BOM、无换行、单行** 的 JSON 文本。写出用 `JSON.stringify(model, "", true, false)`（`indent = ""`、`sort_keys = true`、`full_precision` 无意义因为文件里没有浮点）。

关键约束：**文件中不出现任何 JSON number**。所有整数（GDScript 的 `int` 就是 int64）在序列化时写成十进制字符串：

```json
{"career_id":"c_7f3a91d2e0b4","created_at":"2026-11-04T15:30:45Z",
 "progression":{"upgrade_levels":{"upg_capacity_1":"2"},"wallet":"-9223372036854775808"},
 "schema_version":"1","slot_id":"0","unlocks":{"dishes":["dish_rice"],"stations":["station_canteen"]}}
```

（示例折行只为便于阅读；实际写出是一个不带任何换行的单行。）

为什么这样做：

- **int64 无损**。Godot 4 的 `JSON.parse_string()` 把所有 JSON number 解析成 `float`（`String.to_float()`），超过 2^53 的整数必然失真，这是 4.x 各版本的一贯行为、不是 bug。只要数字走 `float` 通道就没有无损方案，所以数字不能走 JSON number。
- **没有任何浮点格式风险**。文件里不存在 float，就不存在 `full_precision`、小数位截断、`5` 与 `5.0` 之争，也不可能在 4.4 的浮点打印变更后产生不同的字节。
- **逐字节可复现**。`sort_keys = true` 递归地对**每一层** Dictionary 排序（4.7 源码 `core/io/json.cpp` 确认 `p_sort_keys` 会向下传递）。具体比较函数是引擎的 `StringLikeVariantOrder`（自然/字母数字序），本 ADR 不承诺它等于码点序——只承诺它确定、与平台和 locale 无关。为此**所有 Dictionary 键必须是 `String` 或 `StringName`**；其他类型的键会回落到 `Variant` 的 `<`，排序契约不再成立。没有随机 id、没有时间戳参与（`last_saved_at` 是显式排除项）、没有平台换行。`JSON.stringify` 不插入空白。

**解码规则（全局，一条）**：解析出来的树中，**完全匹配** `^-?(0|[1-9][0-9]*)$` 的字符串解码为 `int64`，其余字符串保持 `String`。这条规则同时覆盖了 schema 未知的区域（`records` 由 #14 定义、`extensions` 由写入方定义），使本 ADR 不必依赖 #14 的字段清单。代价是一条明确的限制：**任何值形如规范十进制整数的 String 无法在档中表示**，写入方必须改写（例如加前缀）。`career_id`、所有 id、ISO-8601 时间戳都不符合该模式，因此不受影响——id 命名空间规则 `^[a-z][a-z0-9_]*$`（TR-data-config-004）保证了 id 以字母开头。

解码之后跑一遍**逐字段结构校验**（`SaveSchema`）：每个已知路径的期望类型、`schema_version` 为正整数、`slot_id` 在区间内、等级 ≥ 0、解锁 id 唯一。类型不符（包括本该是整数的位置出现 JSON number，即 float）属于结构错误，走损坏路径，不做逐字段抢救（GDD 规则 15）。

**值域是闭合的**：档中只允许 `null`、`bool`、`String`、`int64`（以十进制字符串表示）、`Array`、`Dictionary`。`float` 与引擎类型（`Vector2`、`StringName`、`Resource` 等）不在值域内——写入方必须先把它们转换成 `String` 或整数。这条规则让「文件里没有 JSON number」成为一个可以被断言的不变式，而不是一条约定。

### 2. 磁盘布局与命名

```
user://save/
  slot_0/                      # MVP: MAX_SLOTS = 1，多档时增加 slot_1…
    career.json                # 主档 primary
    career.bak                 # 上一次成功写入的完整主档
    career.tmp                 # 事务临时文件，失败或崩溃残留在下次启动时删除
    .lock/                     # 写锁（原子创建的目录）
      owner.json               # {instance_id, pid, host, started_unix, started_at}
      heartbeat                # 十进制 unix 秒，持锁期间每 30 s 刷新
    corrupt-<UTC>.json         # 隔离：主档损坏/版本过高后玩家选择「新建生涯」
    corrupt-<UTC>.bak          # 隔离：备份同时不可读时
    deleted-<UTC>.json         # 隔离：玩家删除生涯
    deleted-<UTC>.bak
```

- `<UTC>` 形如 `20261104T153045Z`。**不含冒号**——Windows 文件名禁止 `:`，ISO-8601 原样会失败。
- 同名隔离文件已存在时追加 `-2`、`-3`…；隔离文件永不自动清理，也不计入备份轮换（GDD 规则 23、Open Question 6）。
- 每 slot 一个目录，因此锁的作用域是「一份生涯档」而不是「整个游戏」；未来多档时两个实例可以各开一档而不互相阻塞。
- 主档/备份/临时文件固定为一个槽位内的三个名字，**不按实例或时间戳区分**——单写者保证同一时刻只有一个实例写它们。

> `career.json` 的 `.json` 扩展名与存档 GDD AC 5 举例的 `corrupt-<UTC>.sav` 不同；GDD Open Question 3 自己给的是 `user://save_v1.json`，故取 `.json`。这是命名细化，不改变 AC 5 的语义（旧档改名隔离而非删除）。

### 3. 跨平台写锁：原子创建的锁目录 + 心跳时间戳

锁是 `user://save/slot_0/.lock/` 这个**目录**，内含两个文件：

```
.lock/
  owner.json      # {instance_id, pid, host, started_unix, started_at} —— 身份元数据
  heartbeat       # 一个十进制 unix 秒数；持锁期间每 30 s 刷新 —— 存活信号
```

**获取**：

1. `DirAccess.make_dir_absolute(".lock")`。POSIX 的 `mkdir(2)` 与 Windows 的 `CreateDirectoryW` 在目标已存在时都返回 `ERR_ALREADY_EXISTS`（4.7 源码确认：`drivers/unix/dir_access_unix.cpp` 的 `EEXIST` 分支、`drivers/windows/dir_access_windows.cpp` 的 `ERROR_ALREADY_EXISTS` 分支）——这正是需要的原子 test-and-set。「先 `FileAccess.file_exists` 再 `open`」做不到：两个实例会同时看到「不存在」。**这一步是锁正确性的全部基础。**
2. 成功者写 `owner.json.part` → `rename` 成 `owner.json`，再写 `heartbeat.tmp` → `rename` 成 `heartbeat`。目录里要么没有这两个文件（获取未完成），要么是完整内容，不存在半截文件。

**持锁期间**：一个 `PROCESS_MODE_ALWAYS` 的驱动者每 `LOCK_HEARTBEAT_SECONDS = 30` 秒把当前 unix 秒写进 `heartbeat.tmp` 再 `rename` 覆盖 `heartbeat`（先写后改名，避免读到半截时间戳）。**不能用可暂停的 `Timer`**——暂停菜单下心跳必须继续。心跳写失败不阻断存档，只记一次警告。

**判定表** —— `L` 的取值决定 `load_outcome` 走 `ReadOnly`、`Blocked` 还是正常载入：

| 条件 | 结论 | 理由 |
|---|---|---|
| `.lock` 不存在 | `L = none` | 正常路径 |
| `.lock` 存在但无 `owner.json` | **回收为 `stale`** | 没有任何实例完成过获取，因此不可能有人在写 |
| `owner.json` 存在但无法解析 | `L = live` | 判不了就按存活（GDD 规则 21） |
| `owner.host` 非空、且 ≠ `OS.get_unique_id()` | 转「心跳不可用」列 | 锁不是本机写的，心跳时间戳无意义 |
| `heartbeat` 可读且 `now - hb ≤ LOCK_TIMEOUT_SECONDS (90)` | `L = live` | 心跳仍在推进 |
| `heartbeat` 可读且 `now - hb > 90` | 回收为 `stale` | 三次心跳未到，持锁方已不在 |
| `heartbeat` 不可用（缺失/不可读/跨机），且 `now - owner.started_unix > 86400` | 回收为 `stale` | 兜底上界：24 h |
| `heartbeat` 不可用，且年龄 ≤ 24 h | `L = live` | 判不了就按存活 |
| 任一时间戳在未来（时钟回拨） | `L = live` | 年龄为负，永不满足上界 |

**回收必须是原子的**：把 `.lock` 目录 `rename` 到 `.lock.stale-<instance_id>`（`rename` 是 CAS——只有一个实例能成功，另一个拿到非 `OK` 的错误；`DirAccess.rename` 只回 `FAILED`，不要把「失败」当 `ENOENT` 匹配），再 `make_dir(".lock")` 重新竞争；失败者回落到正常的「读到别人的锁」路径。最后清空并删除 `.lock.stale-*`（`DirAccess.remove` 拒绝非空目录，必须先删里面的文件）。

**持锁自检**：每次心跳顺带读一次 `owner.json`，确认 `instance_id` 仍是自己的。若不是（锁被判定为过期并回收，例如机器睡眠期间），本实例立即把自己降级为不可写、发 `save_status_changed(ReadOnly)`，**不再写任何文件**。

**释放**：正常退出时，读 `owner.json` 确认 `instance_id` 是自己的，然后依次删除 `heartbeat`、`owner.json`、`.lock`。崩溃留下的锁由下次启动按上表回收。

**`ReadOnly` / `Blocked` 的实例绝不写**：不建 `.lock`、不建 `career.tmp`、不改名任何文件。`retry_load()` 是唯一重新探测的入口，且只对「锁存活」与「迁移失败」两类 `Blocked` 开放（GDD 规则 21、UI Requirements 4）。

**为什么存活判定不用 `OS.is_process_running`**：它在 Godot 4.7 里只对自己 `create_process()` 出来的**子进程**有效。Unix 实现走 `waitpid(pid, WNOHANG)`，对非子进程返回 `ECHILD` → `false`；Windows 实现对不在自身 `process_map` 里的 PID 直接 `false`。第二个**独立启动**的游戏实例永远是「非子进程」，探测恒定返回 `false`——按「`false` = 已死」回收，会让两个实例同时写同一份档。该 API 在本场景不可用：`pid` 仍写进 `owner.json` 作为排障元数据，但**不参与判定**。心跳是间接的进程存活信号，代价是每 30 s 一次小文件写，换来的是跨平台一致、且失败方向安全——活着的实例一定在刷新，只有死掉的才会停。

> **与存档 GDD 规则 21 的一处有意偏差（需随本 ADR 一并确认）**：规则 21 的原文是「进程崩溃遗留的锁以进程存活为主、时间戳为辅回收；无法判定时按『存活』处理」。本 ADR 保留这条**意图**——只在持锁方确定不再存活时回收，判不了就按存活——但把判定手段从「OS 进程探测」换成了「心跳时间戳」，因为前者在 Godot 4.7 里对本场景恒定给出「已死」的错误答案。AC 24 仍然通过（进程已死且锁超过 24 h 的锁，其心跳必然早已停摆）。此处不是实现绕开设计，而是设计里点名的机制在引擎上不可用，因此显式记录并请 GDD 在本 ADR 接受时同步措辞。

> 为什么不选命名互斥量：Windows 专有，Godot 的 `OS` 不暴露，macOS/Linux 上无对应物，三平台无法共用一套实现。
> 为什么不选端口锁：会在部分平台上触发防火墙提示；端口是全局命名空间，与「一份生涯档一把锁」不对应；进程死掉后端口释放但锁内容全丢，无法做 PID/时间戳判定。
> 为什么不选 `flock` / `LockFileEx` 咨询锁：最正确，但 Godot 4.7 没有暴露，需要 GDExtension，为单人游戏引入一个原生模块不划算。

### 4. 原子写入：固定五步

```
1. validate(model)                     # schema + 边界 + save_bytes ≤ SAVE_BYTES_MAX
2. bytes = SaveCodec.encode(model)     # 规范 JSON
3. open(career.tmp, WRITE)
4. store_buffer(bytes)                 # 返回值是 bool（4.4 起），必须检查
   flush()
   close()                             # Windows 上必须关闭后才能改名
5. rename(career.json → career.bak)    # 主档不存在（Fresh/Recovered 后首次）时跳过
   rename(career.tmp  → career.json)
```

- 步骤 5 的第一次改名用 `DirAccess.rename` 覆盖旧备份，**应用层不先删除旧备份**。
- 平台差异（4.7 源码确认）：POSIX 走 `::rename(2)`，是原子替换；**Windows 驱动会先 `remove` 目标再 `MoveFileW`**，所以「主档→备份」在 Windows 上不是原子替换，中间存在一个「无备份」窗口。该窗口内主档仍完好，所以载入结果还是 `Loaded`；真正的风险只在「无备份窗口 + 主档随后损坏」这一叠加情形，见 Risks。
- 任一步失败：删除 `career.tmp`，主档与备份保持不动，按 GDD 规则 17/18 返回 `CommitFailed`（购买回滚）或 `SaveDeferred`（结算入账保留 dirty）。
- 崩溃窗口对应关系：步骤 5 之前崩溃 → 主档完好、`career.tmp` 残留（下次启动删除）；两次改名之间崩溃 → 主档缺失、备份是上一次成功内容 → `Recovered`；第二次改名之后 → 新主档生效。第二次改名本身在 Windows 上目标不存在（刚被移走），退化为普通 `MoveFileW`。
- 启动时**无条件删除**遗留的 `career.tmp`——它的存在只可能意味着上次事务没有走完。

### 5. 迁移骨架与版本 0 夹具

- `MigrationChain` 持有有序的 `{from_version, to_version, apply: Callable}` 列表，`apply` 的签名是 `(raw: Dictionary) -> Dictionary` 或返回 `MigrationError(step, field_path)`。**迁移作用于 JSON 解析后的原始树，而不是解码后的模型**——版本 0 的 `wallet` 是字符串 `"120"`，而版本 1 的 `wallet` 也是字符串，二者只在 schema 语义上不同，用原始树才能让 `m_1` 自由改写。
- 链在执行前校验连续性（`from/to` 无空洞）；有空洞说明构建有缺陷，debug 构建直接 `assert`，release 构建把该迁移视作失败。
- **生产构建注册空链**（MVP 的 `SAVE_SCHEMA_VERSION = 1`）；测试构建注入 `m_1`。这是一个注入点而不是 `if OS.is_debug_build()` 分支——存档 GDD AC 8–10 要求测试构建「注册一个 `m_1`」。
- 夹具放在 `tests/fixtures/save/`，每条用例独立目录（AC 前言）：
  - `v0/wallet_string.json` —— `schema_version: 0`、`progression.wallet: "120"`、含未知顶层键 `future_flag`；断言 `Migrated`、`wallet == 120`（int64）、`career_id`/`slot_id` 补齐、`future_flag` 落在 `extensions`。
  - `v0/wallet_invalid.json` —— `wallet: "abc"`；断言 `MigrationError(1, "progression.wallet")` 且原文件逐字节不变。
  - 夹具由测试辅助函数写进临时目录，不依赖仓库里的绝对路径。

### 6. 所有权边界

| 归属 | 内容 |
|---|---|
| **Save 拥有** | 信封字段（`schema_version`、`career_id`、`slot_id`、`created_at`、`last_saved_at`、`extensions`）、编解码器、逐字段结构校验、`save_bytes` 上界、文件布局与命名、锁、原子写入、备份与隔离、迁移链的执行与注册点、`LoadReport` / `CommitResult` / `FlushResult` 的错误词汇 |
| **命名空间所有者拥有** | `progression`（局外升级）、`unlocks`（解锁与档口）、`records`（纪录与里程碑）、未来的 `challenges`（在线挑战）的**字段与语义**；新值的计算；提交时机。它们只经 `commit({ns: value})` 写入，**永不直接碰文件** |
| **既不属于 Save 也不属于命名空间** | 机器级设置（界面流程 / 音效反馈，见 Required ADR #8）；`正常化`（`unlocks.normalize()` 的调用方）；结算事务编排（界面流程，见 ADR-0007） |

Save 做的校验只有**结构性**的：类型、区间、id 唯一性、以及规则 15 要求的「等级超过 `max_level` 钳制并报告」（`max_level` 从数据配置取，这是 Save → Data Config 的合法读取）。Save 不解释任何值的玩法含义。

### Architecture Diagram

```
   ┌────────────────────────────────────────────────────┐
   │  MetaUpgrades · Unlocks · Records · UI Flow        │  命名空间所有者
   └───────────────┬────────────────────────────────────┘
                   │ commit(tx) / get_namespace() / load() / flush()
   ┌───────────────▼────────────────────────────────────┐
   │  SaveSystem (Foundation)                           │
   │  ┌──────────────────┬───────────────────────────┐  │
   │  │ SaveCodec        │ SaveSchema                │  │
   │  │ 规范 JSON        │ 逐字段类型/边界校验       │  │
   │  │ 整数→十进制串    │ 等级钳制（读 DataConfig） │  │
   │  └──────────────────┴───────────────────────────┘  │
   │  ┌──────────────────┬───────────────────────────┐  │
   │  │ SaveLock         │ AtomicWriter              │  │
   │  │ 锁目录 + 心跳    │ tmp→bak→primary 五步      │  │
   │  └──────────────────┴───────────────────────────┘  │
   │  ┌──────────────────────────────────────────────┐  │
   │  │ MigrationChain  (m_1 … m_n，纯函数，可注入)  │  │
   │  └──────────────────────────────────────────────┘  │
   └───────────────┬────────────────────────────────────┘
                   │ FileAccess · DirAccess · OS · Time
   ┌───────────────▼────────────────────────────────────┐
   │  user://save/slot_0/                               │
   │    career.json  career.bak  career.tmp             │
   │    .lock/{owner.json, heartbeat}                   │
   └────────────────────────────────────────────────────┘
```

### Key Interfaces

对外 API 沿用 `architecture.md` 的 `SaveSystem`，本 ADR 只固定它产出的错误词汇与磁盘契约。

```gdscript
class SaveSystem:
    signal save_status_changed(status: SaveStatus)

    func load() -> LoadReport            # 数据配置 Ready 后调用恰好一次；重复调用返回同一结果
    func retry_load() -> LoadReport      # 仅 Blocked 且 (lock_live 或 迁移失败) 时可用
    func get_namespace(owner: StringName) -> Dictionary   # 只读副本
    func commit(tx: Dictionary) -> CommitResult           # 整笔原子；成功即已落盘
    func mark_dirty(namespace: StringName) -> void
    func flush() -> FlushResult
    func delete_career(token: ConfirmToken) -> Result[void, SaveError]

enum SaveError:
    CORRUPT            # 结构不合法、截断、空文件
    VERSION_AHEAD      # schema_version > SAVE_SCHEMA_VERSION
    MIGRATION_FAILED   # 链上某一步返回 error
    LOCK_LIVE          # 有存活写锁且文件不可用
    NOT_WRITABLE       # ReadOnly / Blocked 下的写入请求
    SAVE_TOO_LARGE     # encode 后超过 SAVE_BYTES_MAX
    IO_ERROR           # 文件系统拒绝（权限、磁盘满、被占用）
    DATA_NOT_READY     # 数据配置未 Ready 时调用 load()

enum CommitResult:  OK | COMMIT_FAILED | SAVE_DEFERRED | NOT_WRITABLE | SAVE_TOO_LARGE
enum FlushResult:   OK | SAVE_DEFERRED | NOT_WRITABLE | IO_ERROR
```

`LoadReport` 字段沿用 GDD：`outcome`、`reason`、`warnings`、`writable`、`lock_live`、`source_version`、`source`（`primary` / `backup` / `none`）、`first_run_eligible`。

`first_run_eligible` 按 `save-system.md` 规则 26 计算：`(outcome ∈ {Fresh, Loaded, Migrated}) ∧ records.career.runs_settled == 0`；`Recovered` / `ReadOnly` / `Blocked` 恒为 `false`。Save 只读取 `records` 命名空间的 `runs_settled` 作为报告输入，不解释其余纪录语义；`Recovered` 的 `SETTLED_COUNTER_ROLLED_BACK` 钳制与警告也在此处一次完成。

磁盘契约（对玩家可见、需在 UX 规格中说明）：
- 主档：`user://save/slot_0/career.json`
- 删除生涯后旧档：`user://save/slot_0/deleted-<UTC>.json`（`.bak` 同理）

### Implementation Guidelines

- **MUST** 把每个整数写成十进制字符串；**MUST NOT** 让任何 JSON number 出现在 `career.json` 里。
- **MUST** 用 `JSON.stringify(model, "", true, false)`（`sort_keys = true`）编码；**MUST NOT** 自己拼字符串或依赖 Dictionary 的插入顺序。
- **MUST** 检查 `FileAccess.store_*` 的 `bool` 返回值（4.4 起）；**MUST NOT** 假定写成功。
- **MUST** 在改名之前 `close()` 文件；**MUST NOT** 在文件仍打开时 `rename`（Windows 会失败）。
- **MUST** 在每次 `commit` 前校验 `save_bytes ≤ SAVE_BYTES_MAX` 并拒绝超限；**MUST NOT** 截断或丢弃条目。
- **MUST** 用 `DirAccess.make_dir_absolute` 的返回值判断锁的归属；**MUST NOT** 用「先 `file_exists` 再 `open`」实现锁。
- **MUST** 用 `PROCESS_MODE_ALWAYS` 的驱动者发送心跳；**MUST NOT** 用可暂停的 `Timer`，也 **MUST NOT** 用 `OS.is_process_running` 判断另一个实例是否存活。
- **MUST** 在任何 `writable = false` 的状态下不创建 `career.tmp`、不改名任何文件。
- **MUST** 为档中每一个 Dictionary 使用 `String` / `StringName` 键；**MUST NOT** 用整数或其他类型作键（排序契约会失效）。
- **MUST NOT** 在任何迁移函数里读时钟、读运行期状态、联网、或依赖地图/数组的迭代顺序。
- **MUST NOT** 让 Save 依赖 Unlocks / Records / MetaUpgrades 的任何类型——依赖方向只能是它们 → Save。
- **MUST** 在启动时删除遗留的 `career.tmp`；**MUST NOT** 尝试从 `career.tmp` 恢复。

## Alternatives Considered

### Alternative 1: `FileAccess.store_var`（二进制 Variant 编码）

- **Description**: `store_var(model)` / `get_var()`，整数天然是 int64，无需编码规则，文件比 JSON 小约 30%。
- **Pros**: 零精度风险；任何 Variant 都能存；实现量最小（一个函数调用）。
- **Cons**: 写出的是小端字节，**整数不是十进制**——直接违反存档 GDD 规则 24 的「整数按十进制」；文件不可读、不可 diff，规则 24 的「让 diff 可读」也随之落空；`var_to_bytes` 的编码格式由引擎定义，跨大版本迁移时没有可依赖的稳定契约。
- **Estimated Effort**: 约 0.3×（编码），但迁移与人工排障成本高。
- **Rejection Reason**: 与规则 24 的明文要求冲突，且把格式的稳定性交给引擎实现细节。

### Alternative 2: 自定义二进制（自写 TLV / MessagePack 风格）

- **Description**: 自己定义长度前缀 + 类型标记 + 定长大端整数。
- **Pros**: 体积最小；编码完全可控；int64 精确。
- **Cons**: 需要自写编码器、解码器、结构校验、向前兼容规则；同样是「整数按字节」而不是十进制，仍违反规则 24；出现 bug 时没有现成工具排障；未来每个 schema 变更都要改二进制读写器。
- **Estimated Effort**: 约 3×。
- **Rejection Reason**: 复杂度换来的只有体积，而体积距离 256 KiB 上限有约 30 倍余量（`save_bytes` 公式）。

### Alternative 3: JSON + 逐字段 schema 驱动（只把 int64 字段写成字符串）

- **Description**: 只把 `wallet` 等声明为 int64 的字段写成字符串，其余小整数写 JSON number。
- **Pros**: 文件更接近常见 JSON；小整数可读性更好。
- **Cons**: 「哪些字段是 int64」必须由每个命名空间声明，而 `records` 的字段由 #14 定义、`extensions` 是任意 Variant，二者都无法预先覆盖；于是要么让本 ADR 依赖 #14 的字段清单并在每次 #14 改字段时同步，要么在两处区域另设一套编码。两套机制意味着两条解码路径、两类 bug。
- **Estimated Effort**: 约 1.5×（本方案 + 命名空间 schema 注册机制）。
- **Rejection Reason**: 为了「小整数看起来像数字」付出跨 ADR 依赖与双编码路径，不划算。

### Alternative 4: 写锁用端口锁或命名互斥量

- **Description**: 绑定一个固定 localhost 端口（端口锁）或用 Windows 命名互斥量。
- **Pros**: 端口锁不需要文件；互斥量在进程死亡时由 OS 自动释放。
- **Cons**: 端口锁会触发防火墙提示，端口是全局命名空间而非「一份档一把锁」，且进程死后锁内容全丢，做不了 PID/时间戳判定；命名互斥量是 Windows 专有，Godot 不暴露，macOS/Linux 无对应物。
- **Estimated Effort**: 端口锁约 0.7×，互斥量需要 GDExtension（约 4× 且只覆盖一个平台）。
- **Rejection Reason**: 三平台无法共用一套实现，且都无法满足锁的回收判定。

### Alternative 5: PID 文件 + `OS.is_process_running` 进程探测

- **Description**: 锁是 `owner.json`，存活判定直接问 OS：`OS.is_process_running(owner.pid)`，为 `false` 即回收。
- **Pros**: 不需要周期性写盘；掉落锁立即回收，不用等心跳超时。
- **Cons**: **这个 API 在本场景根本不可用**。4.7 的 Unix 实现走 `waitpid(pid, WNOHANG)`，对非子进程返回 `ECHILD` 即 `false`；Windows 实现对不在自身 `process_map` 里的 PID 直接返回 `false`。它只对 `create_process()` 创建的子进程有效——第二个独立启动的游戏实例永远是非子进程，探测恒定返回 `false`。按「`false` = 已死」回收，等于**保证**双开时两个实例同时写同一份档。
- **Estimated Effort**: 约 0.5×（比心跳少一个文件与一个驱动者）。
- **Rejection Reason**: 语义与名称不符，会在最需要它的场景（两个独立实例）给出完全错误的答案。`pid` 因此只作为排障元数据保留，不参与判定。

## Consequences

### Positive

- **int64 与逐字节复现同时成立**，且不依赖引擎的浮点打印行为——文件里没有浮点。
- **单一编码/解码路径**：一条全局解码规则 + 一遍结构校验，`records` 与 `extensions` 不需要额外契约。
- **锁是所有平台同一套代码**，判定表把「判不了」的每一种情况都写死，行为可验收。
- **原子写入的崩溃窗口与 GDD 的 `Recovered` 语义一一对应**，可以在测试里用注入的失败点穷举。
- **文件可读可 diff**，玩家与开发者都能直接看到档里有什么，符合「不做防篡改」的单机信任模型。

### Negative

- 所有整数带引号，档的观感不如常规 JSON；人读 `"level":"2"` 需要一秒的适应。
- 形如规范十进制整数的 `String` 无法写入档中（需写入方改写）。
- `career.tmp` / `career.bak` / `.lock` 三个伴随物让目录不像「一个存档文件」那么直观；UX 规格需要解释隔离文件在哪。
- 持锁期间每 30 s 一次小文件写（tmp + rename）。对一个每秒只写几次的游戏来说可以忽略，但它确实让「空闲时零磁盘写」不再成立。
- 锁的存活判定依赖一个后台驱动者；它必须不被暂停，否则一次暂停超过 90 s 就会让别的实例认为本实例已退出。

### Neutral

- 存档内容与引擎版本解耦：未来升级 Godot 不改变档的字节，除非 JSON 转义规则变动（`"`、`\`、控制字符的转义是规范的一部分，不会变）。
- 迁移作用在原始树上而不是模型上，迁移函数可以自由改写任何字段。

## Risks

- **心跳超时期间两个实例都可能短暂觉得自己是持锁者**：实例 A 的进程被 OS 挂起（笔记本合盖、调度饥饿）超过 90 s，实例 B 启动并回收了锁。→ 缓解：持锁自检——每次心跳回读 `owner.json`，`instance_id` 不再是自己就立刻降级为不可写；A 醒来后的第一件事就是发现自己已经不是持锁者。窗口内的真实风险只覆盖 A 被挂起前尚未落盘的那一次事务，而 A 在挂起前必然已经走完或被中断在原子写入的五步之内，由备份路径兜底。
- **`FileAccess.flush()` 不是 `fsync`**：Godot 4.7 没有暴露 `fsync`，`flush()` 只把用户态缓冲交给 OS。断电时 OS 仍可能重排写入。→ 缓解：临时文件→备份→主档的改名顺序使 OS 即便重排也很难产生「主档是半截内容」；残余风险（掉电后丢最后一次事务）由备份路径 `Recovered` 覆盖。这是本方案唯一不能靠应用层闭合的洞，接受。
- **Windows 上「主档→备份」不是原子替换**（引擎先 `remove` 目标再 `MoveFileW`），存在一个「无备份」窗口。→ 缓解：该窗口内主档完好，载入仍是 `Loaded`；只有「窗口内崩溃 **且** 主档随后不可读」才同时丢掉两个副本，需要两次独立故障叠加。下一次成功写入会重建备份。
- **跨机同步（未来 Steam Cloud）会让锁在另一台机器上不可判定**。→ 已记录 `host` 字段，非本机锁走「心跳不可用」列并由 24 h 兜底；真正接入云同步时的冲突策略是存档 GDD Open Question 4，不在本 ADR 范围内。
- **排序契约只在键是 `String` / `StringName` 时成立**：引擎的比较器是 `StringLikeVariantOrder`，其他类型的键回落到 `Variant` 的 `<`。→ 缓解：值域规则已禁止非字符串键；解码校验再拒绝一次。
- **`Time.get_datetime_string_from_system(true)` 不带 `Z` 后缀**，直接写会得到非 ISO-8601 的字符串。→ 缓解：`created_at` / `last_saved_at` 必须由调用方补 `Z`；`<UTC>` 文件名用独立格式化（`YYYYMMDD` + `T` + `HHMMSS` + `Z`），不使用该 API 的默认形态。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| save-system.md（TR-save-system-001） | 信封 + `progression` / `unlocks` / `records` 命名空间 | 决策 §1 固定信封字段的线上形态；§6 固定所有权边界：信封归 Save，命名空间内容归各自所有者，只经 `commit` 写入 |
| save-system.md（TR-save-system-002） | 原子写入：校验 → 临时 → flush → 主档→备份 → 临时→主档；失败保持原样 | 决策 §4 给出五步顺序与其失败/崩溃对应关系，明确任一步失败删除 `career.tmp` 且不改动主档与备份 |
| save-system.md（TR-save-system-003） | 六个载入结果 + `LoadReport`（`outcome`、原因、警告、`writable`、`lock_live`） | 决策 §3 的判定表产出 `L`，与 `load_outcome` 公式的判定顺序一致（版本过高 → 存活锁 → 其余）；`Key Interfaces` 固定 `LoadReport` 字段与 `SaveError` 词汇 |
| save-system.md（TR-save-system-004） | 单写者；存活锁的 `Blocked` 只给退出/重试；新建生涯与删除需 `lock_live = false` | 决策 §3：锁的原子获取、心跳存活判定表、持锁自检、`ReadOnly`/`Blocked` 实例不触碰任何文件；删除生涯在持锁时被拒绝 |
| save-system.md（TR-save-system-005） | 单向纯函数迁移链；失败 → `Blocked`；未知 id 保留 | 决策 §5：链执行器 + 纯函数约束（不读时钟/状态/网络）+ 连续性校验 + 版本 0 夹具；未知 id 由「按 id 存储、不解释」天然保留 |
| save-system.md（TR-save-system-006） | `commit({ns: value})` 原子事务；购买回滚 vs 结算延迟重试 | 决策 §4 的失败处理明确返回 `COMMIT_FAILED`（调用方回滚内存）或 `SAVE_DEFERRED`（调用方保留 dirty）；一次 `commit` = 一次五步写入 |
| save-system.md（TR-save-system-007） | 确定性序列化：map 键排序、int64 无损、排除 `last_saved_at` 的逐字节复现 | 决策 §1：`sort_keys = true`、无 JSON number（故无浮点）、int64 走十进制字符串；`last_saved_at` 是唯一被排除的字段 |
| save-system.md（规则 16 / AC 19） | `wallet` 有符号 int64，`9007199254740993` 与 `-9223372036854775808` 逐位往返 | 决策 §1：整数以十进制字符串传输，完全绕开 Godot JSON 的 `float` 解析 |
| save-system.md（规则 15） | 等级越界钳制并报告；类型错误走损坏路径 | 决策 §1 解码后的结构校验：类型不符（含 JSON number 出现在整数字段）→ `CORRUPT`；越界 → 钳制 + `warnings` |
| save-system.md（规则 24 / AC 26） | 同一内存状态两次写出逐字节相同 | 决策 §1：无随机 id、无时间戳参与、无平台换行、键按引擎的确定性顺序（`StringLikeVariantOrder`）递归排序 |
| ui-flow.md（TR-ui-flow-005） | 锁优先的 `Blocked` 出口与 `retry_load` 路径 | 决策 §3：`lock_live = true` 时任何原因都只给「退出」「重试」；`retry_load()` 是唯一重新探测入口，且不隔离文件 |
| records-and-milestones.md（TR-records-001） | `records` 命名空间 schema 由纪录系统定义 | 决策 §6：Save 不拥有 `records` 字段，全局「十进制字符串 → int64」解码规则让 #14 可以在不改本 ADR 的前提下设计其字段 |

## Performance Implications

- **CPU**: 编码 = 一次递归排序 + `JSON.stringify`，解码 = `JSON.parse_string` + 一遍结构校验。8.5 KiB 档在 60 fps 预算下约 < 1 ms；MVP 的 2.5 KiB 档更低。迁移链为空，无额外开销。
- **Memory**: 峰值 = 原始文本（≤ 256 KiB）+ 解析树 + 编码产物。远低于单帧预算。读取时按 `SAVE_BYTES_MAX + 1` 截断，避免一个异常大的文件把内存吃掉。
- **Load Time**: 目标 p95 ≤ 100 ms（AC 28），实际预期 < 10 ms；锁判定只多一次 `make_dir` 与两次小文件读。持锁期间的心跳是每 30 s 一次 tmp+rename 的小文件写，不在载入或写入的临界路径上。
- **Network**: 无。MVP 不接云同步。

## Migration Plan

本 ADR 是首次实现，没有既有格式需要迁移。落地顺序：

1. `SaveCodec`（编码 + 全局解码规则 + 往返测试，先于任何文件 IO）。
2. `SaveSchema`（结构校验 + 等级钳制 + `save_bytes` 上界）。
3. `AtomicWriter`（五步写入 + 可注入失败点，覆盖 AC 11 / 12）。
4. `SaveLock`（锁目录 + 心跳驱动者 + 判定表 + 持锁自检，覆盖 AC 23 / 24 / 24b）。
5. `MigrationChain` + 版本 0 夹具（覆盖 AC 8–10）。
6. `SaveSystem` 组装上述四件，暴露 `architecture.md` 的 API。

`SAVE_SCHEMA_VERSION` 初始为 1，迁移链为空。

**Rollback plan**: 本 ADR 在写代码之前确定，回滚就是改文档。若格式被推翻，已写出的 `career.json` 只能通过「读旧格式 → 迁移」处理；由于首版档只会在本格式下产生，回滚窗口仅存在于首次外发之前。

## Validation Criteria

- AC 19 的两个 int64 夹具往返后逐位相等。
- AC 26：同一内存状态连续写两次（排除 `last_saved_at`）哈希相同，嵌套 map 的键顺序一致。
- AC 11 / 12：注入写失败与两次改名之间的崩溃，主档/备份哈希不变或产生 `Recovered`。
- AC 23 / 24b / 24：双开得到 `ReadOnly` 或 `Blocked`、存活锁优先于版本判定、过期锁被回收。
- AC 8–10：版本 0 夹具迁移成功、非法值返回 `MigrationError`、二次启动不再迁移。
- AC 27 / 28：`save_bytes` 在 16 KiB 内、写入 p95 ≤ 50 ms、载入 p95 ≤ 100 ms。
- Verification Required ②③⑤ 在 4.7.x 实测通过后，本 ADR 的 `Engine Compatibility` 可标记为已闭合。

## Related

- `docs/architecture/architecture.md` — Required ADR #2、API Boundaries（`SaveSystem`）、Open Questions QQ-02 / QQ-03
- `design/gdd/save-system.md` — 规则 8 / 15 / 16 / 21 / 24、Open Question 2 / 3 / 9
- `design/gdd/ui-flow.md` — 规则 20（`Blocked` / `ReadOnly` 的界面分支）
- ADR-0004（Service composition）— 决定启动序列中 `load()` 的调用方，本 ADR 只固定前置条件与「恰好一次」
- ADR-0007（Run state ownership and settlement transaction boundary）— 结算的单一 `Save.commit` 依赖本 ADR 的 `commit` 原子性
- ADR-0008（Settings persistence）— 机器级设置文件可复用本 ADR 的原子写入原语，但其格式与路径由该 ADR 决定
