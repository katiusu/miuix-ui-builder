---
name: miuix-ui-builder
description: 在真实 Android 工程里构建 / 重构 Miuix（HyperOS 设计语言）Compose 界面：先锁定工程实际依赖的 Miuix 版本，再把每个 API 对着该版本的真实构件核验，然后用公开组件与 Defaults 实现，最后编译、lint、核对产物。适用于"用 Miuix 做个界面""照这张图改外观""加毛玻璃 / 液态玻璃""审查现有 Miuix 界面"，尤其是当 miuix 参考技能钉的版本与工程不一致、或涉及模糊/玻璃效果时。
---

# Miuix UI Builder

## 这个技能解决什么

`miuix` 技能是**库的参考手册**（组件目录 + 钉在某个版本的源码路径）。本技能是**干活的工作流**，
补上参考手册管不到的三件事：

1. **版本真相**——参考手册钉在 `v0.9.4`，而你的工程可能是 0.9.3 或别的版本；照手册写会写出编译不过的 API；
2. **核验纪律**——每个参数名、每个 `Defaults`、每个能力检测函数，都要对着**工程实际用的那个版本**的构件核验，不靠记忆、不靠别的库的命名习惯；
3. **验证诚实度**——编译通过不等于观感正确；没有渲染/设备证据就明说"未验证"，不要写成"已完成"。

写代码的落地细节仍然查 `miuix` 技能（组件目录、`Defaults`、Overlay/Window、preference 家族）。

## 铁律（违反任何一条都算失败）

1. **版本真相**：先读工程依赖，确认 Miuix 版本。参考手册版本 ≠ 工程版本时，**以工程为准**，并在报告里点明这个差异。不要混用别的版本的 API。
2. **API 必须对着真实构件核验**：参数名、默认值、重载、能力检测函数，一律从**该版本的 sources jar** 或字节码里核验（见
   `references/api-verification.md`）。签名记不准就 `javap`，别猜。
3. **Defaults 优先**：尺寸、圆角、阴影、颜色先取组件自带的 `XxxDefaults`；要偏离默认值，必须有明确理由（用户指定的视觉、或组件确实不提供该槽位）。
4. **不发明语义 token**：只用主题里真实存在的 color/textStyle。没有 `success`/`warning` 就不要编；需要的语义用真实 token 组合，或把语义交给文案 + 已有角色（如 `error`）。
5. **不手搓组件**：优先公开组件；只有库确实没有该交互时，才在 `MiuixTheme` 下用 Foundation/Layout 原语自组合，并在报告里声明这是自研件、它的语义/禁用态/无障碍由你负责。
6. **状态归属不变**：重构外观时不要顺手改状态所有者、导航、回调、insets 的所有权。要改就单独说明。
7. **验证分层**：编译 / lint / 产物 / 渲染 / 设备 是五种不同强度的证据，不能互相顶替。**没有设备或渲染证据时，视觉结论只能写"未验证"**。

## 工作流

### 0. 摸清工程（只读）

```bash
cat gradle/libs.versions.toml          # 版本目录：miuix / compose / agp
cat app/build.gradle.kts               # compileSdk / minSdk / 依赖 / lint 配置
cat gradle.properties                  # 编译相关的开关（compileSdk 校验、configuration cache…）
cat gradle/wrapper/gradle-wrapper.properties
find app/src/main -name "*.kt" | head -50
```

要确定的六件事：**Miuix 版本**、**AGP/Gradle/Kotlin/Compose 版本**、`minSdk`/`compileSdk`、
**主题与宿主**（`MiuixTheme` 在哪、`Scaffold` 是不是唯一宿主）、**insets 模型**（谁负责状态栏/导航栏内边距）、
**状态所有者**（哪些 state 提升到了外壳）。

### 1. 把版本锁成"可核验的事实"

```bash
# 从 Maven Central 拉工程实际用那个版本的源码（KMP 库的 Android 构件叫 <name>-android）
curl -sO https://repo1.maven.org/maven2/top/yukonga/miuix/kmp/miuix-ui-android/<版本>/miuix-ui-android-<版本>-sources.jar
mkdir -p /tmp/miuix-src && (cd /tmp/miuix-src && unzip -oq ../miuix-ui-android-<版本>-sources.jar)
```

源码拿到后就能直接读组件实现与 `Defaults`（比在线文档和在渲染站点可靠）。详见 `references/api-verification.md`。

### 2. 选组件、定形（查 miuix 技能）

按场景查 `miuix` 技能的 `component-selection` / `usage-patterns` / `design-language`，
挑**公开组件**，并把要用的参数在源码里确认一遍。涉及模糊、玻璃、squircle、图标时读该技能的
`styling-icons-and-effects`。

### 3. 设计（用一段话说清）

动手前用一段话写清：**层级**（页面 → 分区 → 容器 → 行）、**宿主**（用不用新 `Scaffold`/`MiuixTheme`）、
**状态归属**、**insets 谁让开**、**形状与颜色的来源**（哪个 Defaults / 哪个 token）。
说不清就是还没想清，别写代码。

### 4. 实现

- 最小改动、可回退；一次只改一个关注点（结构 / 观感 / 行为）；
- 每个新增的公开 API 调用都能指回源码里的那一行；
- 需要偏离 Defaults 时，把原因写进注释（为什么是 24dp 而不是默认的 16dp）。

### 5. 验证（按强度递增，做到哪一步就报到哪一步）

```bash
./gradlew :app:compileDebugKotlin     # 最快：类型/API 是否正确
./gradlew :app:assembleDebug          # 能装
./gradlew :app:assembleRelease        # R8/资源压缩后是否还成立（含反射/字符串引用的类是否被裁）
./gradlew :app:lintDebug              # 静态检查；先看基线再谈"新增"
```

产物核对不要只看"BUILD SUCCESSFUL"：

```bash
/opt/android-sdk/aapt2-arm64/aapt2 dump badging <apk> | grep -E "^package:|sdkVersion|targetSdk"
unzip -p <apk> classes.dex | strings | grep -c "<效果库特有的着色器字符串>"   # 确认效果代码没被 R8 裁掉
```

**渲染/设备证据**：有模拟器、预览、真机截图才算达到"观感已确认"。拿不到就写"未验证"，并说清缺什么、
怎么才能补上（例如"需要打开无障碍服务才能截图"）。

### 6. 交付报告

按下面四段写，不要多也不要少：

1. **改了什么**（文件 + 一句话职责）；
2. **依据**（哪条是源码契约、哪条是 Defaults、哪条是用户指定的视觉选择、哪条是自研件）；
3. **验证到哪一步**（编译 / lint / 产物 / 渲染 / 设备，逐项给结果，lint 要给"基线 vs 新增"）；
4. **未验证 / 已知取舍**（明确列出；不要用"应该没问题"糊过去）。

## 玻璃效果：先分清毛玻璃和液态玻璃

**把"模糊"叫成"液态玻璃"是常见错误。**两者是不同量级的东西：

| 想要的效果 | 需要的机制 | 常见实现 |
|---|---|---|
| **毛玻璃 / 磨砂**（半透明 + 模糊 + 一层底色） | 只对背景做高斯模糊 | 单次 `textureBlur` / `BlurEffect` / `RenderEffect` |
| **液态玻璃**（玻璃感） | 模糊 **+ 边缘折射/位移** + **高光描边** + 提饱和/色散 | 折射需要 `RuntimeShader`；还要 highlight（描边光）+ vibrancy |

给出结论时必须说清是哪一档。**只做模糊就写"悬浮毛玻璃底栏"，不要写"液态玻璃"。**
具体配方、两套可选库的能力对照与坑，见 `references/glass.md`。

## 玻璃/模糊的三个通用约束

1. **取样源不能包含被模糊物自身**。玻璃卡片如果从"含卡片自身"的图层取样，会拍到自己、出现拖影；
   正确做法是把背景单独录成一层给卡片用，另录一层"背景 + 内容"给顶栏/底栏用。
2. **能力分层降级**。模糊需要 `RenderEffect`（Android 12 / API 31），折射需要 `RuntimeShader`（API 33）。
   低于门槛时**不要只把颜色刷成透明**（会得到一块"什么都看不见的玻璃"），要退回不透明配色。
3. **模糊之上要叠一层容器色**。纯模糊在明暗主题、花哨壁纸下都会让文字/图标读不清；比较基准是
   "半透明容器色 + 模糊"，而不是裸模糊。

## 参考

- `references/api-verification.md` —— 怎么把任意版本的 Miuix / 效果库拉下来核验 API（含字节码手段）
- `references/glass.md` —— 毛玻璃 vs 液态玻璃的配方、可选库对照、性能与降级
- `references/android-release.md` —— 在这台机器上出包、签名、发 GitHub Release 的完整流程

## 本机环境备忘

- Android SDK：`/opt/android-sdk`；**x86 的 `aapt`/`apksigner` 脚本能跑，但原生 `aapt` 跑不了（arm64 机器）**，
  要用 `/opt/android-sdk/aapt2-arm64/aapt2`；`apksigner` / `zipalign` 脚本可用。
- 工程常放在 `/sdcard`（FUSE），小文件 IO 很慢；构建目录若被重定向到内部存储会明显变快（由工程的
  `settings.gradle.kts` / `gradle.properties` 决定，别擅自改）。
- 长构建容易被会话中断打断；产物时间戳要核对，不要拿旧产物当新结果。
- 设备能力（读屏 / 截屏）需要用户在 DSHA「设置 → 设备能力授权 → 设置屏幕操作」开启；
  没开时 `/app/ui/*` 一律返回「无障碍服务未开启」，此时不要反复重试，直接如实报告"无法截图验证"。
