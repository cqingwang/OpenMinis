# 工程记忆 (Engineering Memory)

## [Android] 2026-09-22 — 首次本地编译缺 `app/libs/rclone.aar`，`checkDebugAarMetadata` 失败

**作用域**: `src/android/`、`deps/build_rclone_android.sh`

**症状/原始错误**:
```
Execution failed for task ':app:checkDebugAarMetadata'.
> Could not resolve all files for configuration ':app:debugRuntimeClasspath'.
   > Could not find :rclone:.
     Searched in: file:.../src/android/app/libs/rclone.aar
```

**触发条件**: 全新 clone 后直接按 BUILDING.md Android 节（`build_proot.sh` → `prepare_android_sandbox.sh` → `assembleDebug`）构建。

**根因**: `app/build.gradle.kts` 通过 `flatDir("app/libs")` 依赖本地 `rclone.aar`（备份功能 SMB/WebDAV/SFTP/S3/FTP 后端），该 `.aar` 是构建产物、未入库，须由 `deps/build_rclone_android.sh`（gomobile bind）生成。BUILDING.md 的 Android 步骤清单漏写了这一步（iOS 节同样有对应的 `build_rclone_ios.sh`）。

**最终修复**:
```sh
go install golang.org/x/mobile/cmd/gomobile@latest
gomobile init                                   # 需 GOPATH/GOBIN 可写
export ANDROID_NDK_HOME=... ANDROID_HOME=...    # 脚本要求二者之一
./deps/build_rclone_android.sh                  # → deps/build/rclone/rclone.aar
cp deps/build/rclone/rclone.aar src/android/app/libs/
```

**已排除的无效方案**: 删掉 rclone 依赖（破坏备份功能）；从 Maven 拉取（settings.gradle.kts 注释明确说明上游不发布）。

## [Android] 2026-09-22 — `build_proot.sh` 在非默认 SDK 路径下找不到 NDK

**症状**: `Android NDK not found`，尽管 SDK 里已装 NDK r29。

**根因**: 脚本 `resolve_ndk()` 的自动探测只查 `$HOME/Library/Android/sdk/ndk`；本机 SDK 实际在 `/Users/chan/sdk/android-sdk`。

**修复/规避**: 显式导出 `ANDROID_NDK_HOME` 与 `ANDROID_HOME` 后再运行脚本；Gradle 侧 `local.properties` 的 `sdk.dir` 已覆盖。

**验证命令**: `./deps/build_proot.sh` 末尾的 `verify_artifacts` 段必须输出 4 个 ✓（proot-aarch64、libproot.so、libproot-loader.so、libproot-loader32.so）；loader 两个文件是 vendored Termux 构建、sha256 锁定，禁止用 fork 重编译产物覆盖（见脚本内注释）。

## 本地首次编译成功路径（复现清单）

```sh
export ANDROID_NDK_HOME=/Users/chan/sdk/android-sdk/ndk/29.0.14206865
export ANDROID_HOME=/Users/chan/sdk/android-sdk
./deps/build_proot.sh
./scripts/prepare_android_sandbox.sh
cp src/android/app/provider-customization.properties.example \
   src/android/app/provider-customization.properties
./deps/build_rclone_android.sh
mkdir -p src/android/app/libs && cp deps/build/rclone/rclone.aar src/android/app/libs/
cd src/android && ./gradlew :app:assembleDebug   # JDK 17
adb install -r app/build/outputs/apk/debug/app-debug.apk
```

APK 完整性检查: `unzip -l app-debug.apk | grep -E "libproot|alpine-minirootfs"` — 三个 `.so` + rootfs 缺一不可（缺 loader 的故障模式见 BUILDING.md「[Shell not running]」条目）。
