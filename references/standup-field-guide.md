# StandUp Android Security Field Guide

This note captures durable project-specific lessons from the StandUp security-audit work. Use it when continuing `D:\androidProject\StandUp` or adapting its detector strategy to another first-party Android app.

## Editing Rules

- 不要为了“代码变少”删除有效检测逻辑或中文注释。
- 只删除已验证无效、重复、误报高、或没有接入结果链路的代码。
- 新增检测必须同时考虑日志、结果模型、UI、风险评分和测试设备表现。
- 对可疑但不确定的检测项，宁可标为“人工复核”，不要写成“已检测到”。
- 不要按资料出现时间来覆盖知识。用户新给的仓库、PR、脚本或检测点都只是候选输入，必须通过 app-UID 可见性、误报边界、设备矩阵和 UI 可解释性筛选。
- 如果参考项目的做法只是在旧弱信号外面换了名字，或者会破坏已验证的干净基线，不要移植；最多保留为日志或 `suspicious_only`。

## Important StandUp Files

- Java aggregation and UI: `DeviceRiskAnalyzer.java`, `SecurityProbe.java`, `MyKeyAttestation.java`, `AttestationTamperProbe.java`, `DataAdbDetector.java`, `NativeRiskProbe.java`.
- Native probes: `native-lib.cpp`, `tee_detector.cpp`, `data_adb_detector.cpp`, `sentry_detector.cpp`, plus any dedicated Zygisk/Frida/APatch/KernelSU probe file that currently exists.
- Android package visibility: keep `<queries>` aligned with package-based detectors so Android 11+ release builds can still query known packages.

## Device Matrix Remembered From Testing

- Xiaomi 10 / `4b978729`: APatch environment, often with ZygiskGadget and TrickyStore depending on config. Strong historical APatch evidence included `auth_superkey`; ZygiskGadget may exist as module environment even when not injected into StandUp.
- Redmi Note 11T Pro / `R8LFMF95DEEAKJT8`: SukiSU Ultra / SUSFS device. Kernel version marker such as `Ciallo` is strong when present, but mutable and should be treated as one strong clue, not the only possible basis.
- Pixel 5 / `0C041FDD400009`: often used as “only bootloader unlocked” baseline. Do not treat BL unlock, zero verifiedBootKey, or Unverified state alone as boot image patch or TrickyStore.
- Pixel 3 / `89JX0A8BM`: normal locked baseline. Avoid `/data/adb` parent-path and TrickyStore false positives here. This device proved that Google Hardware Root shared serial `e8fa196314d2fa18` must not be treated as leaked keybox evidence.
- Temporary KSU/root device / `2bfa8119`: temporary root without modules. Avoid weak Zygisk/Frida/Gadget false positives from thread names, Android Studio startup agents, or generic ART dirtiness.

## Detector Boundaries

### `/data/adb` and SELinux AVC

- `/data/adb` parent path denial is weak and common; do not show UI risk from it alone.
- Hunter-style AVC sniffing is only useful when a precise child-path or basename correlation is present, for example `/data/adb/modules` paired with `/proc/modules` in the same sniff window.
- Deduplicate AVC messages before logging/UI, because repeated `ls` or auditd lines can slow the scan and spam the same finding.
- Hunter-style AVC sniffing may call `logcat -c`, which can erase earlier detector logs in the same scan. If another detector's result matters operationally, replay or log it after AVC sniffing so absence of logs is not mistaken for absence of detection.

### SELinux Live Policy / App Zygote

- App Zygote live-policy oracle is strong only when carrier context is truly `app_zygote`, control queries pass, and results are stable.
- `bindIsolatedService` instance names must use legal characters. A name like `sepolicy-<uuid>` can fail with `IllegalArgumentException: Illegal instanceName`; use alphanumeric/underscore-only names.
- Keep `unsupported_or_inconclusive` distinct from `clean`. If the carrier fails to bind, that is lack of usable evidence, not proof that Magisk/KSU/APatch policy is absent.
- Known validation: Xiaomi 10 `4b978729` produced strong APatch/Magisk-policy evidence; locked Pixel 3 `89JX0A8BM` produced clean live-policy evidence.

### TrickyStore / TEESimulator

- Separate these concepts:
  - 当前 UID 的 Keystore/Attestation 返回链路被改写。
  - 证书链命中泄露或吊销 keybox 种子库。
  - 设备环境可能存在完整性修复能力。
- A valid keybox may evade revocation checks, so “未命中泄露库” does not prove no module.
- Do not flag TrickyStore solely because a device is a commonly leaked-keybox model.
- Do not use the root/trust-anchor certificate serial as keybox leak evidence. Shared Google/OEM root serials are not device keyboxes and can create severe false positives on clean devices.
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
- Any hidden result that changes a parent card status must first be converted into a visible child signal. Parent status should be the max severity of visible child signals, not an independent secret verdict.

## Verification Habits

- For Java changes: `.\gradlew.bat :app:compileDebugJavaWithJavac`.
- For native changes: `.\gradlew.bat :app:externalNativeBuildDebug` or targeted NDK `clang++ -fsyntax-only` when practical.
- When compile errors are pasted, patch the reported file first before widening scope.
- Keep expensive probes cached per scan. Repeated auth_superkey, AVC sniff, or attestation probes should not run multiple times for one scan.
