# 毛玻璃 与 液态玻璃

## 先分清档位（这是最容易含糊的地方）

| 档位 | 视觉特征 | 需要的机制 | 能不能叫"液态玻璃" |
|---|---|---|---|
| 半透明 | 一层半透明底色 | alpha | ✗ |
| **毛玻璃 / 磨砂** | 背景被模糊，能透出后面的形状 | 一次高斯模糊（`RenderEffect` / `BlurEffect`） | ✗ |
| **液态玻璃** | 上面全部 **+ 边缘把背景"掰弯"（折射）+ 一圈高光描边 + 提饱和（+可选色散）** | 模糊 **+ `RuntimeShader`(AGSL) 折射 + highlight 描边** | ✓ |

**只做模糊就老实叫"悬浮毛玻璃"。**把 blur 说成 liquid glass 是实打实的货不对板，
用户一眼能看出来（他们会说"这只是毛玻璃"）。

判断库存不存在"真折射"的方法：搜源码里有没有 AGSL / RuntimeShader 参与**边缘位移**，例如
`lens(`、`refractionHeight`、`refractionAmount`、`cornerRadii`、`uniform shader content`。
只有 `blur` 相关字样的，就是毛玻璃。

## 方案 A：miuix-blur（Miuix 官方，只有毛玻璃）

API（0.9.3 / 0.9.4 同形，均已核验）：

```kotlin
val backdrop = rememberLayerBackdrop {        // 录内容；onDraw 里先铺不透明底，避免模糊把颜色扩散进透明区
    drawRect(scheme.surface)
    drawContent()
}
Box(Modifier.fillMaxSize().layerBackdrop(backdrop)) { /* 页面内容 */ }

FloatingNavigationBar(
    modifier = Modifier.textureBlur(          // 形状必须和底栏圆角完全一致，否则玻璃边缘错位
        backdrop = backdrop,
        shape = RoundedCornerShape(FloatingToolbarDefaults.CornerRadius),
        blurRadius = BlurDefaults.BlurRadius,
        colors = BlurDefaults.blurColors(
            blendColors = listOf(BlendColorEntry(color = scheme.surface.copy(alpha = 0.6f))),
        ),
    ),
    color = Color.Transparent,                // 让玻璃透出
    cornerRadius = FloatingToolbarDefaults.CornerRadius,
)
```

关键事实：

- **整个库的地板是 API 33**（`isRuntimeShaderSupported()`）——连 blend/noise 都走 RuntimeShader；
- AAR 声明 `minSdk 33`，工程 `minSdk` 更低时要在 manifest 写 `tools:overrideLibrary="top.yukonga.miuix.kmp.blur"`，
  并在代码里 `if (isRuntimeShaderSupported())` 决定挂不挂 `textureBlur`；**低于门槛时不要只把 color 置透明**，
  要退回 `surfaceContainer` 之类的不透明配色；
- `FloatingNavigationBar` 内部顺序是 `squircleBackground → 调用方的 modifier → padding → 内容`，
  所以传进去的 modifier 正好画在背景之上、图标之下；换成别的组件前先在源码里确认这个顺序；
- 还有 `progressiveTextureBlur`（边缘渐变模糊），适合窄边条，但比均匀模糊更贵。

## 方案 B：backdrop / AndroidLiquidGlass（Kyant0，**才够得上液态玻璃**）

坐标 `io.github.kyant0:backdrop-android`（库名叫 Backdrop，仓库名 AndroidLiquidGlass）。
**它不提供任何高层组件，胶囊/卡片/按钮都得自己拼。**

```kotlin
Modifier.drawBackdrop(
    backdrop = backdrop,
    shape = { RoundedCornerShape(cornerRadius) },   // 必须是 Compose Shape
    effects = {
        vibrancy()                                   // 提饱和（ColorFilter 链）
        blur(14.dp.toPx())                           // BackdropEffectScope 本身是 Density，直接 toPx()
        lens(20.dp.toPx(), 20.dp.toPx())             // ← 真正的折射，需要 API 33
    },
    highlight = { Highlight.Default },               // 边缘高光，默认就有
    shadow = { Shadow.Default },                     // 外阴影；不要和组件自带阴影叠成两层
    onDrawSurface = { drawRect(tint) },              // 模糊之上、内容之下的容器色
)
```

关键事实（均已核验 2.0.0 / 2.0.1）：

- `isRenderEffectSupported()` = API 31（blur/vibrancy 可用），`isRuntimeShaderSupported()` = API 33（lens 可用）；
  两个函数**库内部已自守卫**，但颜色降级要你自己做；
- `lens()` 只认得三类 shape（`CornerBasedShape` / `AbsoluteRoundedCornerShape` / kyant `RoundedRectangularShape`）；
  传别的 shape 会直接 `throw`；
- `rememberLayerBackdrop(graphicsLayer, onDraw)` 内部是 `remember(graphicsLayer, onDraw)`——
  **传内联 lambda 会导致每次重组都重建 backdrop**。要么用 `remember { ... }` 固定 lambda，要么接受重建；
- 没有 `lens` 就只是"模糊 + 高光"，仍然接近毛玻璃；**折射才是液态玻璃的分水岭**；
- 版本坑：`2.0.1` 依赖 Compose 1.12.0 → 要求 AGP ≥ 9.1；`2.0.0` 依赖 Compose 1.11.0。
  先看 `.module` 再决定用哪个版本（见 `api-verification.md`）。

## 通用：取样源不能包含自己

玻璃卡片如果从"含卡片自身"的图层取样，会拍到自己 → 拖影/递归糊。
正确结构是**两层 backdrop**：

```
背景层（纯色 + 装饰）──► backgroundBackdrop ──► 给玻璃卡片/卡片类表面取样
内容层（背景 + 页面内容）──► contentBackdrop ──► 给顶栏 / 悬浮底栏取样（要看到滚动内容穿过）
```

用 Miuix 的组件时，`bottomBar` / `topBar` 是画在 body 之上的兄弟节点，天然满足这个结构。

## 通用：模糊之上叠色

裸模糊在浅色主题 + 白底、深色主题 + 黑底、或者花哨壁纸上都会让文字读不清。
做法是模糊之上叠一层 `surfaceContainer`/`surface` 的半透明色（经验区间 alpha 0.5~0.7），
图标/文字的颜色用主题的 `on*` 角色，而不是写死黑白。

## 性能

- 每个玻璃面 = 一次模糊（+ 一次折射着色器 + 一次阴影）。**卡片数量要控制**，长列表的每一行不要做玻璃；
- 模糊半径越大越贵；`lens` 比 `blur` 更贵；
- 低端机 / 大列表上优先保证滚动流畅：宁可少几个玻璃面，也不要整页玻璃。

## 降级阶梯（照抄这个结构）

```
API ≥ 33  → 模糊 + 折射 + 高光 + 容器色   （液态玻璃）
API 31-32 → 模糊 + 容器色                 （毛玻璃）
API < 31  → 不挂任何效果 + 不透明配色      （普通表面）
```
