# 05 — Frida & Dynamic Analysis

## VERSION FIRST — ALWAYS

```bash
frida --version                                        # client
adb shell su -c "/data/local/tmp/frida-server --version"   # server
```

They must be **identical**.

| Version | Android Java bridge | Notes |
|---|---|---|
| **16.5.7** | **WORKS** | Verified: spawn + `Java.perform` + `ActivityThread` resolve + app context |
| 17.x | reported broken for Android app hooking | Do not "fix" a failing hook by upgrading |

### `--no-pause` DOES NOT EXIST ON FRIDA 16+

```
frida: error: unrecognized arguments: --no-pause
```

It was removed. Spawn resumes by default and `--pause` now opts *in*. Most
tutorials online still pass it. Strip it from everything you have copied.

```bash
frida -U -f com.target.app -l script.js          # correct
frida -U -n com.target.app -l script.js          # attach to running
frida -U -f com.target.app -l script.js --pause  # opt in to pausing
```

### Split brain between client and scripts

`frida --version` (CLI, from the pipx venv) and
`python3 -c "import frida; print(frida.__version__)"` (your own scripts, from
the pip user-site) can disagree. Symptom: hooks fail mysteriously in a script
but the same hook works via the CLI.

```bash
pipx install frida-tools==13.7.1
pip install --user "frida==16.5.7" "frida-tools==13.7.1"
```

### PATH

`frida`, `frida-ps`, `frida-trace`, `frida-ls` live in `~/.local/bin`.
`~/.bashrc` usually exports it; **`~/.zshrc` often does not**. If the tools
report "command not found" despite being installed, it is PATH — not a broken
install.

## MINIMAL VALIDATION SCRIPT

Run this before writing any real hook. If it does not print `WORKING`, the
problem is version/spawn/architecture, not your payload.

```javascript
Java.perform(function () {
    console.log('[OK] Java.perform executed');
    console.log('[OK] ActivityThread: ' + Java.use('android.app.ActivityThread'));
    console.log('[OK] SDK_INT: ' + Java.use('android.os.Build$VERSION').SDK_INT.value);

    var tries = 0;
    var iv = setInterval(function () {
        tries++;
        var app = Java.use('android.app.ActivityThread').currentApplication();
        if (app !== null) {
            clearInterval(iv);
            console.log('[RESULT] JAVA_BRIDGE=WORKING package=' +
                        app.getApplicationContext().getPackageName());
        } else if (tries > 25) {
            clearInterval(iv);
            console.log('[RESULT] JAVA_BRIDGE=WORKING (class resolve ok)');
        }
    }, 200);
});
```

```bash
frida -U -f com.target.app -l validate.js
```

**`currentApplication()` returning `null` is normal at spawn** — the
`Application` object does not exist yet. Poll for it. Do not misread that as a
broken bridge.

## SPAWN vs ATTACH

| Mode | Command | Use when |
|---|---|---|
| Spawn | `-f <pkg>` | hooking code that runs during init (crypto setup, token fetch, root check) |
| Attach | `-n <pkg>` | the app is already running, session exists, shorter loop |

Spawn timing problems are the usual reason "my hook never fires": the interesting
code executed before you attached. Use `-f` for early-init targets, and
`--pause` to freeze before the first frame, then `resume`.

## USEFUL SCRIPTS

### Enumerate classes matching a pattern
```javascript
Java.perform(function () {
    var pat = /Token|Session|Auth|Crypto/i;
    Java.enumerateLoadedClasses({ onMatch: function (n) {
        if (pat.test(n)) console.log(n);
    }, onComplete: function () { console.log('done'); } });
});
```

### Hook a method and log arguments
```javascript
Java.perform(function () {
    var C = Java.use('com.vendor.api.TokenClient');
    C.getToken.overload().implementation = function () {
        console.log('[hook] getToken called');
        return this.getToken();
    };
});
```

### Override a return value (auth bypass)
```javascript
Java.perform(function () {
    var A = Java.use('com.vendor.auth.AuthManager');
    A.isAuthenticated.implementation = function () { return true; };
});
```

### Dump an argument to see the real request shape
```javascript
Java.perform(function () {
    var R = Java.use('com.vendor.net.OkHttpClientWrapper');
    R.enqueue.overload('okhttp3.Call').implementation = function (call) {
        console.log('[req] ' + call.request().url().toString());
        console.log('[hdr] ' + call.request().headers().toString());
        return this.enqueue(call);
    };
});
```

### Enumerate loaded native libraries
```javascript
Process.enumerateModules().forEach(function (m) {
    if (m.name.indexOf('vendor') !== -1 || m.name.indexOf('safe') !== -1)
        console.log(m.name + ' @ ' + m.base + ' size=' + m.size);
});
```

## CERTIFICATE PINNING BYPASS

**Check whether pinning exists first.** Grep the decompiled source and the
manifest `networkSecurityConfig` for `pin-set` / `CertificatePinner`. If a
backend is documented as plain HTTPS it may not pin, and you can skip all of this.

```bash
# codeshare (fastest)
frida -U -f <pkg> -l bypass.js
frida -U -f <pkg> --codeshare pcipolloni/universal-android-ssl-pinning-bypass-with-frida

# objection
objection -g <pkg> explore      # then: android sslpinning disable
```

Hook targets by pinning style:

| Style | Hook |
|---|---|
| OkHttp `CertificatePinner` | `okhttp3.CertificatePinner.check$okhttp` |
| Custom `TrustManager` | `checkServerTrusted` |
| Cronet / native | app-specific, needs `libssl.so` level work |
| Flutter | pinning is compiled native — use `reFlutter` to patch |

### Cert trust without patching
Magisk **Move Certificates** / **trust-user-certs** modules make the Burp CA
system-trusted — persistent, no per-app patching. Faster than Frida for
everyday traffic work.

### Static patch
```bash
apktool d <pkg>.apk -o patched/
# patched/res/xml/network_security_config.xml -> add <certificates src="user" />
apktool b patched/ -o patched.apk
```
Note: a rebuilt APK fails signature checks — fine for local traffic analysis,
never for anything that needs a real session.

## WHEN PINNING IS NOT THE PROBLEM

Common misdiagnoses:
- App never started → check `ps -A`, and `dumpsys window | grep mCurrentFocus`
- Hook attached to the wrong process → app spawns `:remote` services
- Attestation blocking before networking → see preflight
- Traffic still cleartext-dropped by the app itself, not the OS
- **The app does not use the proxy** — check `adb shell settings get global http_proxy`
  and confirm Burp sees the device at all

Debug order: does Burp see traffic at all → is it HTTPS-pinned → is the
app's own client refusing → are you on the right process.

## ROOT / DEVICE DETECTION

Usually **not billable.** Most programs exclude root-detection bypass, pinning
bypass, and obfuscation gaps as non-qualifying. Check the policy gate
(`01_policy_triage_gate.md`) before spending a day here.

If it genuinely matters for reaching functionality: Magisk DenyList per-app, or
hook the detection:
```javascript
Java.perform(function () {
    ['isRooted','checkRoot','isDeviceRooted'].forEach(function (m) {
        try { Java.use('com.vendor.sec.RootUtil')[m].implementation = function () { return false; }; }
        catch (e) {}
    });
});
```

Do not report the bypass itself. Report only what it unlocks.

## PERSISTENCE

`/data/local/tmp` gets wiped. Autostart from Magisk
`/data/adb/service.d/frida-server.sh` — see the script and the Magisk Delta
quirks in `00_environment_preflight.md`.

**Never trust a note that says frida is running.**
```bash
adb shell su -c "ps -A | grep frida-server"
adb shell su -c "cat /data/local/tmp/frida.log"
```

## LOGCAT — THE FREE INSTRUMENT

```bash
adb logcat -c
adb logcat | grep -iE "token|password|secret|key|auth|session|debug"
```

High-signal patterns:
- credentials/tokens in logcat (real finding on stock devices)
- `GoogleCertificatesRslt: not allowed` → sideload noise, **not** a finding
- integrity/attestation failures → app often proceeds anyway; check, don't assume
- stack traces leaking internal paths, and internal hostnames

## iOS

Needs a Mac plus a real device or simulator; a downloaded `.ipa` is usually
FairPlay-encrypted until dumped. `objection -g <bundle> explore` →
`ios keychain dump`, `ios cookies get`, `ios sslpinning disable`. If you have no
Apple hardware, work Android-only rather than shipping an untested iOS claim.