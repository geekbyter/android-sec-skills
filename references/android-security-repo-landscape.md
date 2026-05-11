# Android Security Repository Landscape

This reference summarizes the local repository families used as StandUp security-audit training material. It is intentionally defensive: use these notes to understand what attackers hide and which app-visible signals remain useful. Do not use it to build or improve evasion tooling.

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

## Zygisk Family

Local sources include `ZygiskNext`, `NeoZygisk`, `r0zygisk`, `ReZygisk`, `DetectZygisk`, and `ZygiskDetector`.

Useful concepts:

- Projects use loader/daemon/socket/mountinfo helpers and may use memfd or dl-based loading.
- Simple ptrace event-message and libc internals checks are interesting but device/ROM fragile.

Defensive detector lessons:

- Weak artifacts such as one ART dirty page, JIT noise, or a generic atexit gap should not drive UI high risk.
- Stronger evidence should be current-process injection, explicit zygisk module environment, zygote-derived ptrace residue proven app-visible, or current app native/memfd artifacts.

## TrickyStore / TEESimulator

Local sources include `TrickyStore`, `TEESimulator`, and `TrickyStoreOSS`.

Useful concepts:

- These projects intercept Keystore/KeyMint Binder flows and can patch or generate attestation certificates.
- Keybox, target list, certificate generation, KeyMetadata, AuthorizationList, and RootOfTrust branches are the important surfaces.

Defensive detector lessons:

- Keep three layers separate: current UID attestation tamper, keybox revocation/leak evidence, and broader environment suspicion.
- Valid keyboxes can evade revocation checks, so negative revocation results do not prove no module.
- The strongest app-side strategy is active attestation consistency: challenge echo, key pair/certificate pairing, canary PrivateKeyEntry round-trip, alias/updateSubcomponent consistency, certificate chain shape, and attestationKey branch behavior.

## General Detector Repos

Relevant local sources include `Duck-Detector-Refactoring`, `Kknd_Root_Detector`, `RiskDetector`, `Ruru`, `rootbeer`, `NativeDetector`, `RikkaX`, `launch`, `sentry`, `KeyDetector`, `violet_Box`, `EmulatorDetection`, `EnvCheck`, `Android-Native-Root-Detector`, `UprobeChecker`, and `Vector`.

High-value patterns:

- Duck Detector has useful native memory modules: maps anomaly, fd anomaly, signal handler checks, linker object mismatch, function hook range checks, LSPosed maps/heap probes, and mount parsers.
- XposedDetector and NativeDetector show deeper ART/linker techniques than simple class/package checks.
- KeyDetector is useful for keybox and attestation parsing.
- EnvCheck remains useful for APatch `auth_superkey` and native property/ksu/apatch probes.
- rootbeer is useful as a baseline, but many checks are too generic for high-risk scoring by themselves.

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
