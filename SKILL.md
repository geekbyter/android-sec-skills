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
- 改 StandUp 或类似项目时，保留原有有效逻辑和中文注释；只修正有问题的链路。
- 对复杂检测代码写中文注释，说明“为什么这样设计”，不要只写变量含义。
- UI、日志、风险评分必须和实际检测结论一致，不能出现外层高危、内层全正常。

## Standard Workflow

1. **明确目标与边界**
   - 区分当前进程注入、当前 UID 的 Keystore/Attestation 链路、设备环境嫌疑、包名可见性、root shell 才能看到的证据。
   - 先写出可接受的误报边界，再写检测代码。

2. **静态总览**
   - 先阅读当前项目的聚合层、日志层、UI 显示层和 native JNI 入口，避免只加探针但结果没有接入。
   - 对参考项目先整理“检测项 -> 触发条件 -> 误报风险 -> StandUp 是否已有更好实现”。

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

7. **代码接入**
   - 先接入日志和结果模型，再接 UI 和评分。
   - 不要重复调用昂贵探针；把结果缓存到单次扫描上下文中复用。
   - 能 native 做且 App UID 可见的轻量探针可下沉到 C/C++；需要 Android API/Keystore 的逻辑保留在 Java/Kotlin。

8. **验证**
   - 最少验证 Java 编译和 native 编译入口，必要时只做目标文件级 `-fsyntax-only`。
   - 用已知设备矩阵对比：正常设备、仅 BL 解锁设备、APatch、SukiSU/KernelSU、ZygiskGadget、TrickyStore/TEESimulator、Frida/Gadget 注入。

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

## StandUp-Specific Field Guide

For StandUp or closely related projects, read:

- `references/standup-field-guide.md`

