# 把任意版本的依赖拉下来核验 API

目标：**每一行 Miuix / 效果库的调用，都能指回该版本源码里的那一行。** 不靠记忆，不靠别的库的命名习惯，
不靠在线渲染文档（它通常代表比工程更新的提交）。

## 1. 先确定坐标与版本

```bash
cat gradle/libs.versions.toml
```

Compose Multiplatform 的库在 Maven Central 上是**两个坐标**：

| 坐标 | 内容 |
|---|---|
| `group:name` | KMP 根模块，通常只有一个 POM/JAR + `available-at` 重定向 |
| `group:name-android` | Android 实际构件（`.aar` + `.module`），**核验时用这个** |

注意：**`-android` 构件不一定每个版本都存在**（例如某些库 1.x 只发根坐标，2.0.0-alpha01 起才有 `-android`）。
先列目录确认：

```bash
curl -s "https://repo1.maven.org/maven2/io/github/kyant0/backdrop-android/maven-metadata.xml" | head -30
```

## 2. 拉源码（首选）

```bash
V=0.9.3
G=top/yukonga/miuix/kmp
A=miuix-ui-android
curl -sO "https://repo1.maven.org/maven2/$G/$A/$V/$A-$V-sources.jar"
mkdir -p /tmp/lib-src && (cd /tmp/lib-src && unzip -oq ~/$A-$V-sources.jar)
grep -rn "fun FloatingNavigationBar(" /tmp/lib-src | head
```

源码里能看到**参数名、默认值、中文/英文注释、`Defaults` 对象的真实数值**——这是最可靠的一层。

几个常用动作：

```bash
# 找组件定义与重载
grep -rn "fun <Component>(" /tmp/lib-src

# 看 Defaults 的真实数值（不要凭感觉编尺寸）
grep -rn "object .*Defaults" -A 30 /tmp/lib-src | grep -E "CornerRadius|ShadowElevation|IconSize"

# 看组件内部怎么用调用方传入的 modifier（决定你的 modifier 会被画在哪一层）
sed -n '280,370p' /tmp/lib-src/**/NavigationBar.kt
```

## 3. 只有字节码时用 javap

```bash
unzip -o -q lib.aar classes.jar -d /tmp/lib-jar
javap -classpath /tmp/lib-jar/classes.jar top.yukonga.miuix.kmp.basic.NavigationBarKt | grep -i floating
```

`javap` 看得到签名与重载，**看不到参数名和默认值**——所以能用源码就用源码。

## 4. 能力检测函数

```bash
grep -rn "expect fun is.*Supported\|actual fun is.*Supported" /tmp/lib-src
# 看 Android 实现到底卡在哪个 API：
grep -rn "SDK_INT" /tmp/lib-src/**/androidMain/**/*.kt
```

例：`isRenderEffectSupported()` = `SDK_INT >= S`(31)，`isRuntimeShaderSupported()` = `SDK_INT >= TIRAMISU`(33)。

## 5. 读 AAR 元数据（决定能不能编译进去）

```bash
unzip -p lib.aar META-INF/com/android/build/gradle/aar-metadata.properties
unzip -p lib.aar AndroidManifest.xml | head
```

- `minCompileSdk=37` + 工程 `compileSdk=34` → 需要 `android.experimental.disableCompileSdkChecks=true`；
- `minAndroidGradlePluginVersion` → 与工程 AGP 对不上就会在 `checkDebugAarMetadata` 失败；
- AAR 的 `uses-sdk minSdkVersion=33` + 工程 `minSdk=26` → 需要 `tools:overrideLibrary="<包名>"`，
  **并且要在代码里做能力检测**（见 `glass.md` 的降级阶梯）。

## 6. 读 `.module`（Gradle 元数据）——用它预判版本冲突

这是最容易踩、也最容易提前发现的一类问题：**一个效果库会把 Compose 抬到某个版本，而那个版本可能要求更新的 AGP。**

```bash
curl -s "https://repo1.maven.org/maven2/io/github/kyant0/backdrop-android/2.0.1/backdrop-android-2.0.1.module" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(v['name'], [ (x['group'],x['module'],x['version']['requires']) for x in v.get('dependencies',[])]) for v in d['variants']]"
```

真实例子：`backdrop-android:2.0.1` 依赖 `org.jetbrains.compose.foundation:foundation:1.12.0`
→ 解析成 `androidx.compose.foundation:foundation:1.12.0`
→ 该 AAR 元数据要求 **AGP ≥ 9.1.0**，而工程是 8.9.2，会在 `checkDebugAarMetadata` 直接失败。
同一个库的 `2.0.0` 依赖 Compose 1.11.0（低于工程已有的 1.11.1），因此**降到 2.0.0 就绕开了**——
而且这两个版本的库源码完全一致（`diff -rq` 为空），只差 Compose 依赖。

```bash
# 确认"降级是否安全"：直接比两个版本的源码树
diff -rq /tmp/v200 /tmp/v201 | head
```

## 7. 看工程最终解析出什么

```bash
./gradlew :app:dependencies --configuration debugRuntimeClasspath | grep -E "compose.(foundation|ui):|miuix|backdrop"
```

改依赖前先记下这个输出，改完再比一次：**要能说出"抬了哪个版本、为什么"。**

## 8. 不要被这些红字吓到

- `Module was compiled with an incompatible version of Kotlin … expected 2.1.0`：R8/分析阶段读 Kotlin 元数据的噪音，构建仍然成功；
- `Unable to strip the following libraries`：没有对应架构的 strip 工具，正常；
- `aapt2 daemon failed to shutdown`：收尾超时，不影响产物。

**真正的失败**是 `BUILD FAILED` + `What went wrong` 里那一段。永远看那一段，别只看 tail。
