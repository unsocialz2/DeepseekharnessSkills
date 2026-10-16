# Shared Hunting Environment

Local environment notes shared across targets. **This file describes observed machine/device state, not program policy and not universal tool guidance.**

Last verified: **2026-10-03**

When an exact version, device capability, path, or service state matters, verify it at runtime before relying on this file. Target-specific restrictions belong in `targets/<target>/rules.txt`.

## Host

- Primary OS/tooling: Kali/Linux + Burp Pro + optional VPS.
- `adb`: `/usr/bin/adb`
- `jadx`: `/usr/local/bin/jadx`
- `apktool`: snap install under `/var/lib/snapd/snap/bin/apktool`
- ProjectDiscovery tools such as `nuclei`, `subfinder`, `httpx`: typically under `~/go/bin`
- Java: Temurin 21.0.5 when last verified
- Python/pipx CLIs commonly install under `~/.local/bin`

### Shell PATH gotcha

The interactive shell is zsh. If a pip/pipx-installed CLI exists but reports `command not found`, check that `~/.local/bin` is present in the active zsh PATH before assuming the tool is broken.

## Android lab device

Last verified device endpoint: `192.168.240.112:5555`

- Environment: WayDroid container
- Android: 13 / SDK 33
- Host ABI: x86_64
- Root: Magisk-based lab environment
- Google Play/GMS: present
- ARM translation observed through `libndk_translation.so`
- Advertised ABI list included x86_64/x86 and arm64-v8a/armeabi-v7a

The important operational implication is that an x86_64-looking WayDroid instance may still execute ARM64 app native libraries. Verify with:

```bash
adb shell getprop ro.product.cpu.abi
adb shell getprop ro.product.cpu.abilist
adb shell getprop ro.dalvik.vm.native.bridge
```

Do not infer app compatibility from host ABI alone.

## Frida — dated local pin

On the last verified setup, the working local combination was:

- Frida host library/client: `16.5.7`
- `frida-tools`: `13.7.1`
- device `frida-server`: `16.5.7`

This is a **known-good local snapshot**, not a recommendation that newer versions are universally broken. Before using it, check the currently installed versions and current Frida behavior.

Useful checks:

```bash
frida --version
python3 -c 'import frida; print(frida.__version__)'
adb shell su -c 'ps -A | grep frida-server'
```

Host library/client and device server should be compatible. If CLI and Python report different versions, resolve the split install before debugging hooks.

The last verified server path was:

```text
/data/local/tmp/frida-server
```

A Magisk `service.d` entry was used to restart it after boot. `/data/local/tmp` persistence should not be assumed; verify the process instead of trusting old notes.

## APK / bundle handling notes

Modern Android apps frequently ship as split APK/App Bundle packages. A base APK alone may fail with `INSTALL_FAILED_MISSING_SPLIT`.

For an XAPK/bundle already obtained, inspect its contents and install the required base + ABI + density/language splits together:

```bash
unzip -o app.xapk -d app/
adb install-multiple -r -g app/base.apk app/config.arm64_v8a.apk app/config.xxhdpi.apk
```

Select the actual splits present in the bundle rather than copying this command blindly.

Useful runtime checks:

```bash
adb shell dumpsys package <pkg>
adb shell cmd package resolve-activity --brief -c android.intent.category.LAUNCHER <pkg>
adb shell dumpsys window | grep mCurrentFocus
```

`dumpsys package` is often the fastest way to inspect exported components and intent filters without fully decoding an APK.

### Intent-filter parsing gotcha

In `dumpsys` output, intent-filter host information may appear as an `Authority:` entry rather than a literal `Host:` label. Avoid parsers that assume only `Host:` or they can silently misclassify filters as unhosted.

## Tool confinement gotcha

The snap-packaged `apktool` may be unable to read paths outside its allowed filesystem view (for example some temporary directories). If a file clearly exists but apktool claims it cannot find/read it, test from a snap-visible path before treating the APK as invalid.

## Environment vs target policy

This file answers questions like:
- what tools/device were available,
- what versions last worked locally,
- what lab-specific compatibility quirks were observed.

It does **not** answer:
- whether rooted/emulated testing is accepted,
- whether ADB/WebView/pinning findings qualify,
- what assets are in scope,
- what request rates or techniques are allowed.

Those decisions come from the active target's `rules.txt` and scope files every time.
