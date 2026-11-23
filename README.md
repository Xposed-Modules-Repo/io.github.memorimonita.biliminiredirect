# Bili小程序重定向

还在为群友转发的B站视频在QQ小程序只能跳转到官方 App而麻烦吗？此模块可以跳转到第三方客户端！

默认目标 **PiliPlus** 。别的客户端改个包名也能用。

---

## 2.0 改了什么

**不再往 QQ 里注入任何东西**，改成在系统框架（`system_server`）里改跳转目标。

| 版本 | 做法 | QQ 进程里的注入 |
|---|---|---|
| 1.x | 在 QQ 进程内 hook `JefsClass`，配 Zygisk 模块从 native 层屏蔽风控库读 `smaps` | 有（LSPosed + Zygisk） |
| **2.0** | 在 `system_server` 解析 Intent 时把目标包名换成第三方客户端 | **零** |

为什么换：1.x 的 native 屏蔽只能遮住一条检测路径，QQ 的 `libfekit.so` 还会从 `/proc/self/maps`、ART 完整性、CheckJNI 符号表等地方判断「进程被注入过」，每次都要抢在它上报之前把痕迹盖掉，很难赢。而 2.0 干脆不让 QQ 进程出现任何注入 —— 检测手段再强也没有可检测的东西。

---

## 原理

QQ 小程序点「打开APP」时，发出的是一个**目标包名为 `tv.danmaku.bili`** 的 `Intent`（实测数据见下）。系统在启动 Activity 前会由 `system_server` 的 `ActivityTaskSupervisor` 解析这个 Intent 该落到哪个 Activity —— 我们就在这一步把它改掉：

```java
// ActivityTaskSupervisor#resolveIntent / #resolveActivity（before）
if (目标是官方 B 站包名) {
    intent.setComponent(null);          // 清掉显式 component
    intent.setPackage("com.example.piliplus");   // 就地改成目标客户端
}
```

两个关键细节：

1. **就地改原对象**，不换新 Intent。`resolveIntent` 之后 `ActivityStarter.Request` 里存的还是同一个引用；换成新对象会出现「解析成 PiliPlus、真正启动却是 bili」的错位。
2. **保留 `data` / `extras` / action 原样**，所以 `bilibili://video/BVxxxx` 能直达对应视频，而不是只打开首页。

因为改的是系统层，不依赖 QQ 内部类名，QQ 怎么改版本都不受影响。

---

## 安装

1. 装 `BiliMiniRedirect.apk`
2. LSPosed 管理器 → 模块 → 启用「Bili小程序重定向」
3. **作用域只勾「系统框架」**（`android`）
   - ❌ 不要勾 QQ / 微信
4. **重启手机**（系统框架的 hook 只在开机时注入）
5. 打开模块 App，确认「目标应用包名」= `com.example.piliplus`，保存

> 作用域改动必须重启才生效；只强制停止 QQ 是不够的。

---

## 验证

重启后打开 QQ 小程序，点「开启哔哩哔哩应用」，看日志：

```bash
adb logcat -s LSPosedFramework | grep BiliRedirect
```

**期望看到：**

```
(system)[io.github.memorimonita.biliminiredirect,...] BiliRedirect: [sys] pkg=com.example.piliplus comp=null action=android.intent.action.VIEW data=bilibili://video/BV1rMyGzhE4A?...  (原目标 tv.danmaku.bili → com.example.piliplus)
```

同时确认 QQ 进程里**没有**本模块的记录：

```bash
adb logcat -d | grep LSPosedFramework | grep biliminiredirect | grep com.tencent
# 输出应为空
```

---

## 设置项

| 项 | 默认 | 说明 |
|---|---|---|
| 启用重定向 | 开 | 总开关 |
| 目标应用包名 | `com.example.piliplus` | 换成别的第三方客户端就填它的包名 |

---

## 已知限制

- **接管范围是全局的**：任何 App 显式指定打开官方 B 站（包名定向），都会被改到目标客户端，不只 QQ/微信。这正是「不注入 QQ」的代价。
- 目标客户端没装时不做改动，原流程照常（不会出现「Activity 不存在」）。
- 只接管 B 站系包名：`tv.danmaku.bili` / `tv.danmaku.bilibilihd` / `com.bilibili.app.in` / `com.bilibili.bilithings` / `com.bilibili.app.blue`。
- 系统框架 hook 依赖 `ActivityTaskSupervisor` 的 `resolveIntent` / `resolveActivity`，Android 大版本升级后需要复验（模块日志会打「hook 了 N 个 resolve 方法」，N=0 即不匹配）。
- HyperOS 的「链式启动管理」（`com.miui.securitycenter`）可能在系统层再弹一次确认，本模块不处理。

---

## 1.x 的旧方案（已弃用）

1.x 在 QQ 进程内 hook `JefsClass`，并用内嵌的 `libstealth.so` 把 `libfekit.so` 对 `/proc/self/smaps` 的 `fopen` 重定向到 `/dev/null`。实测仍会被踢（`ACCOUNT_KICKED` / `ACCOUNT_EXPIRED (w21)`，`isSameDevice=false`），因为 QQ 的检测不止这一条路径。

相关源码仍在仓库里（`src/.../Hook.java`、`src/stealth_jni.cpp`、`stealth-zygisk/`），但 2.0 默认不再使用；作用域只勾系统框架时它们不会被加载。

---

## 自己构建

不依赖 Gradle / Android Studio，用 `aapt2` + `javac` + `d8` + `apksigner`，native 部分用 NDK 的 clang。

```bash
ANDROID_SDK=~/Android/Sdk \
ANDROID_NDK=~/Android/Sdk/ndk/27.2.12479018 \
XPOSED_API=~/libs/api-82.jar \
JAVA_HOME=/path/to/jdk17 \
bash build.sh
```

需要：
- Android SDK，含 `build-tools/36.0.0/` 与 `platforms/android-36/android.jar`
- Android NDK r27（2.0 用不到 native，但构建脚本仍会编译它）
- [Xposed API 82](https://api.xposed.info/)（`api-82.jar`）
- JDK 17+

> Windows 上火绒 HIPS 会拦截 `aapt2.exe` / `zipalign.exe` / `ld.lld.exe` 在工程目录写文件
> （报「数据无效」或「Permission denied」）。`build.sh` 里已把它们改名成 `_bt_aapt.exe`
> 等副本，并把 native 产物先链接到临时目录再拷回来规避。

---

## 相关项目

- [PiliPlus](https://github.com/bggRGjQaUbCoE/PiliPlus) —— 第三方 B 站客户端，默认目标、也是实测用的那个。
- [TCQT](https://github.com/callng/TCQT) —— 1.x 的 native 规避思路来自它（`libfekit.so` 的 `fopen` 劫持），PLT hook 结构也参考了它的实现。感谢作者。
- [LSPosed](https://github.com/LSPosed/LSPosed) —— 模块框架。
- [QAuxiliary](https://github.com/cinit/QAuxiliary) —— 早期 hook `JefsClass` 的实践，1.x 的 hook 点选择参考了它。

---

## 许可

MIT
