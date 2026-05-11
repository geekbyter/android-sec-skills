# Android Security Researcher

这个 skill 用于第一方、授权范围内的 Android 安全研究和加固工作，尤其适合 `StandUp` 这类需要把 root、Zygisk、Frida/Gadget、TrickyStore、APatch、KernelSU/SukiSU、TEE/Attestation 检测接入到真实 App 的项目。

## 什么时候使用

- 审计或加固自己拥有的 Android App。
- 分析 APK / native so / JADX / IDA / Binary Ninja / adb 运行态证据。
- 对比本地参考项目，提炼可靠检测思路并移植到现有 App。
- 修复检测误报、漏报、日志缺失、UI/风险评分不一致等问题。
- 需要区分 App UID、adb shell、root shell 能看到的不同证据边界。

不用于绕过第三方防护、隐藏恶意行为、规避检测或攻击未授权目标。

## 使用方式

在任务里提到 `android-security-researcher`，或提出类似下面的需求时，Codex 应主动加载这个 skill：

```text
使用 android-security-researcher 分析这个 APK
帮我增强 StandUp 的 Frida/Gadget 检测
通过 adb 验证这台设备上的 Root/TrickyStore 证据
参考本地检测项目，但要避免误报
```

实际工作时优先阅读：

- `SKILL.md`：核心流程、证据分级、MCP/ADB 使用原则。
- `references/standup-field-guide.md`：StandUp 项目专用经验、设备矩阵、UI/评分约束。
- `references/android-security-repo-landscape.md`：本地 Android 安全参考仓库的取舍原则。

## 核心判断原则

- 新资料只是候选证据，不因为出现得更晚就自动覆盖旧知识。
- 优先采信当前 App / 当前 UID 可见的运行态证据。
- 干净基线设备可以推翻误报型检测，尤其是 Pixel 3 `89JX0A8BM`。
- 强证据才进入 `detected`；弱信号进入 `suspicious_only`，权限/API 不满足进入 `unsupported`。
- UI、日志、聚合结果、风险评分必须一致，父级状态不能高于所有可见子项。
- 保留项目已有有效逻辑和中文注释，只修正有问题的链路。

## 推荐验证

- Java 改动：`.\gradlew.bat :app:compileDebugJavaWithJavac`
- APK 打包：`.\gradlew.bat :app:assembleDebug`
- native 改动：`.\gradlew.bat :app:externalNativeBuildDebug`，或针对单个 C/C++ 文件做 NDK `clang++ -fsyntax-only`
- 误报敏感检测：至少用干净基线、BL-only 基线和已知模块设备做对照。
