# ADR-0008: 机器级设置持久化与本地化呈现管线

## Status

Accepted

> **谁可以把状态改为 `Accepted`：用户，或 `technical-director` 在用户明确确认后。其他 agent 与任何 skill 都不能自行设置。**

## Date

2026-11-04

## Last Verified

2026-11-04

## Decision Makers

user + agents（technical-director 授权；godot-specialist 引擎校验；ux-designer 文案与字号约束；编排会话合并登记）

## Summary

《回本》的设置**不属于生涯档**（`save-system.md` 规则 20），却在启动第一帧就要生效（音量、字号、减少动态）。本 ADR 把它落成一个独立的**机器级文档** `user://config/settings.json`：格式复用 ADR-0002 的规范 JSON 与「临时 → 备份 → 主档」原子替换，七个设置项全部以整数/布尔存储、修改立即生效、**只在设置模态关闭或退出游戏时整体写入一次**，写失败保留内存值并只显示「设置可能未保存」。本地化侧，本 ADR 规定：所有玩家可见字符串（含记录里的 `display_name`）都是 `tr()` key、key 有固定语法与域、中文 MVP 只发 `zh_Hans` 一列、CJK 字体走「内置 Noto Sans SC → SystemFont → ThemeDB」三级回退，文本缩放 100/125/150% 通过 `τ` 只放大**字号与文字相关尺寸**并触发同帧重排，150% 下允许滚动、不允许截断或焦点环被裁切。

## Engine Compatibility

| Field | Value |
|-------|-------|
| **Engine** | Godot 4.7（pin 4.7.x；latest patch 4.7.2；`Installed at pin time = NOT DETERMINED`，本机尚未安装编辑器） |
| **Domain** | UI / Localization / Persistence |
| **Layer** | Foundation（设置文档与字体）+ UI（缩放与焦点呈现） |
| **Knowledge Risk** | HIGH — `Control.accessibility_*`（4.5 AccessKit，4.7 重定类型）、4.6 的 CSV plural/context 列、`ResourceImporterDynamicFont.hinting` 默认值改为 3（4.7）都在训练截止之后；本机也没有编辑器实测 |
| **References Consulted** | `docs/engine-reference/godot/VERSION.md`、`current-best-practices.md`（4.5 AccessKit / 编辑器内实时翻译预览；4.6 CSV plural/context；4.7 `Control.offset_transform_*`）、`breaking-changes.md`（`FileAccess.store_*` 返回 bool；`hinting` 默认 3；`Control.accessibility_live` 重定类型）、`deprecated-apis.md`；`design/gdd/ui-flow.md` 规则 21/24/28、UI Requirements、AC 33–36；`design/gdd/in-game-hud.md`（`hud_ui_scale` / `hud_card_columns` / 不透明度）；`design/gdd/save-system.md` 规则 20/23、OQ8；`design/gdd/data-config.md` OQ6；`docs/architecture/adr-0002-persistence-format.md`、`adr-0003-scene-screen-architecture.md` |
| **Post-Cutoff APIs Used** | `Control.accessibility_name` / `accessibility_description` / `accessibility_live`（4.5 AccessKit，4.7 重定类型）；`ResourceImporterDynamicFont.hinting`（4.7 默认 3）；`FontFile.fallbacks` 与 `SystemFont.font_names`（4.x 稳定，但组合行为需实测）；4.6 CSV 的 plural/context 列 |
| **Verification Required** | ① `FileAccess.store_string` / `store_buffer` 返回 `bool`（4.4 起）——沿用 ADR-0002 的结论，仍需三平台冒烟；② `Control.accessibility_name` 的赋值类型与 `accessibility_live` 在 4.7 的取值域（4.7 重定类型，可能不再是 `bool`）；③ 4.6/4.7 CSV 导入器对 `keys` + `context` 表头的识别方式（plural/context 列的确切表头）；④ 内置 CJK 字体在导出包中未被子集化掉罕见字（用一份覆盖全部文案字符的截图夹具验证）；⑤ `hinting = 1` 在 20 px 与 30 px 下的 CJK 笔画清晰度（4.7 默认已改为 3 = Full）；⑥ `SystemFont` 在目标平台缺失全部 `font_names` 时是否静默回落到 `ThemeDB.fallback_font`（不得崩溃） |

> ⚠️ **引擎知识缺口**：4.5–4.7 均在 LLM 训练截止之后，且本机没有安装编辑器。本 ADR 的每条引擎断言都必须先按 `Verification Required` 逐条实测；未通过前不得写依赖该断言的 story。

## ADR Dependencies

| Field | Value |
|-------|-------|
| **Depends On** | ADR-0002（持久化格式）— 复用其规范 JSON 规则（整数一律十进制字符串、键排序、无浮点）与五步原子写入原语；ADR-0003（场景与屏幕架构）— `ScreenView` 的 `present → apply → grab_focus` 顺序、`FocusSpec`、`focus_restore` 按 id、`reduce_motion` 关闭过渡。ADR-0001（内容数据格式）— 记录的 `display_name` 字段与 `Ready` 门 |
| **Enables** | 界面流程（#8）的设置屏、模态与保存指示分支；音效反馈（#9）的音量/静音/减少重复音效消费；局内 HUD（#16）的 `text_scale_percent` / `hud_opacity_percent` / `reduce_motion` 消费；`TR-ui-flow-003` 的呈现侧验收（焦点环、同帧重排、读屏顺序） |
| **Blocks** | 所有「第一帧就要正确字号与音量」的验收；设置屏（AC 33/34/35）；删除生涯不重置设置（AC 34、存档 AC 25）；任何硬编码中文字符串的 story |
| **Ordering Note** | 由谁调用 `Settings.load()` 归 ADR-0004（服务组合）；本 ADR 只固定前置条件：**`Settings.load()` 必须在第一屏 `present()` 之前、任何音频总线设置之前完成**。本 ADR 也不决定语言切换入口（MVP 没有），只固定 key 与回退链 |

## Context

### Problem Statement

两件事被 GDD 明确留白，且都横跨多个系统。

其一，设置的**物理形态**：`ui-flow.md` OQ3 问「格式、路径与写入时机（如 `user://settings.cfg`；每次修改即写还是关闭时写；损坏时回退默认值）」，`save-system.md` OQ8 问「设置文件由谁拥有」。两者不解决，音量与无障碍选项就无处存放；而如果每个系统各自读文件，第一帧的字号与音量会取决于调用顺序。

其二，本地化的**呈现边界**：`ui-flow.md` OQ12 与 UI Requirements 要求「所有玩家可见字符串使用 `tr()`，不得硬编码」，但 `data-config.md` OQ6 又留着「`display_name` 是否要改为多语言 key？首版只做中文」。如果记录里的菜名是硬编码中文、界面 chrome 是 key，那么 `tr()` 的覆盖率无法断言，Full Vision 加语言时要把内容数据全部重写一遍。

### Current State

没有代码，没有既有设置文件，也没有翻译表。`ui-flow.md` 规则 24 已固定七个设置项、各自范围、「修改立即生效」、「关闭模态或退出时写入」、「写入失败保留内存值」；`in-game-hud.md` 已固定 `hud_ui_scale`、`hud_card_columns`、100/125/150% 三档与 150% 下的重排要求；`save-system.md` 规则 20/23 已固定「设置不属于生涯档、删除生涯不重置设置」。缺的是：文件在哪、什么格式、谁写、什么时候写、坏了怎么办；key 长什么样、记录名怎么接进来；字体从哪来、缺字怎么办；`τ` 到底乘在哪些尺寸上。

### Constraints

- **机器级、非生涯级**：不随生涯删除、不进生涯备份、不进云同步（`save-system.md` 规则 20）；删除生涯后设置文件**逐字节不变**（AC 25）。
- **不显示生涯保存指示**：设置写入成功/失败都不得触发「已保存」/「未保存」的生涯指示（`ui-flow.md` 规则 21）。
- **立即生效**：七个设置项修改当帧生效，与写盘解耦（规则 24）。
- **写盘可失败**：失败保留内存值、显示「设置可能未保存」、下次修改或退出时重试（规则 24、Edge Case）。
- **不阻塞启动**：设置文件缺失、损坏、版本过高都不得把游戏挡在菜单外；没有设置专用的 `Blocked` 状态。
- **纯呈现**：文本缩放、不透明度、减少动态都不改变任何玩法数值（`architecture.md` 原则 2）。
- **中文 MVP**：只发 `zh_Hans`；多语言是 Full Vision 的事，但 key 与表结构必须现在就能容纳第二种语言。
- **无障碍下限**：对比度 ≥ 4.5:1；150% 不截断关键信息、金额列不重叠、焦点环不被裁切；允许滚动。

### Requirements

- 设置文件有唯一路径与唯一格式，且同一内存状态两次写出逐字节相同（可做夹具）。
- 写入是原子的：崩溃在任意窗口都不会留下半截 JSON。
- 损坏 / 版本过高 / 不可写三种失败都有确定行为，且都不阻断启动。
- 七个设置项的 key、类型、范围、默认值、钳制规则全部固定。
- 所有玩家可见字符串有唯一来源（`tr()`），记录名也是 key，且缺 key 在 CI 阶段就被拦住。
- CJK 缺字有确定的三级回退，且字体导入参数被显式钉住（不依赖 4.7 的新默认值）。
- 100/125/150% 三档的字号、最小尺寸、卡片列数、重排行为都有确定数值。
- 文本缩放与不透明度变化的同帧重排不丢焦点、不裁焦点环。

## Decision

### 1. 设置文件的路径、所有权与生命周期

**路径**：`user://config/settings.json`（同目录下 `settings.bak`、`settings.tmp`、`corrupt-<UTC>.settings.json`）。

- 与生涯档的 `user://save/slot_0/` 完全分目录：删除生涯只会操作 `save/`，不可能误伤 `config/`（`save-system.md` 规则 23）。
- `<UTC>` 沿用 ADR-0002 的 `20261104T153045Z`（无冒号，Windows 合法）。
- **不建锁**：设置是用户本机偏好，双开时后写者覆盖先写者，任何一方胜出都可接受；为它引入第二套 `Blocked` 语义会造出一个 GDD 没有设计的界面状态。这是与生涯档**有意不同**的取舍，不是遗漏。
- **不进云同步、不进备份、不参与迁移链**：`schema_version` 只服务本文件自己的向前兼容。

**所有权**：`ui-flow` 拥有 `SettingsModel`、文件编解码与写入时机。其他系统只消费不可变快照：

| 消费方 | 消费内容 | 方式 |
|---|---|---|
| 音效反馈 | `master_volume_db` / `sfx_volume_db` / `muted` / `reduce_repetitive_cues` | `SettingsModel.audio()` 快照 + `changed` 信号 |
| 局内 HUD | `text_scale_percent` / `hud_opacity_percent` / `reduce_motion` | `SettingsModel.presentation()` 快照 + 信号 |
| 场景与流程管理 | `reduce_motion`（过渡时长为 0） | 同上；**不读文件、不持有模型** |
| 存档 | 无 | 存档不得读写设置文件 |

**MVP 没有语言设置项**：七个设置项固定（规则 24），语言不在这七项里。启动时强制 `TranslationServer.set_locale(&"zh_Hans")`，与 OS 区域无关；理由是本版只发一份翻译，跟随 OS 只会多一条「缺 key 回退」的路径，而收益为零。

### 2. 格式：规范 JSON，复用 ADR-0002 的编码与原子写入

```json
{"schema_version":"1","values":{"hud_opacity_percent":"100","master_volume_db":"-6","muted":false,"reduce_motion":false,"reduce_repetitive_cues":false,"sfx_volume_db":"-6","text_scale_percent":"100"}}
```

- **UTF-8、无 BOM、单行、无尾随换行**；`JSON.stringify(model, "", true, false)`（`indent = ""`、`sort_keys = true`）。与 ADR-0002 §1 完全同一套规则：**所有整数以十进制字符串传输**，文件里不出现 JSON number，因此不出现浮点。布尔保持 JSON `true` / `false`。
- 为什么整数也走字符串：`master_volume_db = −80` 与 `text_scale_percent = 100` 必须无损；统一一条规则比「小整数可以用 number」少一次判断，也少一类「4.0 被写成 4」的往返 bug。
- **键集合封闭**：`schema_version`、`values`、以及七个设置键。未知键在解码时忽略并记 `UNKNOWN_KEY` warning（前向兼容：新版本加键、旧版本读不炸）；缺失键取默认值并记 `MISSING_KEY` warning；类型不符走损坏路径。
- **范围与默认值**（与 `ui-flow.md` 规则 24 逐项对应）：

| 键 | 类型 | 范围 | 默认 | 越界行为 |
|---|---|---|---|---|
| `master_volume_db` | int | −80…0（−80 = 静音） | −6 | 钳制 + `VALUE_CLAMPED` |
| `sfx_volume_db` | int | −40…0 | −6 | 钳制 + `VALUE_CLAMPED` |
| `muted` | bool | — | `false` | 类型不符 → 损坏 |
| `reduce_repetitive_cues` | bool | — | `false` | 类型不符 → 损坏 |
| `text_scale_percent` | int | 100 / 125 / 150 | 100 | 吸附到最近的合法档 + `VALUE_SNAPPED` |
| `reduce_motion` | bool | — | `false` | 类型不符 → 损坏 |
| `hud_opacity_percent` | int | 60…100 | 100 | 钳制 + `VALUE_CLAMPED` |

- `text_scale_percent` 用**吸附**而非钳制：`137` 不是「太小」或「太大」，而是无意义；吸附到 125 并在日志里留一条 warning。
- 两个音量都是**整数 dB**。`AudioServer.set_bus_volume_db()` 接受 `float`，但那是一次呈现层的类型转换，不进入文件、不进入模型、不参与相等比较；`−80` 由 `muted` 之外的第二个静音路径表达（GDD 明确「−80 视为静音」），消费方按 `max(db, −80)` 处理。

**原子写入**（与 ADR-0002 §4 同序，本文件独立实现一遍同样的五步）：

1. 编码并校验（键集合、类型、范围、`PAYLOAD_TOO_LARGE`）；
2. 打开 `settings.tmp`（`FileAccess.WRITE`）；
3. `store_string(text)`，检查返回 `bool`；`flush()`；`close()`；
4. 主档存在则 `rename(settings.json → settings.bak)`；
5. `rename(settings.tmp → settings.json)`。

任一步失败：删除 `settings.tmp`，主档与备份保持原样，返回错误。**不得**直接写主档，**不得**在未 `close` 前改名，**不得**把 `flush()` 当成 `fsync`（ADR-0002 Risks 已记录同一限制）。

### 3. 写入时机：模态关闭 / 退出各一次，绝不每帧、绝不每 tick

**模型与磁盘解耦**：`set_*` 只改内存并置 `_dirty = true`，立即发 `changed`；滑杆拖动期间**不写盘**。

写入由三个确定的时机触发，其余都不写：

| 时机 | 调用 | 语义 |
|---|---|---|
| `SettingsModal` 关闭（确认、取消、`ui_cancel`、删除生涯流程结束后回到设置） | `UIRoot` → `SettingsModel.flush(&"modal_close")` | 一次关闭最多写一次 |
| 玩家请求退出游戏（`SceneFlow` 的 `Quit` 请求被接受） | 组合根 → `flush(&"quit")` | 写完后才允许进程退出 |
| 上一次写入失败后，玩家再次修改任一项并再次关闭 | 同上 | 隐式重试；不额外弹窗 |

- **合并**：一次 `flush` 写整个文档，不写差量；`_dirty = false` 只在五步全部成功后设置。
- **空闲不轮询**：没有 500 ms 防抖定时器，也没有 `_process` 里的脏检查。`ui-flow.md` 规则 28 要求「除保存指示与收据滚动外不存在隐藏计时器」，定时写盘会直接违反。
- **失败路径**：保留内存值，`_dirty` 保持 `true`，在**设置模态内**显示「设置可能未保存」。不显示生涯「已保存」/「未保存」指示（规则 21），不阻塞关闭模态，不重试循环。
- **删除生涯不触发设置写盘**，也不重新读取设置文件（规则 24 Edge Case）；设置文件在删除前后逐字节相同。
- **启动时不回写**：`load()` 只读。缺少文件或损坏时用默认值，把「是否落盘默认值」推迟到第一次 `flush`，避免每次启动都写一次文件。

### 4. 载入失败：默认值 + 隔离 + 只读，全部不阻断

```gdscript
enum LoadOutcome {
    LOADED,                        # 正常读取（含缺键取默认、越界钳制）
    DEFAULTS_FIRST_RUN,            # 文件不存在
    DEFAULTS_AFTER_CORRUPT,        # 解析失败或类型不符；旧文件已隔离
    DEFAULTS_AFTER_VERSION_AHEAD,  # schema_version > 当前；旧文件保留不动
    DEFAULTS_NOT_WRITABLE          # 目录不可创建/不可写
}
enum SettingsError { OK, NOT_WRITABLE, IO_ERROR, CORRUPT, VERSION_AHEAD, PAYLOAD_TOO_LARGE }
```

判定顺序（先命中先返回）：

1. `DirAccess.make_dir_recursive_absolute("user://config")` 失败 → `DEFAULTS_NOT_WRITABLE`，`writable = false`。
2. 主档不存在 → `DEFAULTS_FIRST_RUN`，`writable = true`。
3. `JSON.parse_string` 失败、顶层不是 `Dictionary`、`values` 不是 `Dictionary`、任一键类型不符 → 最佳努力把主档改名为 `corrupt-<UTC>.settings.json`（改名失败就保留），返回 `DEFAULTS_AFTER_CORRUPT`。
4. `schema_version` 不是十进制字符串或大于 `SETTINGS_SCHEMA_VERSION` → **不动文件**，返回 `DEFAULTS_AFTER_VERSION_AHEAD`，`writable = false`（本会话只读：绝不覆盖更新版本写下的设置）。
5. 其余 → 逐键解码、钳制/吸附、收集 warnings，`LOADED`。

- **永不阻断**：没有设置专用的错误屏、没有模态、没有 `Blocked`。最坏情况是「这一局用默认音量与 100% 字号」，玩家仍能进菜单。
- **只读会话**：`writable = false` 时所有 `set_*` 仍立即生效（内存），`flush()` 返回 `NOT_WRITABLE`，界面显示「设置可能未保存」。这保证「版本过高」不会被旧版本静默降级覆盖。
- **`PAYLOAD_TOO_LARGE`**：编码结果 > 16 KiB 时拒绝写入（正常文档 < 1 KiB；这是防御，不是预算）。

### 5. `tr()` key 策略：所有玩家可见字符串，含记录名

**key 语法**（CI 与 `DataConfig` 校验器共用同一正则）：

```
^[a-z][a-z0-9_]*(\.[a-z0-9_]+){2,4}$
```

3–5 段、全小写、只用 `.` 与 `_`、总长 ≤ 96。**域**固定为五个：

| 域 | 用途 | 示例 |
|---|---|---|
| `ui` | 菜单、设置、收据、暂停、错误屏的 chrome | `ui.main_menu.start_button`、`ui.settings.text_scale.label` |
| `hud` | 局内 HUD 标签、单位、格式串 | `hud.top_bar.turns_format`（`%d / %d`）、`hud.wave.hidden_count`（`%d 道未揭晓`） |
| `err` | 错误与状态文案，按错误码一一对应 | `err.save.corrupt`、`err.settings.not_saved` |
| `content` | 内容记录的名字/描述 | `content.dish.hongshaorou.name` |
| `a11y` | 仅读屏使用的描述（视觉上不显示） | `a11y.settings.text_scale.description` |

**硬规则**：

- 调用点只允许 `tr(&"字面量")` 或 `tr(KEY_CONST)`；**禁止**用字符串拼接造 key，唯一例外是 `content.<type>.<id>.<field>` 的构造，它由 `TranslationKeys.content(type, id, field)` 这一个**全函数**产出（`id` 已通过 `^[a-z][a-z0-9_]*$`，`type ∈ {dish, upgrade, station}`，`field ∈ {name, description}`）。
- 一句完整的话一个 key，不做「前缀 + 片段」拼接：中文语序与格助词会让拼接在第二种语言下崩掉。
- 带占位符的 key 用 `%` 格式，调用 `tr(key) % [a, b]`；**占位符顺序是 key 契约的一部分**，改顺序等于改 key。
- 数字格式由 `hud_number_format` 常量决定（`+` / `−` / `%` / 小数位），不进翻译表；符号与数字之间的空格属于格式常量。日期/金额的本地化格式在 MVP 不需要（无日期展示）。
- **缺 key 的行为**：`tr()` 找不到时返回 key 本身。正式构建里这会被 CI 拦住（见下）；运行期若仍发生，调试构建 `push_error`，发布构建原样显示 key 并记一次 warning——显示 key 是可见的 bug，比静默显示空字符串或英文更容易被发现。
- **CI 双向检查**：① 提取所有 `tr(` 字面量与 `TranslationKeys.content()` 的可达集合，断言每个 key 在 `zh_Hans` 表里有非空值；② 断言表里没有未被引用的 key（warning，不失败——`a11y.*` 可能只在编辑器夹具里引用）。

**`display_name` 的裁决（关闭 `data-config.md` OQ6）**：`DishData.display_name`、`UpgradeData.display_name`、`UpgradeData.description`、`StationData.display_name` 的值**必须是 `tr()` key**，语法为 `^content\.(dish|upgrade|station)\.[a-z0-9_]+\.[a-z_]+$`。

- 记录结构不变（仍是 `String`），只是**值的语义**从「中文文本」变成「key」。没有 `.tres` 字段改名，没有新增字段。
- 中文文案放进 `zh_Hans` 翻译表，而不是 `.tres`。Full Vision 加语言时只加一列，不改任何记录。
- 不同记录**可以**指向同一个 key（例如两道同名菜），因此字段不是 `id` 的冗余；校验只查语法，不强制等于 `content.<type>.<id>.name`。
- `DataConfig` 只校验**语法**（新增诊断码 `DISPLAY_NAME_NOT_A_KEY`），不做翻译查表——翻译表的加载与 `DataConfig` 的启动门是两条独立路径，互相不构成前置条件。存在性由 CI 检查，理由同上。
- 呈现层解析：`tr(record.display_name)`。**禁止**在 `DataConfig`、结算、存档里解析；记录保持「key 原样往返」。

**翻译表布局**：`res://localization/strings.csv`，表头 `keys,zh_Hans,context`；Godot 导入为 `strings.zh_Hans.translation` 并登记到 `internationalization/locale/translations`。MVP 只有 `zh_Hans` 一列。4.6 的 plural/context 列若要求特定表头（如 `context_` 前缀），按实测调整表头，**key 集合不变**（`Verification Required` ③）。

### 6. 语言回退链：`zh_Hans → zh → 项目默认 → key`

回退由 `TranslationServer` 承担，不自己写查表逻辑：

- 项目设置 `internationalization/locale/fallback = "zh_Hans"`；启动时 `TranslationServer.set_locale(&"zh_Hans")`。
- 解析顺序（引擎语义 + 我们的项目设置）：精确 locale `zh_Hans` → 语言层 `zh` → 项目 fallback `zh_Hans` → 返回 key。重复项去重后顺序固定。
- **MVP 不做语言切换 UI**；若未来加入，切换只调用 `TranslationServer.set_locale()`，随后广播 `NOTIFICATION_TRANSLATION_CHANGED`，每个 `ScreenView` 重新 `present()`，**不重建屏幕、不重置焦点**（按 id 保持，见 ADR-0003 D5）。
- `tr()` 结果长度会变，因此语言切换必须触发与文本缩放同一条重排路径（决策 §8）。
- 读屏文本同样是 key（`a11y.*`），随语言一起换；`Control.accessibility_name` 在每次 `present()` 重新赋值，不缓存旧语言的字符串。

### 7. CJK 字体回退链与导入参数

**三级回退**，挂在 `Theme.default_font` 上：

1. **内置** `res://assets/fonts/NotoSansSC-Regular.ttf`（正文）与 `NotoSansSC-Bold.ttf`（强调），OFL 1.1，许可证文件同目录。数字列另用 `NotoSansMono-Regular.ttf`（`NumberLabel` 类型变体），保证金额/回合等宽、放大时不跳动。
2. **系统** `SystemFont`，`font_names = ["Noto Sans CJK SC", "Source Han Sans SC", "PingFang SC", "Hiragino Sans GB", "Microsoft YaHei", "SimHei"]`，`allow_system_fallback = true`。作为 `FontFile.fallbacks` 的第二项。
3. **引擎** `ThemeDB.fallback_font`，最后兜底（只有拉丁字形，但不会崩）。

- 为什么内置字体全覆盖还要系统回退：内置字体的导出子集、罕见字、符号（`?`、`−`、`×`、`→`）与未来 emoji 都可能缺字；系统回退把「豆腐块」降级为「字形略不同」，比缺字好。
- 为什么不用 MSDF：CJK 笔画密集，MSDF 在 20 px 下会出现笔画断裂；1.5× 放大完全在动态光栅化的舒适区。
- **导入参数显式钉住**（写在 `.import` 里，不依赖 4.7 新默认值）：`antialiasing = 1`（Grayscale）、`hinting = 1`（Light；4.7 默认是 3 = Full，会让 20 px 汉字横画粘连）、`subpixel_positioning = 1`、`multichannel_signed_distance_field = false`、`allow_system_fallback = true`、`force_autohinter = false`。
- 行高：`line_height = ceil(font_size × 1.3)`，按类型变体固定在 `Theme` 里，不用引擎默认行距——否则三档字号的基线会在不同平台漂移。
- 字体文件进导出的方式沿用 ADR-0001 的教训：`.ttf` 必须作为 `Theme` 的依赖被打进导出包，不能只靠 `load("res://...")` 的隐式引用；导出后跑一次「全部文案字符可渲染」的截图夹具（`Verification Required` ④）。

### 8. 文本缩放 100 / 125 / 150%：`τ` 只乘字号与文字相关尺寸

模型存 **int 百分比**，呈现层才转 `τ = text_scale_percent / 100.0`。`τ` 的乘域是**封闭清单**，不在清单里的尺寸一律不乘：

| 对象 | 基准（100%） | 125% | 150% | 规则 |
|---|---|---|---|---|
| 正文字号 | 20 | 25 | 30 | `round(20 × τ)` |
| 数字字号 | 28 | 35 | 42 | `round(28 × τ)` |
| 标签/最小可读 | 16 | 20 | 24 | `round(16 × τ)`，不低于 16 |
| 标题字号 | 36 | 45 | 54 | `round(36 × τ)` |
| 行高 | `ceil(font_size × 1.3)` | 同 | 同 | 按字号推导，不单独缩放 |
| 按钮最小高度 | 44 | 55 | 66 | `max(44, round(44 × τ))` |
| 列表行最小高度 | 32 | 40 | 48 | `max(32, round(32 × τ))` |
| 卡片最小宽度 `w_min` | 200 | 250 | 300 | `200 × s × τ`（`s = hud_ui_scale`） |
| 卡片高度 | 176 | 198 | 220 | 三档常量表（上限 220，不越出 HUD 调参范围） |
| 顶栏高度 | 72 | 84 | 96 | 上限 96（GDD 允许 150% 增至 96） |
| 金额列最小宽度 | 96 | 120 | 144 | `round(96 × τ)`，只增不减 |
| 面板内边距 | 12 | 15 | 18 | `round(12 × τ)` |
| 控件间距 `g` | 16 | 16 | 16 | **不乘**（纯布局常量） |
| 安全边距 | 36 | 36 | 36 | **不乘**（`hud_safe_margin` 独立规则） |
| 焦点环宽度 | 2 | 2 | 2 | **不乘**；改为预留 4 px 外边距 |

- **不做二次缩放**：菜单屏走 `canvas_items` 拉伸，窗口变大时控件已被引擎放大，`τ` 只作用在字号与上表的文字相关尺寸上；把 `τ` 乘到整个 `Control.scale` 上会造成「窗口 × 字号」双重放大，150% 在 1080p 下等于 225%。
- **列数重排**：`hud_card_columns = clamp(floor((G + g) / (w_min + g)), 1, 4)`，`w_min = 200 × s × τ`、`g = 16 × s`。@1280、`s = 1`、`G = 912`：τ=1 → 4 列（卡宽 216 px）；τ=1.25 → 3 列；τ=1.5 → 2 列（4 张卡排成 2×2，与 `in-game-hud.md` AC 11 一致）。
- **重排是同帧的**：`SettingsModel.presentation()` 变化 → 组合根调用 `UIRoot.apply_presentation(prefs)` → 换用预构建的 `Theme` 变体（三个 τ 各一份，启动时构建，避免运行期重建 `Theme` 触发的全树重排抖动）→ 每个活动 `ScreenView.reflow()` → `queue_sort()`。同一帧内完成，下一帧渲染即新字号。
- **不截断**：散句 `Label.autowrap_mode = AUTOWRAP_WORD_SMART`（CJK 按字断行可接受）；金额/回合/百分比 `AUTOWRAP_OFF` + `clip_text = false`，宁可横向滚动也不折行；表格在窄宽度下切换为 `STACKED` 布局（标签在上、金额在下），而不是缩小字号。允许 `ScrollContainer`，不允许「关键信息消失」（AC 35）。
- **焦点环不被裁切**：任何可聚焦控件的容器必须在四边预留 ≥ 4 px（用外层 `MarginContainer`），或把 `focus` `StyleBox` 做成向内描边。150% 下卡片变高、顶栏变高，但焦点环的 2 px 描边与 4 px 预留是常量，不参与缩放。
- **焦点保持**：重排后按 id 重新 `grab_focus()`（ADR-0003 D5 的 `focus_restore` 语义），控件消失时落到 `FocusSpec.order` 中相邻可用项；重排**不得**改变 `FocusSpec.order` 或 `initial_focus`——顺序是声明式的，与几何无关，这正是 150% 下方向键仍然符合直觉的原因。

### 9. `reduce_motion` 与 `hud_opacity_percent` 的消费契约

**`reduce_motion = true`** 时，以下时长**全部为 0**，且状态仍同帧生效：

- 顶层切换的 `FADE_OUT` / `FADE_IN`（ADR-0003 D8）；
- 收据数字滚动 `receipt_roll_duration`（`ui-flow.md` 规则 14 的公式，滚动中打开也立即结束到最终值）；
- 保存指示的淡出（显示时长仍是 1.5 s，但无淡入淡出；`SAVE_INDICATOR_SECONDS` 不变）；
- 任何 `Tween` 驱动的 hover/press 视觉（本作本来就只有 `StyleBox` 直切，无补间）。

不减少信息、不改变任何确认的可用性、不引入新的等待。

**`hud_opacity_percent`**：

- 只作用于 HUD 的**面板底色**（`ColorRect` / `PanelContainer` 的 `StyleBoxFlat.bg_color.a = percent / 100.0`），**不作用于文字**（文字 alpha 恒为 1.0）。
- 60% 是**设计下限**：主题必须保证 60% 不透明度叠在玩法底色上时，每一组文字/背景对比度仍 ≥ 4.5:1（`hud_contrast_min`）。若某个面板在 60% 下达不到，则对该面板取「能达到 4.5:1 的最低不透明度」作为有效值（`opacity_effective = max(requested, panel_floor)`），并把 `panel_floor` 写进主题常量表，不写进代码分支。
- 不影响暂停覆盖层、收据、模态遮罩的遮挡不透明度——那些必须足够不透明以承担「遮挡」职责，与 HUD 的可读性设置是两件事。
- 修改立即生效（同一帧重绘），不需要重排。

**传输方式（关闭 `ui-flow.md` OQ2 的一半）**：`SettingsModel` 发出 `presentation_changed(prefs: PresentationPrefs)`，`PresentationPrefs` 是**不可变值对象**（`text_scale_percent: int`、`hud_opacity_percent: int`、`reduce_motion: bool`）。`SceneFlow` 通过组合根订阅该信号并缓存最新值，**不读设置文件、不持有 `SettingsModel`**；过渡开始时读缓存值决定时长。这样「谁把设置传给场景流程」不再留白，也不引入第二份设置状态。

### 10. 呈现侧的可访问性与读屏

- 每个可交互控件的 `accessibility_name` / `accessibility_description` 在 `present()` 时从 `tr()` 重新赋值（`a11y.*` 或对应 `ui.*` key），不缓存跨语言的旧字符串。
- 焦点环必须同时有**颜色 + 描边**两个通道（纯描边，`draw_center = false`），与 hover 样式互不推导（ADR-0003 D6）；150% 下描边宽度不变、预留边距不变。
- 「设置可能未保存」与生涯「已保存」是两条独立通道：前者只出现在设置模态内，后者只在生涯写盘后出现（`ui-flow.md` 规则 21）。两者都用文字表达，不只靠颜色。
- 数字列永远有 `+` / `−` 符号与文字标签（「净利」「回本」），符号不是唯一通道。

## Architecture Diagram

```
Boot（组合根，ADR-0004 决定调用点）
  ├─ DataConfig.load() ─────────────► READY
  ├─ MilestoneCatalog.load()
  ├─ SettingsModel.load() ──────────► user://config/settings.json
  │     └─ DEFAULTS_* / LOADED → 立即广播 presentation_changed
  └─ Save.load() → SceneFlow Boot → Menu

SettingsModal（ui-flow）
  ├─ 滑杆/开关 → SettingsModel.set_*()  ── 立即发 changed（当帧生效）
  │      ├─ AudioFeedback.apply_settings(audio)   → AudioServer 总线
  │      ├─ HudHost.apply_presentation(presentation)
  │      └─ SceneFlow 缓存 prefs（只用于过渡时长）
  └─ 关闭 → flush("modal_close")
        └─ 编码 → tmp → flush → close → 主档→bak → tmp→主档
              失败 → 保留内存值 + 模态内「设置可能未保存」

Presentation（ui-flow）
  ├─ Theme 变体 ×3（τ = 1.0 / 1.25 / 1.5，启动时预构建）
  ├─ tr() ← strings.zh_Hans.translation（TranslationServer，fallback = zh_Hans）
  └─ FontFile.fallbacks = [内置 Noto Sans SC, SystemFont(CJK), ThemeDB]
```

## Key Interfaces

> 每个 `class_name` 各占一个 `.gd`；此处合并展示只为阅读。`SettingsSnapshot` / `PresentationPrefs` / `AudioSettings` 都是**不可变值对象**：构造后只读，消费方不得持有可变引用。

```gdscript
# UI — src/ui/settings/

class_name SettingsModel extends RefCounted

signal changed(key: StringName, value: Variant)
signal presentation_changed(prefs: PresentationPrefs)

const SETTINGS_SCHEMA_VERSION: int = 1
const SETTINGS_DIR: String = "user://config"
const SETTINGS_PATH: String = "user://config/settings.json"
const PAYLOAD_MAX_BYTES: int = 16384

enum SettingsError { OK, NOT_WRITABLE, IO_ERROR, CORRUPT, VERSION_AHEAD, PAYLOAD_TOO_LARGE }
enum LoadOutcome { LOADED, DEFAULTS_FIRST_RUN, DEFAULTS_AFTER_CORRUPT,
                   DEFAULTS_AFTER_VERSION_AHEAD, DEFAULTS_NOT_WRITABLE }
enum FlushResult { OK, NOTHING_TO_WRITE, NOT_WRITABLE, IO_ERROR, PAYLOAD_TOO_LARGE, VERSION_AHEAD }
enum FlushReason { MODAL_CLOSE, QUIT, EXPLICIT }
enum SetStatus { OK, CLAMPED, SNAPPED, UNKNOWN_KEY, TYPE_MISMATCH }

enum WarningCode { UNKNOWN_KEY, MISSING_KEY, VALUE_CLAMPED, VALUE_SNAPPED, CORRUPT_FILE_KEPT }

func load() -> SettingsLoadReport
func get_int(key: StringName) -> int
func get_bool(key: StringName) -> bool
func set_int(key: StringName, value: int) -> SettingsSetResult     # 钳制/吸附，永不失败
func set_bool(key: StringName, value: bool) -> SettingsSetResult
func snapshot() -> SettingsSnapshot
func audio() -> AudioSettings
func presentation() -> PresentationPrefs
func is_dirty() -> bool
func is_writable() -> bool
func flush(reason: FlushReason) -> FlushResult
func reset_to_defaults() -> void                                   # 只改内存 + 置 dirty


class_name SettingsLoadReport extends RefCounted
var outcome: SettingsModel.LoadOutcome = SettingsModel.LoadOutcome.LOADED
var error: SettingsModel.SettingsError = SettingsModel.SettingsError.OK
var writable: bool = true
var warnings: Array[SettingsWarning] = []
func is_ok() -> bool


class_name SettingsWarning extends RefCounted
var code: SettingsModel.WarningCode
var key: StringName = &""
var detail: String = ""


class_name SettingsSetResult extends RefCounted
var accepted: bool = true
var effective: Variant = null        # 钳制后的实际值；调用方用它刷新控件
var status: SettingsModel.SetStatus = SettingsModel.SetStatus.OK


class_name SettingsSnapshot extends RefCounted            # 不可变
var master_volume_db: int = -6
var sfx_volume_db: int = -6
var muted: bool = false
var reduce_repetitive_cues: bool = false
var text_scale_percent: int = 100
var reduce_motion: bool = false
var hud_opacity_percent: int = 100


class_name AudioSettings extends RefCounted               # 不可变投影
var master_volume_db: int = -6
var sfx_volume_db: int = -6
var muted: bool = false
var reduce_repetitive_cues: bool = false


class_name PresentationPrefs extends RefCounted           # 不可变投影
var text_scale_percent: int = 100
var hud_opacity_percent: int = 100
var reduce_motion: bool = false


# UI — src/ui/localization/

class_name TranslationKeys extends RefCounted

const KEY_PATTERN: String = "^[a-z][a-z0-9_]*(\\.[a-z0-9_]+){2,4}$"

static func content(type: StringName, id: StringName, field: StringName) -> StringName
static func is_valid_key(key: String) -> bool
static func validate_key(key: String) -> bool      # 调试构建：push_error；发布构建：只返回结果，不中断


# UI — src/ui/theme/

class_name TextScale extends RefCounted

const SCALES: Array[int] = [100, 125, 150]
const BODY_PX: int = 20
const NUMBER_PX: int = 28
const LABEL_PX: int = 16
const TITLE_PX: int = 36

static func tau(percent: int) -> float                     # 仅呈现层使用
static func font_px(base_px: int, percent: int) -> int     # round(base × τ)
static func card_columns(main_width: int, s: float, percent: int) -> int
static func theme_variant(percent: int) -> Theme           # 三份预构建之一
```

## Implementation Guidelines

**必须**

1. 设置文件必须只经 `SettingsModel` 读写；任何系统都不得直接打开 `user://config/settings.json`。
2. 编码必须走 ADR-0002 的规范 JSON 规则：单行、无 BOM、`sort_keys = true`、**整数一律十进制字符串**、无浮点。
3. 写入必须是五步原子替换；任一步失败删除 `settings.tmp`，不改动主档与备份。
4. `_dirty` 只能在五步全部成功后清零；失败必须保留，供下次 `flush` 重试。
5. `set_*` 必须立即发 `changed` / `presentation_changed`，并且永不因写盘状态而失败或回滚。
6. 设置写盘必须与生涯保存指示完全隔离：成功与失败都不得触发「已保存」/「未保存」。
7. 删除生涯的路径不得读写设置文件；测试必须断言设置文件删除前后逐字节相同。
8. 所有玩家可见字符串必须经 `tr()`；记录名必须是 `content.*` key，且呈现层才解析。
9. `tr()` 调用点只允许字面量或 `KEY_CONST`；`content.*` 只能由 `TranslationKeys.content()` 构造。
10. 字体导入参数必须在 `.import` 中显式写出（`hinting = 1`、`antialiasing = 1`、`subpixel_positioning = 1`、`msdf = false`），不得依赖 4.7 默认值。
11. `τ` 只能作用在决策 §8 的清单内；清单外的间距、安全边距、焦点环宽度不得缩放。
12. 文本缩放变化必须在同一帧内完成换主题 + `reflow` + 焦点恢复（按 id）。
13. 150% 下关键信息不得被截断；允许 `ScrollContainer`，允许折行，不允许 `clip_text = true` 吃掉金额。
14. `reduce_motion` 必须把过渡、收据滚动、指示淡出三处时长置 0，且不改变任何状态生效时机。
15. 设置载入必须在第一屏 `present()` 之前完成；`load()` 只读，不回写。
16. 三份 `Theme` 变体必须在启动时构建一次，运行期只切换引用。

**不得**

1. 不得把设置写进生涯档、生涯备份或云同步；不得让 `delete_career` 触碰 `config/`。
2. 不得在滑杆拖动、`_process`、`_input` 里写盘；不得引入防抖/轮询定时器。
3. 不得在 `schema_version` 高于当前时覆盖文件；只读会话必须如实返回 `VERSION_AHEAD`。
4. 不得用 `ConfigFile`、`.cfg`、二进制或 `store_var` 存设置——全项目只保留一种序列化格式。
5. 不得在文件里存 `float`（含 `0.0` 形式的音量）；音量是整数 dB。
6. 不得硬编码任何玩家可见中文；不得把 `tr()` 结果拼成 key；不得在 `DataConfig` / 结算 / 存档里解析 `display_name`。
7. 不得用字体缩小（`fit_content` / 动态降字号）来「解决」150% 溢出。
8. 不得让 `τ` 乘到 `Control.scale` 或整个 HUD 的变换上（会造成双重缩放）。
9. 不得假设 `Control.accessibility_live` 仍是 `bool`；按 4.7 的实际类型使用，并列入验证。
10. 不得在设置失败时弹模态、阻塞关闭或重试循环。

## Alternatives Considered

### Alternative 1: `ConfigFile` + `user://settings.cfg`

**Description**：用 Godot 内置 `ConfigFile` 存 `[audio]` / `[ui]` 段，保存到 `user://settings.cfg`。

**Pros**：API 最短；天然分组；人类可读；不需要自己写编码/解码。

**Cons**：引入第二种序列化格式（项目已有规范 JSON）；`ConfigFile` 的键顺序与转义不由我们控制，无法承诺「同一状态逐字节相同」；int64 与布尔混排时缺少统一校验层；`store_*` 的 bool 返回与原子性仍要自己处理。

**Estimated Effort**：0.5 天。

**Rejection Reason**：省下的代码量不足以抵消「两套格式、两套夹具、两套失败路径」的长期成本；而且 GDD 明确把「格式」列为待决问题，选一个能与存档共用原语的答案更有价值。

### Alternative 2: 每次修改立即写盘（或 500 ms 防抖）

**Description**：`set_*` 后直接 `flush()`；或用 500 ms 定时器合并。

**Pros**：崩溃最多丢一次修改；实现直觉。

**Cons**：滑杆一次拖动会产生几十次写盘；防抖定时器直接违反 `ui-flow.md` 规则 28（除保存指示与收据滚动外不得有隐藏计时器）；写盘失败会与「立即生效」纠缠（回滚还是保留？）。而设置丢失的最坏后果只是回到默认音量，与存档丢失不是一个量级。

**Estimated Effort**：0.5 天。

**Rejection Reason**：收益（少丢一次音量调整）远小于成本（IO 抖动、规则冲突、失败语义复杂）。关闭时写一次已经把丢失窗口压到「一局之内未关设置就崩溃」。

### Alternative 3: 设置存入生涯档的一个命名空间

**Description**：在 `career.json` 里加 `settings` 命名空间。

**Pros**：只有一条写盘路径；随存档一起原子提交。

**Cons**：直接违反 `save-system.md` 规则 20（设置不随生涯删除、不进备份、不进云同步）；删除生涯会清空字号与音量；双开时设置被生涯写锁阻塞。

**Estimated Effort**：0.2 天。

**Rejection Reason**：与已批准 GDD 的硬规则冲突，没有讨论空间。

### Alternative 4: `display_name` 保持中文字面量，只对界面 chrome 用 `tr()`

**Description**：记录里继续写「红烧肉」，`tr()` 只覆盖按钮与标签。

**Pros**：内容作者不用查 key；`.tres` 可直接读。

**Cons**：`tr()` 覆盖率无法断言（「所有玩家可见字符串」这条验收永远有例外）；Full Vision 加语言时要把所有记录重写并把中文从数据搬到翻译表；同一个菜名在收据、HUD、升级页三处各写一遍，无法统一改文案。

**Estimated Effort**：1 天（改造 + 数据迁移）。

**Rejection Reason**：把最贵的一次迁移推到「内容已经很多」的时候。现在没有内容，迁移成本为零。

### Alternative 5: MSDF/SDF 字体 + 任意字号

**Description**：用多通道有向距离场字体，字号可任意连续缩放。

**Pros**：任何缩放下边缘都锐利；只需一份图集。

**Cons**：CJK 笔画密集，20 px 下 MSDF 会出现笔画断裂与糊边；本作只需要三档离散缩放，动态光栅化足够；MSDF 还会让 `hinting` 与 `subpixel_positioning` 失效，跨平台字形差异更难控。

**Estimated Effort**：1 天（含视觉返工）。

**Rejection Reason**：为不需要的连续性付出可读性代价。

### Alternative 6: 150% 溢出时自动缩小字号（auto-fit）

**Description**：`Label` 放不下时按容器宽度反算字号。

**Pros**：永远不会溢出；布局简单。

**Cons**：直接违背「150% 是为了让低视力玩家看清」的初衷——一旦自动缩小，最需要放大的玩家拿到的是更小的字；还会造成同一屏内字号不一致。

**Estimated Effort**：0.5 天。

**Rejection Reason**：功能上「通过」验收、目的上完全失败。正确做法是重排 + 滚动。

## Consequences

### Positive

- 设置与生涯彻底解耦：删除生涯、双开、只读存档都不会影响字号与音量；两套文件各自有独立的失败语义。
- 写盘只发生在两个确定时机，没有隐藏计时器，`ui-flow.md` 规则 28 的可测试性得以保持；滑杆拖动不再有 IO 抖动。
- 「所有玩家可见字符串都是 key」在 CI 里可断言，Full Vision 加语言时是加一列而不是改一遍数据。
- 字体参数被钉死，4.7 的 `hinting` 默认值变更不会在升级那天悄悄糊掉全部汉字。
- 150% 的行为被拆成「字号清单 + 列数公式 + 重排规则 + 不截断」，每条都能截图验收（AC 35 的证据链）。
- `PresentationPrefs` 一次性关闭了「`reduce_motion` 怎么传给场景流程」这个悬空问题，且不给 `SceneFlow` 增加文件依赖。

### Negative

- 项目里有两套原子写入实现（存档与设置），虽然步骤相同，但仍是两份代码与两组失败注入测试。
- 关闭设置才写盘意味着「改完音量、直接崩溃」会丢这次修改；这是有意的取舍（见 Alternative 2）。
- 记录名变成 key 后，`.tres` 不再自解释，内容作者必须同时改 CSV；为此需要一条 CI 检查与一份简短的作者说明。
- 内置两套字重 + 等宽字体把安装包增大约数 MB（Noto Sans SC 全字库单字重约 10 MB 量级），需要导出前评估体积；必要时在 Full Vision 做子集化，但子集化本身会引入缺字风险（`Verification Required` ④）。
- 三份 `Theme` 变体常驻内存，换来切换零抖动；内存增量对 720p 项目可忽略，但确实是多了一份状态。

### Neutral

- `schema_version` 与存档的版本号是两套独立编号；它们不共享迁移链，也不承诺同步演进。
- 设置没有锁，双开时后写者胜；这与生涯档的 `Blocked` 语义并存但互不干扰，需要在实现说明里写清楚，避免有人「顺手」给设置加锁。
- `text_scale_percent` 吸附而不是钳制，会在日志里产生 `VALUE_SNAPPED`；它只是诊断，不面向玩家。

## Risks

- **`Control.accessibility_live` 在 4.7 重定类型**：本 ADR 的读屏契约只依赖「有没有这个属性」，但赋值类型写错会编译失败或静默无效。缓解：`Verification Required` ②；实现时先用最小场景验证，再写 story。
- **4.6 CSV 表头规则不确定**：plural/context 列若要求特定前缀，表头会与 `keys,zh_Hans,context` 不同。缓解：把表头做成导入配置的一部分，key 集合不变；`Verification Required` ③ 在写第一条文案前完成。
- **字体子集化掉罕见字**：导出后某句文案出现豆腐块，而编辑器里正常。缓解：`Verification Required` ④ 的「全部文案字符」截图夹具进 CI；系统字体回退作为第二道保险。
- **`hinting = 1` 在三平台观感不一致**：Windows 的 GDI 风格与 macOS 的 CoreText 光栅化本就不同。缓解：钉死导入参数并把「同一字号下三平台截图」列入验收证据；差异超出可接受范围时改为 `hinting = 0`（无 hinting）而不是跟随 4.7 默认的 Full。
- **`tr()` 覆盖率 CI 误报**：动态 key（`content.*`）无法静态提取全部实例。缓解：`TranslationKeys.content()` 是全函数，CI 额外遍历全部记录 id 生成期望 key 集合；漏网的只有「运行时拼 key」，而那已被规则禁止。
- **150% 下某些屏幕仍溢出**：三档字号 + 固定列宽在极端窗口（如 1280×720 + 150%）下可能放不下。缓解：重排优先级写死为「先加高、再减列、再折行、最后滚动」；截断永不作为解。
- **设置文件在云盘目录**：若玩家把 `user://` 放在 OneDrive/iCloud，改名可能被同步进程短暂占用。缓解：`rename` 失败返回 `IO_ERROR`、保留内存值、下次重试；不做指数退避或后台线程。
- **两套写入实现漂移**：设置与存档的五步将来可能被改得不一致。缓解：把五步抽成共享的 `AtomicWrite` 工具（归 ADR-0002 的实现），设置只提供路径与编码；若暂不共享，则两份必须共用同一组失败注入测试。

## GDD Requirements Addressed

| GDD System | Requirement | How This ADR Addresses It |
|------------|-------------|--------------------------|
| `ui-flow.md` | TR-ui-flow-006：机器级设置模型——音量、静音、文本缩放 100/125/150、减少动态、HUD 不透明度 | 决策 §1 所有权与路径；§2 七个键的类型/范围/默认/钳制；§3 写入时机；§4 载入失败；§8 缩放；§9 不透明度与减少动态 |
| `ui-flow.md` | TR-ui-flow-003（呈现侧）：每屏焦点顺序、`initial_focus`、`focus_restore` | 决策 §8「焦点保持」与「焦点环不被裁切」：重排不改变 `FocusSpec.order`/`initial_focus`，按 id 恢复焦点，2 px 描边与 4 px 预留不参与缩放；§10 焦点环双通道 |
| `ui-flow.md` | 规则 24：修改立即生效；关闭模态或退出时写入；失败显示「设置可能未保存」并保留内存值 | 决策 §3：`set_*` 立即广播、`flush` 只在两个时机、失败保持 dirty 且只在模态内提示 |
| `ui-flow.md` | 规则 21：设置写入不显示生涯保存指示 | 决策 §3 失败路径与 `Implementation Guidelines` 必须 6 |
| `ui-flow.md` | 规则 20 Edge Case：删除生涯后设置保持原值、不重新载入设置文件 | 决策 §1 分目录；必须 7 断言设置文件逐字节不变 |
| `ui-flow.md` | UI Requirements：所有玩家可见字符串使用 `tr()`，不得硬编码 | 决策 §5 key 语法/域/构造全函数/CI 双向检查；禁止拼接 |
| `ui-flow.md` | UI Requirements / AC 35：100/125/150%，150% 不溢出、不重叠、不截断，允许滚动 | 决策 §8 的尺寸清单、列数公式、重排顺序与「不截断」规则 |
| `ui-flow.md` | AC 33/34：设置立即生效、写入机器级文件、删除生涯后保持 | 决策 §1–§4；验证表 #1–#4、#9 |
| `ui-flow.md` | AC 36：`reduce_motion` 关闭过渡与数字滚动 | 决策 §9；与 ADR-0003 D8 的时长置 0 对齐 |
| `ui-flow.md` | OQ3：设置文件格式、路径、写入时机、损坏回退 | 决策 §1/§2/§3/§4 全部关闭 |
| `ui-flow.md` | OQ12：正式中文、字符上限、折行规则、`tr()` key 命名 | 决策 §5 key 语法与 `content.*`；折行在 §8（散句折行、数字不折、滚动兜底）；字符上限由 `/ux-design` 文案表持有，本 ADR 固定 key 与表结构 |
| `in-game-hud.md` | `hud_ui_scale` / `hud_card_columns` / 字号三档 / 不透明度下限 / AC 11、30–32 | 决策 §8 的 `w_min` 与列数公式（4/3/2）、顶栏 72/84/96；§9 的 `panel_floor` 与 ≥ 4.5:1 |
| `save-system.md` | 规则 20：设置不属于生涯档，不随删除、不随备份、不进云同步 | 决策 §1：独立目录、独立文件、无锁；`Implementation Guidelines` 不得 1 |
| `save-system.md` | 规则 23 / AC 25：删除生涯隔离旧档，设置文件逐字节不变 | 决策 §1 + 必须 7；验证表 #9 |
| `save-system.md` | OQ8：设置文件由谁拥有 | 决策 §1 明确 `ui-flow` 拥有模型与文件，音效反馈只消费 |
| `data-config.md` | OQ6：`display_name` 是否改为多语言 key | 决策 §5：改为 key，语法固定，结构不变，CI 查存在性；关闭 OQ6 |
| `data-config.md` | `DishData.display_name` / `UpgradeData.display_name`·`description` / `StationData.display_name` | 决策 §5 的字段语义变更与 `DISPLAY_NAME_NOT_A_KEY` 诊断；呈现层 `tr()` 解析 |
| `architecture.md` | 原则 2：呈现只读；原则 8：组合根注入而非单例 | 决策 §1 的所有权表与 `PresentationPrefs` 信号；`SettingsModel` 由组合根构造并注入，不做 autoload |

## Performance Implications

- **CPU**：设置文档 < 1 KiB，编码 = 一次 `JSON.stringify`，解码 = 一次 `JSON.parse_string` + 七个键的校验；写入 < 2 ms，载入 < 1 ms，各发生一次/一局。三份 `Theme` 变体在启动时构建一次（每个 τ 一组字号/最小尺寸），运行期切换只是换引用，重排是 `queue_sort()` 一次布局，不超过一帧。
- **Memory**：设置模型 < 1 KiB；三份 `Theme` + 字形缓存（三档字号 × 两套字体）在 10 MB 量级，其中字体图集占大头。HUD 卡片在 150% 下从 4 列变 2 列，同屏控件数减少，布局节点数不增。
- **Load Time**：`SettingsModel.load()` p95 ≤ 5 ms（含 `make_dir_recursive` 与一次文件读）；字体首次光栅化在首屏出现时完成，冷启动首帧可能多 10–30 ms，已在 `BootWaiting` 内消化。
- **Storage**：`settings.json` < 1 KiB + `settings.bak` < 1 KiB；损坏隔离文件最多与启动次数同量，由未来的「清理隔离文件」入口处理（`save-system.md` OQ6 同类问题）。
- **Network**：无。明确不进云同步。

## Migration Plan

绿地实现，无既有设置文件。落地顺序：

1. `SettingsCodec`（编码/解码 + 七个键的范围/类型表 + 往返夹具），先于任何文件 IO。
2. `AtomicWrite`（与 ADR-0002 共用或同构的五步） + 失败注入（编码失败、`store_string` 返回 false、两次改名之间崩溃）。
3. `SettingsModel`（内存模型、钳制/吸附、`changed` / `presentation_changed`、`flush` 状态机） + AC 33 夹具。
4. `SettingsModel.load()` 的五种 outcome + 损坏隔离 + 版本过高只读 + 不可写，覆盖 AC 33/34 与本文验证表 #1–#6。
5. `TranslationKeys` + `strings.csv`（先只放 `ui.*` 与 `err.*`）+ CI 双向检查。
6. `DataConfig` 增加 `DISPLAY_NAME_NOT_A_KEY` 诊断，记录值改为 key；跑一次「全部记录 key 存在」的 CI。
7. 字体资源与 `.import` 参数 + `Theme` 三档变体 + `hud_card_columns` 重排 + 150% 截图夹具（AC 35）。
8. 接入组合根：`Boot` 顺序插入 `SettingsModel.load()`，音频总线首帧前应用音量；`SceneFlow` 订阅 `presentation_changed`。
9. `reduce_motion` 连通 ADR-0003 D8 与收据滚动；`hud_opacity_percent` 连通 HUD 面板。

**Rollback plan**：本 ADR 在写代码前确定，回滚即改文档。若规范 JSON 方案被推翻，`SettingsCodec` 是唯一接缝——换成 `ConfigFile` 只需改编码层，模型与消费方不动（但会失去「逐字节可复现」的验收）。若「记录名是 key」被推翻，`TranslationKeys` 与 `tr(record.display_name)` 两处是接缝，可退回「中文字面量 + 只翻译 chrome」，代价是放弃 `tr()` 覆盖率的断言。

## Validation Criteria

| # | 验证 | 证据 |
|---|------|------|
| 1 | AC 33：七项任一修改当帧生效；写入 `user://config/settings.json`；不产生生涯保存指示；删除生涯后设置保持 | 注入式 UI 夹具 + 调用记录器（不出现 `save.commit`、不出现生涯指示） |
| 2 | 同一内存状态连续两次 `flush()`（排除备份改名差异）→ 主档字节相同、键顺序相同 | 哈希比对 |
| 3 | 写失败（目录不可写 / `store_string` 返回 false / 改名失败）→ 主档与备份不变、`settings.tmp` 被删除、`_dirty` 仍为 true、模态显示「设置可能未保存」 | 三处失败注入 |
| 4 | 两次改名之间杀进程 → 下次启动得到 `LOADED`（主档）或从备份恢复，绝不出现半截 JSON | 崩溃夹具 |
| 5 | 文件不存在 → `DEFAULTS_FIRST_RUN` + 默认值 + 不写盘；截断 JSON / 类型错误 → `DEFAULTS_AFTER_CORRUPT` + 隔离文件存在 | 五种 outcome 夹具 |
| 6 | `schema_version = 2` → `DEFAULTS_AFTER_VERSION_AHEAD`、`writable = false`、`flush()` 返回 `VERSION_AHEAD`、原文件字节不变 | 版本夹具 |
| 7 | `text_scale_percent = 137` → 吸附到 125 + `VALUE_SNAPPED`；`master_volume_db = -120` → 钳制到 −80 | 边界夹具 |
| 8 | CI：每个 `tr()` 字面量与每条记录的 `content.*` key 在 `zh_Hans` 表中有非空值；不存在未被引用的 key（warning） | CI 脚本 + 失败样例 |
| 9 | AC 34 / 存档 AC 25：删除生涯前后 `settings.json` 逐字节相同；`config/` 不被触碰 | 文件哈希 + 目录快照 |
| 10 | `tr()` 缺 key → 返回 key 本身；调试构建 `push_error`；不返回空串 | 单测 |
| 11 | AC 35：1280×720 + 150% 下逐屏截图，无溢出/重叠/截断，金额列对齐，焦点环完整可见 | `production/qa/evidence/` 截图（主菜单、收据、升级、纪录、设置、暂停、错误屏） |
| 12 | `hud_card_columns`：@1280、`s = 1`，τ = 1/1.25/1.5 → 4/3/2 列；卡宽分别 216 / 约 293 / 448 | 布局单测 + 截图 |
| 13 | 150% 下切换前后焦点控件 id 不变；被移除时落到 `FocusSpec.order` 相邻项 | 焦点夹具 F6 |
| 14 | `hud_opacity_percent = 60` → 所有 HUD 文本/背景 ≥ 4.5:1（含 ≥ 24 px 大字号） | 对比度实测表 |
| 15 | `reduce_motion = true` → 顶层过渡 0 s、收据滚动 0 s、指示无淡出，且状态同帧生效 | 帧索引日志 F1 |
| 16 | 全部文案字符在导出包中可渲染（无豆腐块）；`hinting = 1` 已写入 `.import` | 导出后截图 + `.import` 内容检查 |
| 17 | 设置载入先于第一屏 `present()` 与首次音频总线设置 | 启动顺序断言 |

## Related

- `docs/architecture/architecture.md` — Required ADR #8、QQ-08、原则 2/8
- `design/gdd/ui-flow.md` — 规则 20 / 21 / 24 / 28、UI Requirements、AC 33–36、OQ3 / OQ12
- `design/gdd/in-game-hud.md` — `hud_ui_scale`、`hud_card_columns`、字号三档、不透明度下限、AC 11 / 30–32
- `design/gdd/save-system.md` — 规则 20 / 23、OQ8、AC 25
- `design/gdd/data-config.md` — `display_name` / `description` 字段、OQ6、id 语法
- ADR-0001（内容数据格式）— 记录字段与 `Ready` 门；本 ADR 只改 `display_name` 的**值语义**，不改字段
- ADR-0002（持久化格式）— 规范 JSON 规则与五步原子写入原语；本 ADR 复用其编码与替换顺序
- ADR-0003（场景与屏幕架构）— `FocusSpec` / `focus_restore` / `reduce_motion` 关闭过渡 / `canvas_items` 拉伸前提
- ADR-0009（音效播放与暂停语义，待写）— 消费 `AudioSettings` 快照，本 ADR 只固定快照内容与所有权

## Open Questions

| # | 问题 | 负责人 | 目标解决时间 |
|---|---|---|---|
| 1 | `Control.accessibility_live` 在 4.7 的实际类型与取值域？若不再是 `bool`，读屏播报的写法需要一条独立的 spike。 | godot-specialist | 第一个 UI story 之前 |
| 2 | 4.6/4.7 的 CSV 导入器是否接受 `keys,zh_Hans,context` 表头？plural/context 列的确切拼写与是否需要 `plural` 列？ | godot-specialist + ux-designer | 写第一条文案之前 |
| 3 | 内置 Noto Sans SC 的单字重文件体积较大（10 MB 量级）。是否在 Full Vision 前做子集化？子集化的缺字风险由哪条 CI 夹具兜底？ | technical-director + producer | 首次外发打包前 |
| 4 | 设置缺失或损坏时，是否需要在「设置」屏显示一次「已恢复默认设置」的说明？当前决定是静默（最坏后果只是默认音量与 100% 字号），但玩家可能困惑于音量为什么变了。 | ux-designer | 首次可用性测试后 |
| 5 | `data-config.md` 需要新增 `DISPLAY_NAME_NOT_A_KEY` 诊断并把 `display_name` 的语义改为 key——本 ADR 不代为修改 GDD，需在下次修订时同步（同时关闭 OQ6）。 | technical-director + ux-designer | 下一个 GDD 修订 |
| 6 | 生涯的「清理隔离文件」入口（`save-system.md` OQ6）是否也清理 `config/corrupt-*.settings.json`？两处隔离文件的保留策略应统一。 | ux-designer + producer | 首个外发版本前 |
