# miuix-ui-builder

一个 **agent skill**：在真实 Android 工程里构建 / 重构 **Miuix（HyperOS 设计语言）Compose 界面**的工作流 + 真实失败清单。

它不是组件手册 —— 组件手册是 [`miuix`](https://github.com/limczhh/miuix-skill) 那个 skill 的职责。
它补齐的是参考手册管不到的七件事：

1. **版本真相**：手册钉在某个版本，工程可能用别的版本。照手册写会写出编译不过的 API。先锁定工程实际依赖的版本，并**以工程为准**。
2. **核验纪律**：每个参数名、每个 `Defaults`、每个能力检测函数，都要对着**该版本的真实构件**（sources jar / 字节码 / AAR 元数据 / `.module`）核验，不靠记忆、不靠别的库的命名习惯。
3. **组件契约**：组件内部的副作用（清自己的状态、改 `enabled`、装返回拦截）比它的签名更危险；把交互映射到它的回调之前，先读它实现里那几个 `LaunchedEffect` / `SideEffect`。
4. **验证诚实度**：编译 / lint / 产物 / 渲染 / 设备是五种不同强度的证据，不能互相顶替。没有渲染或设备证据时，视觉结论只能写"未验证"。
5. **宿主要求**（补充）：库会调用宿主（Activity / Window）提供的能力，宿主版本不够时**编译照样全绿、一进那个页面就崩**——所以"能编译"之后还要按页面真机验证。
6. **固定规格**（补充）：有些外观不是"看情况"，而是硬要求。目前有一条：**自带底部导航的外壳必须同时提供悬浮毛玻璃底栏与贴底普通底栏，并给用户开关**（理由与实现见 [`references/bottom-bars.md`](references/bottom-bars.md)）。
7. **现成骨架**（补充）：外壳、页面、配置项管道都有跑通过的参考实现（[`app-shell.md`](references/app-shell.md) / [`page-patterns.md`](references/page-patterns.md) / [`option-pipeline.md`](references/option-pipeline.md)），先抄骨架再改；**已经在用的工程要整体换骨架**的，按 [`retrofit-existing-app.md`](references/retrofit-existing-app.md) 的移植账本走。

## 加载标准（判定门槛已放宽）

一次 grep 就能判定工程用不用 Miuix：

```bash
grep -rn "yukonga" --include=build.gradle.kts --include=*.toml --include=*.kt .
```

判定为**用了 Miuix** 之后，**凡是做 UI 外观就必须加载本技能** —— 放宽之处是：
不再要求"点到了某个 Miuix 组件名"才算触发，只要是外观工作就算。

- 底栏 / 顶栏 / 导航 / 页面布局 / 卡片与列表的组织方式；
- 主题、配色、字号、圆角、间距、`ThemeController`；
- insets（状态栏 / 导航栏 / IME / edge-to-edge）；
- 模糊 / 毛玻璃 / 液态玻璃 / 任何视觉效果；
- 用户说"改外观""照这张图改""审查一下界面""加个开关/选项"。

构建 / 依赖 / `compileSdk`·`targetSdk` / lint / 出包 / 发版出问题也加载。
还没确定工程用不用 Miuix，就先跑上面那条 grep（几秒钟），别靠猜。
跟外观、构建无关的任务（纯后端、纯数据、纯文案）不加载。
理由很实际：**在 Miuix 工程里按 Material 或凭记忆写一遍再返工，代价远大于一次加载。**

## 安装

```bash
npx skills add katiusu/miuix-ui-builder
```

## 内容

| 文件 | 作用 |
|---|---|
| [`SKILL.md`](SKILL.md) | 入口：加载标准 + 11 条铁律 + 23 条真实失败清单 + 6 步工作流 + 交付报告格式 + 底栏规格 + 结构参考 |
| [`references/bottom-bars.md`](references/bottom-bars.md) | **底栏规格的完整实现**：贴底普通 + 悬浮毛玻璃 + 液态玻璃**三档**、开关怎么摆、落盘、backdrop 门控、内容让位、`RowScope` 坑、12 个实测坑 + 验收清单 |
| [`references/app-shell.md`](references/app-shell.md) | **应用外壳骨架**：单 Activity + 多页 pager + 三档底栏（悬浮 / 贴底 / 液态玻璃）+ 宽屏 rail、内容让位三条路径、主题与窗口背景、语言、桌面图标隐藏 |
| [`references/page-patterns.md`](references/page-patterns.md) | **页面结构范式**：三种页面形态、通用骨架（顶栏渐进模糊参数）、元素选择表、主页仪表盘宽窄两套布局、设置页与关于页的实测细节 |
| [`references/option-pipeline.md`](references/option-pipeline.md) | **声明式配置项管道**：`OptionSpec` 字段表 → 渲染分派 → 三条门控 → 默认值语义 → 三个输入对话框 → `AppSettings` 外壳级设置 |
| [`references/retrofit-existing-app.md`](references/retrofit-existing-app.md) | **把既有工程整体换成这套骨架**：移植账本（骨架件 / 管道件 / 示例业务件）、剥业务耦合与偏好键对齐、页面重写顺序、三个编译坑、构建期验收指标、代码之外的交付物 |
| [`references/component-contracts.md`](references/component-contracts.md) | 组件内部会动你的状态：Miuix `SearchBar`/`InputField` 回车清空关键词的完整归因，以及 10 分钟审计一个陌生组件的方法 |
| [`references/glass.md`](references/glass.md) | 毛玻璃 vs 液态玻璃的判定、两套库（miuix-blur / backdrop）配方、**换库迁移的六个坑**、性能与降级阶梯 |
| [`references/edge-to-edge.md`](references/edge-to-edge.md) | 全屏 + 系统栏 / IME 检查单、**系统栏图标跟随应用内主题**的坑（含字节码核验）、被工具链挡住时怎么报 |
| [`references/api-verification.md`](references/api-verification.md) | 把任意版本的依赖拉下来核验 API：sources jar、javap、AAR 元数据、Gradle `.module` 预判版本冲突 |
| [`references/android-release.md`](references/android-release.md) | 出包 → 核对产物（**release 包四件套**、"残包"的根因与修法）→ **工具链闸门** → 签名（含"能不能覆盖升级"、容器里没有 `zipalign` 怎么办）→ 推送 → 发 Release + 匿名复核 → **发版收尾清单** |
| [`references/runtime-host-requirements.md`](references/runtime-host-requirements.md) | **编译通过 ≠ 组合期不炸**：Miuix 组件内部的 `NavigationBackHandler` 需要 `activity ≥ 1.13.0`，以及从 logcat 栈 + `javap` 归因"页面/弹窗打不开"的通用方法 |
| [`references/aarch64-container-toolchain.md`](references/aarch64-container-toolchain.md) | aarch64 容器专属：**aapt2 双命名空间 shim**（argv + daemon stdin 都要翻译）、影子 SDK、镜像 `init.gradle`、老 aapt2 卡 `compileSdk`、Git 推不动时的 **Git Data API 兜底** |
| [`references/review-findings.md`](references/review-findings.md) | **15 类界面不符合项**（结构 / 颜色 / 状态 / 自适应）+ **6 类工程与交付不符合项**（残包、换签名、删 import、文档脱节…），可直接当自查 checklist |

## 十一条铁律（摘要）

1. **版本真相**：以工程实际依赖为准，版本不一致要在报告里点明。
2. **API 对着真实构件核验**，不猜签名。
3. **先读组件实现，再映射你的回调**。
4. **Defaults 优先**——但**用户明确给的视觉 > 库的 Defaults**，别拿"规范"压用户要的效果。
5. **不发明语义 token**（没有 `success`/`warning` 就不要编）。
6. **不手搓组件**；确实要自组合，就声明它是自研件，并自己负责语义 / 禁用态 / 无障碍。
7. **状态归属不变**：重构外观不顺手改状态所有者、导航、insets。
8. **验证分层**，没有渲染 / 设备证据就写"未验证"。
9. **不承诺做不到的前提**：外部 skill 给的前置条件（如 `targetSdk ≥ 35`）如果被工具链挡住，先验证可行性，再如实报告受阻项 + 证据，不要硬改配置把构建推倒。
10. **底栏至少两档**：自带底部导航的应用外壳，必须同时提供**悬浮毛玻璃底栏**与**贴底普通底栏** + 一个开关（用户明确不要才例外）；第三档**液态玻璃**按需求加，档位用两个布尔派生 `navBarMode`，别让用户拼出无意义组合。
11. **结构先抄骨架**：外壳 / 页面 / 配置项管道照已验证的参考写，不要重新发明结构。

## 毛玻璃 ≠ 液态玻璃

这是写这个 skill 的直接原因：

| 档位 | 需要的机制 |
|---|---|
| 毛玻璃 / 磨砂 | 一次高斯模糊（`RenderEffect` / `BlurEffect`） |
| **液态玻璃** | 模糊 **+ 边缘折射（`RuntimeShader`/AGSL）+ 高光描边 + 提饱和** |

**只做模糊就只能叫"悬浮毛玻璃"。** 把 blur 说成 liquid glass 是货不对板，用户一眼能看出来。
`glass.md` 里有从源码判断"一个库到底有没有真折射"的方法（搜 `lens(` / `refractionHeight` / `uniform shader content`），
以及取样源不能包含自身、模糊之上要叠容器色、API 31/33 的降级阶梯这三条通用约束。

## 这个 skill 怎么长出来的

不是凭空写的，是几次真实返工攒出来的（SKILL.md 里的"常见失败"表逐条对应）：

- 把只做模糊的底栏写成"液态玻璃" → 用户当场指出"这只是毛玻璃"；
- 把"回车"映射成组件的"收起搜索"，没读实现 → 关键词被组件自己清掉；
- 按"库 Defaults 优先"把圆角改成库默认值 → 偏离了用户给的参考图；
- 升级效果库前没读 Gradle `.module` → 新版本要求更高 AGP，构建直接失败；
- 承诺 `compileSdk 35` 却没查 aapt2 → 资源链接阶段整段失败。

第二作者的补充（来自另一个真实工程：HyperOS Autofill Fix 2.0.0 的界面重构）：

- 三个页面里两个"打不开"，编译全绿 → 真机 logcat 才看到
  `No NavigationEventDispatcher was provided`，根因是 Miuix 0.9.3 的 `SearchBar` / 弹窗内部调用
  `NavigationBackHandler`，而 `activity` 1.9.3 的 `ComponentActivity` 没有实现对应 owner（1.13.0 才实现）；
- 升级 `activity` 到 1.13.0 后仍有版本漂移：Compose 1.11 移除了 `PagerState.animateToPage`
  （改 `animateScrollToPage`），说明"版本真相"要连着 Compose 一起核；
- Miuix 0.9.x 的 AAR 元数据声明 `minCompileSdk=37`，而本机 aapt2 只吃到 API 34 →
  用 `android.experimental.disableCompileSdkChecks=true` 顶住构建，并把升级路径写进 README；
- 收尾按 `ui-review-workflow` 自查了 4 轮，抓出 14 处不符合项（含"每条记录一张 Card"
  这种设计语言点名的失败做法、用 `error` 红表达"正常工作"的指标），清单收进
  [`references/review-findings.md`](references/review-findings.md)。

第三作者的补充（HyperOS Autofill Fix 2.2.0：底栏做成"两种可切换"）：

- 需求是"两种底栏都要、但要能切"，于是把它固化成**硬规格**（铁律 10）；普通底栏的实现
  **从旧 release 里 `git show <tag>:<path>` 取原始代码**当基准，而不是凭记忆重写 →
  [`references/bottom-bars.md`](references/bottom-bars.md)；
- 实测坑：`NavigationBarItem` 是 **`RowScope` 的扩展**（0.9.3 `basic/NavigationBar.kt`），
  只能写在 `NavigationBar` 的 content 里；两种底栏组件都自己吃导航栏 inset，
  **别再给父容器加 padding**（否则双重让开）；
- 查"两种底栏是否都进了包"时踩了 **multi-dex 假阴性**：debug 包有 `classes..classes6.dex`，
  只搜 `classes.dex` 得到 0，遍历所有 dex 才看到符号 → 收进 `android-release.md`；
- 长构建（7~10 分钟）被会话重启打断了一次，恢复流程（`ps` → 用 `aapt2 dump badging` 读产物里的
  版本号判断上一轮跑到哪 → 只补差的那一步）收进 `android-release.md`；
- 交付报告新增一段 **「我替你定的默认」**（默认值 / 开关位置 / 共用让位逻辑这类代用户做的决定
  要单列出来），否则用户得翻代码才知道怎么改。

第四作者的补充（MiuixGuiExample：把一个 Xposed 模板剥离 Hook 后的纯 GUI 工程）：

- 该工程把"外壳 / 页面 / 配置项"三层写得很规整，于是**按结构而非按内容**蒸成三份新参考：
  [`references/app-shell.md`](references/app-shell.md)、[`references/page-patterns.md`](references/page-patterns.md)、
  [`references/option-pipeline.md`](references/option-pipeline.md)（各自的骨架与实测坑见文件内）；
- 结构层的坑按"事故"记：`WindowDropdownPreference` 不传 `onExpandedChange` 会让展开态与选中态不同步；
  Haze 顶栏的 `backgroundColor` 透明会让卡片硬边缘透出来；关于页的 `logoSpacer` 那个 `key` 就是折叠进度的分母；
- 迁移经验：**"提取 GUI"最省事的做法是保留同名 object 桩**——把 Hook 侧的 `XposedServiceManager` /
  `HookStatusStore` 换成本地实现（同名同签名），界面代码一行不改也能跑起来；
- 这份工程还顺带证明了"外壳三档底栏 + 宽屏 rail + 三条内容让位路径"在真机上能同时成立。

## 第五作者的补充（HyperOS Autofill Fix 2.3.0：照示例骨架整体重构 + 出正式版）

第四作者蒸出了骨架，第五作者把**一个已经在用、已经在发版的工程**整体搬了上去（12 个 `.kt` / 1481 行 →
60 文件 / +6380 行），顺手把工具链升到 Miuix 0.9.4 / AGP 9.4.1 / Gradle 9.7.1 / compileSdk 37，并发布了 2.3.0。攒下来的经验：

- **移植账本**（拷什么 / 不拷什么 / 偏好键怎么对齐 / 页面重写顺序 / 三个编译坑 / 验收指标）单独成篇 →
  [`references/retrofit-existing-app.md`](references/retrofit-existing-app.md)；
- **最贵的一次事故**：release 包一路 `BUILD SUCCESSFUL`，其实**没有 `AndroidManifest.xml` 和 `resources.arsc`**，
  根本装不上；根因是 `optimizeReleaseResources`（aapt2 `optimize`）在容器 aapt2 override 下"成功"却产出 0 个文件，
  修法是 `android.enableResourceOptimizations=false` + `clean` 重出包 → 收进 `android-release.md`，
  并把"四件套核验"写进日常验证；
- **底栏从两档扩到三档**（加了自绘的 iOS 液态玻璃）：档位用
  `navBarMode = if (!isFloatingNavbar) 0 else if (!isLiquidGlass) 1 else 2` 派生，开关用"悬浮"+"液态玻璃"
  两个布尔互相包含，宽屏 rail 只在贴底档替换底栏 → `bottom-bars.md` §2.5；
- **概览页的"分开"需求**（用户要状态块 / 服务名块 / 按钮各自独立）：窄屏用 `weight(1f).aspectRatio(1f)`
  做两个正方形块 + 按钮单独成卡，宽屏三卡等分 → `page-patterns.md` §6.5；
- **发版不只是推代码**：CHANGELOG + annotated tag + GitHub Release（正文含签名证书与 APK 的 SHA-256）+
  回下载复核 + README 对着代码改 + topics 补齐 → `android-release.md` 的"发版收尾清单"；
- 两个真实教训：`pkill -f GradleDaemon` 会**把调用它的 shell 一起杀掉**（用 `pkill -f 'Gradle[D]aemon'`）；
  按"正文里出现简单名"批量删 import 会把委托属性 `getValue` / `setValue` 一起删掉，要到编译期才报错。

## 开源协议

[Apache License 2.0](LICENSE)。

选它的原因：本 skill 的 [`references/edge-to-edge.md`](references/edge-to-edge.md) 是对 Google 官方
[`android/skills`](https://github.com/android/skills)（Apache-2.0）中 `edge-to-edge` skill 检查单的改写与补充，
用同一协议最干净；同时它带明确的专利授权，适合被工具链/agent 广泛消费。

见 [`NOTICE`](NOTICE)：其中说明了对 android/skills 的改编，以及本 skill **只是引用、并未打包**的第三方项目
（Miuix、AndroidLiquidGlass 等，各自保留其许可证）。

## 注意

- **SKILL.md 只放可移植的内容**。机器相关的经验（Android SDK 路径、aarch64 上的 aapt2、依赖镜像、DSHA 开关等）
  统一放 [`references/aarch64-container-toolchain.md`](references/aarch64-container-toolchain.md)，
  不要再往 SKILL.md 里加"本机环境备忘"（那种写法已被删除：换机器的人会照着不存在的路径去调）。
- `references/` 里凡带具体路径或魔法数的，都写了它来自哪个工程、哪个版本；换版本前先按
  [`references/api-verification.md`](references/api-verification.md) 重新核验，别直接照搬数字。
