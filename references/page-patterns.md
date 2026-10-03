# 页面结构范式：三种页面形态 + 通用骨架 + 元素选择

来源同上（MiuixGuiExample）。本文件回答"**一个 Miuix 页面长什么样、用什么拼、放什么**"：
页面分几种形态、通用骨架怎么写、列表内容怎么组织、什么场景用哪个组件、宽窄屏怎么分叉。

外壳（底栏 / rail / pager / 让位）见 `app-shell.md`；选项声明管道见 `option-pipeline.md`。

## 1. 三种页面形态

| 形态 | 载体 | 何时用 | 工程里的模板 |
|---|---|---|---|
| **A. 主页面** | 外壳 pager 里的 `@Composable` | 与底栏同级的顶层页面 | `HomePageView` / `FeaturesPageView` / `SettingsPageView` / `AboutPageContent` |
| **B. 二级页** | 独立 `Activity` + `BaseSubPageActivity` | 从设置/列表点进去的子页面 | `FeatureSubPageActivity` |
| **C. 独立全屏页** | 独立 `Activity`，直接用 `SubPageScaffold` | 关于/许可证这类独立入口 | `LicenseActivity` |

形态 A 的页面**不持有全局状态**（值为参数、改动走回调），也不自己处理底栏让位（由外壳传 `extraBottomPadding`）。
形态 B/C 天生是全屏的（没有底栏），所以它们的 `extraBottomPadding` 恒为 0。

## 2. 通用页面骨架（所有页面都长这样）

```kotlin
val scrollBehavior = MiuixScrollBehavior()
val hazeState = rememberBlurState()                       // 设备不支持时返回 null
val blurActive = isBlurEnabled && hazeState != null
val barColor = if (blurActive) Color.Transparent else MiuixTheme.colorScheme.surface

Scaffold(
    topBar = {
        BlurredBar(hazeState, blurActive, scrollBehavior) {
            TopAppBar(title = title, color = barColor, scrollBehavior = scrollBehavior)
        }
    },
    // 只吃水平方向：纵向 inset 交给下面列表的 contentPadding
    contentWindowInsets = WindowInsets.systemBars.add(WindowInsets.displayCutout)
        .only(WindowInsetsSides.Horizontal),
) { innerPadding ->
    Box(modifier = Modifier.blurSource(if (isBlurEnabled) hazeState else null)) {
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .pageScrollModifiers(showTopAppBar = true, topAppBarScrollBehavior = scrollBehavior),
            contentPadding = PaddingValues(
                top = innerPadding.calculateTopPadding(),
                bottom = innerPadding.calculateBottomPadding() + extraBottomPadding,
            ),
        ) { /* item { ... } */ }
    }
}
```

三条硬约束：

1. `blurSource` 挂在**内容容器**上（这里是最外层 `Box`），不要把 `hazeSource` 挂在 LazyColumn 或整个 Scaffold 上
   —— 顶栏要取的是"内容"，把顶栏自己也登记成来源会自己模糊自己；
2. `contentWindowInsets` 只留 `Horizontal`，纵向内边距由列表自己算（`innerPadding.calculateTopPadding()`），
   否则顶栏高度会被算两次；
3. `pageScrollModifiers(...)` 负责 `scrollEndHaptic()` + `overScrollVertical()` + `nestedScroll(topAppBarScrollBehavior)`，
   缺了它顶栏既不会跟着滚动收起、也没有 Miuix 的滚动手感。

## 3. 顶栏渐进模糊（Haze）的参数集中在一个 object 里

```kotlin
object TopBarBlurConfig {
    const val BlurRadius: Float = 15f            // 模糊最强处的半径（dp）
    const val SurfaceAlpha: Float = 0.3f         // 叠在模糊之上的 surface 透明度：越大栏越实
    const val FullStrengthFraction: Float = 0.55f// 顶部这一段保持满强度，其下线性渐隐
    val ScrollFadeDistance: Dp = 0.dp            // 0 = 顶栏常驻完整模糊（推荐）

    val progressive: HazeProgressive = HazeProgressive.Brush(
        Brush.verticalGradient(
            0f to Color.Black,
            FullStrengthFraction to Color.Black,
            1f to Color.Black.copy(alpha = 0f),
        )
    )
}
```

```kotlin
@Composable fun rememberBlurState(): HazeState? =
    if (isRuntimeShaderSupported() && Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) rememberHazeState()
    else null

Box(Modifier.matchParentSize().hazeEffect(state = state) {
    blurRadius = TopBarBlurConfig.BlurRadius.dp
    backgroundColor = surfaceColor              // ← 必须不透明，见下
    noiseFactor = 0f
    tints = listOf(HazeTint(surfaceColor.copy(alpha = TopBarBlurConfig.SurfaceAlpha)))
    progressive = TopBarBlurConfig.progressive
})
```

**`backgroundColor` 必须是不透明色**：Haze 会先按原样绘制一遍来源内容、再叠加模糊副本；
模糊层透明时，卡片这类硬边缘会从模糊层里透出来，看起来像"组件盖在模糊之上"，而不是"糊在下面"。

参数要**集中一处**、各页面共用：调一次全局生效，不用逐页改。

## 4. 列表内容的默认组织方式：分区 + 卡片

```kotlin
item(key = section.titleRes) {
    Column(modifier = Modifier.fillMaxWidth()) {
        SmallTitle(text = 分区标题)                                  // 小标题在卡片外
        Card(modifier = Modifier.padding(horizontal = 12.dp).padding(bottom = 12.dp)) {
            行组件们                                                   // 一行一个 Preference
        }
    }
}
```

- 分区标题用 `SmallTitle`（灰、小号），**不要**把它塞进卡片里当第一行；
- 一张卡里放"同一主题的若干行"；**不要每条记录套一张 Card**（这是设计语言里点名的失败做法，见 `review-findings.md`）；
- LazyColumn 的 `item` 给稳定 `key`（这里用 `section.titleRes`），否则插入/删除分区时状态会错位。

## 5. 元素选择表（先查这张表，再考虑自己组合）

| 需求 | 用什么 |
|---|---|
| 布尔开关 | `SwitchPreference(title, summary, checked, onCheckedChange, enabled)` |
| 多选/单选列表项 | `CheckboxPreference(..., checkboxLocation = CheckboxLocation.End)` |
| 进入下级页 / 触发动作 | `ArrowPreference(title, summary, onClick, enabled)` |
| 少量选项（含"默认档"） | `WindowDropdownPreference(items = List<String>, selectedIndex, onSelectedIndexChange)` |
| 数值滑杆 | `SliderPreference(value, onValueChange, valueRange, steps, title, valueText, enabled)` |
| 纯展示行（标题 + 值） | `BasicComponent(title, summary, startAction, endActions)` |
| 单行文本输入 | `InputField(query, onQueryChange, onSearch, expanded, onExpandedChange, label)` |
| 搜索栏 | `SearchBar(inputField = { ... }, outsideEndAction = { ... }, expanded, onExpandedChange)` |
| 主按钮 | `Button(colors = buttonColorsPrimary)` |
| 次级/危险按钮 | `TextButton(colors = textButtonColors(textColor = 0xFFDC3545), minWidth = 0.dp)` |
| 弹窗 | `WindowDialog(title, summary) { ... }` |
| 图标按钮 | `IconButton { Icon(imageVector, contentDescription, tint = MiuixTheme.colorScheme.onSurface) }` |
| 文本 | `top.yukonga.miuix.kmp.basic.Text as MiuixText`（别名导入，避免与 Material `Text` 撞名） |

## 6. 主页仪表盘：状态卡 + 计数卡（一套数据、两套布局）

```kotlin
val isWideScreen = shouldShowSplitPane()          // ≥840dp，或 ≥600dp 且高宽比 < 1.2
val cardsModifier = Modifier
    .fillMaxWidth()
    .padding(horizontal = 12.dp).padding(top = 12.dp)
    .height(IntrinsicSize.Min)                    // ← 让同一行的卡片等高

if (isWideScreen) {
    Row(cardsModifier, horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalAlignment = Alignment.CenterVertically) {
        StatusCard(..., compact = true, modifier = Modifier.weight(1f).fillMaxHeight())
        CountCard(作用域数, modifier = Modifier.weight(1f).fillMaxHeight())
        CountCard(功能数, modifier = Modifier.weight(1f).fillMaxHeight())
    }
} else {
    Row(cardsModifier, horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalAlignment = Alignment.CenterVertically) {
        StatusCard(..., modifier = Modifier.weight(1f).fillMaxHeight())
        Column(Modifier.weight(1f).fillMaxHeight()) {
            CountCard(作用域数, Modifier.fillMaxWidth().weight(1f))
            Spacer(Modifier.height(12.dp))
            CountCard(功能数, Modifier.fillMaxWidth().weight(1f))
        }
    }
}
```

- **等高靠 `height(IntrinsicSize.Min)` + `fillMaxHeight`**，不要写死高度；
- `StatusCard` 的装饰大图标：`Box(Modifier.fillMaxSize().offset(xOffset, yOffset), contentAlignment = Alignment.BottomEnd)`
  里放 `Icon(size = iconSize)`，超出卡片的部分由 Card 的圆角裁掉；
  compact 档 `100.dp / offset(22.dp, 26.dp)`，非 compact 档 `170.dp / offset(38.dp, 45.dp)`；
- **语义色三档**（正常 / 警告 / 异常）：动态取色主题用 `secondaryContainer` / `errorContainer`；
  非动态色时用硬编码对（异常 `0xFFFDE8E8`+`0xFF3D1C1C`、警告 `0xFFFDF6E3`+`0xFF3D3520`、正常 `0xFFDFFAE4`+`0xFF1A3825`），
  浅/深由 `isInDarkTheme()` 选；图标色 `0xFFDC3545` / `0xFFE0A800` / `0xFF36D167`（`copy(alpha = 0.8f)`）；
- 字号：卡标题 `20.sp SemiBold`、说明 `13.sp Medium`、计数标题 `15.sp Medium`（`onSurfaceVariantSummary`）、计数 `26.sp SemiBold`；
- 卡片要给 `CardDefaults.defaultColors(color = statusColor)` + `pressFeedbackType = PressFeedbackType.Tilt`（可点卡片的手感来源）；
- **缺图标不要引大库**：三个状态图标是用 `materialIcon(name) + materialPath { ... }` 手写路径自建的（`ui/icons/StatusIcons.kt`），
  只依赖 `material-icons-core`；为几个图标引入 `material-icons-extended` 会让包体涨好几 MB。

### 6.5 变体：正方形状态卡 + 信息卡 + 独立按钮卡（用户说"分开"时）

用户嫌"状态、服务名、按钮挤在一张卡里"，要求"分成方块"时，**不要**继续用 §6 的等高行，改成
**正方形块 + 独立操作卡**（实测于 HyperOS-Autofill-Fix 2.3.0 概览页）：

```kotlin
if (isWideScreen) {                                  // 宽屏：三卡等分
    Row(cardsModifier, horizontalArrangement = Arrangement.spacedBy(12.dp)) {
        StatusCard(modifier = Modifier.weight(1f).fillMaxHeight())
        InfoCard(modifier = Modifier.weight(1f).fillMaxHeight())
        ActionCard(..., stacked = true, modifier = Modifier.weight(1f).fillMaxHeight())
    }
} else {                                             // 窄屏：两个正方形 + 一行按钮
    Row(cardsModifier, horizontalArrangement = Arrangement.spacedBy(12.dp)) {
        StatusCard(modifier = Modifier.weight(1f).aspectRatio(1f))     // 正方形
        InfoCard(modifier = Modifier.weight(1f).aspectRatio(1f))       // 正方形
    }
    ActionCard(
        ...,
        stacked = false,
        modifier = Modifier.fillMaxWidth().padding(horizontal = 12.dp).padding(top = 12.dp),
    )
}
```

- **正方形靠 `Modifier.weight(1f).aspectRatio(1f)`**；不要再叠 `height(IntrinsicSize.Min)`——
  那是"同一行等高"的机制，和正方形冲突（写了也白写）；
- **一张卡只回答一个问题**：状态卡（色块 + 一个大图标 + 一句文案）只答"是否正常"；
  原始值（服务名 / 包名 / 当前档位）放**另一张**信息卡，读不到时用 `onSurfaceVariantSummary` 说明；
  操作是"下一步动作"，单独成卡（`Row` 里两个 `weight(1f)` 的 `Button` / `TextButton`），
  窄屏上下排、宽屏并排都成立；
- 可点进详情的卡用 `pressFeedbackType = PressFeedbackType.Tilt`；纯展示卡**不要**给 `onClick`
  （给了就会有按压反馈，用户以为能点进去）；
- 语义色三档与自建图标同 §6；状态文案要能单独读懂（"已被改回小米自家服务"优于"异常"）。

## 7. 设置页：分区顺序与形态

固定顺序（每段一张 Card）：**功能/模块 → 界面 → 语言 → 更新 → 数据**。

- 下拉：`WindowDropdownPreference`，`selectedIndex = entryValues.indexOf(currentValue)` 记得**兜底 0**（值不在列表里时不能崩）；
- 条件项（例如"液态玻璃"开关只在开启悬浮底栏后才有意义）：
  ```kotlin
  AnimatedVisibility(visible = isFloatingNavbar, enter = expandVertically(MiuixExpandSpec),
                     exit = shrinkVertically(MiuixExpandSpec)) { SwitchPreference(...) }
  ```
  `MiuixExpandSpec` 是工程统一的展开弹性规格（`folmeSpring(damping = 1.0f, response = 0.4f)`），别每条自己调参；
- 危险动作（清缓存 / 重启系统 UI 这类）单独成卡，按钮用红色 `TextButton`，并且要有二次确认弹窗；
- 语言：下拉三项（跟随系统 / 中文 / English），改完 `activity.recreate()` 立即生效；
- 更新：一个自动检查开关 + 一个手动检查 `ArrowPreference`（检查中禁用，结果用 Toast / 对话框）；
- 数据：导出 / 导入 JSON，用 `rememberLauncherForActivityResult(CreateDocument("application/json"))` 与 `GetContent`，
  成功后 Toast + `recreate()`（主题/语言可能已变）；文件名固定（如 `<AppName>_settings.json`），导入时要处理"文件读不出/JSON 非法"两种失败；
- 卡片底部可以放一行版权文本：`MiuixTheme.textStyles.footnote2` + `onSurfaceVariantSummary` + 居中。

## 8. 关于页：折叠头部 + 玻璃卡片

- **折叠进度**：由 `rememberLazyListState()` 的 `firstVisibleItemIndex` / `firstVisibleItemScrollOffset` 推算
  （头部放一个 `key = "logoSpacer"` 的占位 item），归一化成 `scrollProgress`，同时驱动：
  标题渐显、`BgEffectBackground(dynamicBackground = true, isFullSize = true, alpha = { 1f - scrollProgress })`；
- **logo**：`100.dp` + `squircleClip(28.dp)`（不是 `CircleShape`），位图来自 `packageManager.getApplicationIcon(pkg).toBitmap().asImageBitmap()`；
- **应用名**：`textureBlur(blurRadius = 150f, logoBlend = 三色(ColorDodge / LinearLight / Lab), contentBlendMode = DstIn)`；
- **信息卡**：`textureBlur(blurRadius = 60f, cardBlend = ColorBlendToken.Overlay_Thin_Light(深) / Pured_Regular_Light(浅))`，
  两组 token 定义在 `ui/util/BlurUtils.kt`；
- **许可证**：独立 Activity（形态 C），数据是 `LicenseSection(titleRes, libraries: List<LibraryInfo>)`，
  `LibraryInfo(name, version, license, website)` —— 库清单写死在页面里，升级依赖时要同步改（漏改等于许可证声明不实）。

## 9. 二级页模板：BaseSubPageActivity

```kotlin
abstract class BaseSubPageActivity : ComponentActivity() {
    @get:StringRes protected abstract val titleRes: Int
    protected open val topBarActions: (@Composable () -> Unit)? = null
    @Composable protected abstract fun SubPageContent(isBlurEnabled: Boolean, contentPadding: PaddingValues)

    override fun attachBaseContext(newBase: Context) { /* LocaleHelper.wrapContext */ }
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        val settings = AppSettings.load(this)
        val themeMode = try { ColorSchemeMode.valueOf(settings.themeMode) } catch (_: Exception) { ColorSchemeMode.System }
        applyWindowBackground(settings.themeMode)          // 与主页一致，避免打开瞬间闪底
        setContent {
            AppTheme(themeMode = themeMode) {
                SubPageScaffold(
                    title = stringResource(titleRes),
                    isBlurEnabled = settings.isBlurEnabled,
                    onBack = { finish() },
                    topBarActions = topBarActions,
                ) { padding -> SubPageContent(settings.isBlurEnabled, padding) }
            }
        }
    }
}
```

子类只写三样：`titleRes`、可选 `topBarActions`、`SubPageContent`。
**二级页不要自己重写 Scaffold 顶栏**，否则返回键、模糊、主题三处都要各写一遍，且容易与主页不一致。

`SubPageScaffold` 自己会 `CompositionLocalProvider(LocalSubPageScrollBehavior provides scrollBehavior)`，
所以内容页的 LazyColumn 用 `LocalSubPageScrollBehavior.current` 绑定顶栏收起行为，不用层层传参。

## 9.5 关于页补充：折叠头部的精确机制

```kotlin
val scrollProgress by remember { derivedStateOf {
    when {
        lazyListState.firstVisibleItemIndex > 0 -> 1f
        else -> {
            val spacer = lazyListState.layoutInfo.visibleItemsInfo.firstOrNull { it.key == "logoSpacer" }
            if (spacer != null && spacer.size > 0)
                (lazyListState.firstVisibleItemScrollOffset.toFloat() / spacer.size).coerceIn(0f, 1f)
            else 0f
        }
    }
} }
val collapsed by remember { derivedStateOf { scrollProgress == 1f } }
```

- **`key = "logoSpacer"` 的占位 item 就是进度的分母**（`firstVisibleItemScrollOffset / spacer.size`）。
  改这个 key 名或删掉该 item，进度会退化成 0/1 跳变 —— 这是整页动效的隐式契约，改之前先看这段；
- spacer 高度 = logo 实测高度 + `52.dp` + `logoPadding.top - scrollPadding.top` + `126.dp`；
  `logoHeightDp` 由 `onSizeChanged` 实测（初始值 `300.dp` 顶首帧）。**任一 padding 变了，这个表达式也要同步改**；
- 多阶段退场：图标 `(p - 0.35f) / 0.15f`、应用名 `(p - 0.20f) / 0.15f`、版本号/来源文本 `(p - 0.05f) / 0.15f`，
  统一 `coerceIn(0f, 1f)`，缩放幅度统一 `1 - p * 0.05f`；
- **进度以 `scrollProgressProvider: () -> Float` 的 lambda 下传**，读数只发生在 `graphicsLayer {}` / `alpha` lambda 里
  —— 传 `Float` 参数会让整棵子树每帧重组；
- `blurActive` 是"**折叠完成才开**"（`isBlurEnabled && hazeState != null && scrollProgress == 1f`）：
  头部还没收回时开模糊，等于把大标题也糊掉；
- `SmallTopAppBar(..., defaultWindowInsetsPadding = false)`：它自带的内边距会与外层 `BlurredBar` 打架，必须关掉、由外层统一处理；
- **背景与玻璃卡片同生共死**：`textureBlur` 只在 `contentBackdrop != null`（API 33+ 才有）时挂，else 分支是纯色卡片，
  所以 `Card` 的 colors 也要跟着切：
  `CardDefaults.defaultColors(if (backdrop != null) Color.Transparent else colorScheme.surfaceContainer, Color.Transparent)`；
- RTL：横向 padding 一律 `calculateLeftPadding(LayoutDirection.Ltr)`（而不是读 start/end），
  免得与 `Modifier.padding` 的 start/end 语义打架；
- logo 位图直接取 `packageManager.getApplicationIcon(applicationInfo)`（**不要用资源 id**，保证与桌面图标一致），
  并用 `toBitmap(sizePx, sizePx)` 按 `100.dp.roundToPx()` 预缩放。

## 9.6 设置页补充：下拉与列表组织的实测细节

`WindowDropdownPreference` 要**显式接管展开态**，否则 0.9.4 内部会自己管，导致展开/收起与选中状态不同步：

```kotlin
WindowDropdownPreference(
    items = values,                                                  // 文案列表：spec.entryResIds.map { stringResource(it) }
    selectedIndex = values.indexOf(saved).takeIf { it >= 0 } ?: 0,    // ← 索引必须兜底 0
    title = stringResource(spec.titleRes),
    summary = spec.summaryRes.takeIf { it != 0 }?.let { stringResource(it) },
    enabled = enabled,
    onExpandedChange = { },                                          // ← 空 lambda：展开态由调用方持有
    onSelectedIndexChange = { index -> ConfigState.set(spec.key, spec.entryValues.getOrElse(index) { spec.defaultString }) },
)
```

- **文案列表与值列表是两张平行表**（`items` 给用户看、`entryValues` 存盘），任一表增删必须同时改另一张；
- 索引换算一律 `indexOf(x).takeIf { it >= 0 } ?: 0`：存的旧值不在列表里时既不能崩，也不能默默变成 `-1`；
- 选项少于 5 个时，用一列 `CheckboxPreference(checkboxLocation = CheckboxLocation.End)` 拼"单选列表"比下拉少一次点击。

**列表组织有两种写法，按内容规模选**：

- 内容少而固定（设置页那种五段）：LazyColumn 里只放**一个 item**，内容是
  `Column { SmallTitle + Card + SmallTitle + Card + ... }`（每张 Card 自带 `padding(horizontal = 12.dp).padding(bottom = 12.dp)`）；
- 内容多/动态/可搜索（功能页那种）：**一段一个 item**，并 `item(key = section.titleRes)` 给稳定 key（见 `option-pipeline.md` §6）。

**数据管理段的导入导出**：

```kotlin
val exportLauncher = rememberLauncherForActivityResult(ActivityResultContracts.CreateDocument("application/json")) { uri ->
    uri ?: return@rememberLauncherForActivityResult
    runCatching { context.contentResolver.openOutputStream(uri)?.use { it.write(ConfigBackup.exportJson().toByteArray()) } }
        .onSuccess { toast(导出成功) }.onFailure { toast(导出失败) }
}
val importLauncher = rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri ->
    uri ?: return@rememberLauncherForActivityResult
    runCatching { context.contentResolver.openInputStream(uri)?.bufferedReader()?.readText() }
        .mapCatching { ConfigBackup.importJson(it) }
        .onSuccess { toast(导入成功); activity?.recreate() }    // ← 主题/语言可能已变，必须重建
        .onFailure { toast(导入失败) }                           // ← 失败时不动任何状态
}
```

两类失败都要覆盖：**文件读不出来**（URI 失效 / 权限）与 **JSON 非法**。

## 10. 页面自查清单

- [ ] 顶栏走 `BlurredBar`，`barColor` 由 `blurActive` 决定（透明 / surface 二选一）；
- [ ] `blurSource` 挂在内容容器上；`contentWindowInsets` 只留 Horizontal；
- [ ] 每个 LazyColumn 都挂了 `pageScrollModifiers(...)`；
- [ ] 列表内容按「`SmallTitle` + 一张 `Card(h12/b12)`」组织，一条记录不套一张卡；
- [ ] 列表项有稳定 `key`；
- [ ] 主页面在**宽屏与窄屏**都看过（"三卡横排 / 一大两小"两套布局都要成立）；
- [ ] 状态色有语义、深浅主题都取过值，没用红表达"正常"；
- [ ] 缺图标时用 `materialIcon + materialPath` 自建，没为几个图标引入 `material-icons-extended`；
- [ ] 二级页继承 `BaseSubPageActivity`，没有自己重写顶栏；
- [ ] 许可证页的库清单与本工程实际依赖一致。
