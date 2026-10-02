# miuix-ui-builder

一个 **agent skill**：在真实 Android 工程里构建 / 重构 **Miuix（HyperOS 设计语言）Compose 界面**的工作流 + 真实失败清单。

它不是组件手册 —— 组件手册是 [`miuix`](https://github.com/limczhh/miuix-skill) 那个 skill 的职责。
它补齐的是参考手册管不到的六件事：

1. **版本真相**：手册钉在某个版本，工程可能用别的版本。照手册写会写出编译不过的 API。先锁定工程实际依赖的版本，并**以工程为准**。
2. **核验纪律**：每个参数名、每个 `Defaults`、每个能力检测函数，都要对着**该版本的真实构件**（sources jar / 字节码 / AAR 元数据 / `.module`）核验，不靠记忆、不靠别的库的命名习惯。
3. **组件契约**：组件内部的副作用（清自己的状态、改 `enabled`、装返回拦截）比它的签名更危险；把交互映射到它的回调之前，先读它实现里那几个 `LaunchedEffect` / `SideEffect`。
4. **验证诚实度**：编译 / lint / 产物 / 渲染 / 设备是五种不同强度的证据，不能互相顶替。没有渲染或设备证据时，视觉结论只能写"未验证"。
5. **宿主要求**（补充）：库会调用宿主（Activity / Window）提供的能力，宿主版本不够时**编译照样全绿、一进那个页面就崩**——所以"能编译"之后还要按页面真机验证。
6. **固定规格**（补充）：有些外观不是"看情况"，而是硬要求。目前有一条：**自带底部导航的外壳必须同时提供悬浮毛玻璃底栏与贴底普通底栏，并给用户开关**（理由与实现见 [`references/bottom-bars.md`](references/bottom-bars.md)）。

## 加载标准（已放宽）

**只要沾到 Android 界面或构建，就先加载，别先花时间确认"这是不是 Miuix 工程"** —— 本技能第 0 步自己会判断工程用的是哪套组件、哪个版本：

- 工程里有 Compose 界面代码（`*Screen.kt` / `@Composable` / `androidx.compose.*`）；
- 要动底栏 / 顶栏 / 导航 / 主题 / 页面布局；
- 要动 insets（状态栏 / 导航栏 / IME / edge-to-edge）；
- 要动模糊 / 毛玻璃 / 液态玻璃 / 任何视觉效果；
- 构建、依赖、`compileSdk`/`targetSdk`、lint、出包、发版出问题；
- 用户说"改外观""照这张图改""审查一下界面""加个开关/选项"。

唯一不该加载的是和 Android 界面/构建完全无关的任务（纯后端、纯数据处理、纯文案）。
理由很实际：**按错组件版本写一遍再返工的代价，远大于一次加载。**

## 安装

```bash
npx skills add katiusu/miuix-ui-builder
```

## 内容

| 文件 | 作用 |
|---|---|
| [`SKILL.md`](SKILL.md) | 入口：加载标准 + 10 条铁律 + 13 条真实失败清单 + 6 步工作流 + 交付报告格式 + 底栏规格 |
| [`references/bottom-bars.md`](references/bottom-bars.md) | **底栏规格的完整实现**：悬浮 + 贴底两种形态、开关、落盘、backdrop 门控、内容让位、`RowScope` 坑、8 个实测坑 + 验收清单 |
| [`references/component-contracts.md`](references/component-contracts.md) | 组件内部会动你的状态：Miuix `SearchBar`/`InputField` 回车清空关键词的完整归因，以及 10 分钟审计一个陌生组件的方法 |
| [`references/glass.md`](references/glass.md) | 毛玻璃 vs 液态玻璃的判定、两套库（miuix-blur / backdrop）配方、**换库迁移的六个坑**、性能与降级阶梯 |
| [`references/edge-to-edge.md`](references/edge-to-edge.md) | 全屏 + 系统栏 / IME 检查单、**系统栏图标跟随应用内主题**的坑（含字节码核验）、被工具链挡住时怎么报 |
| [`references/api-verification.md`](references/api-verification.md) | 把任意版本的依赖拉下来核验 API：sources jar、javap、AAR 元数据、Gradle `.module` 预判版本冲突 |
| [`references/android-release.md`](references/android-release.md) | 出包 → 核对产物 → **工具链闸门** → 签名（含"能不能覆盖升级"）→ 推送 → 发 Release + 匿名复核 |
| [`references/runtime-host-requirements.md`](references/runtime-host-requirements.md) | **编译通过 ≠ 组合期不炸**：Miuix 组件内部的 `NavigationBackHandler` 需要 `activity ≥ 1.13.0`，以及从 logcat 栈 + `javap` 归因"页面/弹窗打不开"的通用方法 |
| [`references/aarch64-container-toolchain.md`](references/aarch64-container-toolchain.md) | aarch64 容器专属：**aapt2 双命名空间 shim**（argv + daemon stdin 都要翻译）、影子 SDK、镜像 `init.gradle`、老 aapt2 卡 `compileSdk`、Git 推不动时的 **Git Data API 兜底** |
| [`references/review-findings.md`](references/review-findings.md) | 一轮 UI review 实际抓到的 **15 类不符合项**（结构 / 颜色 / 状态 / 自适应），可直接当自查 checklist |

## 十条铁律（摘要）

1. **版本真相**：以工程实际依赖为准，版本不一致要在报告里点明。
2. **API 对着真实构件核验**，不猜签名。
3. **先读组件实现，再映射你的回调**。
4. **Defaults 优先**——但**用户明确给的视觉 > 库的 Defaults**，别拿"规范"压用户要的效果。
5. **不发明语义 token**（没有 `success`/`warning` 就不要编）。
6. **不手搓组件**；确实要自组合，就声明它是自研件，并自己负责语义 / 禁用态 / 无障碍。
7. **状态归属不变**：重构外观不顺手改状态所有者、导航、insets。
8. **验证分层**，没有渲染 / 设备证据就写"未验证"。
9. **不承诺做不到的前提**：外部 skill 给的前置条件（如 `targetSdk ≥ 35`）如果被工具链挡住，先验证可行性，再如实报告受阻项 + 证据，不要硬改配置把构建推倒。
10. **底栏两种形态**：自带底部导航的应用外壳，必须同时提供**悬浮毛玻璃底栏**与**贴底普通底栏** + 一个开关（用户明确不要才例外）。

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
  版本号判断上一轮跑到哪 → 只补差的那一步）收进 `android-release.md` 与 SKILL.md 的环境备忘；
- 交付报告新增一段 **「我替你定的默认」**（默认值 / 开关位置 / 共用让位逻辑这类代用户做的决定
  要单列出来），否则用户得翻代码才知道怎么改。

## 开源协议

[Apache License 2.0](LICENSE)。

选它的原因：本 skill 的 [`references/edge-to-edge.md`](references/edge-to-edge.md) 是对 Google 官方
[`android/skills`](https://github.com/android/skills)（Apache-2.0）中 `edge-to-edge` skill 检查单的改写与补充，
用同一协议最干净；同时它带明确的专利授权，适合被工具链/agent 广泛消费。

见 [`NOTICE`](NOTICE)：其中说明了对 android/skills 的改编，以及本 skill **只是引用、并未打包**的第三方项目
（Miuix、AndroidLiquidGlass 等，各自保留其许可证）。

## 注意

- SKILL.md 末尾的「本机环境备忘」（Android SDK 路径、arm64 上的 aapt2、DSHA 无障碍开关）是**作者机器相关**的部分，
  换机器请按自己的环境调整，其余内容是通用的。
