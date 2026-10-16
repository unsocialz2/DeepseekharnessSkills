# 00 — Environment Preflight & Toolchain Reality

Everything here was learned by hitting the failure. Verify, don't assume.

## 1. DEVICE PREFLIGHT (30 seconds, run first)

```bash
adb devices -l
adb shell getprop ro.product.cpu.abi                 # host arch
adb shell getprop ro.product.model
adb shell getprop ro.dalvik.vm.native.bridge         # ARM translation
adb shell getprop ro.product.cpu.abilist             # supported ABIs
adb shell getprop persist.sys.locale                 # region gate
adb shell getprop ro.build.version.sdk
adb shell su -c "which su"                           # root present?
adb shell pm list packages | grep -cE "vending|gms"  # Play ecosystem
adb shell ps -A | grep frida-server                  # is frida ACTUALLY running?
```

## 2. THE ARM64 / x86_64 TRAP — do not drop the target

**Symptom:** you download an APK, look at the libs, see `arm64-v8a` only. Your
emulator is x86_64. You conclude the app cannot run and abandon the target.

**Reality:** Android-x86, Waydroid, and current-gen emulators ship an ARM
translation layer. If this is set:

```
ro.dalvik.vm.native.bridge = libndk_translation.so
ro.product.cpu.abilist     = x86_64,x86,arm64-v8a,armeabi-v7a,armeabi
```

then **arm64 native code executes.** Near-universal now — `armeabi-v7a` and
`x86` ABIs are gone from current apps, so an x86_64-only emulator is useless
*only if it lacks translation*.

**Verified:** an Indonesian arm64-only fintech app (7.8 MB of arm64 `.so`
including facial-liveness and anti-fraud libs) installed on x86_64 Waydroid via
NDK translation and ran to its live login screen.

Interpretation table:

| `native.bridge` | `abilist` contains | Verdict |
|---|---|---|
| `libndk_translation.so` | `arm64-v8a` | ARM apps run. Go. |
| empty | `arm64-v8a` | Translation present some other way. Try it. |
| empty | x86_64 only | No ARM. Need arm64 hardware. |

## 3. APP STORE REGION GATE

Apps are region-scoped in the Play catalog. An `en-US` store account cannot
resolve them even though the APK exists and is publicly downloadable.

| Program/app | Region |
|---|---|
| Grab, Gojek | SEA |
| OVO | Indonesia |
| iFood, PagSeguro, Mercado Pago | Brazil / LatAm |
| Just Eat | UK/EU/AU/CA/BR — **not US** |

Two ways out:
1. `adb install-multiple` an APK you fetched directly (works regardless of store
   account; bypasses Play's licensing but the app usually tolerates it).
2. Change the Play Store country on the Google account (adds a payment profile).

Sideloading produces benign log noise — **do not report it**:
```
FlagStore: SecurityException: GoogleCertificatesRslt: not allowed: pkg=<pkg>
```
That is just "this APK is not Play-signed."

## 4. APK ACQUISITION

Modern apps ship as **App Bundles**, split by ABI, language, and density.
APIs that hand you a single APK usually give you the **base only**, which will
not install:

```
adb: failed to install base.apk: Failure [INSTALL_FAILED_MISSING_SPLIT: Missing split for <pkg>]
```

That is not a corrupt download. Get the bundle container:

```bash
UA="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0 Safari/537.36"

# XAPK endpoint => base + all splits.  /b/APK/ => base only, will NOT install.
curl -sL -A "$UA" -o app.xapk "https://d.apkpure.com/b/XAPK/<pkg>?version=latest"
unzip -o app.xapk -d app/
ls -la app/
```

- **A browser User-Agent is mandatory.** curl's default UA gets HTTP 403.
- The container's `manifest.json` records package, version, and the split list.

Install with the ABI split included:

```bash
adb install-multiple -r -g \
  app/base.apk app/config.arm64_v8a.apk app/config.xxhdpi.apk
```

Verify:

```bash
adb shell dumpsys package <pkg> | grep -iE "primaryCpuAbi|secondaryCpuAbi"
# arm64-v8a is CORRECT and expected on an x86_64 host with translation.
```

Mirror notes: APKPure needs the UA; APKMirror and Google-dork mirrors commonly
403; `apkeep` release asset names move and its GitHub API rate-limits;
**`gplaycli` is dead** (token-dispenser service shut down, plus a Python 3.14
`exceptions must derive from BaseException` crash). Do not build a pipeline on it.

## 5. LAUNCHING — resolve the real activity

```bash
adb shell monkey -p <pkg> -c android.intent.category.LAUNCHER 1   # often fails silently
adb shell cmd package resolve-activity --brief -c android.intent.category.LAUNCHER <pkg>
# -> <pkg>/.common.activity.StartingActivity
adb shell am start -n <pkg>/.common.activity.StartingActivity
sleep 15
adb shell ps -A | grep <pkg>                 # is the process alive?
adb shell dumpsys window | grep mCurrentFocus # what actually opened
```

`monkey` can exit `-5` with no reason. Always resolve explicitly.

## 6. ATTESTATION IS USUALLY NOT THE GATE

Do not assume Play Integrity blocks you. Test empirically:

```bash
adb logcat -c
adb shell am start -n <pkg>/<activity> && sleep 15
adb logcat -d | grep -iE "play.?integrity|attest|safetex|device.?integrity|Finsky"
```

Observed on a real fintech app:

```
E Finsky: [83] requestIntegrityToken() failed for ovo.id.
I Finsky: [170] Integrity key attestation record generated successfully.
```

The integrity token request **failed** — and the app proceeded to onboarding
and a live login screen. Many apps treat integrity as telemetry, not a gate.

Spend your time on the app logic and API instead of fighting attestation.

## 7. FRIDA: VERSIONS, PATHS, PERSISTENCE

### Version matrix

| Version | Android Java bridge | Status |
|---|---|---|
| **16.5.7** | **works** | Verified: spawn + `Java.perform` + `ActivityThread` resolve |
| 17.x | reported broken for Android app hooking | Do not "fix" a hook failure by upgrading |

### Client must match server

```bash
frida --version                                  # client
adb shell su -c "/data/local/tmp/frida-server --version"   # server
# must be identical, e.g. 16.5.7 / 16.5.7
```

Binary for the device must match the device ABI — for x86_64 Android:
`frida-server-<ver>-android-x86_64.xz`.

Install:

```bash
curl -sL -o fs.xz "https://github.com/frida/frida/releases/download/16.5.7/frida-server-16.5.7-android-x86_64.xz"
xz -dk fs.xz && mv fs frida-server-16.5.7-android-x86_64 && chmod +x frida-server-16.5.7-android-x86_64
adb push frida-server-16.5.7-android-x86_64 /data/local/tmp/frida-server
adb shell su -c "chmod 755 /data/local/tmp/frida-server"
adb shell su -c "nohup /data/local/tmp/frida-server > /data/local/tmp/frida.log 2>&1 &"
```

Verify both sides by checksum — this is the actual test:
```bash
md5sum frida-server-16.5.7-android-x86_64 | cut -d' ' -f1
adb shell su -c "md5sum /data/local/tmp/frida-server" | cut -d' ' -f1
```

### `--no-pause` is GONE on Frida 16+

```
frida: error: unrecognized arguments: --no-pause
```

Spawn resumes by default; `--pause` opts *in*. Most tutorials online are wrong
here. Correct form: `frida -U -f <pkg> -l script.js`

### Split brain

Two installs, two versions:

| Install | Provides |
|---|---|
| pipx venv | the `frida`, `frida-ps` CLI |
| pip user-site | `import frida` in your own scripts |

If `frida --version` and `python3 -c "import frida; print(frida.__version__)"`
disagree, you have this. Pin both:
```bash
pipx install frida-tools==13.7.1
pip install --user "frida==16.5.7" "frida-tools==13.7.1"
```

### PATH trap

`frida`, `frida-ps`, `frida-trace`, `frida-ls` live in `~/.local/bin`. If
`~/.zshrc` lacks the export, they report "command not found" while being fully
installed. **`~/.bashrc` usually has it and zsh does not** — and zsh is the
common default shell.

```zsh
if ! [[ "$PATH" =~ "$HOME/.local/bin:" ]]; then
    export PATH="$HOME/.local/bin:$PATH"
fi
```

Symptom pattern: *installed but not found* → check PATH before reinstalling.

### Persistence — `/data/local/tmp` gets wiped

The server will vanish and any note claiming "frida is running" becomes a lie.
Autostart from Magisk instead:

```bash
adb push frida-server.sh /data/local/tmp/frida-server.sh
adb shell su -c "cp /data/local/tmp/frida-server.sh /data/adb/service.d/frida-server.sh"
adb shell su -c "chmod 755 /data/adb/service.d/frida-server.sh"
```

```sh
#!/system/bin/sh
# wait for the binary, let the framework settle, then start
i=0
while [ ! -f /data/local/tmp/frida-server ] && [ $i -lt 60 ]; do sleep 2; i=$((i+1)); done
sleep 20
chmod 755 /data/local/tmp/frida-server 2>/dev/null
pkill -f frida-server 2>/dev/null
sleep 1
/data/local/tmp/frida-server >/data/local/tmp/frida.log 2>&1 &
```

**Rule: never trust a note that says frida is running. Check it.**
```bash
adb shell su -c "ps -A | grep frida-server"
```

Magisk Delta quirks: heredocs through `adb shell su -c "cat > file"` fail with
Permission denied — push the file instead. And reading files back out of
`/data/adb` via `su -c cat` can return Permission denied even when the write and
chmod succeeded. Verify with `ls -la`, not by cat'ing.

## 8. TOOL GOTCHAS

| Symptom | Cause | Fix |
|---|---|---|
| `apktool`: "Input file was not found" on a file that exists | apktool installed via **snap**; snap cannot read `/tmp` or hidden dirs | Run from a snap-visible path, or use the `apktool.jar` directly with `java -jar`, or use jadx |
| `aapt`/`apkanalyzer` not found | not installed | `android-sdk-build-tools`, or use `dumpsys` |
| gplaycli crashes / `TOKEN_DISPENSER_SERVER_ERROR` | service shut down; Python 3.14 incompat | use XAPK + `adb install-multiple` |
| `adb: more than one device` | multiple targets | `adb -s <serial> ...` |
| Frida hooks never fire | `--no-pause`, or version mismatch, or app not spawning | remove `--no-pause`; match versions |

## 9. iOS PREFLIGHT

A downloaded `.ipa` is normally **FairPlay-encrypted** — strings are garbage
until dumped from a paired device. iOS dynamic testing needs a Mac plus a real
device or simulator. If you have neither, say so and work Android-only rather
than shipping an untested iOS claim.