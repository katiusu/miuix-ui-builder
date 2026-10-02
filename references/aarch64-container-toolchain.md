# aarch64 容器里的工具链生存指南（aapt2 双命名空间 / 影子 SDK / 镜像 / Git 兜底）

适用场景：**agent 跑在手机上的 aarch64 容器里**（PRoot / proot-distro 之类），
SDK 里多数二进制是 x86 跑不了，只能借用一个手工编的 aarch64 `aapt2`。

另一份 `android-release.md` 讲的是"正常机器"上的出包流程；这一页讲**这台机器为什么不正常、以及怎么绕过去**。
换机器请先判断自己是否真需要这些招。

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

## 5. 老 aapt2 还卡住 `compileSdk` 上限

本机可用的 aapt2 是 2.19。它**解析不了新 platform 的 `resources.arsc`**：

```bash
aapt2 dump resources /sdcard/.shadow-sdk/platforms/android-35/android.jar
# error: illegal map type 'string' (22)  → 链接阶段就是 failed to load include path
```

结论：**compileSdk 只能停在老 aapt2 能吃的档位（本机 34）**。
需要更新的 compileSdk 时，先解决 aapt2 版本，别改 `compileSdk` 硬试。
`compileSdk` 与库的 `minCompileSdk` 冲突时用 `android.experimental.disableCompileSdkChecks=true`，
但**只在你确认界面没用到那些新 API** 时用；升级路径（AGP 9.1+ / Gradle 9.x / compileSdk 37）写进 README。

## 6. Gradle 依赖：必须"清空 + 换镜像"

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
