# 声明式配置项管道：一个 data class 驱动整页设置

来源同上（MiuixGuiExample）。这套管道解决的是"**选项越加越多、每个都手写一遍状态 + 落盘 + 搜索**"的问题：
把选项写成数据，渲染、持久化、搜索、备份全部由管道统一处理。

```
OptionSpec（声明）→ OptionRegistry（检索 / 依赖解析）
                  → HookOptionsPage + HookOptionView（渲染）
                  → ConfigState（响应式状态，写它即重组）
                  → PrefsStore（SharedPreferences 落盘）
                  → ConfigBackup（JSON 导入导出）
```

页面外壳（顶栏 / 底栏 / insets）见 `app-shell.md`；页面骨架见 `page-patterns.md`。

## 1. 分层职责

| 层 | 文件角色 | 职责 | 关键实现 |
|---|---|---|---|
| 声明 | `prefs/OptionSpec.kt` | 一个 `data class` 描述一个选项 | 22 个字段，见 §2 |
| 类型 | `prefs/OptionType.kt` | `enum`：SWITCH / CHECKBOX / ARROW / DROPDOWN / SPINNER / RADIO / SLIDER / TEXT / PACKAGE_LIST | 渲染层按它分派 |
| 注册表 | `prefs/OptionRegistry.kt` | `mutableStateMapOf<String, OptionSpec>`；`register` / `registerAll` / `all()` / `find(key)` / `search(context, query)` | search 按 title / summary 小写包含匹配 |
| 状态 | `prefs/ConfigState.kt` | `mutableStateMapOf<String, Any?>` + `@Volatile initialized`；`init` / `reload` / `get` / `bool` / `int` / `float` / `string` / `set` | **`set` 双写**：内存（驱动重组）+ `PrefsStore`（落盘） |
| 存储 | `prefs/PrefsStore.kt` | `SharedPreferences`，名 `miuix_template_prefs`，键前缀 `prefs_key_`，远程镜像组 `miuix_template_remote` | `init` / `get*` / `put` / `remove` / `getAll` / `clearAll` |
| 备份 | `prefs/ConfigBackup.kt` | `exportJson` / `importJson`；**跳过 `runtime_` 前缀键**；导入后调 `ConfigState.reload()` | 导入完通常还要 `recreate()` 让主题/语言重生效 |
| 渲染 | `ui/component/pref/HookOptionsPage.kt`、`HookOptionView.kt`、`Hook*Card.kt` | 页面 = 顶栏 + 搜索 + 分区卡片；`HookOptionView` 按 `type` 分派到卡片 | 见 §3 / §6 |

`ConfigState` 的读取要**带默认值**（`ConfigState.bool(key, spec.defaultBoolean)`），
磁盘没有该键时由调用方给默认 —— 声明里的默认值与读取时的默认值必须是同一个来源，别写两份。

## 2. OptionSpec 字段（照抄这张表）

```kotlin
data class OptionSpec(
    val key: String,                              // 裸键；落盘时由 PrefsStore 自动加前缀
    val type: OptionType,
    val titleRes: Int,                            // 只存资源 id，不存字符串
    val summaryRes: Int = 0,                      // 0 = 无副标题
    val defaultBoolean: Boolean = false,
    val defaultInt: Int = 0,
    val defaultFloat: Float = 0f,
    val defaultString: String = "",
    val entryResIds: List<Int> = emptyList(),     // 下拉/单选/选择器的文案
    val entryValues: List<String> = emptyList(),  // 与 entryResIds 一一对应；[0] 约定为"默认/不生效"
    val targetPackages: List<String> = emptyList(),
    val deviceScope: Set<DeviceType>? = null,     // 设备形态白名单；null/空 = 通用；非白名单禁用不隐藏
    val showStatus: Boolean = false,              // 副标题追加「已生效 / 未生效」
    val dependsOn: String? = null,                // 布尔依赖键
    val dependsOnValue: Boolean = true,           // 依赖键需等于该值才启用
    val masterKey: String? = null,                // 滑块的主开关（关 → 隐藏滑块）
    val sliderMin: Float = 0f,
    val sliderMax: Float = 100f,
    val sliderStep: Float = 1f,
    val sliderDecimals: Int = 0,
    val sliderUnitRes: Int = 0,
    val sliderValueLabelRes: Int = 0,
)
```

为什么这么设计：

- **只存资源 id**：`titleRes` / `summaryRes` 若改成 `String`，声明就离不开 `Context`，全局搜索也没法在非组合期索引；
- **`entryValues[0]` 是约定**：下拉/单选的第一项代表"默认档（不做任何事）"，见 §4；
- **`deviceScope` 是白名单而不是开关**：不匹配时组件**可见但禁用**（灰显），用户能看到"这个功能在我的设备上不存在"，比凭空消失可解释。

## 3. 渲染：一个 `when` 分派 + 三条门控

```kotlin
@Composable
fun HookOptionView(spec: OptionSpec, modifier: Modifier = Modifier, onArrowClick: () -> Unit = {}) {
    when (spec.type) {
        OptionType.SWITCH       -> HookSwitchCard(spec, modifier)
        OptionType.CHECKBOX     -> HookCheckboxCard(spec, modifier)
        OptionType.ARROW        -> HookArrowCard(spec, modifier, onArrowClick)
        OptionType.DROPDOWN     -> HookDropdownCard(spec, modifier)
        OptionType.RADIO        -> HookRadioCard(spec, modifier)
        OptionType.SLIDER       -> HookSliderCard(spec, modifier)
        OptionType.TEXT         -> HookTextCard(spec, modifier)
        OptionType.PACKAGE_LIST -> HookPackageListCard(spec, modifier)
        OptionType.SPINNER      -> HookDropdownCard(spec, modifier)   // SPINNER 复用下拉
    }
}
```

所有卡片的 `enabled` 都收口到同一个函数：

```kotlin
@Composable
fun rememberOptionEnabled(spec: OptionSpec): Boolean =
    rememberDependencyEnabled(spec) && rememberDeviceScopeEnabled(spec)
```

1. **布尔依赖**：`dependsOn` 的默认值从注册表取（`OptionRegistry.find(dependsOn)?.defaultBoolean ?: false`），
   不要硬编码；`dependsOnValue = false` 表示"依赖项为假时才启用"（取反）。
2. **设备形态**：`rememberEffectiveDeviceType()`（用户覆盖值 `DeviceType.fromKey(ConfigState.string(DeviceContext.KEY_DEVICE_TYPE, OVERRIDE_AUTO))`，无覆盖则用自动判定）
   是否落在 `spec.deviceScope` 内。读的是 `ConfigState`，所以设置页改设备类型会**实时重组**，无需重启页面。
3. **滑块主开关**：由卡片自己处理（见 §4），不进 `rememberOptionEnabled`。

卡片改动的固定三步（顺序别换）：

```kotlin
ConfigState.set(spec.key, value)                                   // 1. 写状态（内存 + 落盘）
HookStatusStore.removeKeys(context, listOf(spec.key))              // 2. 清掉旧的「已生效」证据
if (新值离开默认档) { ensureScopeFor(spec); recordOptionApplied(context, spec) }  // 3. 触发副作用 + 记录
```

副标题的状态拼接：

```kotlin
private fun optionSummary(spec: OptionSpec): String? {
    val base = spec.summaryRes.takeIf { it != 0 }?.let { stringResource(it) }
    if (!spec.showStatus) return base
    val status = stringResource(if (rememberHookApplied(spec.key)) R.string.hook_status_applied
                                else R.string.hook_status_not_applied)
    return listOfNotNull(base, status).joinToString(" · ")
}
```

**纯 GUI 工程的变体**：没有后端进程回报"生效"时，把"已生效"降级为**本机记录**——
在原本该触发副作用的时机（即"离开默认档"那一刻）调一次 `HookStatusStore.record(context, listOf(key))`，
文案与交互时序就与原来完全一致，不需要改任何界面代码。

## 4. 默认值语义（最容易写错的地方）

| 类型 | 约定 | 实现要点 |
|---|---|---|
| DROPDOWN / RADIO | `entryValues[0]` = 默认档 | `defaultIndex(spec) = entryValues.indexOf(defaultString)`，兜底 0；选中 index **不等于** defaultIndex 时才触发副作用（原注释：「第一个选项为默认（不 hook）」） |
| SLIDER | `masterKey` 关 → 滑块隐藏且不生效 | 主开关画在滑块上方；滑块用 `AnimatedVisibility(visible = masterEnabled && enabled, enter = expandVertically(MiuixExpandSpec), exit = shrinkVertically(MiuixExpandSpec))` 包住（`MiuixExpandSpec` 是工程里统一的展开弹簧规格） |
| SLIDER 刻度 | `steps = 中间刻度数` | `((sliderMax - sliderMin) / sliderStep).toInt() - 1`，`coerceAtLeast(0)`；`sliderStep <= 0` 直接给 0 |
| TEXT / PACKAGE_LIST | 空字符串 = 未设置 | 摘要显示"未设置"占位文案，不要把空串直接显示 |

数值格式化（滑块通用）：

```kotlin
fun roundToDecimals(value: Float, decimals: Int): Float =
    BigDecimal(value.toString()).setScale(decimals, RoundingMode.HALF_UP).toFloat()

fun formatValue(value: Float, decimals: Int, unit: String) =
    String.format(Locale.US, "%.${decimals}f", value) + unit    // ← 必须 Locale.US
```

`Locale.US` 不是洁癖：德语/法语区默认 locale 的小数点是逗号，直接 `String.format` 会输出 `1,5%`，
再被解析回数值时就出错。

## 5. 输入对话框：一套骨架，三处复用

三处输入（滑块定值、文本、包名列表）共用同一套骨架：`WindowDialog` + **等宽三按钮**
「取消 / 恢复默认 / 确定」（确定用 `buttonColorsPrimary`），恢复默认走 `ConfigState.set(key, 默认值)`。

- **滑块定值**：三栏展示「最小值 / 当前值 %1$s · 默认值 %2$s / 最大值」+ `TextField(singleLine = true)`；
  非法或越界时 `isError = true` 并显示范围提示；解析用 `toBigDecimalOrNull()`
  （**不要用 `toFloatOrNull`**：`"1e"`、超长数字这类边界要能判非法）。
- **文本**：`TextField` + 「当前：%1$s / 默认：%1$s」两行 + 占位提示；空串合法（回到"未设置"）。
- **包名列表**：`TextField(singleLine = false, maxLines = 6)`；解析用统一函数
  `parsePackageList(raw)` = `split(',', '，', ';', '；', '\n', ' ', '\t')` → `trim` → 去空 → `distinct`
  —— **中英文标点都要吃**，用户从不同地方粘贴的分隔符不一样。

## 6. 页面级结构（HookOptionsPage）

- **顶栏**：`BlurredBar` + `TopAppBar`，`actions` = 调用方扩展槽 + （页内存在 PACKAGE_LIST 选项时）自动出现的"刷新/重启"按钮；
- **搜索**：`SearchBar(inputField = { InputField(query, onQueryChange, onSearch, expanded, onExpandedChange, label) },
  outsideEndAction = { "取消" }, expanded, onExpandedChange)`；
- **搜索命中处理**：`OptionRegistry.search(context, query)` → `.filter { searchTargets.containsKey(it.key) }`
  → `distinctBy { 分区 "s:titleRes" / 子页 "p:path" }`；命中分区 → `pendingScrollIndex = sections.indexOf(section) + 1`，
  收起搜索后 `animateScrollToItem(target)`；命中子页 → 直接 `subPage.onOpen()`；最后统一 `expanded = false; query = ""`；
- **分区渲染**：`SmallTitle(text = 标题) + Card(Modifier.padding(horizontal = 12.dp).padding(bottom = 12.dp))`，
  采用 `item(key = section.titleRes)` 给 LazyColumn 稳定 key；
- **子页面并入搜索**：`HookSubPage(titleRes, specs, onOpen, subPages)` 递归展开（`flattenSubPages`），
  搜索结果摘要用 `父 / 子` 路径；子页面自己**不放搜索栏**；
- **注册时机**：`LaunchedEffect(sections, flatSubPages) { OptionRegistry.registerAll(allSpecs) }`，
  `allSpecs` 含子页面项**与各 `masterKey`**（否则主开关搜不到，滑块只能搜到滑块本身）。

## 7. 加一个选项要改哪里

1. `strings.xml` 加 title / summary（`values/` 中文 + `values-en/` 英文，缺英文会回退中文）；
2. 在对应分组的声明列表里加一条 `OptionSpec(...)`；
3. **不需要动渲染层** —— 类型已有就完事；确实需要新交互时才加 `OptionType` 值 + `HookOptionView` 的分支；
4. 主题 / 语言 / 更新这类"外壳级"开关不走这套管道（它们的值属于 `AppSettings`，由外壳回调持有，见 `app-shell.md`）。

## 8. 坑

- 状态写在页面里（`remember { mutableStateOf(...) }`）→ 切页/旋屏就丢；配置项一律走 `ConfigState`。
- `ConfigState.set` 只写内存、漏了 `PrefsStore.put` → 重启回到默认值（双写必须成对）。
- 搜索搜不到：被 `dependsOn` 依赖的键、子页面里的键、`masterKey` 都必须注册进 `OptionRegistry`。
- 导入备份后忘记 `ConfigState.reload()` → 界面仍显示旧值（内存缓存没刷新）。
- 导出时没排除 `runtime_` 前缀键 → 把进程运行态（开机号、临时计数）当配置迁移到新设备。
- 把默认值写在两处（声明里一处、读取兜底一处）→ 迟早分叉；统一从 `spec` 取。

## 9. 外壳级设置：`AppSettings` 三段式

功能选项走 `OptionSpec` 管道；**外壳级设置**（主题 / 底栏档位 / 模糊 / 语言 / 更新检查 / 桌面图标）
用一个小 data class 承载，写法固定三段：

```kotlin
data class AppSettings(
    val themeMode: String = "System",              // 存枚举名，读回来要兜底
    val isFloatingNavbar: Boolean = false,
    val isLiquidGlass: Boolean = false,
    val isBlurEnabled: Boolean = true,
    val checkUpdateOnLaunch: Boolean = true,
    val language: String = "",                     // 与 LocaleHelper.Language.SYSTEM 的空串对齐
    val hideLauncherIcon: Boolean = false,
) {
    fun toJson(): String = JSONObject().apply { /* 字段名与属性名逐字相同 */ }.toString(2)   // 缩进便于人工编辑
    companion object {
        fun fromJson(json: String): AppSettings = try { /* optString / optBoolean(..., 默认值) */ } catch (_: Exception) { AppSettings() }
        fun load(context: Context): AppSettings       // SharedPreferences
        fun save(context: Context, settings: AppSettings)   // androidx.core.content.edit { }
        fun importFromJson(context: Context, json: String): AppSettings   // 先解析 → 再 save → 再返回
    }
}
```

规矩：

1. **默认值只写一处**：全写在 data class 主构造里；`load` 与 `fromJson` 复用同一组字面量。
   （`fromJson` 用 `optString` / `optBoolean(name, 默认值)`，整体再包一层 `try/catch → AppSettings()`：
   字段缺失、类型错、整段 JSON 坏，都退化成默认值，**不抛异常**——导入的是用户手上的文件，不能信。）
2. **`importFromJson` 先解析成功再 `save`**，解析失败不污染当前配置。
3. **语言是双写来源**：`AppSettings.language` 只是镜像，真值在 `LocaleHelper`（`app_prefs` / `language_code`），
   `load` 时从 `LocaleHelper.getSavedLanguage(context).code` 取，`save` 时同步写回。
4. 读枚举名（主题）时一律"查表 + 兜底"：`try { ColorSchemeMode.valueOf(saved) } catch (_: Exception) { ColorSchemeMode.System }`；
   枚举查表（语言）用 `entries.find { it.code == code } ?: Language.SYSTEM`。旧版本写过的脏值不能让应用起不来。

什么时候该用哪个：**用户能"搜索 / 依赖 / 分组"的功能选项**用 `OptionSpec`（见 §1–§8）；
**只影响应用外观与外壳行为、不进功能列表**的用 `AppSettings`。
