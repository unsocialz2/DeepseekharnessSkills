# 04 — Static Analysis Workflow

Static-first. It is faster than dynamic, works with no device, no account, no
proxy and no root, and it produces the map that makes dynamic work worthwhile.

## ORDER OF OPERATIONS

```
1. bundle inventory      -> what am I actually holding?
2. packer/protector ID   -> is jadx output even readable?
3. native/ABI triage     -> what can I NOT test?
4. manifest via dumpsys  -> routes + exported components, no decoder
5. decompile             -> backend map, secrets, crypto, auth logic
6. secret sweep          -> replayable credentials
7. THEN decide if you need a device at all
```

## 1. BUNDLE INVENTORY

```bash
ls -la app/                     # base.apk + config.<abi>.apk + config.<locale>.apk
unzip -p app.xapk manifest.json | python3 -m json.tool | head -40
```

Modern apps ship as App Bundles split by ABI / language / density. A base-only
download fails with `INSTALL_FAILED_MISSING_SPLIT` — that is the bundle, not a
corrupt file. See `00_environment_preflight.md`.

## 2. PACKER / PROTECTOR CHECK — BEFORE jadx

A packed APK decompiles to noise. Check first:

```bash
APKiD app.apk
# or quick tells:
unzip -l app/base.apk | grep -iE "libjiagu|libshell|libsecexe|libDexHelper|libprotect|assets/.+\.dat"
unzip -l app/base.apk | grep -iE "libdexprotector|libPhoenixAndroid|libsecsigner|libsafex"
```

Seen in the wild:
| Signal | Meaning |
|---|---|
| `libdexprotector.so` | DexProtector — commercial packer, expect bad jadx output |
| `libPhoenixAndroid.so` | Phoenix packer |
| `libsafexEx.so`, `libosec.so`, `libconcealjni.so` | Safex/anti-tamper |
| `libbarhopper*.so` | Barhopper obfuscation |
| `libsecsigner.so` | signature verification |
| `assets/*.{dat,bin}` + stub dex | Jiagu / Bangcle style |

Packed → options: dump from a running device (`frida-dexdump`), find the real
dex in memory, or accept that static analysis is limited and pivot to the API
and the manifest, which are never packed.

## 3. NATIVE / ABI TRIAGE

```bash
python3 - <<'PY'
import zipfile
for f in ('app/base.apk',):
    z = zipfile.ZipFile(f)
    libs = [n for n in z.namelist() if n.startswith('lib/')]
    print(f, 'ABIs:', sorted({n.split('/')[1] for n in libs}))
    print('libs:', [n.split('/')[-1] for n in libs][:40])
    print('dex :', len([n for n in z.namelist() if n.endswith('.dex')]))
PY
```

Read the list as a **do-not-attack** list. Native crypto, liveness detection,
root detection, and anti-fraud are unreachable on any normal device — and you do
not need them. The exploitable logic is in the dex. Knowing this saves a day.

App scale indicators: 5 dex = normal; 20+ dex = heavily modular, expect long
jadx runs (start it in the background) and heavy obfuscation.

## 4. MANIFEST VIA DUMPSYS (fastest triage in the mode)

Needs the app installed. No decoder, no apktool, no snap problems.

```bash
adb shell dumpsys package <pkg> > pkg.txt
grep -E 'Scheme:|Authority:|Path|Action:|Category:' pkg.txt
grep -iE 'debuggable|allowBackup|cleartext|permission' pkg.txt | head
```

When the app is not installed, decode with apktool — but check whether it is a
snap first, because snaps cannot read `/tmp`:
```bash
which apktool && snap list 2>/dev/null | grep -i apktool
```

## 5. DECOMPILE

```bash
nohup jadx --no-res -d java_out/ app/base.apk > jadx.log 2>&1 &   # big APK
jadx -d java_out/ app/base.apk                                    # small, want resources
```

- `--no-res` skips resource decoding and is dramatically faster; you lose the
  manifest, which is why step 4 uses dumpsys instead.
- Watch the tail: `finished with errors, count: N` is normal on obfuscated apps.
- Fully obfuscated (single-letter packages) → you are reading noise. Fall back to
  the manifest, strings, and the API.

## 6. BACKEND MAP — highest value per second

```bash
grep -rhoE "https?://[a-z0-9.-]+\.[a-z]{2,}[a-zA-Z0-9/._-]*" java_out/ \
  | sed -E 's#(https?://[^/]+).*#\1#' | sort | uniq -c | sort -rn | head -40
```

Reference count ≈ importance. Top hosts are the primary API and the API gateway.
Cross-check against the **web app**: hosts with no web equivalent are mobile-only
and disproportionately under-tested — that is the whole lead.

Then extract paths, which is where the mobile-only surface lives:
```bash
grep -rhoE '"(/(v[0-9]+/)?[a-z0-9_/-]{4,60})"' java_out/ | sort -u | head -80
```

Look specifically for:
- `/api/`, `/mobile/`, `/app/` prefixes absent from the site
- versioned roots (`/v1/` vs `/v3/`) — the older one may be unauthenticated
- dev/staging hosts (`staging.`, `-dev.`, `uat.`, `sandbox.`, `internal.`)
- GraphQL endpoints and `/graphql`

## 7. SECRET SWEEP

```bash
grep -rniE "api[_-]?key|apikey|client[_-]?secret|client[_-]?id|app[_-]?secret" java_out/ | head -40
grep -rnoE "AKIA[0-9A-Z]{16}" java_out/ | sort -u          # AWS access key id
grep -rnoE "AIza[0-9A-Za-z_-]{35}" java_out/ | sort -u      # Google API key
grep -rniE "firebaseio\.com|firestore\.googleapis" java_out/ | head -20
grep -rniE "eyJ[A-Za-z0-9_-]{10,}\." java_out/ | head      # hardcoded JWTs
grep -rniE "Bearer [A-Za-z0-9._-]{20,}" java_out/ | head   # baked tokens
apkleaks -f app/base.apk -o apkleaks.txt
```

**Program filter matters.** Many programs exclude "disclosed/misconfigured
Google API key", "3rd party secrets", and "leaks sourced from archive.org,
wayback, urlscan, VirusTotal or dorking". Only report secrets that come from
their own asset and that you verified.

Verify, don't assume:
```bash
curl -s "https://<project>.firebaseio.com/.json" | head
aws sts get-caller-identity --profile <extracted>
```

## 8. CRYPTO AND AUTH LOGIC

```bash
grep -rniE "\"AES\"|\"DES\"|Cipher\.getInstance|ECB|IvParameterSpec" java_out/ | head -20
grep -rniE "TrustAllCerts|checkServerTrusted|HostnameVerifier|ALLOW_ALL" java_out/ | head
grep -rniE "setJavaScriptEnabled|addJavascriptInterface|setAllowFileAccessFromFileURLs" java_out/ | head
grep -rniE "MODE_WORLD_READABLE|getExternalStorageDirectory|openFileOutput" java_out/ | head
```

## 9. PINNING — DETECT BEFORE YOU BYPASS

```bash
grep -rniE "CertificatePinner|networkSecurityConfig|TrustManager|X509TrustManager|pinning" java_out/ | head -20
```

Check the manifest for `networkSecurityConfig` pointing at a
`<pin-set>` / `<pins>` block. If the backend you care about shows no pinning
evidence, **you may not need a bypass at all** — test that first.

## 10. STATIC-ONLY EXIT CRITERIA

You can skip the device entirely if the program wants secrets leakage / mobile
misconfigurations, or if you already have: a backend map, a route inventory,
replayable credentials, and a staging-leftover finding. State clearly in the
report that no device was used, which is a **strength** when the program requires
stock-device PoCs.