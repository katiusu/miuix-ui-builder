# 把既有 Miuix 工程整体重构成骨架（实测移植账本）

适用：工程**已经能用、甚至在发版**，需求是"UI 整体对齐某个示例骨架"，而不是只改一屏。
本文件来自真实案例：HyperOS-Autofill-Fix 2.2.0 → 2.3.0（12 个 `.kt` / 1481 行 → 60 文件 / +6380 行，
同时把工具链从 Miuix 0.9.3 / AGP 8.9.2 / compileSdk 34 升到 Miuix 0.9.4 / AGP 9.4.1 / compileSdk 37）。

## 0. 先让基线构建跑通（否则后面的失败归因不清）

```bash
printf 'sdk.dir=/opt/android-sdk\n' > local.properties   # 容器里 ANDROID_HOME 通常只在 login shell 里
./gradlew :app:assembleDebug --console=plain
```

- 基线失败必须先解决；本机典型报错是 `SDK location not found ... local.properties`；
- 记下基线产物的版本号 / 大小 / 条目数，后面用它判断"改造有没有把既有能力弄丢"。

## 1. 移植账本：明确拷什么、不拷什么

示例工程是**完整应用**（含更新检查、Hook 回执、崩溃后重启、桌面图标切换……），整包拷等于把无关依赖、
跨进程假设和死代码一起搬进你的工程。按"UI 是否引用 / 是否示例业务"两问筛：

| 分类 | 示例里的件 | 处理 |
|---|---|---|
| 骨架件（必拷） | `ui/theme/Theme.kt`、`ui/util/{BlurUtils,MiuixAnimations,WindowBackground}.kt`、`ui/component/{liquid,animation,effect}/*`、`device/*`、`util/SystemVersionDetector.kt`、`ui/component/SubPageScaffold.kt`、`ui/screen/subpage/BaseSubPageActivity.kt`、`ui/icons/StatusIcons.kt` | 直接拷，只改包名 |
| 管道件（拷了要改） | `prefs/*`、`ui/component/pref/*` | 拷，再按 §2 剥掉示例业务耦合 |
| 示例业务件（不拷） | `bridge/{HookStatusStore,XposedServiceManager,AppRestarter}.kt`、`UpdateChecker.kt`、`ui/component/UpdateDialog.kt`、`LauncherIconController.kt`、`TemplateApp.kt`、`ui/component/{SystemRestartConfirmDialog,pref/QuickActionDialog}.kt`、`ui/screen/features/*`、`LicenseActivity.kt`、`LocaleHelper.kt` | 删；UI 里有引用就按"直接删入口 / 换同名桩"处理 |
| 页面（抄样式、内容自写） | `ui/screen/{home,settings,about}/*` | 不整体拷；按 `page-patterns.md` 重写，保留本工程原有功能与文案 |

批量改包名（`/sdcard` 这类 FUSE 卷上 `sed -i` 不可靠，用临时文件替换）：

```bash
for f in $(find <src> -name '*.kt'); do
  sed 's/com\.other\.example/com.mine.app/g' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done
```

拷完**立刻单独编译一次**（`./gradlew :app:compileDebugKotlin`），把"缺依赖 / 缺桩"的报错就地解决，
不要和后面的页面重写混在一轮里排查。

## 2. 剥掉示例的业务耦合（三处，缺一就会出现"设置了不生效"）

1. **偏好存储要对齐既有键**：`PrefsStore` 的 `PREFS_NAME` / `KEY_PREFIX` 必须指向**本应用原本那份
   SharedPreferences 与键名**——这是跨进程契约（Hook / 后台侧读的就是这些键）。改之前先 grep 一遍旧键名，
   改完再确认两边读写的是同一批 key。
2. **Hook 回执类封装整段删除**：`rememberHookApplied` / `ensureScopeFor` / `recordOptionApplied` /
   `HookStatusStore` 这类"把改动回执给目标进程"的件在纯 GUI 工程里没有对应方，只会留下一堆空实现；
   只保留 `rememberEffectiveDeviceType` / `rememberDeviceScopeEnabled` / `rememberDependencyEnabled` /
   `rememberOptionEnabled`（`OptionSpec.showStatus` 字段可以留着，默认 `false`）。
3. **外壳级 `AppSettings` 按需精简**：留下主题 / 底栏 / 模糊这些真正用到的字段，其余（语言、更新检查、
   桌面图标）删掉；**旧键迁移写进 `load()`**（本次：旧 `theme_mode` Int `1/2/3` → `Light`/`Dark`/`MonetSystem`，
   `floating_nav_bar` → `isFloatingNavbar`，迁移完立刻 `save()`，否则每次启动都要重迁）。

## 3. 页面重写顺序

外壳（`MainActivity` + 底栏组件）→ 概览（信息密度最高）→ 日志（列表 + 搜索）→ 设置（声明式管道）→ 关于（折叠头部，可最后做）。

- 每页的**功能与文案以旧实现为准**：`git show HEAD:<旧文件>` 取原文，不要凭记忆重写
  （与 `bottom-bars.md` 坑 5 同理）；
- 页面清单、顶栏操作这类要写进文档的信息，**回读自己刚写的代码**再写（本次就抓出 README 里
  "日志页有复制按钮"与实现不符：复制在设置页，日志页顶栏只有"切换原文/直观"和"清空"）。

## 4. 移植期最常踩的三个编译坑

1. **不要按"正文里出现简单名"批量删 import**：`getValue` / `setValue`（委托）、`kotlinx.coroutines.job`
   这类是隐式使用，删了会报 `has no method 'getValue(Nothing?, KProperty0<*>)'`。
   让编译器报 unused，或者删完立刻编译一遍。
2. **Miuix `Button` 没有 `text` 参数**（内容走尾随 lambda）；`TextButton(text = ...)` 才有。
3. **`MiuixTheme.textStyles` 没有 `title4`**。0.9.4 实测可用：`main` / `paragraph` / `body1` / `body2` /
   `button` / `footnote1` / `footnote2` / `headline1` / `headline2` / `subtitle` / `title1` / `title2` / `title3`。

## 5. 验收（"只做构建期验证"也要有硬指标）

- **残留引用 grep 必须为 0**：`HookStatusStore|XposedServiceManager|ensureScopeFor|LocaleHelper|QuickAction|AppRestarter|UpdateChecker|LauncherIconController`；
- **字符串对账**：代码里的 `R.string.*` 与 `strings.xml` 双向核对，MISSING / unused 都应为空（本次 128 个键）；
- **两个包都出得来**，且 release 包里**确实有** `AndroidManifest.xml` 与 `resources.arsc`
  （见 `android-release.md` 的"残包"一节——这一条不查就会发出装不上的包）；
- `aapt2 dump badging` 的 `versionCode` / `versionName` / `minSdk` / `compileSdk` 与构建配置一致。

## 6. 收尾：代码之外的交付物

整体重构通常会顺带改版本号与发版，别只推代码：

- `CHANGELOG.md` 加一段（新增 / 变更 / 修复）+ annotated tag `vX.Y.Z` + GitHub Release
  （正文含**签名证书 SHA-256** 与 **APK SHA-256**，让用户能确认可覆盖升级、能校验产物）；
- `README.md` 的界面表、代码结构表、构建段、坑清单同步更新——**照着代码核**，不要照上一版 README 转述；
- 仓库 topics / description 顺手补齐（发版后一次 `PUT /repos/{repo}/topics` 即可）。

一句话教训：**照抄骨架要连"豁免清单"一起抄**——示例里那些与 UI 无关的件（更新检查、Hook 回执、
重启应用）不但没用，还会把无关依赖和跨进程假设带进你的工程；而"偏好键要对齐"这种跨进程契约，
是重构里唯一不能凭示例抄的东西。
