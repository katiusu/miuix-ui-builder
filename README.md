# miuix-ui-builder

一个 **agent skill**：在真实 Android 工程里构建 / 重构 **Miuix（HyperOS 设计语言）Compose 界面**的工作流。

它不是组件手册 —— 组件手册是 [`miuix`](https://github.com/limczhh/miuix-skill) 那个 skill 的职责。
它补齐的是参考手册管不到的三件事：

1. **版本真相**：手册钉在某个版本，工程可能用别的版本。照手册写会写出编译不过的 API。本 skill 要求先锁定工程实际依赖的版本，并**以工程为准**。
2. **核验纪律**：每个参数名、每个 `Defaults`、每个能力检测函数，都要对着**该版本的真实构件**（sources jar / 字节码 / AAR 元数据 / `.module`）核验，不靠记忆、不靠别的库的命名习惯。
3. **验证诚实度**：编译 / lint / 产物 / 渲染 / 设备是五种不同强度的证据，不能互相顶替。没有渲染或设备证据时，视觉结论只能写"未验证"。

## 安装

```bash
npx skills add katiusu/miuix-ui-builder
```

## 内容

| 文件 | 作用 |
|---|---|
| [`SKILL.md`](SKILL.md) | 入口：7 条铁律 + 6 步工作流 + 交付报告格式 + 玻璃效果判定 |
| [`references/glass.md`](references/glass.md) | 毛玻璃 vs 液态玻璃的判定标准、两套库（miuix-blur / backdrop）的配方与坑、性能与降级阶梯 |
| [`references/api-verification.md`](references/api-verification.md) | 把任意版本的依赖拉下来核验 API：sources jar、javap、AAR 元数据、Gradle `.module` 预判版本冲突 |
| [`references/android-release.md`](references/android-release.md) | 出包 → 核对产物 → 签名（含"能不能覆盖升级"的判定）→ 推送 → 发 GitHub Release + 匿名复核 |

## 七条铁律

1. **版本真相**：以工程实际依赖为准，版本不一致要在报告里点明。
2. **API 对着真实构件核验**，不猜签名。
3. **Defaults 优先**，偏离默认值必须有理由。
4. **不发明语义 token**（没有 `success`/`warning` 就不要编）。
5. **不手搓组件**；确实要自组合，就声明它是自研件，并自己负责语义 / 禁用态 / 无障碍。
6. **状态归属不变**：重构外观不顺手改状态所有者、导航、insets。
7. **验证分层**，没有渲染/设备证据就写"未验证"。

## 毛玻璃 ≠ 液态玻璃

这是写这个 skill 的直接原因：

| 档位 | 需要的机制 |
|---|---|
| 毛玻璃 / 磨砂 | 一次高斯模糊（`RenderEffect` / `BlurEffect`） |
| **液态玻璃** | 模糊 **+ 边缘折射（`RuntimeShader`/AGSL）+ 高光描边 + 提饱和** |

**只做模糊就只能叫"悬浮毛玻璃"。** 把 blur 说成 liquid glass 是货不对板，用户一眼能看出来。
`glass.md` 里有从源码判断"一个库到底有没有真折射"的方法（搜 `lens(` / `refractionHeight` / `uniform shader content`），
以及取样源不能包含自身、模糊之上要叠容器色、API 31/33 的降级阶梯这三条通用约束。

## 注意

- SKILL.md 末尾的「本机环境备忘」（Android SDK 路径、arm64 上的 aapt2 用法、DSHA 无障碍开关）是**作者机器相关**的部分，换机器请按自己的环境调整，其余内容是通用的。
- 仓库暂未附加开源协议；如需分发请先加一个 LICENSE。
