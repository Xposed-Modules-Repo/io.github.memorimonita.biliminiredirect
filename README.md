# Bili小程序重定向

还在为群友转发的B站视频在QQ小程序只能跳转到官方 App而麻烦吗？此模块可以跳转到第三方客户端！

默认目标 **PiliPlus** 。别的客户端改个包名也能用。

---

## 装了什么

**一个 APK 就够**，不用刷 Magisk 模块。

它同时做两件事：

| 层 | 做什么 |
|---|---|
| **Java（Xposed）** | 拦截 `JefsClass`，把「打开哔哩哔哩」改成打开 PiliPlus |
| **Native（内嵌 .so）** | 屏蔽 QQ 风控库读 `/proc/self/smaps`，避免被检测踢下线 |

---

## 为什么光有 Java hook 不够

QQ 小程序点「打开APP」走的是**包名定向** Intent，系统层的默认应用设置根本没有介入点，只能在宿主进程内 hook。这部分由 Xposed 完成。

但只做这个会被 QQ **踢下线**（实测错误码 `ACCOUNT_KICKED`、`ACCOUNT_EXPIRED (w21)`，`isSameDevice=false`）。

根因藏在 native 层：QQ 的风控库 **`libfekit.so`（7.6 MB）会 `fopen("/proc/self/smaps")` 读取进程内存映射**，扫里面的注入痕迹（LSPosed 的 so、模块路径）上报服务端。

**LSPosed 的 Java hook 拦不住 native 的 `fopen`** —— 这就是为什么之前试过「主进程不 hook」「只在小程序进程 hook」「改 hook 到 system_server」全部无效。

解法来自 [TCQT](https://github.com/callng/TCQT)：在 native 层把 `libfekit.so` 对 `/proc/self/smaps` 的读取重定向到 `/dev/null`。

---

## 实现

### Java 层：拦截跳转

QQ 的跳转链路（从 QQ 的 `classes.dex` 反编译确认）：

```
JefsClass.startActivity(Context,Intent)        [PUBLIC STATIC]
  └→ checkAndDoSync(Context,Intent,Runnable)   [PUBLIC]
       └→ checkAndDoAsyn(...)                  [PUBLIC]
            └→ intercept(...)                  [PRIVATE]
```

hook 点选公开的 `startActivity` / `checkAndDoSync`（签名稳定），不碰 private 的 `intercept`：

1. `JefsClass#startActivity(Context, Intent)` 和 `(Context, Intent, Bundle)` —— 静态入口
2. `JefsClass#checkAndDoSync(Context, Intent, Runnable)` —— 实例入口
3. `android.app.Instrumentation#execStartActivity` —— 所有 Activity 启动的收口
4. `ContextWrapper` / `ContextImpl#startActivity` —— 直接调 Context 的那批

命中后自己 `ctx.startActivity(重定向后的 Intent)` 再 `setResult(null)` 掐掉原流程，连「即将离开QQ」弹窗一起跳过。

> 为什么不能只改 `param.args` 里的 Intent：原流程那个 `Runnable` 是调用方构造的闭包，
> 里面捕获的是旧 Intent 的引用，改参数对它无效。

重定向时保留原始 `data` URI，所以能直达对应视频。

### Native 层：屏蔽 smaps 读取

`libstealth.so` 随 APK 打包，模块在 `Application#onCreate` 后 `System.load()` 加载，然后：

1. 轮询 `dl_iterate_phdr` 等 `libfekit.so` 加载（它通常要 0.5~1.5 秒）
2. 解析它的 `PT_DYNAMIC` → `DT_STRTAB` / `DT_SYMTAB` / `DT_JMPREL`
3. 遍历 `.rela.plt`，找符号名为 `fopen` 的 `R_AARCH64_JUMP_SLOT`
4. `mprotect` 让 GOT 页可写，原子替换成自己的 `stealth_fopen`，原地址存起来
5. `stealth_fopen` 遇到 `/proc/self/smaps` 就返回 `/dev/null` 的句柄，其他路径原样转发

只改 `libfekit.so` 自己那张 GOT，**不影响全局 `fopen`**。

> 为什么不直接用 Zygisk 自带的 `pltHookRegister`：实测在 ZygiskNext 上
> `pltHookCommit` 一直返回 false（库已加载、dev/inode 也对）。TCQT 自己实现
> PLT hook 也是同样原因。

---

## 安装

1. 装 `BiliMiniRedirect.apk`
2. LSPosed 管理器 → 模块 → 启用「Bili小程序重定向」
3. **作用域勾选**：
   - `QQ`（`com.tencent.mobileqq`）—— 必须
   - `微信`（`com.tencent.mm`）—— 可选
   - ❌ 不要勾「系统框架」
4. 强制停止 QQ（设置 → 应用 → QQ → 强制停止）
5. 打开模块 App，确认「目标应用包名」= `com.example.piliplus`，保存

---

## 验证

装好后强制停止 QQ 再打开，看日志：

```bash
adb logcat -s BiliStealth BiliRedirect
```

**期望看到：**

```
# native 规避生效
BiliStealth: nativeStart: lib=libfekit.so block=/proc/self/smaps
BiliStealth: PLT hook installed for libfekit.so (poll #33)
BiliStealth: intercept fopen(/proc/self/smaps) -> /dev/null (count=1)

# Java 跳转生效
BiliRedirect: 注入 com.tencent.mobileqq / com.tencent.mobileqq:mini
BiliRedirect [Jefs.checkAndDoSync] pkg=tv.danmaku.bili ...  =>  pkg=com.example.piliplus ...
```

`intercept fopen(...)` 这行是关键 —— 说明真的拦到了风控库的读取。

---

## 备选：分离版（Zygisk 模块）

如果合并版的 native 部分在你的设备上加载不了（`System.load` 失败、或 `libstealth.so` 没被解压），
可以用分离版：`BiliStealth-zygisk-v1.0.zip` 是独立的 Zygisk 模块，功能相同。

1. KernelSU / Magisk → 模块 → 从本地安装 → 选这个 zip
2. 完整重启
3. 装 `BiliMiniRedirect.apk`（此时它的 native 部分会因 `libstealth.so` 已存在而跳过或重复，建议先确认）

源码在 [`stealth-zygisk/`](stealth-zygisk/)。

**这个版本实测通过的证据**（小米 25060RK16C / HyperOS 3.0 / Android 16）：

```
BiliStealth: preAppSpecialize: target process com.tencent.mobileqq:MSF
BiliStealth: PLT hook installed for libfekit.so (poll #30)
BiliStealth: PLT hook installed for libfekit.so (poll #33)
BiliStealth: PLT hook installed for libfekit.so (poll #27)
BiliStealth: PLT hook installed for libfekit.so (poll #23)
BiliStealth: intercept fopen(/proc/self/smaps) -> /dev/null (count=1)
```

---

## 设置项

| 项 | 默认 | 说明 |
|---|---|---|
| 启用重定向 | 开 | 总开关 |
| 同时接管隐式链接 | 开 | `bilibili://`、`b23.tv`、`*.bilibili.com` 这类没写死包名的 Intent |
| 跳过「即将离开QQ」确认弹窗 | 开 | 关掉则保留 QQ 的确认框，只换目标 |
| 日志取证 | 关 | 记录所有跳转 Intent，排查用 |
| 目标应用包名 | `com.example.piliplus` | 换成别的第三方客户端就填它的包名 |

---

## 已知限制

- QQ 版本更新后 `JefsClass` 可能改名/改签名 → 日志会打「JefsClass 里没找到可 hook 的方法，退回通用 hook」，此时靠第 3/4 层兜底（但会保留 QQ 的确认弹窗）
- 只屏蔽 `/proc/self/smaps`。如果 QQ 改读别的文件做检测（`/proc/self/maps`、`/proc/self/status`），需要在 `stealth_jni.cpp` 的 block 列表里补
- 如果 QQ 把风控挪到别的 so（不走 `libfekit.so` 的 `fopen`），这套方案会失效
- HyperOS 的「链式启动管理」（`com.miui.securitycenter`）可能在系统层再弹一次确认，本模块不处理
- 只接管 B 站系包名：`tv.danmaku.bili` / `tv.danmaku.bilibilihd` / `com.bilibili.app.in` / `com.bilibili.bilithings` / `com.bilibili.app.blue`

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
- Android NDK r27（编译 `libstealth.so`）
- [Xposed API 82](https://api.xposed.info/)（`api-82.jar`）
- JDK 17+

> Windows 上火绒 HIPS 会拦截 `aapt2.exe` / `zipalign.exe` / `ld.lld.exe` 在工程目录写文件
> （报「数据无效」或「Permission denied」）。`build.sh` 里已把它们改名成 `_bt_aapt.exe`
> 等副本，并把 native 产物先链接到临时目录再拷回来规避。

---

## 相关项目

- [TCQT](https://github.com/callng/TCQT) —— 本项目的 native 规避思路直接来自它（`libfekit.so` 的 `fopen` 劫持），PLT hook 结构也参考了它的实现。感谢作者。
- [PiliPlus](https://github.com/bggRGjQaUbCoE/PiliPlus) —— 第三方 B 站客户端，默认目标、也是实测用的那个。
- [LSPosed](https://github.com/LSPosed/LSPosed) —— 模块框架。
- [QAuxiliary](https://github.com/cinit/QAuxiliary) —— 它的 `JumpController` 是 hook `JefsClass` 的早期实践，hook 点选择参考了它。

---

## 许可

MIT
