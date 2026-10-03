# 出包、签名、发 Release（本机实操）

## 1. 构建

```bash
cd <工程根>
ANDROID_HOME=/opt/android-sdk ANDROID_SDK_ROOT=/opt/android-sdk \
  ./gradlew :app:assembleDebug :app:assembleRelease
```

- 工程若开了 `org.gradle.configuration-cache=true`，第一次仍会跑完整配置阶段，第二次才快；
- **不要同时跑两条 Gradle 命令**：wrapper 对发行版目录有独占锁，第二条会以
  `ExclusiveFileAccessManager.access` 失败告终。串行跑，或者用 `--continue` 合成一条。

## 2. 核对产物（不要只看 BUILD SUCCESSFUL）

```bash
A=/opt/android-sdk/aapt2-arm64/aapt2          # 注意：build-tools 里的 aapt/aapt2 是 x86，跑不了
$A dump badging app/build/outputs/apk/release/app-release-unsigned.apk | grep -E "^package:|sdkVersion|targetSdk"

# 确认效果库的着色器没被 R8 裁掉（AGSL 源码字符串是稳定标记）
unzip -p <apk> classes.dex | strings | grep -c "half4 main"
unzip -p <apk> classes.dex | strings | grep -c "refractionHeight"   # 换成你要找的库的特征串

# 确认 Xposed 入口类没被改名
unzip -p <apk> assets/xposed_init
```

### release 包必须"有内容"（四件套）

`BUILD SUCCESSFUL` 不等于包能装。这四样缺一不可：`AndroidManifest.xml`、`resources.arsc`、
`classes*.dex`，以及 `aapt2 dump badging` 能读出 `package:`（若它回你一句
`could not identify format of APK.`，说明手上是个残包）：

```bash
unzip -l app/build/outputs/apk/release/app-release-unsigned.apk | grep -E "AndroidManifest.xml|resources.arsc|classes.*\.dex"
/opt/android-sdk/build-tools/36.0.0/aapt2 dump badging app/build/outputs/apk/release/app-release-unsigned.apk | head -3
```

### `BUILD SUCCESSFUL` 但包是残的：`optimizeReleaseResources` 静默产出 0 个文件

**症状**（实测：AGP 9.4.1 + `android.aapt2FromMavenOverride` 指向 build-tools 的 aapt2）：
release APK 只有 1.49 MB / 80 条目，**既没有 `AndroidManifest.xml` 也没有 `resources.arsc`**，装不上；
`aapt2 dump badging` 报 `could not identify format of APK.`；
而 Gradle 全程 `BUILD SUCCESSFUL`、一句警告都没有。`--no-build-cache` 也修不掉
（构建缓存会把这份空输出一起缓存下来）。

**定位**（顺着 `intermediates/` 查哪个任务把内容丢了）：

```bash
# 1) 资源链接产物：应该含 manifest（proto 格式，没有 arsc 是正常的）
unzip -l app/build/intermediates/linked_resources_proto_format/release/*/linked-resources-proto-format-release.ap_
# 2) 二进制资源包：manifest + arsc 都该在（好包）
unzip -l app/build/intermediates/shrunk_resources_binary_format/release/*/shrunk-resources-binary-format-release.ap_
# 3) 资源优化任务的输出目录：metadata 里声明了 resources-release-optimize.ap_，目录里却只有 metadata 本身
ls -la app/build/intermediates/optimized_processed_res/release/optimizeReleaseResources/
```

第 2 步的资源包是好的、第 3 步声明的输出不存在 → 根因是 `optimizeReleaseResources`（aapt2 `optimize`）
在这个 aapt2 组合下"成功"但产出 0 文件。手工 `aapt2 optimize <shrunk ap_> -o /tmp/x.ap_` 却完全正常，
所以不是 aapt2 二进制坏了，而是该任务与 `aapt2FromMavenOverride` 的交互问题。

**修法**：在 `gradle.properties` 里关掉资源优化：

```properties
# 容器 aapt2 override 组合下 optimizeReleaseResources 会「成功」但产出 0 文件，
# 导致 release APK 缺 AndroidManifest.xml / resources.arsc；官方 x86_64 aapt2 环境可删掉此行。
android.enableResourceOptimizations=false
```

改完必须 `./gradlew :app:clean :app:assembleRelease --no-build-cache`（只加参数、不清 `build/` 不够），
再复验条目数（本次 80 → 92）与上面那套四件套。

## 3. 签名

`assembleRelease` 出来的是 **未签名** 包（工程没有 signingConfig）。而 Android 11+ 要求 targetSdk ≥ 30 的包
**必须有 v2+ 签名**——`jarsigner` 只加 v1，装了会失败；必须用 `apksigner`。

```bash
BT=/opt/android-sdk/build-tools/35.0.0
$BT/apksigner sign \
  --ks /root/.android/debug.keystore --ks-pass pass:android --key-pass pass:android \
  --ks-key-alias androiddebugkey \
  --out HyperOS-Autofill-Fix-2.1.0.apk app/build/outputs/apk/release/app-release-unsigned.apk

$BT/apksigner verify --verbose --print-certs HyperOS-Autofill-Fix-2.1.0.apk
# 期望：v2 scheme true、v3 scheme true、Signer #1 certificate DN: CN=Android Debug
```

实用参数：`--v1-signing-enabled true --v2-signing-enabled true`（`minSdk ≥ 24` 时 v1 不是必须，
留着对老设备更稳；只加 v1 的话 Android 11+ 会拒绝安装——`jarsigner` 就是这种情况）。

**容器里 `zipalign` 通常跑不了**（它是原生 x86_64 二进制，arm64 上直接 `bad machine`；
`apksigner` 是脚本包装，所以可用）。对齐由 AGP 的打包环节完成，核验时用 `unzip -v <apk>` 看
`AndroidManifest.xml` / `resources.arsc` / `classes*.dex` 是不是 `Stored`（未压缩）即可；
要精确核验偏移就自己读 zip 头（Python `zipfile` 也可）。

**升级路径**：先取上一版已发布 APK 的证书指纹，和本机 keystore 比：

```bash
keytool -printcert -jarfile 上一版.apk | grep -E "SHA256:|Owner"          # 已发布包的证书
keytool -list -v -keystore /root/.android/debug.keystore -storepass android \
  -alias androiddebugkey | grep -E "SHA256:|Owner"                        # 本机 keystore
```

两者 SHA-256 一致 → **同签名**，用户可以直接覆盖升级；不一致 → 用户必须先卸载（要在 Release 说明里写清楚，
或者不要擅自换 key）。

`apksigner` 会另外生成 `*.idsig`（v4 尝试的副产物），别一起传上去。

## 4. 推送代码

```bash
# token 不要写进仓库、不要留在 shell history 里
umask 077; printf 'https://x-access-token:%s@github.com\n' "$TOKEN" > /root/.git-cred
git -c credential.helper='store --file=/root/.git-cred' push origin main
git -c credential.helper='store --file=/root/.git-cred' push origin v2.1.0
shred -u /root/.git-cred 2>/dev/null || rm -f /root/.git-cred
```

自检：`git grep -I -n "github_pat_" $(git rev-list --all)` 必须为空。
token 一旦在明文里出现过，就建议用户去 GitHub 撤销/轮换。

工程既有约定：标签是 annotated 的 `vX.Y.Z`（不是裸 `X.Y.Z`）。

## 5. 发 GitHub Release（API，不用网页）

```bash
TOKEN=... ; REPO=katiusu/HyperOS-Autofill-Fix
# 1) 建 release（body 用 --data-urlencode 或 python json，别手工拼引号）
curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/$REPO/releases \
  -d '{"tag_name":"v2.1.0","name":"2.1.0","body":"...","draft":false,"prerelease":false}'
# 2) 传附件（注意是 uploads.github.com，Content-Type 用 package-archive）
curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/vnd.android.package-archive" \
  --data-binary @HyperOS-Autofill-Fix-2.1.0.apk \
  "https://uploads.github.com/repos/$REPO/releases/<ID>/assets?name=HyperOS-Autofill-Fix-2.1.0.apk"
```

**发布后必须匿名复核**（去掉 token 再下一次，确认公开可访问、且哈希对得上）：

```bash
curl -sIL "https://github.com/$REPO/releases/download/v2.1.0/<asset>.apk" | grep -iE "^HTTP/|content-length"
curl -sL "…/<asset>.apk" -o /tmp/a.apk && sha256sum /tmp/a.apk    # 与 Release 说明里写的 SHA-256 对比
```

Release 说明里写清：**签名证书 SHA-256**（让用户确认能覆盖升级）、**APK SHA-256**（让用户校验），
以及从哪个旧版本可以直接升级、哪个必须卸载重装。

## 工具链闸门：改 SDK / 依赖前先查

**教训**：把 `compileSdk` 从 34 提到 35 后，构建挂在资源链接阶段：

```
ERROR: AAPT: error: failed to load include path …/platforms/android-35/android.jar
# 直接读那个 jar 才看清根因：error: illegal map type 'string' (22)  ← aapt2 版本太老
```

排查顺序（30 秒，能省一轮构建）：

```bash
aapt2 version                                   # 能跑的 aapt2 是哪一代
file /opt/android-sdk/build-tools/*/aapt2       # e_machine 3e=x86-64, b7=aarch64
grep -rn aapt2FromMavenOverride ~/.gradle/gradle.properties /root/.gradle/gradle.properties 2>/dev/null
ls /opt/android-sdk/platforms/                  # 目标 platform 是否已装
```

经验：

- SDK 里多数二进制是 **x86-64**；arm64 容器里只有手工编的 `aapt2`（版本可能很旧）。
  `build-tools;3x.0.0` 自带的 aapt2 是 x86 → 在 arm64 上直接 `bad machine`，换不了。
- aapt2 太老的表现是**解析不了新 platform 的 `resources.arsc`**，不是权限问题（文件可读、`unzip -t` 通过）。
- 结论：**先确认工具链能覆盖目标 API level，再动 `compileSdk/targetSdk`**；做不到就保持原样并把受阻项写清楚。

## 产物与源码是否一致（比复述 BUILD SUCCESSFUL 有力）

```bash
./gradlew :app:assembleRelease        # 再跑一次
# 期望：compileReleaseKotlin UP-TO-DATE（Gradle 按输入哈希判定"产物=当前源码"），packageRelease 可执行
```

注意：**时间戳不可靠**（FUSE 挂载 + 会话中断后时钟/ mtime 会乱），所以别只看 mtime，
用 `UP-TO-DATE` + 产物内容标记（dex 里的特征字符串 / `aapt2 dump badging` 的版本号）双向核对。

### 查"某个组件/能力是否真的进了包"

```bash
# ❌ 只查 classes.dex：debug 包是 multi-dex（实测有 classes..classes6），会得到假的 0
unzip -p app-debug.apk classes.dex | strings | grep -c FloatingNavigationBarItem     # 0 ← 假阴性

# ✅ 遍历所有 dex
for d in $(unzip -l app-debug.apk | grep -oE "classes[0-9]*\.dex"); do
  echo -n "$d: "
  unzip -p app-debug.apk "$d" | strings | grep -cE "FloatingNavigationBarItem"
done        # → classes5.dex: 1, classes6.dex: 10  ← 真在包里

# ✅ 混淆过的 release 包：符号名会被 R8 改掉，改查**数据字符串**
unzip -p app-release.apk classes.dex | strings | grep -c "floating_nav_bar"          # 偏好键会留下来
```

选哪一种取决于你要证明什么：证明"代码路径存在"用 debug 包的类名；证明"配置/资源/字符串生效"
用 release 包里的数据字符串。**"搜不到"不等于"没有"**，先排除 multi-dex 与混淆这两个假阴性来源再说结论。

### 构建被会话中断之后怎么接着做

长时间构建（本环境实测 7~10 分钟）容易被会话重启打断，恢复时**不要凭记忆假定成败**：

1. `ps aux | grep gradle` —— 还有没有在跑；
2. 用 `aapt2 dump badging <apk> | grep ^package:` 读**产物里的版本号**：
   - debug 包已是新版本、release 还是旧版本 → 编译过了，只差 release 打包，补跑 `assembleRelease`；
   - 两个都是旧版本 → 没跑完，整条重跑（幂等）；
3. `git status` / `git log -1` —— 别把已经提交的改动再做一遍。

## 发版收尾清单（代码之外的交付物）

一次发版通常不止推代码，按顺序做完这些（2.3.0 实测走通的完整链路）：

1. **版本号**：`versionCode` 唯一且单调（本工程用日期式 `2026100300`），`versionName` 与 CHANGELOG、README 徽章一致；
2. **`CHANGELOG.md`**：加 `## [X.Y.Z] — YYYY-MM-DD` 段，分「新增 / 变更 / 修复」三类；
3. **提交 + annotated tag**：`git tag -a vX.Y.Z -m "..."`（工程既有约定是 annotated 的 `vX.Y.Z`，不是裸版本号）；
4. **Release 正文**：按上一版的模板写（✨ 新增 / ♻️ 变更 / 🐛 修复 / 📦 安装 / 🔍 校验），
   **校验段必须含签名证书 SHA-256 与 APK SHA-256**，并写明"从哪些旧版本可以直接覆盖升级"；
5. **资产上传后回下载复核**：去掉 token 再下一次，`sha256sum` 与正文里写的一致才算发布成功（见第 5 节）；
6. **README 同步**：界面表、代码结构表、构建段、坑清单都要**照着代码核**再改（本次就抓出
   README 里"日志页有复制按钮"与实现不符——复制其实在设置页，日志页顶栏只有切换视图与清空）；
7. **仓库元数据**：description / topics 顺手补齐（topics 用 `PUT /repos/{repo}/topics`，
   只能是小写字母数字加连字符；"最新版号"这类会过期的信息不要写进 description）。


