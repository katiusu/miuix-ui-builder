# aarch64 容器里的工具链生存指南（aapt2 双命名空间 / 影子 SDK / 镜像 / Git 兜底）

适用场景：**agent 跑在手机上的 aarch64 容器里**（PRoot / proot-distro 之类），
SDK 里多数二进制是 x86 跑不了，只能借用一个手工编的 aarch64 `aapt2`。

另一份 `android-release.md` 讲的是"正常机器"上的出包流程；这一页讲**这台机器为什么不正常、以及怎么绕过去**。
换机器请先判断自己是否真需要这些招。
**2026-10 复测后新增下面的 §0.5：三条新路径能省掉这一页大部分手工活，先读它。**

## 0. 先判断你处在哪种命名空间

```bash
# 容器（glibc）看到的根
ls /opt /root /tmp | head
# bionic 二进制（Android 程序）看到的根：同一路径是否存在？
/system/bin/linker64 /system/bin/toybox ls /opt      # "No such file or directory" → 双命名空间确认
```

本机的映射：容器根 = Android 侧 `/data/data/<pkg>/files/linux/ubuntu`；
`/sdcard`、`/system`、`/proc` 两边**同名**（共享），其余容器路径 bionic 看不见。
**aapt2 是 bionic 程序，它只认 Android 那一侧。**

## 0.5 2026-10 复测：这一页有一半的招已经不必用了（先试新路）

同一个容器类环境重测，下面这几条把"shim + 影子 SDK + 强制镜像"整条链都省掉了。
**先按这里试，失败了再回来看后面的手工方案。**

1. **先找 glibc 版 aapt2，别急着编 shim。** `build-tools/36.0.0/aapt2` 实测是 **aarch64 + glibc**
   （`readelf -l` 的 INTERP 是 `/lib/ld-linux-aarch64.so.1`），在 PRoot 里**直接可执行、而且吃容器路径**。
   各版本实测：`34.0.0` aarch64 但缺 `libm.so`（跑不起来）、`35.0.2` 正常、`36.0.0` 正常、`37.0.0` 是 **x86_64**（`bad machine`）。

   ```bash
   for v in 34.0.0 35.0.2 36.0.0 37.0.0; do
     printf "%-7s " $v; od -j18 -N2 -An -tu2 /opt/android-sdk/build-tools/$v/aapt2 2>/dev/null   # 183=arm64 62=x86_64
   done
   readelf -l /opt/android-sdk/build-tools/36.0.0/aapt2 | grep -A1 INTERP                        # glibc 版才有 /lib/ld-linux-aarch64.so.1
   ```

   指过去（写在**与 AGP 同一个 GRADLE_USER_HOME** 的 `gradle.properties` 里）：
   `android.aapt2FromMavenOverride=/opt/android-sdk/build-tools/36.0.0/aapt2`
   —— glibc aapt2 **不需要路径翻译**，第 1~4 节的 shim 与影子 SDK 全部不需要。

2. **新 aapt2 能解析新 platform。** 那句 `error: illegal map type 'string' (22)` 是**旧 bionic shim** 的锅，
   不是 aapt2 2.19 的锅：
   ```bash
   /opt/android-sdk/build-tools/36.0.0/aapt2 dump resources /opt/android-sdk/platforms/android-37.0/android.jar
   # → Binary APK  Package name=android id=01   （android-34/35/37.0 全部通过）
   ```
   所以 **compileSdk 可以跟着库的 `minCompileSdk` 走**。本轮用 `compileSdk = 37` + `buildToolsVersion = "36.0.0"`
   的组合跑通（SDK 里的 37.0.0 是 x86_64 跑不了，就显式钉住能跑的那个）。第 5 节的"只能停在 34"作废。

3. **别全局强制替换仓库。** `dl.google.com` / `repo.maven.apache.org` / `repo1.maven.org` / `services.gradle.org`
   现在都通。全局 `init.gradle` 里 clear + 换镜像反而会**漏包**：实测 `dev.chrisbanes.haze:haze:1.7.3`
   在阿里 `gradle-plugin` 镜像里只有 `.module`/`.aar`、没有 `haze-1.7.3.jar`，解析直接失败。
   更稳的做法是**给工程一个独立的 `GRADLE_USER_HOME`**（如 `/root/.gu-<proj>`：拷/链 `wrapper/dists`、
   `caches/modules-2` 复用已有缓存，自带一份 `gradle.properties`，**不放 init.gradle**），
   让它走工程自己声明的 `google()` / `mavenCentral()`。第 6 节只在"外网确实不通"时才用。

4. **PRoot 下的文件锁只允许一个 Gradle 进程。** `fcntl` 在 PRoot 里不可靠，
   全局 `~/.gradle/caches/journal-1/journal-1.lock` 同时只能被一个守护进程持有，第二个进程会报：
   ```
   Could not create service of type FileAccessTimeJournal using GradleUserHomeScopeServices.createFileAccessTimeJournal().
   > java.io.IOException: Operation not permitted
   ```
   （`modules-2.lock` / `jars-9.lock` 同理。）做法：
   ```bash
   for p in $(ps -ef | awk '/[G]radleDaemon/ {print $2}'); do kill -TERM $p; done  # [G] 防自匹配
   ./gradlew --no-daemon --console=plain :app:assembleDebug                       # 全程单进程
   ```
   注意：**不要用 `pkill -f <模式>`** 在这个 shell 里杀进程（模式会匹配到工具自己的 shell，实测把自己也杀了）。
   另外 `/sdcard` 这类 FUSE 卷上 Gradle 会报 `Failed to load native library 'libnative-platform.so'`——
   **只是告警**，把 `GRADLE_USER_HOME` 放容器内即可。

5. **AGP 9.x 的两个硬性前提**：Gradle 要 ≥ 9.6.0（用 9.3.1 会报
   `Minimum supported Gradle version is 9.6.0. Current version is 9.3.1.`）；
   并且**不要再加 `org.jetbrains.kotlin.android` 插件**（AGP 9 内置 Kotlin 支持，加了直接报
   `The 'org.jetbrains.kotlin.android' plugin is no longer required for Kotlin support since AGP 9.0`）。

**什么时候仍需回看后面的手工方案**：容器里确实只有 bionic aapt2（SDK 只有旧 build-tools，
或根本拿不到 aarch64 的 glibc 版）时——那时第 1~4 节的 shim + 影子 SDK 就是唯一出路。

## 1. aapt2 直接执行失败：`deps: cannot find libm.so` / `proroot-ldso: failure rc=2`

这是容器的 stub loader 把 bionic 二进制当 glibc 二进制加载（找到 `libm.so` 后又抱怨 `_ctype_` 未定义）。
**不要**去 glibc 目录里补 `libm.so` 符号链接——那会让 bionic 链接器反过来找不到 `libc.so`，
把 `toybox` 之类也弄坏（真实踩过）。

正确做法：**用 Android 自己的动态链接器启动**：

```bash
BIN=/data/data/<pkg>/files/linux/ubuntu/opt/android-sdk/aapt2-arm64/aapt2.bin
/system/bin/linker64 "$BIN" version     # 能打印版本 = 走通了
```

## 2. 但仅仅"能跑"不够：**路径要翻译**

`linker64` 起来的 aapt2 在 Android 命名空间里工作，AGP 传给它的却是**容器路径**：

```
ERROR: AAPT: error: failed to open APK: I/O error.                      # -I <容器路径>/android.jar
ERROR: AAPT: error: failed to load include path /…/platforms/android-35/android.jar
ERROR: AAPT: error: /root/.gradle/caches/…/transformed/androidx.core: No such file or directory
```

翻译规则（本机）：

| 容器路径 | bionic 可见路径 |
|---|---|
| `/sdcard/...`、`/system/...`、`/proc/...` | 原样 |
| 其他绝对路径 `/X/...` | `/data/data/<pkg>/files/linux/ubuntu/X/...` |

## 3. 关键：AGP 用的是 aapt2 **守护进程**，stdin 也要翻译

只看文档很容易只翻译 `argv`，然后发现**编译资源时照样报错**——因为 AGP 启动的是
`aapt2 m`（守护模式），命令通过 **stdin** 逐行下发。

从 AGP 字节码可以坐实协议（`com.android.tools.build:builder` → `Aapt2DaemonUtil`）：

```
DAEMON_MODE_COMMAND = "m"
request(Writer, command, args) 写的是：command "\n" (每个 arg "\n") "\n" "\n"
```

即：`命令行 → 每行一个参数 → 两个空行结束`。

所以要写一个 **argv + stdin 双向翻译**的 shim，放在 AGP 的 `aapt2FromMavenOverride` 指向的位置：

```python
#!/usr/bin/env python3
# /opt/android-sdk/aapt2-arm64/aapt2  （aapt2.bin 是原始二进制）
import os, subprocess, sys

HOST_ROOT = "/data/data/<pkg>/files/linux/ubuntu"       # 容器根在 Android 侧的路径
PASSTHROUGH = ("/sdcard", "/storage", "/mnt", "/system", "/vendor", "/product",
               "/apex", "/proc", "/dev", "/sys", "/linkerconfig", HOST_ROOT)
BIN, LINKER = HOST_ROOT + "/opt/android-sdk/aapt2-arm64/aapt2.bin", "/system/bin/linker64"
DAEMON = ("m", "daemon")

def tr(p):
    if not p.startswith("/") or p.startswith(HOST_ROOT):
        return p
    for pre in PASSTHROUGH:
        if p == pre or p.startswith(pre + "/"):
            return p
    return HOST_ROOT + p

raw = sys.argv[1:]
is_daemon = bool(raw) and raw[0] in DAEMON          # ← 必须是 "m"，不是 "daemon"
proc = subprocess.Popen([LINKER, BIN] + [tr(a) for a in raw],
                        stdin=subprocess.PIPE if is_daemon else subprocess.DEVNULL,
                        stdout=sys.stdout.buffer, stderr=sys.stderr.buffer)
if is_daemon:
    try:
        for line in sys.stdin.buffer:                # 逐行翻译 stdin
            out = tr(line.rstrip(b"\r\n").decode("utf-8", "surrogateescape"))
            proc.stdin.write(out.encode("utf-8", "surrogateescape") + b"\n")
            proc.stdin.flush()
    except (BrokenPipeError, ValueError, OSError):
        pass
sys.exit(proc.wait())
```

自检：`aapt2 version` 能打印；构建时用 `AAPT2-shim` 日志确认收到的行里出现的是**翻译后**的路径。

## 4. 影子 SDK：让 Gradle 和 bionic 看到同一个 SDK

`~/.gradle/caches/<ver>/transforms/...` 这类**容器私有路径** aapt2 读不到，
`AarResourcesCompilerTransform` 会直接失败。做法是把 SDK 放到两边同名的地方（`/sdcard`）：

```bash
S=/sdcard/.shadow-sdk
mkdir -p $S/platforms/android-34 $S/build-tools/34.0.0 $S/licenses
P=/opt/android-sdk/platforms/android-34
cp $P/android.jar $P/core-for-system-modules.jar $P/framework.aidl $P/build.prop \
   $P/sdk.properties $P/source.properties $P/package.xml $P/uiautomator.jar $S/platforms/android-34/
cp -r $P/optional $S/platforms/android-34/
cp /opt/android-sdk/build-tools/34.0.0/source.properties $S/build-tools/34.0.0/
cp /opt/android-sdk/licenses/* $S/licenses/
# 构建时：
ANDROID_HOME=/sdcard/.shadow-sdk ANDROID_SDK_ROOT=/sdcard/.shadow-sdk gradle assembleDebug
```

坑：`/sdcard` 是 FUSE，**建不了符号链接**（`ln: Permission denied`），只能实拷贝；
`package.xml` / `build.prop` / `source.properties` 少一个就是 "Failed to find target with hash string 'android-34'"；
Gradle 守护进程会缓存 SDK 解析结果，改了影子 SDK 结构后 `gradle --stop` 再跑。

## 5. 老 aapt2 还卡住 `compileSdk` 上限（※ 2026-10 复测：换 glibc aapt2 后此条作废，见 §0.5-2）

本机可用的 aapt2 是 2.19。它**解析不了新 platform 的 `resources.arsc`**：

```bash
aapt2 dump resources /sdcard/.shadow-sdk/platforms/android-35/android.jar
# error: illegal map type 'string' (22)  → 链接阶段就是 failed to load include path
```

结论：**compileSdk 只能停在老 aapt2 能吃的档位（本机 34）**。
需要更新的 compileSdk 时，先解决 aapt2 版本，别改 `compileSdk` 硬试。
`compileSdk` 与库的 `minCompileSdk` 冲突时用 `android.experimental.disableCompileSdkChecks=true`，
但**只在你确认界面没用到那些新 API** 时用；升级路径（AGP 9.1+ / Gradle 9.x / compileSdk 37）写进 README。

## 6. Gradle 依赖：必须"清空 + 换镜像"（※ 2026-10 复测：先试 §0.5-3 的独立 GRADLE_USER_HOME + 官方仓库）

直连 `dl.google.com` / `repo.maven.apache.org` 会被 TLS 掐断，而**只往列表里追加镜像没用**——
Gradle 会先试那些连不上的仓库。要在全局 init 脚本里**先 clear 再添加**：

```groovy
// ~/.gradle/init.gradle
def mirrors = ['https://maven.aliyun.com/repository/google',
               'https://maven.aliyun.com/repository/gradle-plugin',
               'https://maven.aliyun.com/repository/public']
settingsEvaluated { settings ->
    try { settings.pluginManagement.repositories.clear() } catch (Throwable ignored) { }
    settings.pluginManagement.repositories { mirrors.each { m -> maven { url m } }; gradlePluginPortal() }
    try { settings.dependencyResolutionManagement.repositories.clear() } catch (Throwable ignored) { }
    try { settings.dependencyResolutionManagement.repositories { mirrors.each { m -> maven { url m } } } } catch (Throwable ignored) { }
}
```

另外：`repo1.maven.org` 常通（拉 miuix / androidx 的 sources jar 走它），
`codeload.github.com` 通（下整个仓库 tarball 比一个个 raw 文件快）。

## 7. Git 推不上去时的兜底：GitHub Git Data API

症状：`curl -I https://github.com` 有时 200、有时连不上；`git push` 报
`GnuTLS recv error (-110)` 或 `Failed to connect to github.com port 443`，
而 `api.github.com` 一直可用。别耗在重试 `git push` 上，直接用 API 建提交：

```
GET   /repos/{repo}/git/ref/heads/{branch}        → 取父提交 sha
POST  /repos/{repo}/git/blobs   (base64 content)  → 每个文件一个 blob
POST  /repos/{repo}/git/trees   {"tree":[…]}
      ★ 不要带 base_tree：这棵树就是全部内容 → 旧的（包括误提交的 build/ 产物）自然消失
POST  /repos/{repo}/git/commits {"message","tree","parents":[父],"author","committer"}
PATCH /repos/{repo}/git/refs/heads/{branch} {"sha":新提交,"force":true}
```

要点：

- 文件模式从本地 git 取：`git ls-files -s` 的 `100755`（`gradlew`）/ `100644`；
- 推完**校验**：`GET /repos/{repo}/git/trees/{branch}?recursive=1` 数文件数、
  确认 `build/` 之类已不存在；再 `git fetch && git reset --hard origin/<branch>` 让本地与远端同 sha；
- Release 附件**下载回来算哈希**（见 `android-release.md` 第 5 节），别只看上传返回 200；
- token 放 `umask 077` 的临时文件，别写进仓库、别进 shell history；用完提醒用户轮换。

## 8. 会话会中断：恢复后先核对，不要重跑

长构建（本机 release 常 10~20 分钟）很容易被会话重启打断。中断后**先取证再动作**：

```bash
tail -5 /tmp/<你的构建日志>; grep -c "GRADLE_EXIT" /tmp/<你的构建日志>   # 有没有跑到收尾
ls -la app/build/outputs/apk/release/                                   # 产物时间戳变了没
ps -ef | grep -c "[G]radleDaemon"                                       # 还有没有在跑
git -C <repo> status --short; git -C <repo> log --oneline -1            # 推送到底成没成
```

时间戳在 FUSE + 时钟漂移下不可靠，**以"产物内容标记 + UP-TO-DATE + 远端 API 查询"三者交叉为准**。
