# Android Security Repository Landscape

This reference summarizes the local repository families used as StandUp security-audit training material. It is intentionally defensive: use these notes to understand what attackers hide and which app-visible signals remain useful. Do not use it to build or improve evasion tooling.

## Corpus Intake Verdict

Use this local corpus as a judged evidence library, not as a source to copy blindly.

- **Best detector references to adapt carefully**: `Duck-Detector-Refactoring`, `XposedDetector`, `RiskEngine`, `sentry`, `AntiFrida`, `DetectFrida`, `frida-detection`, `KeyDetector`, `EnvCheck`, `NativeDetector`, and targeted parts of `Android-Native-Root-Detector`.
- **Best attacker-model references**: official `frida`, modified Frida variants (`ajeossida`, `Florida`, `phantom-frida`, `strongR-frida-android`, `undetected-frida`), LSPosed variants, ZygiskNext/NeoZygisk/r0zygisk/ReZygisk, TrickyStore/TEESimulator, APatch/KSU/SukiSU/SUSFS, `HideApk`, `MyInjector`, `Zygisk-MyInjector`, and `kpm-panda-hide`.
- **Best research workflow references**: `AlgoKiller` for large ARM64 trace reasoning, `SVCMonitors` for syscall/runtime observation, `tiny-dec` for decompiler principles, and `decx` for JADX-based code-analysis workflow.
- **Historical or low-confidence references**: `rootbeer`, old fixed-port/name Frida demos, broad package/path scanners, and detector projects that only repackage `/data/adb`, `su`, or keyword checks. Keep as context, not high-risk logic.

If a repo is mainly a hiding implementation, the useful question is: "after the hiding step, what app-visible side effect remains?" If that answer is unclear, do not port a detector yet.

## Frida / Gadget Family

Local sources include official Frida plus modified variants such as `ajeossida`, `Florida`, `phantom-frida`, `strongR-frida-android`, and `undetected-frida`, and detector projects such as `AntiFrida`, `DetectFrida`, and `frida-detection`.

Common hidden features in modified Frida builds:

- Process and binary names such as `frida-server`, `frida-agent`, and `libfrida-agent`.
- Thread names such as `gum-js-loop`, `gmain`, `gdbus`, `pool-frida`, and `pool-spawner`.
- Memory strings and symbols such as `frida_agent_main`, `LIBFRIDA`, `frida:rpc`, `GumScript`, and `GumV8`.
- memfd / fd names such as `frida-agent-64.so`, often changed to benign names like `jit-cache`.
- Default ports and protocol surface such as `27042`, `27043`, and D-Bus-style interaction.
- Temporary paths and asset paths containing `frida`.

Defensive detector lessons:

- Strong current-process evidence is better than process-name scanning: maps/fd/memfd, loaded ELF shape, Gadget config JSON, unexpected native lib directory residue, and modified executable code pages.
- Thread names and default ports are useful but easy to rename; treat them as supporting evidence unless combined with stronger process-local artifacts.
- A robust Gadget check should detect `xxx.so + xxx.config.so` in the current app native directory and parse JSON fields such as `interaction`, `type`, `listen/connect`, `on_load`, `port`, and `address`.
- Android Studio `code_cache/startup_agents` is usually benign and should not be promoted without explicit Frida/Gadget markers.
- Modified builds such as `ajeossida` and `phantom-frida` explicitly patch memfd names, thread names, agent/gadget strings, D-Bus names, ports, temp paths, and binary residual strings. Therefore a future detector must assume names are mutable and lean on structure: executable memfd ELF, fd/socket state, loaded payload shape, and app-local Gadget residue.

## Xposed / LSPosed Family

Local sources include `LSPosed`, `LSPosed_mod`, `LSPosed-Irena`, `ReLSPosed`, `rxposed`, `MyInjector`, `Zygisk-MyInjector`, and detector `XposedDetector`.

Common hidden features:

- Java/API strings such as `XposedBridge`, `XposedHelper`, `LSPosed`, `LSPosed-Bridge`, `lspd`, and package names.
- Runtime library names such as `libxposed`, `liblspd`, `edxposed`, and bridge dex/so files.
- Hook framework classes visible through normal ClassLoader paths.

Defensive detector lessons:

- String/package checks are low-cost but easy to rename.
- Stronger checks inspect ART roots, JNI weak/global references, ClassLoader instances, linker solist/preload state, runtime maps, and heap strings.
- Xposed/LSPosed UI detection should avoid broad keyword-only high risk; require runtime evidence in the current process when labeling current-app injection.
- `XposedDetector` is more valuable for its ART/ClassLoader/global-ref/PLT approach than for any one literal string. LSPosed forks and `rxposed` should be read mainly to understand how framework state can be moved or renamed.

## Magisk / APatch / KernelPatch

Local sources include `MagiskDetector`, `MagiskTest`, `APatch`, modified APatch/FolkPatch, and `KernelPatch`.

Useful concepts:

- Classic Magisk module detection compares `/proc/self/mountinfo` data partition device with `/proc/self/maps` entries that look like `/system`, `/vendor`, `/product`, or `/system_ext` but are backed by data.
- APatch / KernelPatch expose supercall and KPM concepts, but app-UID probing can be gated by superkey, allowlist, seccomp, or fixes to side-channel behavior.
- APatch has characteristic management/package/module paths and can change `su` path to custom names such as `kp`, so `su` path checks are insufficient.

Defensive detector lessons:

- Do not rely on old syscall timing side channels as universal evidence.
- Strong APatch evidence should be explicit package, validated `auth_superkey`, app-visible mount/source mismatch, precise AVC/path correlation, or root-shell read-only module confirmation.
- Treat inaccessible `/data/adb` parent paths as weak and do not show UI risk from EACCES alone.
- APatch/KPM sources are useful for understanding `supercall`, `resetprop`, module directories, sepolicy loading, and kernel module capability. A normal app must not assume it can query those directly.

## KernelSU / SukiSU / SUSFS

Local sources include `KernelSU`, `KernelSU-Next`, `ReSukiSU`, `SukiSU-Ultra`, and `susfs4ksu-module`.

Useful concepts:

- KSU/SukiSU use supercall/ioctl-style interfaces with allowlist and seccomp constraints.
- SUSFS is designed to hide filesystem and mount traces, so app-visible signals may be deliberately reduced.
- Kernel version markers such as `Ciallo` are high-signal when present but mutable.

Defensive detector lessons:

- A blocked supercall is `unsupported`, not detection.
- Stronger conclusions should correlate package, root shell behavior, mount/proc visibility, SUSFS side effects, and attestation/root state.
- Avoid making kernel version string the only high-risk trigger.
- `susfs4ksu-module` is a hiding implementation; treat it as a reason to prefer corroborated negative/positive evidence, not as a stable string list.

## Zygisk Family

Local sources include `ZygiskNext`, `NeoZygisk`, `r0zygisk`, `ReZygisk`, `DetectZygisk`, and `ZygiskDetector`.

Useful concepts:

- Projects use loader/daemon/socket/mountinfo helpers and may use memfd or dl-based loading.
- Simple ptrace event-message and libc internals checks are interesting but device/ROM fragile.

Defensive detector lessons:

- Weak artifacts such as one ART dirty page, JIT noise, or a generic atexit gap should not drive UI high risk.
- Stronger evidence should be current-process injection, explicit zygisk module environment, zygote-derived ptrace residue proven app-visible, or current app native/memfd artifacts.
- `DetectZygisk` ptrace and `ZygiskDetector` libc/AtexitArray style checks are research probes. They need ROM/API validation and should start as `suspicious_only` unless paired with stronger app-visible evidence.
- ReZygisk is best learned as a concrete example of "hiding changes names, but stable policy side effects remain": it defines `zygisk_file`, labels `/data/adb/rezygisk` socket state, uses `cp64.sock` / `cp32.sock`, `zygisk-ptrace*`, and `zygiskd*`, and clears ptrace event messages. For StandUp, the durable detector lesson is the App Zygote raw-context oracle for `u:object_r:zygisk_file:s0`, not broader process-name scanning.
- A negative ptrace-message result is especially weak against ReZygisk because clearing that message is part of the design. Treat ptrace, JIT, and atexit probes as research/supporting signals unless live policy or current-process injection artifacts corroborate them.

## TrickyStore / TEESimulator

Local sources include `TrickyStore`, `TEESimulator`, and `TrickyStoreOSS`.

Useful concepts:

- These projects intercept Keystore/KeyMint Binder flows and can patch or generate attestation certificates.
- Keybox, target list, certificate generation, KeyMetadata, AuthorizationList, and RootOfTrust branches are the important surfaces.

Defensive detector lessons:

- Keep three layers separate: current UID attestation tamper, keybox revocation/leak evidence, and broader environment suspicion.
- Valid keyboxes can evade revocation checks, so negative revocation results do not prove no module.
- The strongest app-side strategy is active attestation consistency: challenge echo, key pair/certificate pairing, canary PrivateKeyEntry round-trip, alias/updateSubcomponent consistency, certificate chain shape, and attestationKey branch behavior.
- TEESimulator moves from leaf patching toward generated virtual keys. That makes "certificate looks plausible" weaker and increases the value of consistency probes across alias, challenge, key usage, certificate chain, and KeyStore side effects.

## General Detector Repos

Relevant local sources include `Duck-Detector-Refactoring`, `Kknd_Root_Detector`, `RiskDetector`, `Ruru`, `rootbeer`, `NativeDetector`, `RikkaX`, `launch`, `sentry`, `KeyDetector`, `violet_Box`, `EmulatorDetection`, `EnvCheck`, `Android-Native-Root-Detector`, `UprobeChecker`, and `Vector`.

High-value patterns:

- Duck Detector has useful native memory modules: maps anomaly, fd anomaly, signal handler checks, linker object mismatch, function hook range checks, LSPosed maps/heap probes, and mount parsers.
- Duck Detector also has useful Zygisk/Root ideas such as fd trap, atexit/linker drift, thread/fd probes, smaps/namespace/seccomp probes, and KSU supercall handling. Import only with StandUp-style result states and false-positive guards.
- XposedDetector and NativeDetector show deeper ART/linker techniques than simple class/package checks.
- KeyDetector is useful for keybox and attestation parsing.
- EnvCheck remains useful for APatch `auth_superkey` and native property/ksu/apatch probes.
- rootbeer is useful as a baseline, but many checks are too generic for high-risk scoring by themselves.
- RiskEngine and sentry are especially useful for engineering shape: Java orchestration plus native syscall-first probes, redundant channels, weighted/warn-only scoring, and interpretable evidence. Do not copy their broad checks without adapting StandUp's device matrix.

## Injection and Hiding Tooling

- `HideApk` shows memory dex loading and custom linker in-memory so loading. The detector lesson is to look beyond file paths: anonymous ELF, nonstandard executable mappings, linker/soinfo mismatch, and JNI registration/runtime shape.
- `MyInjector` and `Zygisk-MyInjector` are useful to understand Xposed/Zygisk module lifecycle, target scoping, and native payload loading. Do not use them as detector code directly.
- `kpm-panda-hide` shows kernel-level hiding of debugger, Frida ports, maps, memory reads, `openat`, `faccessat`, and `connect`. This means userspace detectors should expect `/proc`, maps, and port probes to be filtered, and should correlate multiple channels instead of trusting one clean result.
- `Vector` is an ART hooking framework reference. Use it to understand method entry/trampoline behavior, not as a direct high-risk keyword source.

## Reverse Engineering Workflow Repos

- `AlgoKiller` is valuable for constrained, line-anchored ARM64 trace analysis. Future algorithm restoration should preserve evidence anchors, avoid giant context dumps, and reconstruct data flow before writing code.
- `SVCMonitors` is valuable for syscall and runtime observation on owned devices. Treat its results as research/adb/root evidence unless an app-side signal can reproduce the finding.
- `tiny-dec` and `decx` are learning references for decompiler/JADX workflows. They belong in analysis methodology, not runtime detection.

## StandUp Porting Rule

Only port a detector when at least one of these is true:

- It produces app-UID-visible strong evidence on the user's device matrix.
- It improves false-positive control over existing StandUp logic.
- It consolidates duplicated expensive probes.
- It gives clearer UI/log attribution without broadening the risk surface.

Do not port broad keyword lists, adb-only `/proc` observations, or root-only paths unless the result is clearly marked as root-shell-only environmental evidence.

When multiple references propose overlapping checks, do not pick the newest or loudest one by default. Rank candidates by:

1. Direct app-process or app-UID visibility.
2. Passing clean-baseline validation.
3. Clear separation between current-process compromise, module environment, and root-shell-only evidence.
4. UI/risk-score explainability.
5. Maintenance cost and runtime overhead.

Reject or downgrade a reference idea if it breaks a known clean baseline, depends on mutable strings without corroboration, or only restates a previously rejected weak signal.
