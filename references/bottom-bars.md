# 底栏规格：贴底普通 + 悬浮毛玻璃 + 液态玻璃，可切换

SKILL.md 的铁律 10：**自带底部导航的应用外壳，必须同时提供"贴底普通"与"悬浮毛玻璃"两档 + 一个开关**
（用户明确不要才例外）。示例骨架里还有第三档 **液态玻璃**（自绘高光/折射/内阴影），做法见 §2.5。
本文件是完整实现、核验过的 API 与实测踩到的坑。

## 1. 先核验 API（别凭记忆）

```bash
# Miuix 0.9.3（KMP：Android 构件是 miuix-ui-android）
curl -sO https://repo1.maven.org/maven2/top/yukonga/miuix/kmp/miuix-ui-android/0.9.3/miuix-ui-android-0.9.3-sources.jar
# 读 basic/NavigationBar.kt 里的三个签名
```

实测签名（0.9.3，`basic/NavigationBar.kt`）：

```kotlin
@Composable
fun NavigationBar(                                   // 贴底普通底栏
    modifier: Modifier = Modifier,
    color: Color = MiuixTheme.colorScheme.surface,   // 不透明主题色，不要自己填
    showDivider: Boolean = true,                     // 顶部分隔线
    defaultWindowInsetsPadding: Boolean = true,      // ← 它自己吃导航栏 inset
    mode: NavigationBarDisplayMode = NavigationBarDisplayMode.IconAndText,
    content: @Composable RowScope.() -> Unit,
)

@Composable
fun RowScope.NavigationBarItem(                      // ← RowScope 扩展！
    selected: Boolean, onClick: () -> Unit, icon: ImageVector, label: String,
    modifier: Modifier = Modifier, enabled: Boolean = true, badge: (@Composable () -> Unit)? = null,
)

@Composable
fun FloatingNavigationBar(...)                       // 悬浮胶囊（毛玻璃那档）
```

`NavigationBarItem` **是 `RowScope` 的扩展**，因此只能写在 `NavigationBar` 的 content 里；
写在外面要么编译不过、要么被 IDE 提示"找不到"。这也是为什么"普通底栏"不能只是换个 `Modifier` 了事。

0.9.4 实测这三个签名未变（`FloatingNavigationBarItem` 为 `(selected, onClick, icon, label, modifier, enabled, colors, content)`），
三档的实现差异全部在**外壳的档位派生 + 玻璃层怎么加**，见 §2.5。

## 2. 实现配方（四步）

### 2.1 偏好落盘

```kotlin
// data/AppPrefs.kt
/** 底栏形态：true = 悬浮毛玻璃胶囊，false = 贴底普通底栏。 */
var floatingNavBar: Boolean
    get() = sp.getBoolean(KEY_FLOATING_NAV_BAR, true)     // 默认保持"当前发布的形态"
    set(value) = sp.edit { putBoolean(KEY_FLOATING_NAV_BAR, value) }
// companion object 里补：private const val KEY_FLOATING_NAV_BAR = "floating_nav_bar"
```

### 2.2 经状态持有者暴露（不要在页面里直接读 SharedPreferences）

```kotlin
// ui/UiPrefsState.kt —— 沿用工程里既有的模式
private var floatingNavBarState by mutableStateOf(prefs.floatingNavBar)

var floatingNavBar: Boolean
    get() = floatingNavBarState
    set(value) { floatingNavBarState = value; prefs.floatingNavBar = value }
```

### 2.3 设置页给开关（放在"外观"分组里，跟着主题）

```kotlin
SwitchPreference(
    title = "悬浮底栏",
    summary = "开：悬浮的毛玻璃胶囊，内容从它后面穿过；关：贴底普通底栏（图标带文字标签）",
    checked = state.floatingNavBar,
    onCheckedChange = { state.floatingNavBar = it },     // 即时生效，不需要"重启生效"
)
```

### 2.4 外壳里分支（顺带把 backdrop 门控掉）

```kotlin
val useGlassBar = isRenderEffectSupported() && state.floatingNavBar
val backdrop: LayerBackdrop? = if (useGlassBar) rememberLayerBackdrop(onDraw = backdropDraw) else null

Scaffold(
    bottomBar = {
        if (state.floatingNavBar) {
            GlassNavigationBar(backdrop = backdrop, currentPage = currentPage, onSelect = { goToPage(it) })
        } else {
            PlainNavigationBar(currentPage = currentPage, onSelect = { goToPage(it) })
        }
    },
) { innerPadding ->
    val contentBottomPadding = innerPadding.calculateBottomPadding()   // 两种形态共用
    ...
}

@Composable
private fun PlainNavigationBar(currentPage: Int, onSelect: (Int) -> Unit) {
    NavigationBar {                                    // inset 由它自己吃，别再给父容器加 padding
        DESTINATIONS.forEachIndexed { index, destination ->
            NavigationBarItem(                         // RowScope 扩展，只能写在这里
                selected = currentPage == index,
                onClick = { onSelect(index) },
                icon = destination.icon,
                label = destination.label,
            )
        }
    }
}
```

### 2.5 三档底栏（Miuix 0.9.4 + miuix-blur，实测于 HyperOS-Autofill-Fix 2.3.0）

在"两档"之上加第三档时，**档位用两个布尔派生，不要写成三选一的枚举硬编码在组件里**：

```kotlin
val navBarMode = if (!isFloatingNavbar) 0 else if (!isLiquidGlass) 1 else 2
// 0 = 贴底普通 NavigationBar
// 1 = 悬浮毛玻璃 FloatingNavigationBar + textureBlur
// 2 = iOS 液态玻璃（自绘高光/折射/内阴影）
val useNavigationRail = shouldShowSplitPane() && navBarMode == 0   // 宽屏只在贴底档换成 rail
val blurActive = isBlurEnabled && isRuntimeShaderSupported() && Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU
```

- mode 1 用 **miuix-blur 的 `textureBlur`**（0.9.4 起不再需要 kyant backdrop）：

```kotlin
FloatingNavigationBar(
    modifier = Modifier.textureBlur(
        backdrop,
        shape = RoundedCornerShape(FloatingToolbarDefaults.CornerRadius),
        blurRadius = 25f,
        colors = BlurDefaults.blurColors(listOf(BlendColorEntry(surfaceContainer.copy(alpha = 0.4f)))),
        highlight = if (isInDarkTheme()) Highlight.GlassStrokeMiddleDark else Highlight.GlassStrokeMiddleLight,
    ),
    color = if (blurActive) Color.Transparent else surfaceContainer,
) {
    items.forEachIndexed { index, item ->
        FloatingNavigationBarItem(
            selected = selectedIndex == index,
            onClick = { onItemSelected(index) },
            icon = icons[index],
            label = item,
        )
    }
}
```

- mode 2 的 `IosLiquidGlassNavigationBar(items, selectedIndex, onItemClick, backdrop, isBlurActive, modifier, badge)`
  是 **internal**（同模块可直接调）；它自带高光/折射/内阴影，**不要再套一层 `textureBlur`**（两套玻璃叠起来会糊成一团）；
- **开关要互相包含，别让用户拼出无意义组合**：`isLiquidGlass` 只在 `isFloatingNavbar = true` 时生效
  （关掉悬浮后液态档自然失效），所以设置页把"悬浮底栏"放上面、"液态玻璃"紧跟其后，
  并在摘要里写明"仅在悬浮底栏开启时生效"——这样两个开关就覆盖了三种真实形态，不需要第三个分支；
- **三档共用同一套内容让位**（各页的 `contentBottomPadding`）；mode 0 不创建 backdrop、不挂 `layerBackdrop`；
- 降级链条：`blurActive == false` → mode 2 退回 mode 1 的不透明配色（或按产品决定直接落回 mode 0 的观感），
  **永远不要只把颜色刷透明**。

## 3. 坑（都实测过）

1. **inset 归属**：`NavigationBar` 默认 `defaultWindowInsetsPadding = true`，和 `FloatingNavigationBar`
   一样**自己吃导航栏 inset**。所以别给底栏或它的父容器再加 `navigationBarsPadding()`——
   会双重让开、底栏浮在半空。
2. **内容让位只写一套**：两种形态都用各页的 `contentBottomPadding`。普通底栏是不透明的，
   内容从它下面滚过看不见，观感与"只让开"等价 —— 为它再写一套 `Modifier.padding(innerPadding)`
   只会让两条分支行为分叉、以后难改。
3. **关掉悬浮时要门控 backdrop**：不透明底栏用不到"每帧录一次图层"的开销，`useGlassBar` 里带上形态判断。
   切换时 `rememberLayerBackdrop` 随分支创建/释放，不会泄漏。
4. **降级别写成刷透明**：悬浮形态在 API < 31（无 `RenderEffect`）时 `backdrop == null`，
   底栏要退回 `surfaceContainer` 不透明配色，而不是 `Color.Transparent`（否则是一块看不见的玻璃）。
5. **"取回旧形态"要拿原始实现，不要凭记忆重写**：
   ```bash
   git show <旧 tag>:app/src/main/java/.../HafApp.kt | sed -n '/bottomBar = {/,/},/p'
   ```
   旧 release 的底栏写法就是需求方所指的"那个版本"，照抄比"我觉得应该是这样"可靠。
6. **默认值别偷偷换**：新增开关时，默认值应当等于**当前已发布形态**，否则用户升级后会莫名其妙变样。
   要换默认就得单独说明并让用户确认。
7. **验证要查所有 dex**：证明"两种底栏都进了包"时，debug 包是 multi-dex，只查 `classes.dex` 会得到假阴性
   （见 `android-release.md`）。
8. **无障碍**：两个组件的 `label` 都是可见文字标签，不要为了"简洁"只留图标；
   只靠图标/颜色的状态表达在 review 清单里是要被打回的（见 `review-findings.md`）。
9. **档位派生写在"外壳"一处**：`navBarMode` / `useNavigationRail` 都要在外壳算好再往下传，
   不要一半在底栏组件里、一半在 `Scaffold` 里（宽屏 rail 与窄屏底栏的判定会各写一份、必然分叉）。
10. **模糊开启时 `color` 必须透明**：`FloatingNavigationBar(color = surfaceContainer)` 会把
    自己那层毛玻璃整个盖住，看起来像"模糊没生效"——`color = if (blurActive) Color.Transparent else surfaceContainer`。
11. **`textureBlur` 的 `highlight` 跟主题走**：`Highlight.GlassStrokeMiddleDark` / `GlassStrokeMiddleLight`
    要用 `isInDarkTheme()` 选，浅色主题套深色描边会显脏。
12. **液态玻璃档不要"建在"毛玻璃档上**：mode 2 是自绘玻璃，若再叠 mode 1 的 `textureBlur + Highlight`，
    高光与描边会重叠成双影；两档是并列关系，不是叠加关系。

## 4. 验收清单

- [ ] 设置里有开关，能即时切换、重启后保持（偏好落盘）；
- [ ] 悬浮形态：内容从玻璃后穿过、滚动末端不压字、API < 31 退回不透明；
- [ ] 普通形态：贴底、不透明、有分隔线、图标 + 文字；
- [ ] 关掉悬浮时不再创建 backdrop / 不挂 `layerBackdrop`；
- [ ] 两种形态都不额外加 inset padding（不双重让开）；
- [ ] 有第三档（液态玻璃）时：关掉"悬浮底栏"后它不再影响观感，且模糊不可用时整体退成不透明而不是透明；
- [ ] mode 0 时宽屏走 rail、窄屏走底栏（rail 与 mode 2 不共存）；
- [ ] 产物里两种底栏的代码都在（遍历所有 `classes*.dex`）；
- [ ] 交付报告里写明默认值与开关位置（用户改起来只需一句话）。
