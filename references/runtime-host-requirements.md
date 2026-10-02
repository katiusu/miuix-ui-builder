# 宿主要求：编译通过 ≠ 组合期不炸

**这一页的结论只有一句：组件会调用宿主（Activity / Window）提供的能力；宿主版本不够时，
`compileDebugKotlin` 一样全绿，但用户一进那个页面就崩。**

## 真实事故（Miuix 0.9.3 + activity 1.9.3）

现场：模块 App 的三个页面里，「概览」正常，「日志」「设置」**打不开**。

设备 logcat（`adb-shell "logcat -d"`）里唯一的 FATAL：

```
java.lang.IllegalStateException: No NavigationEventDispatcher was provided via LocalNavigationEventDispatcherOwner
    at androidx.navigationevent.compose.NavigationEventHandlerKt.NavigationEventHandler(NavigationEventHandler.kt:82)
    at androidx.navigationevent.compose.NavigationEventHandlerKt.NavigationBackHandler(NavigationEventHandler.kt:148)
    at top.yukonga.miuix.kmp.layout.DialogContentLayoutKt.DialogContentLayout(DialogContentLayout.kt:184)
    at top.yukonga.miuix.kmp.overlay.OverlayDialogKt.OverlayDialog(OverlayDialog.kt:75)
    at top.yukonga.miuix.kmp.utils.MiuixPopupUtils$Companion.DialogEntry(MiuixPopupUtils.kt:408)
```

顺着栈在 **0.9.3 源码**里把受影响组件找全：

| 组件 | 证据（0.9.3 源码） | 触发时机 |
|---|---|---|
| `SearchBar`（`InputField` 同文件） | `basic/SearchBar.kt:55-57` import、`:90` `rememberNavigationEventState`、`:121` `NavigationBackHandler` | **组合期**就调用 → 含它的页面一打开就崩 |
| `OverlayDialog` / `WindowDialog` | 两者共用 `layout/DialogContentLayout.kt:184` | 弹窗**显示**时 |
| `OverlayDropdownPreference` → `OverlayDropdownPopup` | 走同一套 popup 宿主 | 下拉**展开**时 |

根因：`NavigationBackHandler` 需要 `LocalNavigationEventDispatcherOwner`，
而 `androidx.activity` 1.9.x / 1.10.x / 1.11.x 的 `ComponentActivity` **没有**实现
`androidx.navigationevent.NavigationEventDispatcherOwner`。**1.13.0 才开始实现。**

## 怎么在 10 分钟内定位这类问题

```bash
# 1) 抓崩溃栈（有真机就一定有；别只看自己代码的异常）
adb-shell "logcat -d" | grep -nE "FATAL|Process: <你的包名>"        # 取行号
adb-shell "logcat -d" | sed -n '9,40p'                              # 打印完整栈

# 2) 从栈里挑出「库的哪个文件哪一行」调用了一个需要宿主能力的东西
grep -rn "NavigationBackHandler\|NavigationEventHandler" <库源码目录>

# 3) 确认宿主的哪个版本才提供该能力（javap 最快）
for v in 1.10.1 1.11.0 1.13.0; do
  curl -s -o /tmp/act-$v.aar "https://repo1.maven.org/maven2/androidx/activity/activity/$v/activity-$v.aar"
  T=$(mktemp -d); (cd $T && unzip -oq /tmp/act-$v.aar classes.jar && unzip -oq classes.jar \
    "androidx/activity/ComponentActivity.class" && javap androidx/activity/ComponentActivity.class \
    | head -2 | grep -o NavigationEventDispatcherOwner) && echo "  $v 已实现"
done
# 输出：只有 1.13.0 打印"已实现"
```

修法：`gradle/libs.versions.toml` 里 `activityCompose = "1.13.0"`（一行，不改任何 UI 代码）。

## 通用规律（下次遇到别的库同样适用）

1. **库会消费宿主能力**：返回拦截（`NavigationBackHandler`）、`ViewTree*Owner`、
   `WindowInsets` 控制器、窗口/弹窗宿主、`SavedStateRegistry` 等。它们由 Activity / Window 提供，
   **版本不够就只在"进到那个页面/弹出那个浮层"时炸**。
2. **编译绿不代表运行时稳**：所有"库 + 宿主版本配对"的问题都不在编译期暴露。
   出包前至少把每个新页面**真机打开一次**；打不开就是这类问题，先抓栈，别猜布局。
3. **"弹窗没内容"常常是崩溃**：`OverlayDialog` 缺 owner 时用户看到的是"点了没反应 / 里面是空的"。
   在把它当成"组件没渲染"之前，先去 logcat 找 FATAL。
4. **诊断顺序固定**：logcat FATAL → 栈里库的那一行 → 源码 grep 该 API → `javap` 找第一个实现该接口的宿主版本 → 升依赖（或避开这些组件）。
5. **升不了宿主时的退路**：换掉会装返回拦截的组件（自组合一个不装拦截的浮层），
   或在组合树上自己提供 owner——后者让组件行为依赖自研实现，属"自研件"，要在报告里声明并自己负责语义 / 无障碍。

## 同类风险清单（改依赖前扫一眼）

| 风险 | 症状 | 判据 |
|---|---|---|
| Activity 太老 | `No NavigationEventDispatcher was provided` | `javap` 宿主 AAR 看是否实现 owner 接口 |
| Window 太老 / 无 `WindowNavigationEventScope` | Window* 系列弹窗的返回键行为异常或崩 | 查库的 `window/` 源码是否用 `WindowNavigationEventScope` |
| `minSdk` 低于库的效果要求 | AAR 合并失败，或运行时无效果 | 读 AAR `AndroidManifest.xml` 的 `uses-sdk` + 库的能力检测函数 |
| 宿主不支持 edge-to-edge | 内容被系统栏盖住 | 见 `edge-to-edge.md` |
| Compose 版本漂移 | 编译期 "Unresolved reference"（如 1.11 移除了 `PagerState.animateToPage`，改用 `animateScrollToPage`） | `references/api-verification.md` 第 7 节：先看工程解析出的 Compose 版本 |
