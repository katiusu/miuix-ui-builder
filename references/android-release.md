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
