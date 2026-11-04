# Art Bible: 回本 (Break Even)

> **Status**: Draft — sections 1–4 authored (standard tier); 5–9 not authored
> **Owned By**: art-director
> **Last Updated**: 2026-11-04
> **Art Director Sign-Off (AD-ART-BIBLE)**: SKIPPED 2026-11-04 — lean mode

`/art-bible` writes these nine sections in this order and replaces each
section's `[To be designed]` line as that section is approved. A section still
holding its placeholder is incomplete; a later run fills only those.

## 1. Visual Identity Statement

**视觉身份陈述**

一块食堂价目板：米白纸底上，价格与饱腹值永远最大声，食物退成高饱和剪影，所有读数像一张刚打印的收据。

**原则一：数字永远优先于图标** —— 服务支柱 1「每口都是算术题」

设计测试：当食物图标与价格争夺玩家的第一眼时，这条原则要求选数字——图标缩小、降饱和，或退成底色块。时间格与胃容量同样是数字，享受同等优先。

**原则二：食物只给剪影，不给质感** —— 服务支柱 2「没有白吃的菜」

设计测试：当插画细节让某道菜看起来比它的数值更高级时，这条原则要求选统一剪影——画面不许替玩家做价值判断，贵不贵只能由数字说。

**原则三：读数像收据，反馈不靠动画** —— 服务支柱 4「数字看得见地涨」

设计测试：当反馈想靠动画、闪烁或粒子表达时，这条原则要求选静态数字直接跳到新值——结算读数必须在 0.2 秒内看清，装饰若干扰读数就删掉。

## 2. Mood & Atmosphere

**启动等待 BootWaiting**｜情绪目标：平静、可信、不催人。光照：清晨尚未营业的空食堂，米白纸面占九成，只有一条 1px 冷白进度线，低对比、无阴影、无红。氛围：空桌、灯管初亮、点钞机待机、翻页前的静默。能量：contemplative。

**主菜单 Menu**｜情绪目标：熟悉、被邀请、想先看看价。光照：打烊后开灯的门口，米白加墨黑，中性偏暖，中对比，顶部一层极淡的中性暖光；红色只落在价格与“回本线”读数上，按钮靠交互蓝焦点环识别。氛围：完整价目板、粉笔待写、菜单边角微卷、玻璃反光。能量：measured。

**局内取餐 Run**｜情绪目标：专注、紧张但可控——压力来自时间格，不来自手速。光照：无窗的人工顶光，冷白，高对比；数字区最亮，背景压暗约 15%，让红色读数跳出来。氛围：收银台灯、计算器屏、盘子剪影、时间格、胃容量条。能量：frenetic（认知密度高，但手不快）。

**暂停 Paused**｜情绪目标：喘口气、复盘、不丢进度。光照：整屏降饱和并覆一层冷灰，对比降到中低，背后的 Run 仍隐约可见；红色收到 60% 明度，停止“喊”。氛围：手按住收据、暂停营业牌、灯管半亮、笔尖悬停。能量：contemplative。

**结算收据 Receipt**｜情绪目标：满足、可验证、想再算一遍。光照：全游戏最亮最干净的一屏——纯米白、无背景、高对比，红与橄榄绿达到最高饱和，像刚打印还带热感。氛围：打印纸、撕纸边、油墨未干、小票卷、净利盖章。能量：measured。

**错误／存档受阻 Error / Blocked**｜情绪目标：明确、不指责、知道下一步——不是“你失败了”，是“这张单打不出来”。光照：去饱和冷灰，整体低对比，仅错误块高对比；纸面像缺纸、卡纸。氛围：缺纸、卡纸、被划掉的条目、收银机故障灯、暂停营业。能量：contemplative。

## 3. Shape Language

**菜品剪影：外轮廓即身份。** 所有菜共用一套剪影语法：纯色块、无内绘、无高光，缩到 48×48 仍可辨类别。每类只给一个区分特征：主食=圆顶堆；肉=厚块加骨缺口；蔬菜=细茎分叉的叶伞；甜点=三角切角；饮料=直筒杯加斜吸管；汤=椭圆碗口加两道平行弧。同类别内只靠凸起、缺口、比例区分，不靠颜色与文字——对应原则二与支柱 2「没有白吃的菜」：画面不得替玩家判断哪个更值。菜品卡固定三区（剪影／价格／饱腹），剪影不超过卡面 40%。

**餐厅环境几何。** 只在首页与结算背景出现，全部平面正投影：收银台=一条横贯屏幕的深色带，把画面分成“上方菜单／下方操作”；价目板=网格加更粗的顶部分割线；档口牌=左上角直角小矩形，无圆角、无图标。三者只用直线与直角，呼应“刚打印的收据”这一视觉身份。

**UI 形状语法。** 圆角 ≤4px；层级用 1px 实线而非阴影；标签页为直角标签，选中态是回本红下划线加反白，不填充整块；焦点环为 2px 实线外框加 2px 间距，不做发光。形状永远“平、直、线”，与原则三一致：反馈不靠动效，靠位置直接切换。

**主角形与配角形。** 主角形只有数字块：最大字号、最重字重、四周留白最多，每屏只允许一个“最大数字”（净利或当前关键读数）。价格与饱腹值同级，为第二层；时间格与胃容量条是等宽条形，与数字同级优先。菜剪影、装饰线、档口牌全部退为配角：统一尺寸、统一描边、不抢亮度。对应原则一与支柱 1「每口都是算术题」——第一眼落在数字，第二眼才是菜。

## 4. Color System

**核心色板**

| 色名 | Hex | 角色 | 用／不用 |
|---|---|---|---|
| 米白纸底 | `#F5F1E6` | 主背景、价目板、收据 | 占 80–90%；不作警示填充 |
| 墨黑字 | `#1F1F1C` | 正文、数字、1px 线 | 可满屏文字；不铺大底 |
| 回本红 | `#C63A2F` | 正收益、净利、关键读数 | 只上数字与细线；不作背景、菜色、错误 |
| 橄榄绿 | `#6E7C3C` | 胃容量、安全区、余量 | 只作容量条；不表示“正确” |
| 冷灰 | `#8C918E` | 次级文字、禁用、暂停遮罩 | 不上关键数字 |
| 警示橙 | `#B5651D` | 负收益、超载、时间告急 | 只小面积；不作文字主体或底色 |
| 交互蓝 | `#2F6F8F` | 仅 UI 悬停／焦点／选中 | 2px 以内、不填充 |

**语义逻辑。** 红=收益与关键读数，因为中文语境里红即“红利／红盘”；它只出现在数字上，不作背景、菜色或错误。橙=代价，专门与红错开，让“赚”和“要付”永不同色。绿=容量而非“正确”，用橄榄是因为它接近食堂搪瓷与旧价目板的漆色，避开荧光绿的廉价感。错误不给高饱和色：它是“这张单打不出来”，用冷灰压暗加 45° 斜线纹（缺纸）加明确文案，而不是警报红。中性信息=墨黑正文加冷灰注释，用明度而非色相分层。全案不以暖黄为底：大面积暖黄会压掉米白纸感，并削弱红色数字的对比；橙只作小面积警示。

**分状态色温。** 主菜单中性偏暖、中对比；局内冷白、高对比、背景压暗 15%，让红更跳；结算最亮最中性，红绿满饱和，像刚打印；错误去饱和冷灰、低对比，仅错误块高对比。同一红色跨四态明度不变，只变背景明度，保证“红=赚”始终成立。

**UI 与世界的分歧。** 世界色板只描述食物、价目板、收据；UI 多一个交互蓝，且只出现在焦点环与选中下划线上，避免“把红色当按钮”。禁用态用 40% 冷灰，绝不用淡红——淡红会被读成“亏了一点”。UI 不引入阴影与渐变，层级只靠明度差和 1px 线。

**色盲安全。** 危险对：红／橙、红／绿、绿／橙。备份：正收益数字恒带“+”与上三角，价格等关键读数靠金额列右对齐与货币单位区分，警示恒带“!”与下三角；容量条以“填充／空缺”形状加文字“可装／超载”，并加粗边界；红只占金额区、绿只占容量条，位置本身即编码。所有语义须过灰度截图测试：转灰度后仍靠明度、形状、文字可辨。错误另配斜线纹与文案；提示音由 audio-director 定义，此处只预留位置。

## 5. Character Design Direction

[To be designed] — the player character's visual archetype, how players tell
enemies, NPCs and allies apart at a glance, expression and pose style, and how
much detail survives at game camera distance (LOD philosophy).

## 6. Environment Design Language

[To be designed] — architectural style and its tie to the world's culture,
texture philosophy (painted, PBR or stylised, and why), prop density rules per
area type, and environmental storytelling guidelines.

## 7. UI/HUD Visual Direction

[To be designed] — diegetic versus screen-space HUD, typography (personality,
weight, size hierarchy, minimum readable size), iconography style, UI animation
feel, and any art-direction versus readability conflict and how it was decided.

## 8. Asset Standards

[To be designed] — file formats, naming convention, texture resolution tiers,
LOD levels, export settings, and the engine's hard limits (poly budgets per
asset category, texture memory, material slots). Where an art preference and a
technical limit disagree, record both and the trade-off chosen.

## 9. Reference Direction

[To be designed] — 3–5 references (games, films, art styles or artists). For
each: the specific element to take from it, and what to avoid so the result
does not read as a copy.
