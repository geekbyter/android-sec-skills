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
- 对同类新资料只吸收能通过 App 可见性和基线验证的机制，不按时间顺序覆盖旧结论。

## StandUp 近期要点

- Frida/Gadget 当前注入优先看当前进程：`/proc/self/maps`、`fd`、`memfd`、`/proc/net/tcp*` 当前 UID 端口、native 目录残留和内存字符串。
- APatch WebUI 版 zygiskGadget 可能把载荷复制到 App 私有目录后 `dlopen` 并删除文件；强特征是 `/data/data/<pkg>/libhhh.so (deleted)` 或 `/data/user/0/<pkg>/libhhh.so (deleted)`，再叠加 `frida:rpc` / `LIBFRIDA` / `Gum*` 更强。
- `/data/adb/zygisk_gadget/config.json` 中 `inject=true` 是明确目标配置；`inject=false` 是配置残留或开关关闭，属于异常环境，但不能写成当前进程已注入。
- APatch/zygiskGadget WebUI 是否打开只做上下文，不做主检测。UI 关闭后注入仍可存在，UI 打开也不等价于目标进程已被注入。
- `code_cache/startup_agents/...-agent.so` 默认只记录信息；只有路径、内存、fd 或线程里出现明确 Frida/Gadget/Gum/linjector 等标记时才升级。
- ReZygisk 不应靠宽泛进程名或 `/data/adb` 父路径定罪；优先看 App Zygote live policy 中 `RAW_ZYGISK_FILE_VALID=1` / `FAMILY_REZYGISK_POLICY=1`。命中和未命中都要在 logcat 打 `Zygisk 模块环境[SELinux] status=...`，方便现场判断探针是否真正跑完。

## 推荐验证

- Java 改动：`.\gradlew.bat :app:compileDebugJavaWithJavac`
- APK 打包：`.\gradlew.bat :app:assembleDebug`
- native 改动：`.\gradlew.bat :app:externalNativeBuildDebug`，或针对单个 C/C++ 文件做 NDK `clang++ -fsyntax-only`
- 误报敏感检测：至少用干净基线、BL-only 基线和已知模块设备做对照。
