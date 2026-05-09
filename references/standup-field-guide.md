# StandUp Android Security Field Guide

This note captures durable project-specific lessons from the StandUp security-audit work. Use it when continuing `D:\androidProject\StandUp` or adapting its detector strategy to another first-party Android app.

## Editing Rules

- 不要为了“代码变少”删除有效检测逻辑或中文注释。
- 只删除已验证无效、重复、误报高、或没有接入结果链路的代码。
- 新增检测必须同时考虑日志、结果模型、UI、风险评分和测试设备表现。
- 对可疑但不确定的检测项，宁可标为“人工复核”，不要写成“已检测到”。

## Important StandUp Files

- Java aggregation and UI: `DeviceRiskAnalyzer.java`, `SecurityProbe.java`, `MyKeyAttestation.java`, `AttestationTamperProbe.java`, `DataAdbDetector.java`, `NativeRiskProbe.java`.
- Native probes: `native-lib.cpp`, `tee_detector.cpp`, `data_adb_detector.cpp`, `sentry_detector.cpp`, plus any dedicated Zygisk/Frida/APatch/KernelSU probe file that currently exists.
- Android package visibility: keep `<queries>` aligned with package-based detectors so Android 11+ release builds can still query known packages.

## Device Matrix Remembered From Testing

- Xiaomi 10 / `4b978729`: APatch environment, often with ZygiskGadget and TrickyStore depending on config. Strong historical APatch evidence included `auth_superkey`; ZygiskGadget may exist as module environment even when not injected into StandUp.
- Redmi Note 11T Pro / `R8LFMF95DEEAKJT8`: SukiSU Ultra / SUSFS device. Kernel version marker such as `Ciallo` is strong when present, but mutable and should be treated as one strong clue, not the only possible basis.
- Pixel 5 / `0C041FDD400009`: often used as “only bootloader unlocked” baseline. Do not treat BL unlock, zero verifiedBootKey, or Unverified state alone as boot image patch or TrickyStore.
- Pixel 3 / `89JX0A8BM`: normal locked baseline. Avoid `/data/adb` parent-path and TrickyStore false positives here.
- Temporary KSU/root device / `2bfa8119`: temporary root without modules. Avoid weak Zygisk/Frida/Gadget false positives from thread names, Android Studio startup agents, or generic ART dirtiness.

## Detector Boundaries

### `/data/adb` and SELinux AVC

- `/data/adb` parent path denial is weak and common; do not show UI risk from it alone.
- Hunter-style AVC sniffing is only useful when a precise child-path or basename correlation is present, for example `/data/adb/modules` paired with `/proc/modules` in the same sniff window.
- Deduplicate AVC messages before logging/UI, because repeated `ls` or auditd lines can slow the scan and spam the same finding.

### TrickyStore / TEESimulator

- Separate these concepts:
  - 当前 UID 的 Keystore/Attestation 返回链路被改写。
  - 证书链命中泄露或吊销 keybox 种子库。
  - 设备环境可能存在完整性修复能力。
- A valid keybox may evade revocation checks, so “未命中泄露库” does not prove no module.
- Do not flag TrickyStore solely because a device is a commonly leaked-keybox model.
- Good active probes include challenge echo, key-pair/certificate pairing, canary PrivateKeyEntry round-trip, alias enumeration consistency, updateSubcomponent behavior, and attestation-key branch shape. Treat weak inconsistency as review-only unless it crosses the designed threshold.

### Bootloader / TEE / RootOfTrust

- BL unlock is medium-low risk by itself per user policy, not high risk.
- On BL-unlocked Pixels, `deviceLocked=false`, `verifiedBootState=Unverified`, or zero boot key can be expected. Do not call that boot-image patch unless there is additional evidence.
- TEE/RootOfTrust explanations should be separate from TrickyStore family attribution.

### APatch

- `auth_superkey` side signal is historically strong when validated, but APatch/KernelPatch fixes can reduce reliability. Keep it as one APatch-specific evidence item, not a universal root detector.
- Avoid old syscall-45 timing logic if it showed false positives across BL-only and APatch devices.
- Package evidence and root-shell module evidence are useful, but keep them separate from current UID attestation evidence.

### KernelSU / SukiSU / SUSFS

- Normal App UID supercall probes are often blocked by seccomp. A blocked supercall is `unsupported`, not detection.
- Kernel version markers such as `Ciallo` are strong if present but mutable.
- Stronger conclusions should correlate package, kernel/mount/proc behavior, known SUSFS side effects, and attestation/root state when app-visible.

### Zygisk / ReZygisk / r0zygisk

- Do not resurrect old broad Zygisk checks that only produce weak suspicious logs.
- Current-process detection should require stronger evidence such as zygote-derived artifacts, explicit memfd/fd/maps markers, module-injected native payload, or multiple independent runtime anomalies.
- Weak items such as one ART Private_Dirty hit, generic JIT cache count, or AtexitArray gap should stay `suspicious_only` unless combined with stronger evidence.

### Frida / Gadget / ZygiskGadget

- Split two layers:
  - Module environment: packages such as `com.xiaojia.xgj`, root-visible `module.prop`, config JSON, or gadget module files.
  - Current process injection: explicit maps/fd/socket/memfd/port/native-dir residue in StandUp itself.
- Strong current-process evidence includes `libgadget`, `libhhh`, suspicious memfd ELF such as a renamed Frida payload, fd/socket to Frida/Gadget, standard ports `27042/27043` or observed module port `14725`, and unexpected `.so` / `.config.so` in StandUp native lib directory.
- Android Studio `code_cache/startup_agents/...-agent.so` is usually a debugging artifact; do not flag it unless explicit Frida/Gadget/Gum/froda/linjector markers exist.
- Thread names such as `Timer-0` or generic `Binder:*` are weak. They can assist only when stronger evidence exists.

## UI and Risk Scoring

- Top-level ordering preferred by the user:
  1. Root 环境检测
  2. 核心完整性检测
  3. 内核/挂载模块痕迹
  4. 风险检测
  5. Hook/动态调试检测
  6. 设备标识一致性
- Sort cards and subitems by severity: 高危 -> 警告 -> 正常.
- Root, Zygisk, TrickyStore, Frida/Gadget, LSPosed, APatch, KernelSU/SukiSU/SUSFS should be high risk only when strong evidence is present.
- BL 解锁、ADB/开发者、普通风险 App、设备指纹差异 are medium-low risk unless correlated with stronger evidence.
- UI must not show an outer warning when every expanded child is normal.

## Verification Habits

- For Java changes: `.\gradlew.bat :app:compileDebugJavaWithJavac`.
- For native changes: `.\gradlew.bat :app:externalNativeBuildDebug` or targeted NDK `clang++ -fsyntax-only` when practical.
- When compile errors are pasted, patch the reported file first before widening scope.
- Keep expensive probes cached per scan. Repeated auth_superkey, AVC sniff, or attestation probes should not run multiple times for one scan.

