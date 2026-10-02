---
name: miuix-ui-builder
description: "Use for any Android UI work in this workspace — building, restyling, or reviewing a Jetpack Compose screen or app shell, bottom navigation bars (悬浮底栏与普通底栏都必须提供), theming, insets & edge-to-edge, blur / 毛玻璃 / 液态玻璃 surfaces, and Android build or library-version problems. Load it even when you are not sure yet whether the project uses Miuix / HyperOS components — it starts by identifying what the project actually uses. Keywords — Miuix, HyperOS, MiuixTheme, ThemeController, Scaffold, TopAppBar, NavigationBar, FloatingNavigationBar, SearchBar, InputField, Card, Compose, AGP, compileSdk, lint, edge-to-edge, insets, blur, glass, release, review."
---

# Miuix UI Builder

## 什么时候该加载（**放宽后的标准**）

满足**任意一条**就加载，不要先花时间去确认"这是不是 Miuix 工程"：

- 工程里有 Compose 界面代码（`*Screen.kt` / `@Composable` / `androidx.compose.*` 依赖）；
- 要动底栏 / 顶栏 / 导航 / 主题 / 任何页面布局；
- 要动 insets（状态栏、导航栏、IME、edge-to-edge）；
- 要动模糊 / 毛玻璃 / 液态玻璃 / 任何视觉效果；
- 构建、依赖、`compileSdk`/`targetSdk`、lint、出包、发版出了问题；
- 用户说"改外观""照这张图改""审查一下界面""加个开关/选项"。

**不确定就加载。** 本技能的第 0 步就是判断工程到底用哪套组件（Miuix 还是纯 Compose、版本多少），
加载的成本远低于按错版本写一遍再返工。唯一不该加载的是和 Android 界面/构建完全无关的任务
（纯后端、纯数据处理、纯文案）。

## 这个技能解决什么

`miuix` 技能是**库的参考手册**（组件目录 + 钉在某版本的源码路径）。本技能是**干活的工作流 + 真实失败清单**，
补上参考手册管不到的五件事：

1. **版本真相**——手册钉在 `v0.9.4`，而工程可能是 0.9.3；照手册写会写出编译不过的 API。
2. **核验纪律**——参数名、`Defaults`、能力检测函数，一律对着**工程实际用的那个版本**的构件核验。
3. **组件契约**——组件内部的副作用（清自己的状态、改 `enabled`、装返回拦截）比它的签名更危险；
   把交互映射到它的回调之前，先读它实现里那几个 `LaunchedEffect` / `SideEffect`。
4. **验证诚实度**——编译 / lint / 产物 / 渲染 / 设备是五档不同强度的证据；
   没有渲染或设备证据时，视觉结论只能写"未验证"。
5. **固定规格**——有些外观不是"看情况"，而是**硬要求**（见"底栏规格"）。

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
10. **底栏两种形态**：自带底部导航的应用外壳，**必须同时提供悬浮毛玻璃底栏与贴底普通底栏 + 一个开关**
    （用户明确说不要才例外）。见下方"底栏规格（硬性）"。

## 底栏规格（硬性要求）

| 形态 | 组件 | 特征 |
|---|---|---|
| **悬浮毛玻璃**（默认） | Miuix `FloatingNavigationBar` + AndroidLiquidGlass 的 `drawBackdrop` | 悬浮胶囊、内容从玻璃后面穿过、API < 31 自动退回不透明配色 |
| **贴底普通** | Miuix `NavigationBar` + `RowScope.NavigationBarItem` | 不透明底色、顶部分隔线、图标 + 文字标签 |

为什么两个都要：悬浮玻璃是"好看的那档"，但它依赖 `RenderEffect`（API 31+）、更重、
在低端机上更吃性能；贴底普通底栏是**兼容与可读性的兜底**（老设备、低端机、就是不喜欢玻璃的人）。
做成开关比替用户选死一档好。

实现约束（细节与代码见 `references/bottom-bars.md`）：

- 偏好要**落盘**（`AppPrefs` 这类），经状态持有者（`UiPrefsState` 这类）暴露给界面，设置页给一个开关，**即时生效**；
- 关掉悬浮形态时**不创建 backdrop、也不挂 `layerBackdrop`**——不透明底栏用不到"每帧录一次图层"的开销；
- 两种形态**共用同一套"内容让位"逻辑**（各页的 `contentBottomPadding`），不要写两套 padding；
- 两边的 inset 都由组件自己吃（`defaultWindowInsetsPadding = true`），**不要再给父容器加 padding**。

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
| 每条记录套一张 `Card`、红色表达"正常工作"的指标 | 设计语言里点名的失败做法 + 颜色角色错用 | `references/review-findings.md` 逐条 checklist |
| 改完 `SKILL.md` 的 frontmatter / 只 `curl` raw 地址就宣布发布成功 | 描述里的 `: `（如 `Keywords: ...`）让 YAML 变成嵌套映射 → 加载器报 `No skills found`，**整个技能静默失效**；raw 地址还会给你 CDN 上的旧内容 | 用消费方命令实跑 + 干净 clone 核对 sha（见"验证"一节） |

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
说不清就是还没想清，别写代码。有底栏导航时，先把"底栏两种形态 + 开关"排进方案。

### 4. 实现

- 最小改动、可回退；一次只改一个关注点（结构 / 观感 / 行为）；
- 每个新增的公开 API 调用都能指回源码里的那一行；
- 偏离 Defaults 时把原因写进注释（为什么是 24dp 而不是默认 16dp）；
- 涉及模糊/玻璃时先读 `references/glass.md`；涉及系统栏/全屏时先读 `references/edge-to-edge.md`；
  涉及底栏/导航时先读 `references/bottom-bars.md`。

### 5. 验证（按强度递增，做到哪步报到哪步）

```bash
./gradlew :app:compileDebugKotlin     # 最快：API/类型是否正确
./gradlew :app:assembleDebug          # 能装
./gradlew :app:assembleRelease        # R8 + 资源压缩后是否还成立
./gradlew :app:lintDebug              # 静态检查；先看基线，再谈"新增"
```

产物不要只看 `BUILD SUCCESSFUL`：

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
- `references/glass.md` —— 毛玻璃 vs 液态玻璃、两套库配方、换库迁移的坑、降级阶梯
- `references/edge-to-edge.md` —— 全屏 + 系统栏 / IME 内边距检查单、系统栏图标与主题的坑、被工具链挡住时怎么报
- `references/android-release.md` —— 出包 → 核对 → 签名 → 推送 → 发 Release；工具链闸门
- `references/runtime-host-requirements.md` —— 宿主要求：编译通过 ≠ 组合期不炸；从 logcat 栈 + `javap` 归因"页面打不开"
- `references/aarch64-container-toolchain.md` —— aarch64 容器专属：aapt2 双命名空间 shim（含 daemon stdin 协议）、影子 SDK、镜像、Git 兜底
- `references/review-findings.md` —— 一轮 UI review 实际会抓到的 15 类不符合项（可直接当自查 checklist）

## 本机环境备忘（换机器请自行调整）

- Android SDK `/opt/android-sdk`：原生 `aapt`/`aapt2` 是 **x86**（arm64 机器上 `bad machine`），
  能跑的是 `apksigner` / `zipalign` 脚本和手工编的 `/opt/android-sdk/aapt2-arm64/aapt2`（版本低，见 release 参考）。
- 工程常在 `/sdcard`（FUSE，小文件 IO 慢）；构建目录是否重定向由工程配置决定，**别擅自改**。
- 长构建容易被会话中断打断（本项目实测一次 7~10 分钟）。恢复流程：先 `ps` 看有没有 gradle 在跑，
  再**用 `aapt2 dump badging` 读产物里的版本号**判断上一轮跑到哪：debug 包已是新版本 = 编译过了、
  只差 release 打包；release 还是旧版本 = 没跑完，直接重跑（幂等）。最后核 `git status`，别把已提交的改动重做一遍。
- 读屏/截屏需要用户在 DSHA「设置 → 设备能力授权 → 设置屏幕操作」开启；没开时 `/app/ui/*` 一律返回
  「无障碍服务未开启」——不要反复重试，按"无法截图验证"如实报告。
- 本机是 **aarch64 容器 + 双命名空间**：aapt2 要经 `/system/bin/linker64` 启动、且 argv 与 **daemon stdin**
  都要做路径翻译；SDK 需要一个落在 `/sdcard` 的"影子 SDK"（`ANDROID_HOME` 指过去）。
  完整 shim、影子 SDK 组装、镜像 `init.gradle`、Git 推不上去时的 Git Data API 兜底，见
  `references/aarch64-container-toolchain.md`。
- 依赖坐标只要 `repo1.maven.org` 通就能拉（sources jar 首选）；整仓源码用 `codeload.github.com` 的 tarball 更快；
  `git push` 失败不等于网络全断——先分别探 `api.github.com` / `github.com` 再决定走哪条路。
