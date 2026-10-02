# edge-to-edge 与系统栏 / IME

检查单与做法改写自 Google 官方 [android/skills](https://github.com/android/skills) 的 `edge-to-edge` skill
（Apache-2.0），并补上本工程里实测到的坑。**不要只看清单点完，先做第 0 步。**

## 0. 先查前置条件能不能满足

该 skill 的硬前提是 **`targetSdk ≥ 35`**（`compileSdk` 必须 ≥ targetSdk）。**在承诺之前先验证工具链**：

```bash
ls /opt/android-sdk/platforms/                      # 需要的 platform 装了吗
/opt/android-sdk/aapt2-arm64/aapt2 version          # 能跑的 aapt2 是哪一代
grep -rn "aapt2FromMavenOverride" /root/.gradle/gradle.properties ~/.gradle/gradle.properties 2>/dev/null
```

实测教训：把 `compileSdk` 提到 35 后构建挂在资源链接：

```
ERROR: AAPT: error: failed to load include path …/platforms/android-35/android.jar
# 直接读那个 jar：error: illegal map type 'string' (22)
```

原因是容器里唯一能跑的 aapt2 是手工编的 arm64 **2.19**（解析不了 API 35 平台的 `resources.arsc`），
而 `build-tools;35.0.0` 自带的 aapt2 是 **x86-64**，在 arm64 容器里被加载器拒绝（`bad machine`）。
SDK 里其它二进制同样是 x86-64，`sdkmanager` 不会给 arm64 的 build-tools。

**这种时候的正确动作**：保持原 `compileSdk/targetSdk`，代码按 edge-to-edge 写好，
把**受阻项 + 原始报错 + 解除条件**写进注释和交付报告，而不是硬改配置。

## 1. 分析（只读，先做）

```bash
grep -rn "<activity\|windowSoftInputMode" app/src/main/AndroidManifest.xml
grep -rln "ComponentActivity\|: Activity(" app/src/main/java --include=*.kt
grep -rn "InputField\|BasicTextField\|TextField(" app/src/main/java --include=*.kt
```

三个问题：**有几个 Activity**（每个都要 edge-to-edge）、**有没有软键盘输入**（IME 检查对象）、
**输入框在布局的什么位置**（在顶部就不会被键盘盖住；在底部必查）。

## 2. 检查单

| 检查项 | 做法 | 常见错 |
|---|---|---|
| 每个 Activity 调 `enableEdgeToEdge()` | `onCreate` 里、`setContent` 之前 | 只在部分 Activity 调；用 `WindowCompat.setDecorFitsSystemWindows(false)` 手工拼 |
| manifest 有 `android:windowSoftInputMode="adjustResize"` | 用 Activity 属性；**不要**用已废弃的 `SOFT_INPUT_ADJUST_RESIZE` | 用 `adjustPan` 后 IME 顶起整页 |
| 让开系统栏 | 二选一：把 `PaddingValues` 传给内容的 `contentPadding`（**首选**，或材料组件的自动 insets） | 给列表的**父容器**加 `Modifier.padding()` → 内容被裁、无法滚到系统栏后面 |
| 输入框不被 IME 挡住 | 内容容器加 `Modifier.fitInside(WindowInsetsRulers.Ime.current)`（首选）或 `imePadding()`（放在 `verticalScroll()` **之前**） | 上游已经用 `contentWindowInsets` 吃过 IME 又加 `imePadding()` → 双重内边距 |
| 首/末列表项让开系统栏 | insets 走 `contentPadding`，不要走父容器 padding | 同上 |
| FAB / 悬浮控件在导航栏之上 | 放进 `Scaffold`，或自己加 `safeDrawingPadding()` | 直接 `align(BottomCenter)` |

**Miuix 特有的两点**：

- Miuix 的 `Scaffold` 默认 `contentWindowInsets = WindowInsets.systemBars ∪ displayCutout`，
  **不含 IME**；而且当 `topBar`/`bottomBar` 存在时，`innerPadding` 取的是**栏的实测高度**，
  系统栏 insets 根本不会流进 `innerPadding`。所以别指望改 `contentWindowInsets` 来拿 IME 内边距。
- Miuix 的 `TopAppBar` / `FloatingNavigationBar` 默认 `defaultWindowInsetsPadding = true`，
  **它们自己吃状态栏/导航栏 inset**。别再给它们的父容器加 padding —— 会双重让开。
  内容要"铺到栏下面再从玻璃后穿过"时，用各页自己的 `contentPadding` 让开，而不是给内容父容器加 padding。

## 3. 系统栏图标必须跟随"应用内"主题（实测坑）

`ComponentActivity.enableEdgeToEdge()` 的默认样式是 `SystemBarStyle.auto(...)`，它**只读系统 dark mode**。
如果应用自己提供"跟随系统 / 浅色 / 深色"的覆盖，用户强制浅色而系统是深色时，
就会出现**浅底 + 白图标**（导航栏同理）。

修法：按**有效主题**下发显式样式（`HafTheme` 这类主题 Composable 里做）：

```kotlin
val darkTheme = when (themeMode) {
    ThemeMode.LIGHT -> false
    ThemeMode.DARK -> true
    else -> isSystemInDarkTheme()          // 跟随系统 / 动态取色
}
val activity = LocalActivity.current as? ComponentActivity
if (activity != null) {
    SideEffect {
        val style = if (darkTheme) SystemBarStyle.dark(Color.TRANSPARENT)
                    else SystemBarStyle.light(Color.TRANSPARENT, Color.TRANSPARENT)
        activity.enableEdgeToEdge(statusBarStyle = style, navigationBarStyle = style)
    }
}
```

用字节码核验过的两条事实（`androidx.activity:activity:1.13.0`）：

- `EdgeToEdgeApi29.setUp` 里是 `window.setNavigationBarContrastEnforced(navStyle.nightMode == 0)`：
  **只有 `auto` 样式（nightMode=0/UNDEFINED）才打开**导航栏对比度强制，显式 `light/dark` 自动关闭。
  这正合"有底栏 / 玻璃要铺到屏幕底"的场景 —— 系统不会再叠一层半透明底。
- `EdgeToEdge$enableEdgeToEdge$1$2`（一个 `View` 子类，`onConfigurationChanged`）**只在 `detectDarkMode`
  返回 null 时才会被安装**，也就是只有 `auto` 样式会跟随系统配置变化重刷。用显式样式时，
  **你的选择会一直生效**，不会被系统 dark mode 变化覆写。

`LocalActivity` 在 `activity-compose` 里有**从 `LocalContext` 沿 `ContextWrapper` 链推导 Activity 的默认值**
（字节码 `LocalActivity$lambda$0`），所以即便没人显式 provide 也能拿到；`as? ComponentActivity` 之后再调扩展函数。

**别**在同时用 ComponentActivity 版 `enableEdgeToEdge` 的情况下再去写 `WindowCompat.getInsetsController(...).isAppearanceLightStatusBars`
—— 两套机制会互相打架（官方 skill 也明确说二选一）。

## 4. 报告方法（前置条件受阻时）

```
已做：<逐项列检查单的结果，符合的写"本来就符合，未改动">
受阻：targetSdk ≥ 35
证据：<原始报错> + <aapt2 版本> + <为什么是环境问题（x86/arm64、aapt2 太老）>
解除：<换环境后改哪几行>
```
