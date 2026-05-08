# RikkaHub Android Debug 构建成功流程

本文记录这次在 Android Termux + Ubuntu proot + ARM64 设备上成功构建并安装 RikkaHub Debug APK 的完整流程。

## 最终结果

- 项目目录：`/root/Projects/rikkahub`
- 成功命令：`./gradlew assembleDebug --stacktrace`
- 构建结果：`BUILD SUCCESSFUL`
- 输出 APK：
  - `app/build/outputs/apk/debug/app-arm64-v8a-debug.apk`
  - `app/build/outputs/apk/debug/app-universal-debug.apk`
  - `app/build/outputs/apk/debug/app-x86_64-debug.apk`
- 已复制到 Android 可见目录：
  - `/sdcard/Download/rikkahub-arm64-debug.apk`
- 包名：`me.rerere.rikkahub.debug`
- 版本：`2.1.12`

## 环境特征

这次不是普通 x86_64 Linux 构建环境，而是：

- 设备架构：`aarch64`
- Android：16
- 容器环境：Ubuntu proot
- Termux 前缀：`/data/data/com.termux/files/usr`
- Android SDK：`/root/android-sdk`
- 项目 `local.properties`：

```properties
sdk.dir=/root/android-sdk
```

这个环境的核心问题是：官方 Android SDK/Gradle 下载的 `aapt2` 是 x86_64 Linux 二进制，但当前设备是 ARM64，不能直接执行。

## 1. 拉代码

项目统一放到 `/root/Projects/`：

```bash
mkdir -p /root/Projects
cd /root/Projects
git clone https://github.com/Horizon00001/rikkahub.git
cd /root/Projects/rikkahub
```

如果需要保留上游地址：

```bash
git remote add upstream https://github.com/rikkahub/rikkahub.git
git remote -v
```

## 2. 安装基础依赖

```bash
apt-get update
apt-get install -y openjdk-17-jdk wget unzip zip curl
```

Gradle 后续会自动下载项目需要的 JDK 21 toolchain。

## 3. 安装 Android SDK

安装 command line tools：

```bash
mkdir -p /root/android-sdk/cmdline-tools
cd /root/android-sdk/cmdline-tools
wget -O commandlinetools.zip https://dl.google.com/android/repository/commandlinetools-linux-13114758_latest.zip
unzip commandlinetools.zip
mv cmdline-tools latest
```

配置环境变量：

```bash
export ANDROID_HOME=/root/android-sdk
export ANDROID_SDK_ROOT=/root/android-sdk
export PATH="$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"
```

接受 license 并安装 SDK 包：

```bash
yes | sdkmanager --licenses
sdkmanager \
  "platform-tools" \
  "platforms;android-37.0" \
  "build-tools;37.0.0"
```

项目构建过程中 AGP 还可能自动安装：

```bash
sdkmanager "platforms;android-36" "build-tools;36.0.0"
```

写入项目本地 SDK 配置：

```bash
cd /root/Projects/rikkahub
printf 'sdk.dir=/root/android-sdk\n' > local.properties
```

## 4. 安装前端依赖

项目的 `web` 模块会在 Gradle 构建时调用 `web-ui` 的构建流程。

安装 Bun：

```bash
curl -fsSL https://bun.sh/install | bash
ln -sf /root/.bun/bin/bun /usr/local/bin/bun
```

安装 `web-ui` 依赖：

```bash
cd /root/Projects/rikkahub/web-ui
bun install --frozen-lockfile
```

这次 `bun install` 后 `pathe` 的 nested dependency 有解析问题，用 npm 补了一次依赖树：

```bash
npm install --package-lock=false
```

验证前端可以单独构建：

```bash
bun run build
```

这里有 sourcemap 和 chunk size warning，但不影响构建。

## 5. 处理 google-services.json

仓库 README 提到构建需要 `app/google-services.json`。

如果没有真实 Firebase 配置，debug 本地构建可以放一个占位文件到：

```text
app/src/debug/google-services.json
```

示例内容：

```json
{
  "project_info": {
    "project_number": "000000000000",
    "project_id": "rikkahub-debug",
    "storage_bucket": "rikkahub-debug.appspot.com"
  },
  "client": [
    {
      "client_info": {
        "mobilesdk_app_id": "1:000000000000:android:0000000000000000000000",
        "android_client_info": {
          "package_name": "me.rerere.rikkahub.debug"
        }
      },
      "oauth_client": [],
      "api_key": [
        {
          "current_key": "debug-placeholder-api-key"
        }
      ],
      "services": {
        "appinvite_service": {
          "other_platform_oauth_client": []
        }
      }
    }
  ],
  "configuration_version": "1"
}
```

注意：这是 debug 占位配置，只用于本地打包。正式 Firebase Analytics、Crashlytics、Remote Config 等功能需要真实 `google-services.json`。

## 6. 处理 Google Maven 下载不稳定

第一次构建卡在 Gradle 从 Google Maven 下载 AndroidX/Firebase 依赖时 TLS handshake 失败。

解决方式是在用户级 Gradle init 脚本里加镜像，不改项目源码：

文件：`/root/.gradle/init.d/rikkahub-mirrors.gradle.kts`

```kotlin
settingsEvaluated {
    pluginManagement {
        repositories {
            maven("https://maven.aliyun.com/repository/google")
            maven("https://maven.aliyun.com/repository/gradle-plugin")
            maven("https://maven.aliyun.com/repository/public")
        }
    }

    dependencyResolutionManagement {
        repositories {
            maven("https://maven.aliyun.com/repository/google")
            maven("https://maven.aliyun.com/repository/public")
        }
    }
}
```

加完后重新跑：

```bash
cd /root/Projects/rikkahub
./gradlew assembleDebug --stacktrace
```

这一步解决了依赖下载问题。

## 7. 处理 AAPT2 架构问题

### 现象

依赖下载解决后，构建失败在：

```text
AAPT2 aapt2-9.2.0-15009934-linux Daemon startup failed
Cannot run program ".../aapt2": Exec failed, error: 2 (No such file or directory)
```

检查机器和 AAPT2：

```bash
uname -m
file /root/android-sdk/build-tools/37.0.0/aapt2
```

结果：

```text
aarch64
aapt2: ELF 64-bit LSB pie executable, x86-64
```

也就是说当前机器是 ARM64，但官方 SDK 的 `aapt2` 是 x86_64。

### 尝试过但不采用的方案

Termux 仓库里有 ARM64 `aapt2`：

```bash
# Termux 包：aapt2_13.0.0.6-23_aarch64.deb
```

这个 ARM64 `aapt2` 能执行 `version`，但在链接项目资源、读取 `android.jar` 时会失败或崩溃，所以最终没有采用。

### 成功方案：qemu 跑官方 x86_64 AAPT2

启用 amd64 包源。

先把 `/etc/apt/sources.list` 的 Ubuntu ports 源限制为 arm64，并新增 amd64 源：

```text
deb [arch=arm64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://ports.ubuntu.com/ubuntu-ports questing main universe multiverse
deb [arch=arm64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://ports.ubuntu.com/ubuntu-ports questing-updates main universe multiverse
deb [arch=arm64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://ports.ubuntu.com/ubuntu-ports questing-security main universe multiverse
deb [arch=amd64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://archive.ubuntu.com/ubuntu questing main universe multiverse
deb [arch=amd64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://archive.ubuntu.com/ubuntu questing-updates main universe multiverse
deb [arch=amd64 signed-by="/usr/share/keyrings/ubuntu-archive-keyring.gpg"] http://security.ubuntu.com/ubuntu questing-security main universe multiverse
```

安装 qemu 和 x86_64 运行库：

```bash
dpkg --add-architecture amd64
apt-get update
apt-get install -y \
  qemu-user \
  qemu-user-binfmt \
  libc6:amd64 \
  libstdc++6:amd64 \
  libgcc-s1:amd64 \
  zlib1g:amd64
```

验证 qemu 能跑官方 AAPT2：

```bash
qemu-x86_64 /root/android-sdk/build-tools/37.0.0/aapt2 version
qemu-x86_64 /root/android-sdk/build-tools/37.0.0/aapt2 dump resources /root/android-sdk/platforms/android-37.0/android.jar >/tmp/aapt2-dump.out
```

## 8. 给 Gradle 配置 AAPT2 wrapper

AGP 的 `android.aapt2FromMavenOverride` 要求路径文件名必须以 `aapt2` 结尾。shell 脚本 wrapper 或 `aapt2-qemu` 这种名字会被拒绝。

所以写一个 ARM64 原生小 wrapper，文件名必须叫 `aapt2`。

文件：`/root/bin/aapt2-qemu.c`

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(int argc, char **argv) {
    const char *qemu = "/usr/bin/qemu-x86_64";
    const char *aapt2 = "/root/android-sdk/build-tools/37.0.0/aapt2";

    char **args = calloc((size_t)argc + 2, sizeof(char *));
    if (args == NULL) {
        perror("calloc");
        return 127;
    }

    args[0] = (char *)qemu;
    args[1] = (char *)aapt2;
    for (int i = 1; i < argc; i++) {
        args[i + 1] = argv[i];
    }
    args[argc + 1] = NULL;

    execv(qemu, args);
    perror("execv");
    free(args);
    return 127;
}
```

编译成 `/root/bin/aapt2`：

```bash
gcc -O2 -Wall -Wextra -o /root/bin/aapt2 /root/bin/aapt2-qemu.c
```

验证：

```bash
/root/bin/aapt2 version
/root/bin/aapt2 dump resources /root/android-sdk/platforms/android-37.0/android.jar >/tmp/aapt2-wrapper-dump.out
```

配置 Gradle：

文件：`/root/.gradle/gradle.properties`

```properties
android.aapt2FromMavenOverride=/root/bin/aapt2
```

## 9. 正式打包

```bash
cd /root/Projects/rikkahub
./gradlew assembleDebug --stacktrace
```

成功输出：

```text
> Task :app:assembleDebug

BUILD SUCCESSFUL in 1m 23s
258 actionable tasks: 93 executed, 165 up-to-date
```

构建时可能看到这类 warning：

```text
Unable to strip the following libraries, packaging them as they are
```

这是 native library strip warning，不影响 APK 产出。

## 10. APK 路径和安装

查看 APK：

```bash
ls -lh app/build/outputs/apk/debug/*.apk
```

这次产物：

```text
app-arm64-v8a-debug.apk
app-universal-debug.apk
app-x86_64-debug.apk
```

复制 ARM64 包到 Android 可见目录：

```bash
cp app/build/outputs/apk/debug/app-arm64-v8a-debug.apk /sdcard/Download/rikkahub-arm64-debug.apk
```

这台环境里直接静默安装失败：

```bash
pm install -r /sdcard/Download/rikkahub-arm64-debug.apk
```

失败原因是当前 proot/root 身份调用 Android package service 时 AppOps 校验不通过：

```text
Failure calling service package: Failed transaction
AppOpsService.checkPackage
```

可行方式是拉起系统安装器：

```bash
termux-open --view --content-type application/vnd.android.package-archive /sdcard/Download/rikkahub-arm64-debug.apk
```

如果没弹出安装器，强制 chooser：

```bash
termux-open --chooser --view --content-type application/vnd.android.package-archive /sdcard/Download/rikkahub-arm64-debug.apk
```

然后在手机界面手动点安装。

安装后可检查：

```bash
pm path me.rerere.rikkahub.debug
pm list packages | rg 'rikkahub|rerere'
```

## 关键点总结

1. `google-services.json` 缺失会导致 `:app:processDebugGoogleServices` 失败；debug 本地构建可用占位文件，正式功能要换真实配置。
2. Google Maven TLS 不稳定时，用用户级 Gradle init 脚本增加阿里 Maven 镜像。
3. ARM64 Termux/proot 环境不能直接运行官方 SDK 的 x86_64 `aapt2`。
4. Termux ARM64 `aapt2` 能启动但不适合这个项目的完整资源链接。
5. 最终成功方案是：安装 `qemu-user` + amd64 运行库，用 ARM64 原生 wrapper `/root/bin/aapt2` 转发到 `qemu-x86_64 /root/android-sdk/build-tools/37.0.0/aapt2`。
6. `android.aapt2FromMavenOverride` 的文件名必须以 `aapt2` 结尾，否则 AGP 会拒绝。
7. 静默安装在当前 proot 身份下失败，用 `termux-open` 拉起系统安装器安装。

