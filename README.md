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
- 用户的判断、我自己的历史判断、参考项目的结论都可能出错；先把它们当线索验证，必要时要直接纠正，而不是顺着错误前提继续加检测。
- 优先采信当前 App / 当前 UID 可见的运行态证据。
- 干净基线设备可以推翻误报型检测，尤其是 Pixel 3 `89JX0A8BM`。
- 强证据才进入 `detected`；弱信号进入 `suspicious_only`，权限/API 不满足进入 `unsupported`。
- UI、日志、聚合结果、风险评分必须一致，父级状态不能高于所有可见子项。
- 保留项目已有有效逻辑和中文注释，只修正有问题的链路。
- 对同类新资料只吸收能通过 App 可见性和基线验证的机制，不按时间顺序覆盖旧结论。
- `adb shell`、root shell、App UID 是三种视角。能在 `kp` / root 下看到的东西，可以证明设备环境，但不能自动改写成当前 App UID 强证据。

## StandUp 近期要点

- Frida/Gadget 当前注入优先看当前进程：`/proc/self/maps`、`fd`、`memfd`、`/proc/net/tcp*` 当前 UID 端口、native 目录残留和内存字符串。
- APatch WebUI 版 zygiskGadget 可能把载荷复制到 App 私有目录后 `dlopen` 并删除文件；强特征是 `/data/data/<pkg>/libhhh.so (deleted)` 或 `/data/user/0/<pkg>/libhhh.so (deleted)`，再叠加 `frida:rpc` / `LIBFRIDA` / `Gum*` 更强。
- `/data/adb/zygisk_gadget/config.json` 中 `inject=true` 是明确目标配置；`inject=false` 是配置残留或开关关闭，属于异常环境，但不能写成当前进程已注入。
- APatch/zygiskGadget WebUI 是否打开只做上下文，不做主检测。UI 关闭后注入仍可存在，UI 打开也不等价于目标进程已被注入。
- `code_cache/startup_agents/...-agent.so` 默认只记录信息；只有路径、内存、fd 或线程里出现明确 Frida/Gadget/Gum/linjector 等标记时才升级。
- ReZygisk 不应靠宽泛进程名或 `/data/adb` 父路径定罪；优先看 App Zygote live policy 中 `RAW_ZYGISK_FILE_VALID=1` / `FAMILY_REZYGISK_POLICY=1`。命中和未命中都要在 logcat 打 `Zygisk 模块环境[SELinux] status=...`，方便现场判断探针是否真正跑完。
- APatch/KP 不能只看管理器包名。卸载 `me.bmax.apatch` 不代表 root 核心消失；`/system/bin/kp`、`kp -c id -> uid=0`、root-visible `/data/adb/ap` 和 `/data/adb/modules` 说明环境仍在。包名、root-shell 证据、App UID 证据要分层展示。
- Hunter-style VFS timing 只能在验证过的阈值上作为强证据；`1.25x~1.4x` 一类校准弱区间只做 logcat 诊断。`/proc/modules` AVC 是通用拒绝，不能被 timing 单独带成 APatch modules 高危。
- 工具层面优先用 `rg` 搜索代码；如果当前 Codex 子进程仍命中 WindowsApps 包内 `rg.exe` 并报 `Access is denied`，先用 `Select-String` / `Get-ChildItem` 继续工作，再单独修 PATH，不要让搜索工具问题影响判断。

## 推荐验证

- Java 改动：`.\gradlew.bat :app:compileDebugJavaWithJavac`
- APK 打包：`.\gradlew.bat :app:assembleDebug`
- native 改动：`.\gradlew.bat :app:externalNativeBuildDebug`，或针对单个 C/C++ 文件做 NDK `clang++ -fsyntax-only`
- 误报敏感检测：至少用干净基线、BL-only 基线和已知模块设备做对照。
