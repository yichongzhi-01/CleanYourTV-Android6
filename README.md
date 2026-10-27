# CleanYourTV — Android 6 适配版（老电视救星）

> **让 2016-2018 年间的安卓电视（Android 6.0）也能用上清爽的新桌面**

本仓库是 [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv) 的 **Android 6 源码适配分支**。

原版 CleanYourTV 要求 **Android 8.0+（API 26）**，大量 2016~2018 年的老电视（如小米电视 4/4C、小米电视 3S、各品牌 Android 6 盒子）被拒之门外。本项目把整个 launcher **降级适配到 Android 6.0.1（API 23）**，并在真机上完整验证：**启动不崩溃、遥控器全键可用、时钟正常、应用可打开**。

---

## ✨ 特性

- 🧹 **无广告、无推荐流**的极简电视桌面：问候语 + 时钟 + 常用应用 + 定时提醒
- 📺 **专门适配 Android 6.0.1**：原版一装就崩（`NoSuchMethodError`），本版开箱即用
- 🎮 **遥控器完整支持**：方向键导航、OK 键打开、HOME 键直达桌面
- 📦 **开箱即用**：`releases/` 目录提供已编译好的 APK，ADB 安装即可
- 🏠 **HOME 键直达**（小米电视）：卸载 PatchWall 老桌面后，按 HOME 键直接进新桌面（详见下文）

## ✅ 实测机型

| 机型 | 系统 | 结果 |
|---|---|---|
| 小米电视 4C 32 英寸（MiTV4A / xmen） | Android 6.0.1（SDK 23，build 20.3.6.2057） | ✅ 全部功能实测通过 |

理论上所有 **Android 6.0 及以上** 的电视/盒子均可运行（API 23+）。

---

## 🚀 快速开始（三分钟装好）

### 方法一：直接安装 APK（推荐，不用编译）

1. 从 [`releases/`](releases/) 下载 APK：
   - **`app-release-home.apk`** —— 注册为 HOME 的版本（推荐，配合下方"HOME 直达"步骤使用）
   - `app-release-android6.apk` —— 普通版本（仅作为应用启动）
2. 电视上开启 **ADB 调试**（设置 → 账号与安全 → ADB 调试），电脑与电视同一局域网
3. 电脑上安装（用任意 adb 工具）：

```bash
adb connect <电视IP>:5555
adb install app-release-home.apk
```

> 装完后桌面图标显示为 **「主屏幕 / Home」**（包名 `com.gathin.tvhome6`）。

### 方法二：从源码构建

需要 JDK 17 + Android SDK（platform 34 / build-tools 34.0.0）：

```bash
cd launcher
./gradlew assembleRelease
# 产物：launcher/app/build/outputs/apk/release/app-release.apk
```

### 🏠 让 HOME 键直达新桌面（小米电视专用）

Android 6 的系统 HOME 固定为系统自带桌面（PatchWall），**不会自动切换到第三方桌面**。实测有效方案：

1. 安装本 APK（HOME 版）
2. 卸载小米的桌面守卫 TvMgr（否则它会强制把前台切回 PatchWall）：

```bash
adb shell pm uninstall --user 0 com.xiaomi.mitv.tvmanager
```

3. 卸载老桌面 PatchWall（系统只剩新桌面一个 HOME，HOME 键即直达）：

```bash
adb shell pm uninstall --user 0 com.mitv.tvhome
```

> ⚠️ 卸载前建议备份：`adb pull /system/vendor/app/TvHome/TvHome.apk`（恢复时重新安装即可）。
>
> 其他品牌电视若同样锁 HOME，思路一致：卸载系统自带桌面，让新桌面成为唯一 HOME。

---

## 🛠 适配内容（Android 6 做了什么）

| # | 原版问题 | 修复 |
|---|---|---|
| 1 | `minSdk 26`（Android 8+ 才能装） | 降到 **minSdk 23** |
| 2 | 6 个文件用了 `java.util.function`（Java 8 API，Android 6 没有） | 开启 `coreLibraryDesugaring` + `desugar_jdk_libs` |
| 3 | R8 裁剪会丢掉 desugar 合成的 lambda 类（启动即崩） | 关闭 `minifyEnabled / shrinkResources` |
| 4 | `View.forceHasOverlappingRendering()`（API 26）3 处 | `SDK_INT >= 26` 守卫 |
| 5 | `registerDefaultNetworkCallback`（API 26） | `SDK_INT >= 26` 守卫 |
| 6 | `Configuration.getLocales()`（API 24）2 处 | 自建 `getLocalesCompat()` 兼容方法 |
| 7 | `PackageManager.getChangedPackages`（API 26）2 处 | 守卫 + 老系统降级为总是刷新 |

详细技术文档见 [`ANDROID6-PORT.md`](ANDROID6-PORT.md)。

---

## 📂 目录结构

```
CleanYourTV-Android6/
├── README.md              # 本文件
├── ANDROID6-PORT.md       # 适配技术文档（改动明细/安装/HOME直达/已知限制）
├── LICENSE                # MIT（原项目许可证，保留版权声明）
├── releases/              # 预编译 APK（免编译直接装）
│   ├── app-release-home.apk      # HOME 直达版（推荐）
│   └── app-release-android6.apk  # 普通版
└── launcher/              # 适配后的完整 Android 工程（独立可构建）
    ├── app/               # 应用源码（含全部适配改动，注释标明）
    ├── gradle/
    ├── build.gradle
    └── gradlew / gradlew.bat
```

---

## ❓ 常见问题

**Q：为什么包名是 `com.gathin.tvhome6` 而不是原来的？**
A：小米 TvMgr 桌面服务会拦截原包名（`com.gathin.tvhome`）并强制切回 PatchWall；换包名后不再被拦截。同时旧包与新版可共存，卸载不冲突。

**Q：装了之后原来桌面还能用吗？**
A：能。本版只新增一个桌面入口；只有当你主动卸载 PatchWall 后 HOME 键才直达新桌面。

**Q：新桌面里能打开哪些应用？**
A：所有可启动的第三方应用（如云视听快TV），以及系统设置等入口。

**Q：返回键被视频应用拦截怎么办？**
A：部分视频应用拦截返回键，多按几次或直接按 HOME 键即可回到桌面。

---

## 📄 许可与致谢

- 本项目基于 [Gathin23/cleanyourtv](https://github.com/Gathin23/cleanyourtv)（MIT License）改编
- **保留原作者版权声明**，见 [`LICENSE`](LICENSE)
- 字体 Outfit 遵循 SIL Open Font License 1.1，见 `launcher/FONT-LICENSE-OFL.txt`

> 分享让老电视重获新生 ✨ 如果你喜欢，请给原项目 ⭐
