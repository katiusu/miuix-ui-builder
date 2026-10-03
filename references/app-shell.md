# 应用外壳：单 Activity + 多页 pager + 三档底栏 + 宽屏 rail

本文件是一套**可直接照抄的应用外壳骨架**。它取自一个真实跑通的纯 GUI 工程
（MiuixGuiExample，从 MiuixGuiTemplate 提取、剥离 Hook 后的版本），
下面每个 API 与数值都在 **Miuix 0.9.4 / Compose BOM 2026.09.00 / AGP 9.4.1 / compileSdk 37**
上编译通过并出过可安装 APK。

配套阅读：底栏两种形态的完整规格见 `bottom-bars.md`；玻璃/模糊分档见 `glass.md`；
insets 与系统栏图标见 `edge-to-edge.md`；页面内部结构见 `page-patterns.md`。

## 1. 分层与职责

```
MainActivity : ComponentActivity
├─ attachBaseContext → LocaleHelper.wrapContext(...)          # 语言（必须在 super 之前）
├─ onCreate
│   ├─ enableEdgeToEdge()
│   ├─ window.isNavigationBarContrastEnforced = false          # 关掉系统给导航栏加的遮罩
│   ├─ AppSettings.load(this)                                  # 偏好总入口
│   ├─ LauncherIconController.apply(this, hideLauncherIcon)    # 桌面图标与偏好对齐
│   ├─ applyWindowBackground(savedSettings.themeMode)          # 必须在 setContent 之前
│   └─ setContent { AppTheme(themeMode) { MainScreen(...) } }
│
MainScreen（外壳，@Composable private）
├─ 4 页 HorizontalPager（Home / Features / Settings / About）
├─ 底栏三档 或 左侧 NavigationRail
└─ rememberLayerBackdrop → 挂在 pager 容器上，供玻璃底栏取样
│
HomePageView / FeaturesPageView / SettingsPageView / AboutPageContent（页面）
└─ 只接收「值 + 回调 + extraBottomPadding」，不持有任何全局状态
```

**规则**：状态只提升到 Activity；页面按签名接收。外壳向设置页传参的写法就是模板：

```kotlin
SettingsPageView(
    currentMode = themeMode,             onModeChange = { themeMode = it; persistState() },
    isFloatingNavbar = isFloatingNavbar, onFloatingNavbarChange = { ...; persistState() },
    isLiquidGlass = isLiquidGlass,       onLiquidGlassChange = { ...; persistState() },
    isBlurEnabled = isBlurEnabled,       onBlurEnabledChange = { ...; persistState() },
    extraBottomPadding = navBarHeight,   // ← 内容让位由外壳量出来、传下去
)
```

`persistState()` 把 6 个开关一次性写回 `AppSettings`：
**任何开关改动立即落盘**，不要攒到 `onPause`（进程被杀就丢）。

## 2. 多页 pager 外壳

```kotlin
val pagerState = rememberPagerState(pageCount = { 4 })
var selectedIndex by remember { mutableIntStateOf(0) }   // 底栏选中态（不等于 pagerState）
var isNavigating by remember { mutableStateOf(false) }
var navJob by remember { mutableStateOf<Job?>(null) }

val onItemSelected: (Int) -> Unit = select@{ index ->
    if (index == selectedIndex) return@select            // 重复点当前页不重放动画
    navJob?.cancel()                                     // 连点：后一次取消前一次
    selectedIndex = index
    isNavigating = true
    navJob = scope.launch {
        val myJob = coroutineContext.job
        try {
            pagerState.springAnimateToPage(index)        // Miuix 提供的弹性翻页
        } finally {
            if (navJob == myJob) {                       // 只有自己还是"当前任务"才收尾
                isNavigating = false
                if (pagerState.currentPage != index) selectedIndex = pagerState.currentPage
            }
        }
    }
}

// 用户/系统从别的路径换了页（手势、恢复、代码），把选中态追回来
LaunchedEffect(pagerState.currentPage) {
    if (!isNavigating && selectedIndex != pagerState.currentPage) {
        selectedIndex = pagerState.currentPage
        homeRefreshKey++                                 // 顺带给"每次切回主页刷新"一个信号
    }
}

HorizontalPager(
    state = pagerState,
    beyondViewportPageCount = 1,
    contentPadding = pagerPadding,
    userScrollEnabled = false,
    pageNestedScrollConnection = PagerGestureNestedScrollConnection,
    modifier = Modifier
        .fillMaxSize()
        .pagerGestureOverride(
            pagerState = pagerState,
            mode = PagerInterceptionMode.CrossAxisInterceptor,
        ),
) { page -> when (page) { 0 -> HomePageView(...); /* 1/2/3 */ } }
```

为什么是这几个参数：

| 写法 | 作用 |
|---|---|
| `userScrollEnabled = false` + 自己调 `springAnimateToPage` | 底栏点击是唯一翻页入口；横向手势不抢页面内 LazyColumn / 横向组件的滑动 |
| `beyondViewportPageCount = 1` | 预组合相邻页，切页不闪白 |
| `pagerGestureOverride(CrossAxisInterceptor)` | 纵向滚动不被 pager 误判为翻页 |
| `PagerGestureNestedScrollConnection` | 让 pager 与页面内滚动容器协同（缺它会出现"滚不动"或"抢手势"） |
| `isNavigating` + `navJob` 三件套 | 连点两次不会两条动画互相打架；动画被取消时用 `pagerState.currentPage` 回写，避免"底栏亮着 A、内容停在 B" |
| `selectedIndex` 与 `pagerState.currentPage` 分开 | 底栏要高亮"用户刚点的那个"，pager 要报告"真正停在的那个"，中间态必须能分叉 |

`homeRefreshKey` 的模式值得抄：外壳 `homeRefreshKey++` → 页面 `LaunchedEffect(refreshKey) { refresh() }`。
**"每次切回来都要刷新一次"不要写在 `onResume`**（pager 里的页面不会被 resume），要由外壳发信号。

## 3. 底栏三档阶梯（满足铁律 10，且多给一档）

```kotlin
val navBarMode = if (!isFloatingNavbar) 0 else if (!isLiquidGlass) 1 else 2
val isWideScreen = shouldShowSplitPane()
// 横屏时只有"贴底普通"这一档换左侧 rail；悬浮与液态玻璃保持底部（它们的形态就是卖点）
val useNavigationRail = isWideScreen && navBarMode == 0
```

| mode | 形态 | 组件 | 关键写法 |
|---|---|---|---|
| 0 | 贴底普通（默认） | `NavigationBar` + `RowScope.NavigationBarItem` | 模糊开启时给外层 `Box` 挂 `textureBlur(shape = RectangleShape, blurRadius = 25f, colors = BlurDefaults.blurColors(listOf(BlendColorEntry(surface.copy(0.5f)))))`；再 `.background(barColor)`；再 `.clickable(interactionSource = remember { MutableInteractionSource() }, indication = null) {}` 吃掉空白点击 |
| 1 | 悬浮毛玻璃 | `FloatingNavigationBar` + `FloatingNavigationBarItem` | 形状 `RoundedCornerShape(FloatingToolbarDefaults.CornerRadius)`；模糊时挂 `textureBlur(shape = 上面那个, blurRadius = 25f, BlendColorEntry(surfaceContainer.copy(0.4f)), highlight = if (isInDarkTheme()) Highlight.GlassStrokeMiddleDark else Highlight.GlassStrokeMiddleLight)`；关模糊时 `color = surfaceContainer`（**不是** `Color.Transparent`） |
| 2 | 液态玻璃 | `IosLiquidGlassNavigationBar` | `Modifier.padding(horizontal = 12.dp).widthIn(max = 440.dp)` 之后用 `Box(fillMaxWidth, contentAlignment = Center)` 居中——**横屏也不许拉长**，保持与竖屏一致的尺寸 |

- `isInDarkTheme()` 不要用 `isSystemInDarkTheme()`：应用内可以强制深色，判定必须看 **`MiuixTheme.colorScheme.surface` 的相对亮度**（见 `BlurUtils.kt` 的实现）；高光/描边这类玻璃参数按它二选一。
- 底栏图标用 `MiuixIcons.Home / ListView / Settings / Info`（`top.yukonga.miuix.kmp.icon.extended.*`，逐个 import），别用 Material 图标混搭。
- 三档共用**同一个 backdrop**：外壳里 `val backdrop = rememberLayerBackdrop { drawRect(surfaceColor); drawContent() }`，
  pager 容器挂 `.then(Modifier.layerBackdrop(backdrop)).background(surfaceColor)`。
  **关掉模糊时不创建也不挂**（`blurActive = isBlurEnabled`，交给分支判断），省掉每帧录图层的开销。

宽屏 rail：

```kotlin
val railState = rememberNavigationRailState()
val expandRail = shouldExpandNavigationRail()          // 窗口宽 ≥ 1200dp
LaunchedEffect(expandRail) { if (expandRail) railState.expand() else railState.collapse() }

NavigationRail(
    modifier = Modifier
        .onSizeChanged { railWidthPx = it.width }      // 实测宽度，供 pager 让位
        .then(if (blurActive) Modifier.textureBlur(backdrop = backdrop!!, shape = RectangleShape,
            blurRadius = 25f, colors = BlurDefaults.blurColors(listOf(BlendColorEntry(surface.copy(0.5f))))) else Modifier),
    color = if (blurActive) Color.Transparent else surface,
    state = railState,
) { items.forEachIndexed { i, label -> NavigationRailItem(selected = selectedIndex == i, onClick = { onItemSelected(i) }, icon = icons[i], label = label) } }
```

## 4. 内容让位：三条路径，别写混

| 场景 | 让位写法 |
|---|---|
| Scaffold 底栏（mode 0/1/2） | `Scaffold(popupHost = {}, bottomBar = { BottomNavigationBar(...) }) { globalPadding -> pagerContent(globalPadding.calculateBottomPadding(), PaddingValues(0.dp)) }`，外壳把 `navBarHeight` 透传给**每一页**的 `extraBottomPadding`，页面把它加进 LazyColumn 的 `contentPadding.bottom` |
| NavigationRail | pager 的 `contentPadding = PaddingValues(start = railWidth)`（宽度来自 `onSizeChanged`），外层再 `consumeWindowInsets(WindowInsets.systemBars.add(WindowInsets.displayCutout).only(WindowInsetsSides.Start))` |
| 顶栏 | `Scaffold(contentWindowInsets = WindowInsets.systemBars.add(WindowInsets.displayCutout).only(WindowInsetsSides.Horizontal))`：**纵向 inset 不吃**，由页面用 `innerPadding.calculateTopPadding()` 自己放进列表头部 |

坑：**同一个边不要既吃 innerPadding 又加自定义 padding**。两种底栏形态必须共用同一套 `extraBottomPadding` 参数
（见 `bottom-bars.md` 坑 2），否则两条分支的观感会逐渐分叉。

## 5. 主题 / 窗口背景 / 语言 / 桌面图标

- **主题**：`AppTheme(themeMode = ColorSchemeMode)` 包住整棵树。落盘的是 `themeMode.name`，
  读回来必须兜底：`try { ColorSchemeMode.valueOf(saved) } catch (_: Exception) { ColorSchemeMode.System }`
  ——枚举增删或存了坏值不能让应用起不来。
- **窗口背景**：`applyWindowBackground(savedSettings.themeMode)` 要在 `setContent` **之前**调用，
  它覆盖 XML 主题的 `windowBackground`（Dark/MonetDark → `0xFF000000`，Light/MonetLight → `0xFFF7F7F7`，其余跟随 `uiMode`）。
  不做的后果：系统浅色 + 应用内强制深色时，启动瞬间闪一下白底；二级页打开时也会闪。
  **每个 Activity（含二级页）都要调**，不只是主页。
- **语言**：`attachBaseContext(newBase)` 里先读已存语言，再 `super.attachBaseContext(LocaleHelper.wrapContext(newBase, language))`；
  抽象基类 `BaseSubPageActivity` 也照做，否则二级页语言不一致。设置页改完语言调 `activity.recreate()` 立即生效。
- **桌面图标**：靠 `activity-alias`（`.LauncherAlias` → `MainActivity`）+ `setComponentEnabledSetting(alias, ENABLED/DISABLED, DONT_KILL_APP)`，
  启动时用偏好对齐一次（`LauncherIconController.apply(this, hideLauncherIcon)`）。
  **坑**：隐藏图标后如果应用没有任何其它入口，用户就再也打不开它了 —— 要么别暴露这个开关，
  要么隐藏时留通知/快捷方式等回程入口。Manifest 里主 Activity 与 alias 的 intent-filter 要分开写（alias 才是 LAUNCHER 入口）。

### 5.1 主题 + 系统栏图标（Theme.kt 的完整写法）

```kotlin
@Composable
fun AppTheme(themeMode: ColorSchemeMode = ColorSchemeMode.System, content: @Composable () -> Unit) {
    val controller = remember(themeMode) { ThemeController(themeMode) }   // 必须 remember(themeMode)
    MiuixTheme(controller = controller, content = {
        val isDark = isInDarkTheme()
        val view = LocalView.current
        if (!view.isInEditMode) {                       // Preview 里没有 Activity/Window
            SideEffect {                                // 写系统状态：用 SideEffect，不要在组合里直接改
                val window = (view.context as? Activity)?.window ?: return@SideEffect
                WindowCompat.getInsetsController(window, view).apply {
                    isAppearanceLightStatusBars = !isDark
                    isAppearanceLightNavigationBars = !isDark
                }
            }
        }
        content()
    })
}
```

两条结论（源码注释里的原话）：

- **`enableEdgeToEdge()` 的 `SystemBarStyle.auto` 不够用**：它只跟随系统深浅模式；应用内手动强制深色/浅色时，
  状态栏图标不会跟着变，会出现"深色页面配深色图标"。系统栏图标必须由 `isInDarkTheme()` 驱动。
- `isInDarkTheme()` 以 **`MiuixTheme.colorScheme.surface` 的相对亮度**判定
  （`0.2126*r + 0.7152*g + 0.0722*b < 0.5f`），不是看主题名 —— 所以 Monet 动态取色下同样正确。

### 5.2 外壳级设置与功能选项是两条线

- **外壳级**（主题 / 底栏档位 / 模糊 / 语言 / 更新 / 桌面图标）→ `AppSettings` data class，
  模式见 `option-pipeline.md` §9；
- **功能级**（出现在功能页、要搜索/依赖/分组/落盘前缀）→ `OptionSpec` 管道，见 `option-pipeline.md` 全文。

两者都遵循同一条铁律：**默认值只写一处**，读取时复用（外壳级写在 data class 主构造里，功能级写在 `spec` 里）。

## 6. 外壳检查清单

- [ ] 状态全在外壳/Activity，页面只收「值 + 回调 + extraBottomPadding」；
- [ ] 每个开关改动**立即落盘**（`persistState()` 模式）；
- [ ] 底栏档位可切，默认值 = 当前发布形态（铁律 10 + `bottom-bars.md`）；
- [ ] `blurActive` 为假时不创建 backdrop、不挂 `layerBackdrop`；
- [ ] 三档底栏 + rail 的内容让位都只走一条路径（`extraBottomPadding` / `contentPadding`）；
- [ ] `selectedIndex` 与 `pagerState.currentPage` 有同步逻辑，连点不打架；
- [ ] 主题枚举读取有 try/catch 兜底；`applyWindowBackground` 在 `setContent` 之前、且每个 Activity 都调；
- [ ] `attachBaseContext` 的包语言在**所有** Activity 里都做了；
- [ ] 隐藏桌面图标的开关有回程入口，或干脆不做。
