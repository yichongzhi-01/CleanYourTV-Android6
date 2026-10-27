# CleanYourTV Android 6 适配技术文档

本文档记录将 CleanYourTV（原要求 Android 8.0+）适配到 **Android 6.0.1（API 23）** 的全部技术细节，供开发者参考与复现。

---

## 1. 背景

CleanYourTV 原版 `minSdk 26`（Android 8.0+），在 Android 6.0.1（小米电视 4C 等大量 2016-2018 老机型）上**无法安装**；即使强行降级安装，启动即崩溃：

```
java.lang.NoSuchMethodError: No virtual method forceHasOverlappingRendering(Z)V in class Landroid/view/View;
```

根因：代码直接调用了 API 26 的 `View.forceHasOverlappingRendering()`、API 24 的 `Configuration.getLocales()` 等方法，且使用了 Java 8 的 `java.util.function` 包。

## 2. 适配改动总览

| 文件 | 改动 |
|---|---|
| `launcher/build.gradle` | `minSdk 23`；`compileSdk 34`；`coreLibraryDesugaringEnabled true` + `desugar_jdk_libs 2.0.4`；`minifyEnabled false`、`shrinkResources false` |
| `launcher/app/build.gradle` | `applicationId com.gathin.tvhome6`；同上 desugar/minify 配置 |
| `launcher/app/src/main/AndroidManifest.xml` | MainActivity 恢复 `HOME + DEFAULT` category（并保留 `LEANBACK_LAUNCHER`，形成双 filter） |
| `MainActivity.java` | 见下方逐项 |

### 2.1 MainActivity.java 逐项修复（7 处）

| 位置 | 原代码 | 修复 |
|---|---|---|
| 3 处 `forceHasOverlappingRendering(false)` | API 26 方法，Android 6 直接崩 | 包裹 `if (Build.VERSION.SDK_INT >= 26) { ... }` |
| `registerDefaultNetworkCallback(...)` | API 26，Android 6 直接崩 | `if (Build.VERSION.SDK_INT >= 26)` 守卫；老系统跳过网络监视回调 |
| 2 处 `getResources().getConfiguration().getLocales()` | API 24，Android 6 崩 | 替换为自建 `getLocalesCompat(Configuration)`：`SDK_INT >= 24 ? getLocales().get(0) : locale` |
| 2 处 `PackageManager.getChangedPackages(...)` | API 26，Android 6 崩 | `SDK_INT < 26` 时返回 `true`（总是刷新应用列表，安全降级） |

## 3. 为什么换包名 `com.gathin.tvhome6`

小米电视固件中的 **TvMgr（com.xiaomi.mitv.tvmanager）** 会拦截原包名 `com.gathin.tvhome` 的桌面启动器：
- 第三方桌面一旦进入前台，TvMgr 立即强制切回 PatchWall；
- 且可能对原包名进行"拉黑"处理。

换用新包名 `com.gathin.tvhome6` 后不再被拦截。卸载 TvMgr 后该守卫彻底消失。

## 4. HOME 键直达新桌面（小米电视实测方案）

### 问题
Android 6 的 HOME intent 固定解析到系统自带桌面（PatchWall），注册了多个 HOME 候选也**不会弹出选择器**，第三方桌面永远无法通过 HOME 键进入。

### 实测有效的三步
1. **安装 HOME 版 APK**（MainActivity 带 HOME + LEANBACK 双 filter）
2. **卸载桌面守卫 TvMgr**：
   ```
   adb shell pm uninstall --user 0 com.xiaomi.mitv.tvmanager
   ```
   （否则任何第三方 HOME 都会被强制切回）
3. **卸载系统桌面 PatchWall**：
   ```
   adb shell pm uninstall --user 0 com.mitv.tvhome
   ```
   系统此时只剩新桌面一个 HOME 候选 → 按 HOME 键必进新桌面。

### 备份与恢复
```bash
# 卸载前备份（可选但推荐）
adb shell pm path com.mitv.tvhome        # 输出 package:/system/vendor/app/TvHome/TvHome.apk
adb pull /system/vendor/app/TvHome/TvHome.apk backup-tvhome.apk

# 恢复（需要重开 ADB 后操作）
adb push backup-tvhome.apk /data/local/tmp/
adb shell pm install /data/local/tmp/backup-tvhome.apk
```

## 5. 小米电视系统极致精简（可选，配合桌面替换）

实测可卸载清单（小米电视 4C / build 20.3.6.2057），全部走 `pm uninstall --user 0` 通道：

```
# 系统广告 / 商店 / 娱乐 / 工具
com.miui.systemAdSolution        # 系统广告（拦截开机+桌面广告）
com.xiaomi.mitv.appstore         # 应用商店
com.mitv.shoplugin               # 商城插件
com.xiaomi.mitv.shop / com.xiaomi.mitv.pay  # 商城 / 快捷支付
com.xiaomi.mitv.payment          # 支付服务
com.xiaomi.tweather              # 天气
com.duokan.videodaily            # 视频日报
com.xiaomi.gamecenter.sdk.service.mibox  # 游戏中心
com.xiaomi.mitv.handbook         # 使用手册
com.mitv.gallery / com.xiaomi.tv.gallery  # 时尚画报
com.xiaomi.mitv.karaoke.service  # K歌
com.xiaomi.screenrecorder        # 录屏
com.xiaomi.miplay                # 投屏
com.xiaomi.voicecontrol          # 语音控制
com.miui.tv.analytics            # 统计上报
com.xiaomi.mitv.tvmanager        # 桌面守卫（HOME 替换必卸）
com.mitv.tvhome                  # 老桌面 PatchWall（HOME 直达必卸）
```

> ⚠️ **自动重装机制**：小米固件内置「远程部署」，重启可能自动装回已卸载应用。实测需同时卸载部署链路三件套：
> ```
> com.xiaomi.mitv.tvpush.tvpushservice   # 小米推送（接收部署指令）
> com.xiaomi.mitv.upgrade                # 系统升级（执行预置应用恢复）
> com.xiaomi.mitv.appstore               # 应用商店（下载静默安装）
> ```
> 保留系统必需件：设置、系统UI、播放器、遥控服务、编解码器更新。

## 6. 已知限制（诚实说明）

- **HOME 直达仅小米电视实测**：其他品牌 Android 6 电视的默认 HOME 锁定机制可能不同，思路可复用（卸载系统桌面）。
- 视频类应用（如快TV）可能拦截返回键，多按几次或按 HOME 键。
- 新桌面应用区只显示**可启动的第三方应用**（符合原版设定），系统内部组件不显示。
- `com.xiaomi.mitv.upnp` 卸载始终失败（保留，无 UI，不影响使用）。

## 7. 构建环境（已验证）

- JDK 17（Temurin 17.0.20）
- Android SDK：platforms;android-34 + build-tools;34.0.0
- Gradle：项目自带 wrapper（国内可配腾讯镜像 / 阿里云 Maven）

```bash
cd launcher
./gradlew assembleRelease --console=plain --no-daemon
```

产物：`launcher/app/build/outputs/apk/release/app-release.apk`

## 8. 变更记录

| 版本 | 内容 |
|---|---|
| v1（本轮） | Android 6 全量适配 + HOME 直达 + 小米精简方案文档化 |

---

© 适配工作基于 Gathin23/cleanyourtv（MIT）。请在您的发布中保留原 LICENSE。
