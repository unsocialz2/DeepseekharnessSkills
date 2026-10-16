# 02 — Real-World Scenarios

End-to-end scenarios. Scenario A is a verified session transcript including the
mistakes, so you can pattern-match the failure modes.

---

## SCENARIO A — Fintech app triage (VERIFIED SESSION, all 3 mistakes included)

**Target:** `ovo.id`, an Indonesian e-wallet, in scope on a public HackerOne
program with a mobile tier advertised up to $15,000 for Critical.
**Box:** Waydroid x86_64, Android 13, Magisk rooted.
**Time to first real signal:** ~25 minutes, zero live traffic.

### A1. The mistake that nearly killed the engagement

Downloaded the APK. Base had **0 native libs**, so assumed pure Java and
installable. The XAPK's ABI split was `arm64-v8a` only.

Near-conclusion written down: *"apps are arm64-only, device is x86_64, they
cannot run here — the dynamic half of this program is unreachable."*

**Wrong.** The box reported:

```
ro.dalvik.vm.native.bridge = libndk_translation.so
ro.product.cpu.abilist     = x86_64,x86,arm64-v8a,armeabi-v7a,armeabi
```

ARM translation. Installed with `install-multiple`, app ran to a live login
screen.

**Lesson:** check `ro.dalvik.vm.native.bridge` before concluding an arm64-only
app is dead on an x86_64 host. Almost every guide gets this backwards.

### A2. The mistake that produced a fake finding

Grepped the class list for exported activities and matched:
`androidx.test.core.app.InstrumentationActivity` in a **production** APK —
a classic test-harness-in-release misconfiguration.

Tried to launch it:
```
Error: Activity class {ovo.id/androidx.test.core.app.InstrumentationActivity} does not exist.
```

The regex had matched substrings inside `InstrumentationActivityInvoker$BootstrapActivity`.
**The activity never existed.** A real-sounding lead produced entirely by a
sloppy pattern.

**Lesson:** a lead is not real until you have *fired or resolved it*. Cheap
verification before you build a theory.

### A3. The mistake that corrupted an analysis

Wrote a parser for `dumpsys package` intent filters and grepped for:
```
Scheme: "..."     -> matched
Host: "..."       -> returned NOTHING
Path: "..."       -> matched
```

Concluded the deep-link filters had **no host restriction** — a serious
"unrestricted deep link" story. Then noticed resolution failures and read the
raw block:

```
Authority: "ovo.id": -1
Path: PatternMatcher{LITERAL: /oauth-callback}
```

dumpsys prints the host as `Authority:`, never `Host:`. My parser dropped it,
which made every filter look wide open.

**Lesson:** when a result says "no X found", first confirm you were grepping for
the right token. A silent empty match is a parser bug far more often than it is
a target bug.

### A4. URI composition — three wrong forms before one right

Filter: `scheme=gxsdev`, `authority=ovo.id`, `path=/oauth-callback`.

| Tried | Result | Why |
|---|---|---|
| `gxsdev://oauth-callback?code=X` | no match | `oauth-callback` parsed as **authority**, path empty |
| `gxsdev:///oauth-callback?code=X` | no match | empty authority, fails `Authority: "ovo.id"` |
| `gxsdev://ovo.id/oauth-callback?code=X` | **resolved** | scheme + authority + literal path all present |

**Lesson:** custom schemes use the authority slot for a route name. If the filter
declares an authority, the authority must be in the URI.

### A5. The check that made everything cheap

Before firing anything at a service:
```bash
adb shell cmd package query-activities -a android.intent.action.VIEW -d "gxsdev://ovo.id/oauth-callback?code=X&state=Y"
```
Non-destructive. Confirmed all three schemes (`gxs`, `gxsdev`, `gxsstg`) resolved
into `ovo.id.loyalty.app.Ovo`. No traffic to a third party, no state change.

**Lesson:** always separate *resolution* (cheap, local) from *firing* (may touch
a real service).

### A6. What the finding actually was

Fired `gxsdev://ovo.id/oauth-callback?code=TESTAUTH123&state=attacker` on the
local device. App state **did not change** — handler is behind an authenticated
session; the box was at the login screen.

Honest status:

| Claim | Status |
|---|---|
| Dev/staging OAuth callback schemes in a production APK | **CONFIRMED REACHABLE** |
| Handler is exported + browsable | **CONFIRMED REACHABLE** |
| Attacker `code`/`state` reaches the handler | **CONFIRMED REACHABLE** |
| Authorization-code injection / ATO | **NOT PROVEN — needs a session** |

Reported as a misconfiguration, not ATO. Correct call.

### A7. What actually produced value

The static pass, needing no device and no session:

```bash
grep -rhoE "https?://[a-z0-9.-]+\.[a-z]{2,}[a-zA-Z0-9/._-]*" java_out/ \
  | sed -E 's#(https?://[^/]+).*#\1#' | sort | uniq -c | sort -rn
```
```
  21  api.cp1.ovo.id      19  static.ovo.id     17  agw.ovo.id
  13  api.ovo.id           7  webview.ovo.id     4  finserve.ovo.id
```

Plus a corrected intent-filter map of **38 browsable routes**, where `ovo://`'s
"authority" is really a route name:

```
ovo://login   ovo://transfer  ovo://withdrawal  ovo://topup
ovo://paybill ovo://paylater  ovo://kyc  ovo://scan  ovo://checkout
ovo://webview ovo://jsbridgewebcontainer   ovo://upgrade
```

Highest-value lead: `ovo://jsbridgewebcontainer` — a JavaScript-bridge web
container behind a browsable deep link. The same program on a competitor's scope
explicitly excluded "WebView exploitation"; this one did not.

**Lesson:** the static pass gave the backend map, the route inventory, and the
lead list. Dynamic work only confirmed reachability of one route and found
nothing further. **Budget 70/30 static/dynamic, not the reverse.**

### A8. Native layer triage told me where NOT to look

```
libfinauthlivenessv5Ex.so   facial liveness for financial auth
libcsgfaceEx.so             face
libsafexEx.so libosec.so libconcealjni.so libbarhopper_v3.so   anti-tamper
```
All arm64, all unreachable, all irrelevant. The exploitable logic was in the
5 dex files of Kotlin. Knowing this saved an afternoon.

---

## SCENARIO B — Static-only sweep when the device can't help

Use when: no spare device, region-blocked store, or app won't run.

```bash
# 1. bundle map
unzip -o app.xapk -d app/ && ls app/
# 2. ABIs — know what you cannot test
python3 -c "
import zipfile; z=zipfile.ZipFile('app/base.apk')
print(sorted({n.split('/')[1] for n in z.namelist() if n.startswith('lib/')}))"
# 3. decompile (--no-res is far faster)
jadx --no-res -d java_out/ app/base.apk
# 4. manifest from an installed copy — no decoder needed
adb shell dumpsys package <pkg> > pkg.txt
# 5. backend map + secrets
grep -rhoE "https?://[a-z0-9.-]+\.[a-z]{2,}[a-zA-Z0-9/._-]*" java_out/ \
  | sed -E 's#(https?://[^/]+).*#\1#' | sort | uniq -c | sort -rn
grep -rniE "api_key|client_secret|AKIA|firebase" java_out/ | head -40
# 6. dev/staging leftovers in a release build — a real, cheap misconfiguration
grep -rniE "staging|\.dev\.|uat|sandbox|test\." java_out/ | grep -iE "host|url|endpoint" | head
```

Scenario B yields the backend map, route inventory, secret candidates, and
staging-leftover leads — with no device, no root, no proxy, no account.
On programs whose mobile focus areas are "secrets leakage" and "mobile
misconfigurations", Scenario B alone can carry a submission.

---

## SCENARIO C — Exported component abuse (classic, still works)

Requires an installed app. Prefer `query-activities` / `dumpsys` over blind
`am start` so you do not trigger actions on a real service.

```bash
# 1. inventory without decoding
adb shell dumpsys package <pkg> | grep -iE "Activity|Service|Receiver|Provider"
# 2. resolve before firing
adb shell cmd package query-activities -a android.intent.action.VIEW -d "<uri>"
# 3. local-only tests are safe
adb shell am start -n <pkg>/.InternalActivity --es key value
# 4. read a provider you already own
adb shell content query --uri content://<pkg>.provider/table
```

**Order of operations:** local-only intents first. Anything that reaches the
vendor's backend — money movement, auth callbacks, order placement — needs a
session, and on a program with "own accounts only, or you are banned" language,
needs care and a realistic account.

---

## SCENARIO D — When the app is region-locked or the store refuses

```
en-US store account + Grab (SEA)  -> not listed
```
Do not burn hours on the store. Go direct: XAPK + `install-multiple`. Registering
a test account may still need a local phone number — establish that constraint
early, because it decides whether the dynamic half of the program is available
at all.

---

## SCENARIO E — When pinning is the wall

```bash
# 1. does pinning even exist? check before building a bypass
grep -rniE "CertificatePinner|network_security_config|TrustManager|X509TrustManager" java_out/
grep -A5 networkSecurityConfig <(adb shell dumpsys package <pkg>)
```
```bash
# 2. verify CA is trusted — Magisk "Move Certificates"/trust-user-certs module
# 3. only then bypass (Frida 16.x, NO --no-pause)
frida -U -f <pkg> -l bypass.js
frida -U -f <pkg> --codeshare pcipolloni/universal-android-ssl-pinning-bypass-with-frida
# 4. confirm the hook actually attaches before assuming pinning is the cause
```

A backend documented as plain HTTPS may not pin at all — check the second
endpoint before fighting the first. And if the program lists "SSL pinning
bypass" as non-qualifying, the bypass is a means, never the finding.

---

## ANTI-PATTERNS OBSERVED

- Concluding an arm64 app cannot run on x86_64 without checking for translation.
- Building a theory from a class-list grep without firing or resolving it.
- Writing a parser and trusting its empty results without reading raw output.
- Treating a resolved intent as a proven vulnerability.
- Reporting Play Store signature-mismatch log noise as an integrity finding.
- Spending a day on pinning bypass for a program that lists it as non-qualifying.
- Retrying deep-link URIs by guessing forms instead of reading the filter block.