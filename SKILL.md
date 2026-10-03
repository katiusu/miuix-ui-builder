---
name: miuix-ui-builder
description: "Use for Miuix / HyperOS (top.yukonga.miuix) Android UI appearance work — once the project is confirmed to use Miuix, load it for any building, restyling, or reviewing of a Compose screen or app shell: bottom navigation bars (贴底普通 / 悬浮毛玻璃 / 液态玻璃三档都必须提供), top bars, layout, theming, insets & edge-to-edge, blur / 毛玻璃 / 液态玻璃 surfaces, retrofitting an existing app onto the proven shell / page / option skeleton (整体重构与迁移), plus Android build, release packaging and library-version problems. Keywords — Miuix, HyperOS, MiuixTheme, ThemeController, Scaffold, TopAppBar, NavigationBar, FloatingNavigationBar, SearchBar, InputField, Card, Compose, edge-to-edge, insets, blur, glass, release, review."
---

# Miuix UI Builder

## 什么时候该加载（**判定门槛已放宽**）

先做一次判定，成本只有几秒：

```bash
grep -rn "yukonga" --include=build.gradle.kts --include=*.toml --include=*.kt .   # 命中即工程用了 Miuix
```

**判定为"用了 Miuix"（`top.yukonga.miuix.*` 依赖或 import）之后，凡是做 UI 外观，就必须加载本技能** ——
门槛放宽的地方是：**不再要求"点到了某个 Miuix 组件名"才算触发**，只要是外观工作就算。外观包括：

- 底栏 / 顶栏 / 导航 / 页面布局 / 卡片与列表的组织方式；
- 主题、配色、字号、圆角、间距、`ThemeController`；
- insets（状态栏 / 导航栏 / IME / edge-to-edge）；
- 模糊 / 毛玻璃 / 液态玻璃 / 任何视觉效果；
- 用户说"改外观""照这张图改""审查一下界面""加个开关或选项"。

另外两类也加载：

- **构建 / 依赖 / `compileSdk`·`targetSdk` / lint / 出包 / 发版**出问题（本技能有工具链闸门与验证纪律）；
- **还没确定用不用 Miuix** —— 先跑上面那条 grep 再决定，不要靠"我觉得它是 Material"猜。

理由：在 Miuix 工程里，外观工作处处受库的版本、`Defaults` 和组件副作用约束；
按 Material 或凭记忆写一遍再返工，代价远大于一次加载。跟外观/构建无关的任务（纯后端、纯数据、纯文案）不加载。

## 这个技能解决什么

`miuix` 技能是**库的参考手册**（组件目录 + 钉在某版本的源码路径）。本技能是**干活的工作流 + 真实失败清单**，
补上参考手册管不到的七件事：

1. **版本真相**——手册钉在 `v0.9.4`，而工程可能是 0.9.3；照手册写会写出编译不过的 API。
2. **核验纪律**——参数名、`Defaults`、能力检测函数，一律对着**工程实际用的那个版本**的构件核验。
3. **组件契约**——组件内部的副作用（清自己的状态、改 `enabled`、装返回拦截）比它的签名更危险；
   把交互映射到它的回调之前，先读它实现里那几个 `LaunchedEffect` / `SideEffect`。
4. **验证诚实度**——编译 / lint / 产物 / 渲染 / 设备是五档不同强度的证据；
   没有渲染或设备证据时，视觉结论只能写"未验证"。
5. **宿主要求**——库会调用宿主（Activity / Window）的能力，宿主版本不够时编译全绿、一进页面就崩。
6. **固定规格**——有些外观不是"看情况"，而是**硬要求**（见"底栏规格"）。
7. **现成骨架**——外壳、页面、配置项管道都有跑通过的参考实现（见"结构参考"）；先抄骨架再改，
   比每个工程重新发明一套写法便宜得多。**已经在用的工程要整体换骨架**时，按
   `references/retrofit-existing-app.md` 的移植账本走：拷什么、不拷什么、示例业务耦合怎么剥、偏好键怎么对齐。

## 铁律

1. **版本真相**：先读工程依赖。参考手册版本 ≠ 工程版本时**以工程为准**，并在报告里点明差异；不要混用别的版本的 API。
2. **API 必须对着真实构件核验**：参数名、默认值、重载、能力检测函数，从该版本的 **sources jar** 或字节码核验
   （`references/api-verification.md`）。记不准就 `javap`，别猜。
3. **先读组件实现，再映射你的回调**：你想让组件做的事，和组件在同一个回调里自己做的事，可能冲突
   （典型：组件在"收起"时清空你拥有的状态）。读它的 `LaunchedEffect`/`SideEffect`/`enabled` 分支
   （`references/component-contracts.md`）。
4. **Defaults 优先**：尺寸、圆角、阴影、颜色先取组件自带的 `XxxDefaults`；要偏离默认值，必须写清理由。
   注意层级：**用户明确给的视觉 > 库的 Defaults**——两者冲突时别拿"规范"压用户要的效果。
5. **不发明语义 token**：只用主题里真实存在的 color/textStyle；没有 `success`/`warning` 就不要编，
   改用语义组合或让文案承担语义。
6. **不手搓组件**：优先公开组件；库确实没有该交互时才在 `MiuixTheme` 下用 Foundation/Layout 原语自组合，
   并在报告里声明这是自研件、它的语义 / 禁用态 / 无障碍由你负责。
7. **状态归属不变**：重构外观时不要顺手改状态所有者、导航、回调、insets 的所有权。
8. **验证分层**：编译 / lint / 产物 / 渲染 / 设备不能互相顶替（见"验证"一节）。
9. **不承诺做不到的前提**：外部 skill 或文档给的前置条件（如 `targetSdk ≥ 35`、某个 API level 的新能力）
   如果被本机工具链挡住，**先验证可行性，再如实报告受阻项 + 证据**，不要硬改配置把构建推倒。
10. **底栏至少两档**：自带底部导航的应用外壳，**必须同时提供悬浮毛玻璃底栏与贴底普通底栏 + 一个开关**
    （用户明确说不要才例外）；第三档**液态玻璃**按需求加，档位用"悬浮底栏 / 液态玻璃"两个布尔派生
    （`navBarMode`），别让用户拼出无意义组合、也别把档位判定散在两处。见下方"底栏规格（硬性）"
    与 `references/bottom-bars.md` §2.5。
11. **结构先抄骨架**：外壳 / 页面 / 配置项管道都有已验证的参考写法（见"结构参考"）；
    先照抄骨架，再按需求改——重新发明一套结构，等于把别人踩过的坑再踩一遍。

## 底栏规格（硬性要求）

| 形态 | 组件 | 特征 |
|---|---|---|
| **悬浮毛玻璃**（至少要有的这档） | Miuix `FloatingNavigationBar` + `Modifier.textureBlur`（miuix-blur 0.9.4+）或 AndroidLiquidGlass 的 `drawBackdrop` | 悬浮胶囊、内容从玻璃后面穿过、API < 31 自动退回不透明配色 |
| **贴底普通** | Miuix `NavigationBar` + `RowScope.NavigationBarItem` | 不透明底色、顶部分隔线、图标 + 文字标签 |
| **液态玻璃**（可选的第三档） | 自绘 `IosLiquidGlassNavigationBar`（同模块 `internal`） | 悬浮胶囊 + 高光描边 / 边缘折射 / 内阴影，需要 AGSL（API 33+）；**不要**再叠一层 `textureBlur` |

为什么两个都要：悬浮玻璃是"好看的那档"，但它依赖 `RenderEffect`（API 31+）、更重、
在低端机上更吃性能；贴底普通底栏是**兼容与可读性的兜底**（老设备、低端机、就是不喜欢玻璃的人）。
做成开关比替用户选死一档好。三档的档位派生（`navBarMode`）、开关怎么摆、模糊门控与降级链条，
都写在 `references/bottom-bars.md` §2.5。

实现约束（细节与代码见 `references/bottom-bars.md`）：

- 偏好要**落盘**（`AppPrefs` 这类），经状态持有者（`UiPrefsState` 这类）暴露给界面，设置页给一个开关，**即时生效**；
- 关掉悬浮形态时**不创建 backdrop、也不挂 `layerBackdrop`**——不透明底栏用不到"每帧录一次图层"的开销；
- 两种形态**共用同一套"内容让位"逻辑**（各页的 `contentBottomPadding`），不要写两套 padding；
- 两边的 inset 都由组件自己吃（`defaultWindowInsetsPadding = true`），**不要再给父容器加 padding**。

## 结构参考：外壳 / 页面 / 配置项（先抄再改）

三份参考来自一个真实跑通、出过可安装 APK 的纯 GUI 工程
（Miuix 0.9.4 / Compose BOM 2026.09.00 / AGP 9.4.1 / compileSdk 37）。
写外壳或页面之前先读对应那份，**照抄骨架再改**：

| 要做的东西 | 读哪份 | 一句话骨架 |
|---|---|---|
| 应用外壳（多页 + 底栏 + 横屏） | `references/app-shell.md` | 单 Activity 持状态 → `HorizontalPager(userScrollEnabled = false)` 装 4 页 → `navBarMode = if (!isFloatingNavbar) 0 else if (!isLiquidGlass) 1 else 2` 三档底栏 → 宽屏只在 mode 0 换 `NavigationRail` |
| 任何一个页面的内部结构 | `references/page-patterns.md` | `Scaffold(topBar = BlurredBar { TopAppBar }, contentWindowInsets = systemBars + displayCutout .only(Horizontal))` → `Box(blurSource)` → `LazyColumn(pageScrollModifiers)` → 分区 = `SmallTitle` + `Card(h12/b12)` |
| 一堆设置项 / 功能开关 | `references/option-pipeline.md` | 声明 `OptionSpec` → `HookOptionsPage` 渲染 + 全局搜索 → `ConfigState.set` 双写内存与 `PrefsStore` → 门控只走 `rememberOptionEnabled(spec)` |
| **把一个已经在用的工程整体换成这套骨架** | `references/retrofit-existing-app.md` | 移植账本（拷什么 / 不拷什么）→ 剥示例业务耦合（**偏好键必须对齐既有契约**）→ 页面重写顺序 → 收尾（CHANGELOG / tag / Release / README） |

三条最容易违反的结构约定：

1. **状态只提升到外壳**：页面是无状态受控组件（值 + 回调 + `extraBottomPadding`），开关改动**立即落盘**；
2. **insets 只吃一边**：页面 `Scaffold` 的 `contentWindowInsets` 只留 `Horizontal`
   （Miuix 默认是 `systemBars.union(displayCutout)` 全量，**必须显式覆盖**），纵向内边距由内容自己算；
3. **模糊层必须不透明**：顶栏 Haze 的 `backgroundColor = surfaceColor`；做成透明会让卡片硬边缘透出来，
   看起来像"组件盖在模糊之上"而不是"糊在下面"。

## 常见失败（都真实发生过，照单避开）

| 失败 | 症状 | 对策 |
|---|---|---|
| 把只做模糊的效果叫"液态玻璃" | 用户一眼看出"这只是毛玻璃" | 先按 `references/glass.md` 分档，**名字跟着机制走** |
| 把交互映射到组件回调时没读实现 | 刚输入的内容/刚设的状态被组件自己清掉 | `references/component-contracts.md` |
| 拿"库 Defaults 优先"压用户给的参考图 | 越改越不像用户要的效果 | 见铁律 4 的层级 |
| 升库版本前没读 Gradle `.module` | `checkDebugAarMetadata` 因 AGP/Compose 要求直接失败 | `references/api-verification.md` |
| Modifier 工厂函数没用 receiver | lint `ModifierFactoryUnreferencedReceiver` | 必须 `this.then(...)` / `this.xxx(...)` |
| 承诺 `targetSdk`/`compileSdk` 提升却没查 aapt2 | 资源链接阶段整段失败，浪费一轮构建 | `references/android-release.md` 的"工具链闸门" |
| 拿旧产物当新结果 | APK 时间戳早于源码 mtime | Gradle `UP-TO-DATE` + 产物内容双向核对 |
| **只搜 `classes.dex` 就断言"某组件没打进包"** | debug 包是 **multi-dex**，符号在 `classes5/6.dex` 里 → 假阴性 | 遍历 `classes*.dex`；混淆包改查**数据字符串**（偏好键、shader 常量） |
| 只报"构建通过"就收工 | 用户装上去发现布局/观感不对 | 五档证据逐项报，缺的明说"未验证" |
| **把"我替用户定的默认"藏在代码里** | 用户得翻代码才知道默认值/开关位置 | 交付报告里单列一节：代定的默认 + 一句话怎么改 |
| 只做编译期核验，不查宿主版本 | 某个页面/弹窗"打不开"或"没内容"，其实一进就 `IllegalStateException` | `references/runtime-host-requirements.md`（Miuix 需要 `activity ≥ 1.13.0`） |
| 只在"正常机器"的前提下调工具链 | aapt2 读不到容器路径 / 解析不了新 platform 的 `resources.arsc` | `references/aarch64-container-toolchain.md` |
| 顶栏模糊层设成透明 | 卡片硬边缘从模糊层里透出，像"组件盖在模糊之上" | `backgroundColor = surfaceColor`（`references/page-patterns.md`） |
| `WindowDropdownPreference` 没传 `onExpandedChange` | 展开态被组件内部接管，展开/收起与选中不同步 | 显式传 `onExpandedChange = { }`（`references/page-patterns.md`） |
| 搜索只注册"当前页可见项" | 子页面里的项、滑块主开关搜不到 | 子页面与 `masterKey` 也要 `registerAll`（`references/option-pipeline.md`） |
| 隐藏桌面图标后没留回程入口 | 用户自己切完就再也打不开应用 | `activity-alias` + `DONT_KILL_APP`，并留通知/快捷方式入口（`references/app-shell.md`） |
| 每条记录套一张 `Card`、红色表达"正常工作"的指标 | 设计语言里点名的失败做法 + 颜色角色错用 | `references/review-findings.md` 逐条 checklist |
| 改完 `SKILL.md` 的 frontmatter / 只 `curl` raw 地址就宣布发布成功 | 描述里的 `: `（如 `Keywords: ...`）让 YAML 变成嵌套映射 → 加载器报 `No skills found`，**整个技能静默失效**；raw 地址还会给你 CDN 上的旧内容 | 用消费方命令实跑 + 干净 clone 核对 sha（见"验证"一节） |
| release 包 `BUILD SUCCESSFUL` 但装不上 | 包缺 `AndroidManifest.xml` / `resources.arsc`，`aapt2 dump badging` 报 `could not identify format of APK.` —— `optimizeReleaseResources` 静默产出 0 文件 | `android.enableResourceOptimizations=false` + `clean` 重出包（`references/android-release.md`） |
| 换了签名 key 就发版 | 老用户安装报"应用未安装"，只能卸载重装（本机数据全丢） | 发版前比对上一版 APK 的证书 SHA-256；不擅自换 key |
| 按"正文里出现简单名"批量删 import | 委托属性 `getValue` / `setValue` 是隐式使用，删完报 `has no method 'getValue(Nothing?, KProperty0<*>)'` | 让编译器报 unused，或删完立刻编译 |
| 交付文档照上一版转述 | 用户按 README 找不到入口（本次：复制按钮其实在设置页） | 写文档前回读实现，按钮位置与数字都对着源码核 |
| 用 `pkill -f GradleDaemon` 清守护进程 | 模式匹配到调用它的 shell，任务以 `[killed by signal: SIGTERM]` 收尾 | `pkill -f 'Gradle[D]aemon'` |

## 工作流

### 0. 摸清工程（只读）

```bash
cat gradle/libs.versions.toml      # miuix / compose / agp 版本
cat app/build.gradle.kts           # compileSdk / minSdk / targetSdk / 依赖 / lint
cat gradle.properties              # compileSdk 校验、configuration cache 等开关
cat gradle/wrapper/gradle-wrapper.properties
find app/src/main -name "*.kt" | head -50
grep -rn "ComponentActivity" app/src/main/java --include=*.kt   # 有几个 Activity（决定 insets 改造面）
```

要确定的六件事：**Miuix 版本**、**AGP/Gradle/Kotlin/Compose 版本**、`minSdk`/`compileSdk`/`targetSdk`、
**主题与宿主**（`MiuixTheme` 在哪、`Scaffold` 是否唯一）、**insets 模型**（谁吃状态栏/导航栏/IME 内边距）、
**状态所有者**（哪些 state 提升到了外壳）。

### 1. 把版本锁成"可核验的事实"

```bash
curl -sO https://repo1.maven.org/maven2/top/yukonga/miuix/kmp/miuix-ui-android/<版本>/miuix-ui-android-<版本>-sources.jar
mkdir -p /tmp/miuix-src && (cd /tmp/miuix-src && unzip -oq ../miuix-ui-android-<版本>-sources.jar)
```

拿到源码后直接读组件实现与 `Defaults`（比在线文档、渲染站点可靠）。细节见 `references/api-verification.md`。

### 2. 选组件、定形

按场景查 `miuix` 技能（`component-selection` / `usage-patterns` / `design-language` / `preferences-and-menus`），
挑**公开组件**，并在源码里确认每个要用的参数。

### 3. 设计（用一段话说清）

动手前写清：**层级**、**宿主**（要不要新增 `Scaffold`/`MiuixTheme`）、**状态归属**、
**insets 谁让开**（含 IME）、**形状与颜色来源**（哪个 Defaults / 哪个 token）。
结构本身先查"结构参考"那三份：外壳抄 `app-shell.md`、页面骨架抄 `page-patterns.md`、设置项抄 `option-pipeline.md`。
说不清就是还没想清，别写代码。有底栏导航时，先把"底栏两种形态 + 开关"排进方案。

### 4. 实现

- 最小改动、可回退；一次只改一个关注点（结构 / 观感 / 行为）；
- 每个新增的公开 API 调用都能指回源码里的那一行；
- 偏离 Defaults 时把原因写进注释（为什么是 24dp 而不是默认 16dp）；
- 涉及模糊/玻璃时先读 `references/glass.md`；涉及系统栏/全屏时先读 `references/edge-to-edge.md`；
  涉及底栏/导航时先读 `references/bottom-bars.md`；
  涉及外壳结构读 `references/app-shell.md`、页面内部读 `references/page-patterns.md`、设置项列表读 `references/option-pipeline.md`。

### 5. 验证（按强度递增，做到哪步报到哪步）

```bash
./gradlew :app:compileDebugKotlin     # 最快：API/类型是否正确
./gradlew :app:assembleDebug          # 能装
./gradlew :app:assembleRelease        # R8 + 资源压缩后是否还成立
./gradlew :app:lintDebug              # 静态检查；先看基线，再谈"新增"
```

产物不要只看 `BUILD SUCCESSFUL`：**release 包还要过"四件套"**——`AndroidManifest.xml`、`resources.arsc`、
`classes*.dex` 都在，且 `aapt2 dump badging` 能读出 `package:`（残包长什么样、怎么修见 `references/android-release.md`）。

```bash
/opt/android-sdk/aapt2-arm64/aapt2 dump badging <apk> | grep -E "^package:|sdkVersion|targetSdk"
# 查"某个组件/能力是否真的进了包"：debug 包是 multi-dex，必须遍历所有 dex
for d in $(unzip -l <apk> | grep -oE "classes[0-9]*\.dex"); do
  unzip -p <apk> "$d" | strings | grep -c "<组件名或 shader 常量>"
done
./gradlew :app:assembleRelease        # 再跑一次；compileXxxKotlin UP-TO-DATE 即"产物=当前源码"
```

**渲染/设备证据**：有预览、模拟器、真机截图才算"观感已确认"。拿不到就写"未验证"，并说清缺什么、怎么补。

**发布物证据（推 skill / 推文档 / 发 Release 同理）**：不要用 `curl` 某个 raw 地址当唯一证据
——它在 CDN 上会缓存旧内容，会让你"验证"到一个已经不存在的版本（实测踩过）。要在干净目录里
`git clone --depth=1` 核对 sha，或用**真正的消费方命令**跑一遍（skill → `npx -y skills add <owner>/<repo> -l`），
看到它被正确识别才算发布成功。改 `SKILL.md` 的 frontmatter 尤其要跑：
描述里出现 `: `（例如 `Keywords: ...`）会让 YAML 把描述当成嵌套映射，
加载器报 `No skills found`，**整个技能静默失效**，而文件看起来完全正常。

### 6. 交付报告（五段，不要多）

1. **改了什么**（文件 + 一句话职责）；
2. **依据**（哪条是源码契约、哪条是 Defaults、哪条是用户指定的视觉、哪条是自研件、哪条是硬性规格）；
3. **我替你定的默认**（默认值 / 开关位置 / 交互选择，逐条写"想改成什么就动哪里"）；
4. **验证到哪一步**（编译 / lint / 产物 / 渲染 / 设备逐项给结果；lint 要给"基线 vs 新增"）；
5. **未验证 / 已知取舍**（明确列出，不要用"应该没问题"糊过去）。

## 玻璃：先分清毛玻璃和液态玻璃

| 想要的效果 | 需要的机制 |
|---|---|
| 毛玻璃 / 磨砂（半透明 + 模糊 + 一层底色） | 只对背景做高斯模糊（`BlurEffect` / 一次 `textureBlur` / backdrop 的 `blur()`） |
| **液态玻璃** | 模糊 **+ 边缘折射/位移** + **高光描边** + 提饱和（+ 可选色散）；折射需要 AGSL `RuntimeShader` |

**只做模糊就写"悬浮毛玻璃"，不要写"液态玻璃"。** 配方、两套库的对照、迁移与降级见 `references/glass.md`。

三个通用约束：① 取样源不能包含被模糊物自身（否则拖影，需分两层 backdrop）；
② 能力分层降级（模糊 API 31 / 折射 API 33，低于门槛退回不透明配色，**不要只把颜色刷透明**）；
③ 模糊之上要叠半透明容器色，否则明暗主题下文字/图标读不清。

## 参考

- `references/api-verification.md` —— 把任意版本依赖拉下来核验 API；用 Gradle `.module` 预判版本冲突
- `references/component-contracts.md` —— 组件内部会动你的状态：怎么找、怎么改
- `references/bottom-bars.md` —— **底栏规格的完整实现**：两种形态、开关、落盘、backdrop 门控、内容让位、RowScope 坑
- `references/app-shell.md` —— **应用外壳骨架**：单 Activity + 多页 pager + 三档底栏（悬浮/贴底/液态玻璃）
  + 宽屏 rail、内容让位三条路径、主题与窗口背景、语言、桌面图标隐藏
- `references/page-patterns.md` —— **页面结构范式**：三种页面形态、通用骨架、顶栏渐进模糊参数、元素选择表、
  主页仪表盘两套布局、设置页与关于页的实测细节
- `references/option-pipeline.md` —— **声明式配置项管道**：`OptionSpec` 字段表、9 种类型渲染、三条门控、
  默认值语义、三个输入对话框、外壳级 `AppSettings` 模式
- `references/glass.md` —— 毛玻璃 vs 液态玻璃、两套库配方、换库迁移的坑、降级阶梯
- `references/edge-to-edge.md` —— 全屏 + 系统栏 / IME 内边距检查单、系统栏图标与主题的坑、被工具链挡住时怎么报
- `references/android-release.md` —— 出包 → 核对 → 签名 → 推送 → 发 Release；工具链闸门
- `references/runtime-host-requirements.md` —— 宿主要求：编译通过 ≠ 组合期不炸；从 logcat 栈 + `javap` 归因"页面打不开"
- `references/aarch64-container-toolchain.md` —— aarch64 容器专属：aapt2 双命名空间 shim（含 daemon stdin 协议）、影子 SDK、镜像、Git 兜底
- `references/retrofit-existing-app.md` —— **把既有工程整体换成这套骨架**：移植账本（骨架件 / 管道件 / 示例业务件）、
  剥业务耦合与偏好键对齐、页面重写顺序、移植期三个编译坑、构建期验收指标、代码之外的交付物
- `references/review-findings.md` —— 一轮 UI review 实际会抓到的 15 类界面不符合项 + 6 类工程与交付不符合项（可直接当自查 checklist）
