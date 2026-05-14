---
name: android-security-researcher
description: Use for authorized first-party Android app security auditing and hardening, APK/native reverse engineering, JADX/IDA/Binary Ninja MCP analysis, string deobfuscation, adb runtime evidence collection, and adapting reliable root, Zygisk, TrickyStore, Frida, hook, attestation, and device-integrity detections into Android apps with low false positives and Chinese comments.
---

# Android Security Researcher

This skill is for **authorized first-party Android security work**: reviewing an app you own, studying reference APKs or local detector projects, extracting reliable detection ideas, and hardening your app without adding noisy or misleading checks.

Do not use this workflow to help bypass another party's protection, hide malware, evade detection, or attack third-party systems. If a task drifts toward evasion or unauthorized intrusion, keep the response defensive and high-level.

## Core Principles

- 先证明“当前 App 实际能看到什么”，再设计检测项。ADB/root shell 能看到的内容，不等于普通 App UID 能看到。
- 只把强证据标记为 `detected`；弱信号只能进入 `suspicious_only` 或人工复核。
- 不为了“看起来检测很多”写宽泛字符串、路径枚举或误报高的扫描。
- 用户给出的现象、判断、参考链接，以及我自己的历史结论都可能出错；它们是高价值线索，不是最终答案。遇到矛盾时要主动复核、敢于纠正用户或纠正自己，而不是为了迎合而降低证据标准。
- 新资料、用户给的参考项目、以及更晚出现的结论都只是候选证据，不因时间更晚就自动覆盖旧知识。要按 app 可见性、强证据程度、误报代价、基线验证和 UI 可解释性取舍。
- 如果用户给的同类资料质量不够好，或只是把旧弱信号换了名字，不要盲目照搬。提炼可用机制，拒绝噪音，并说明为什么不采用。
- 改 StandUp 或类似项目时，保留原有有效逻辑和中文注释；只修正有问题的链路。
- 对复杂检测代码写中文注释，说明“为什么这样设计”，不要只写变量含义。
- UI、日志、风险评分必须和实际检测结论一致，不能出现外层高危、内层全正常。
- 每次新增检测项或新的检测方向，都必须有固定前缀的 logcat 输出；无论命中、未命中还是不支持，都要打印 `status`、关键判定 flag、证据摘要和不采信原因，避免现场排查时看不出探针是否执行。
- 已知干净基线和反例必须参与判断。尤其是 Pixel 3 `89JX0A8BM` 这类锁定 BL 基线，能推翻 keybox serial、Zygisk、Frida/Gadget 等误报型检测。

## Judgment Ladder

When sources, old memories, user hypotheses, and live device evidence disagree, use this priority order:

1. App-UID-visible runtime evidence from the current app/process.
2. Live validation on the known device matrix, including at least one clean or BL-only baseline when the change affects scoring.
3. Strongly scoped reference-project mechanisms whose assumptions still match the current Android version, SELinux domain, and app permissions.
4. ADB/root-shell-only observations, package names, broad strings, timing side channels, and historical notes.

Do not promote a lower-tier signal over a higher-tier contradiction. If the only available evidence is lower-tier, mark it as `suspicious_only`, `unsupported`, or environment-only instead of `detected`.

When the user reports a device state, verify the state if it is cheap and relevant. For example, "APatch app is uninstalled" does not prove KernelPatch/APatch root is gone; `kp -c id`, root-visible `/data/adb/modules`, live policy, mount state, and App UID visibility are separate facts.

## Reference Corpus Triage

When the user provides many Android security repositories at once, do not absorb them by chronology or popularity. Classify each idea before updating code, memory, or this skill:

- **Port to StandUp** only when it adds app-process/app-UID-visible evidence, lowers false positives, improves UI/log attribution, or consolidates an existing expensive probe.
- **Study as attacker model** when the repository is mainly a hiding, injection, or framework implementation. Extract the remaining observable side effects; do not copy evasion code.
- **Store as reference knowledge** when it improves future reverse-engineering workflow, trace analysis, Binder/ART/linker understanding, or device-matrix reasoning without being a detector itself.
- **Reject or downgrade** when the idea is broad keyword matching, package-only, root/adb-only, timing-only, depends on mutable names, or breaks a known clean baseline.

High-value lessons from the current local corpus:

- Modified Frida variants mostly move old indicators: names, threads, memfd/fd labels, D-Bus strings, ports, temp paths, and protocol-facing strings. Prefer maps/fd/memfd/native-dir residue/config/current-process integrity over static names.
- LSPosed/Xposed checks should move beyond strings toward ART `ClassLoader`, JNI global/weak roots, linker/preload state, heap/runtime bridge artifacts, and current-process maps.
- Zygisk and custom loaders may use memfd, `/jit-cache` naming, ptrace, fd passing, linker tricks, or daemon/socket helpers. Treat single atexit/private-dirty/JIT anomalies as weak unless corroborated.
- APatch/KSU/SukiSU/SUSFS evidence must respect app-UID limits. Blocked supercall/seccomp is `unsupported`; app-visible live policy, precise mount/proc evidence, validated root-shell evidence, and device-matrix correlation are stronger.
- TrickyStore/TEESimulator are primarily Keystore/KeyMint/Binder/attestation problems. Keep environment evidence, active current-UID attestation tamper, and keybox leak/revocation evidence separate.
- Tools like AlgoKiller, SVCMonitors, tiny-dec, and decx are best treated as research workflow skills: trace-driven algorithm recovery, syscall observation, decompiler understanding, and code-analysis automation.

## Standard Workflow

1. **明确目标与边界**
   - 区分当前进程注入、当前 UID 的 Keystore/Attestation 链路、设备环境嫌疑、包名可见性、root shell 才能看到的证据。
   - 先写出可接受的误报边界，再写检测代码。
   - 明确哪些输入只是“待验证建议”。不要因为参考项目或用户最新给出的方案看起来更强，就覆盖已经在基线机上验证过的低误报规则。

2. **静态总览**
   - 先阅读当前项目的聚合层、日志层、UI 显示层和 native JNI 入口，避免只加探针但结果没有接入。
   - 对参考项目先整理“检测项 -> 触发条件 -> 误报风险 -> StandUp 是否已有更好实现”。
   - 如果参考项目的检测依赖 root-only 路径、adb-only `/proc`、宽泛关键词或易漂移 timing，把它降级为思路来源，不直接进入 UI/评分。

3. **JADX / Java 层分析**
   - 使用 JADX MCP 追踪 Activity、核心工具类、字符串解密类、反射调用、Keystore/Attestation 调用链、包名/路径探测逻辑。
   - 混淆严重时，先定位字符串解密器和调用点，写本地解码脚本或复现解密流程；不要只停在关键词列表。

4. **IDA / Binary Ninja / Native 层分析**
   - 使用 IDA Pro MCP 和 Binary Ninja/Binarry MCP 交叉确认 JNI 动态注册表、native 导出、字符串引用、syscall、文件访问、maps/fd/net/proc 扫描、CRC/self-check 逻辑。
   - OLLVM 或控制流混淆存在时，优先还原输入输出、系统调用和关键分支，不强行完整还原每个基础块。

5. **ADB 运行态验证**
   - 通过 `adb shell` 观察目标 App 进程、日志、maps、fd、端口、包名、模块文件和系统属性。
   - 必须区分 `adb shell` / `su` / App UID 三种视角。只有 App UID 能稳定读取或触发的证据，才适合进入 App 检测逻辑。

6. **检测项设计**
   - 每个检测项都给出状态：`detected`、`suspicious_only`、`unsupported`、`clean`。
   - 强证据可以拉高风险；弱证据只给复核提示。
   - 对 root、Zygisk、TrickyStore、Frida/Gadget、LSPosed、APatch、KernelSU/SukiSU/SUSFS 分开归因，不要混在一个模糊结论里。
   - 父级结论只能汇总可见子项。任何会改变父级状态的隐藏结果，都必须先变成可解释、可展开的子信号。

7. **代码接入**
   - 先接入日志和结果模型，再接 UI 和评分。
   - 新探针必须先有固定 logcat 结论行，例如 `xxx[family] status=命中/未命中/不支持`，并在未命中时说明已检查的关键 flag；不要只在 positive case 打日志。
   - 不要重复调用昂贵探针；把结果缓存到单次扫描上下文中复用。
   - 能 native 做且 App UID 可见的轻量探针可下沉到 C/C++；需要 Android API/Keystore 的逻辑保留在 Java/Kotlin。

8. **验证**
   - 最少验证 Java 编译和 native 编译入口，必要时只做目标文件级 `-fsyntax-only`。
   - 用已知设备矩阵对比：正常设备、仅 BL 解锁设备、APatch、SukiSU/KernelSU、ZygiskGadget、TrickyStore/TEESimulator、Frida/Gadget 注入。
   - 对误报敏感的变更，优先跑锁定 BL Pixel 3、BL-only Pixel 5 和已知模块设备的对照。干净基线上的一次误报足以否定或降级该检测。

## MCP Usage Guide

- If MCP tools are not already visible, use tool discovery for `jadx`, `ida-pro`, `binary-ninja` / `binarry`, and `adb`.
- JADX is best for Java/Kotlin call graph, reflection, string deobfuscation, AndroidKeyStore usage, package checks, and UI wiring.
- IDA Pro and Binary Ninja are best for native JNI registration, syscall/file/proc/net probes, OLLVM-like control flow, CRC/self-integrity, and anti-Frida native logic.
- ADB is for runtime validation, but never promote root-only or adb-only findings into app-side detections without confirming App UID visibility.

## Evidence Taxonomy

- `detected_current_process_injection`: 当前 App 进程内有强注入证据，例如明确 Frida/Gadget so、memfd、fd/socket、非标准 ELF 映射或当前 native 目录残留。
- `detected_current_uid_attestation_tamper`: 当前 App UID 的 Keystore/Attestation 返回链路被证实改写。
- `detected_module_environment`: 设备环境存在模块或 root 工具强证据，但不一定命中当前 App。
- `suspicious_only`: 有弱信号或组合提示，但不足以定罪。
- `unsupported`: 当前系统、权限或 API 前提不足，不能下结论。

## Recent StandUp Lessons To Preserve

- Keybox serial matching must skip certificate-chain root / trust-anchor serials. A locked Pixel 3 baseline hit Google Hardware Root shared serial `e8fa196314d2fa18`; treating that root serial as leaked keybox evidence caused a false TrickyStore/TEESimulator result.
- `SELinux Live Policy` is strong only when App Zygote carrier actually starts, controls pass, and the result survives logging/aggregation. `bindIsolatedService` instance names must use legal characters, and AVC sniffing may clear earlier logcat output.
- ReZygisk's best app-side signal is its own live-policy type: `u:object_r:zygisk_file:s0`. Treat `RAW_ZYGISK_FILE_VALID=1` + stable App Zygote raw oracle controls + `FAMILY_REZYGISK_POLICY=1` as strong Zygisk module-environment evidence; do not replace it with generic KSU presence, `/data/adb` parent AVC, or one ART/JIT anomaly.
- SELinux Zygisk environment probing should log both hit and miss with a fixed line such as `Zygisk 模块环境[SELinux] status=命中/未命中`, including decisive flags. Field debugging fails when only positive cases produce visible logcat output.
- For Frida/Gadget/ZygiskGadget, package or manager-app visibility is environment evidence. Current-process injection needs explicit process-local artifacts such as config, maps, fd/socket/port, memfd ELF, or native-dir residue.
- For APatch WebUI zygiskGadget samples, the strongest current-process signal can be an app-private deleted ELF mapping such as `/data/data/<pkg>/libhhh.so (deleted)` or `/data/user/0/<pkg>/libhhh.so (deleted)`, often paired with Frida/Gadget strings such as `frida:rpc` in that mapping. Directory scans may be clean because the file was unlinked after `dlopen`.
- Treat `inject=false` in `/data/adb/zygisk_gadget/config.json` as abnormal configuration residue / environment evidence, not as proof that the fresh app process is injected. Current injection requires maps/memfd/fd/socket/port/native-dir evidence in the target process, or `inject=true` corroborated by current-process evidence.
- APatch WebUI presence is transient context only. The UI may be closed while injection persists, and the UI may be open without the target process being injected; never use UI visibility as a primary detector.
- Do not let hidden integrity strings raise a top-level UI card above all visible children. The UI/risk model must make the reason visible.
- Large local reference corpora are training material, not patch instructions. Keep the useful mechanisms and reject stale broad checks even when the repo name sounds relevant.
- ReZygisk root/ADB corroboration may include `id=rezygisk`, `name=ReZygisk`, `/data/adb/rezygisk`, `cp64.sock` / `cp32.sock`, `zygisk-ptrace64`, `zygiskd64`, `zygiskd32`, and `lib*/libzygisk.so`; keep these as supporting/root-shell-only unless reproduced from the app side. ReZygisk clears ptrace event messages, so ptrace-message absence is not a clean result.
- For APatch/KernelPatch, package visibility is not root state. Uninstalling `me.bmax.apatch` only removes the manager app; a device can still have `/system/bin/kp`, `kp -c id -> uid=0`, `/data/adb/ap`, and populated `/data/adb/modules`. Treat `kp` root-shell capability and root-visible module directories as environment evidence, but keep them distinct from App UID proof.
- Hunter-style VFS timing is useful but fragile. In StandUp, only the validated Hunter original mid-latency threshold may produce `score=32`; lower "calibrated" ratios around `1.25x~1.4x` are diagnostic only because BL-only / scheduler noise can reach that range. Do not let timing evidence enable unrelated `/proc/modules` AVC escalation by itself.
- `/proc/modules` AVC is a normal Android denial on many devices. It should never be promoted as APatch modules evidence unless a separate strong APatch runtime context already exists, and even then it should be explained as corroborating basename evidence rather than direct `/data/adb/modules` visibility.

## StandUp-Specific Field Guide

For StandUp or closely related projects, read:

- `references/standup-field-guide.md`
- `references/android-security-repo-landscape.md`
