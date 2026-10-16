# 01 — Policy Triage Gate

**Run this before installing anything.** A mobile finding is worth exactly what
the program says it is worth, and the classes hunters report first are the
classes that get closed Informative.

## THE CORE QUESTION

> Does this program accept a finding that only reproduces on a device you had to
> modify to get it?

If the answer is no, every rooted-device finding is dead weight and every hour
spent on Frida, root bypass, or `allowBackup` was misallocated.

## THE FIVE-QUESTION GATE

Pull the program's mobile section, then answer:

| # | Question | If NO |
|---|---|---|
| 1 | Does the scope list the app **as an asset** (Play Store ID / bundle ID)? | Don't test it. Reports are Not Applicable. |
| 2 | Do they exclude "**rooted/jailbroken** device" findings? | Everything you can only prove while rooted is out. |
| 3 | Do they exclude "**ADB access**" or "physical device access"? | `allowBackup`, `adb backup`, file-pull findings are out. |
| 4 | Do they exclude "**WebView exploitation**"? | The richest Android class is gone. Rethink the target. |
| 5 | Do they exclude "**SSL pinning bypass / root detection / obfuscation**"? | Standard pinning-bypass work is not billable. Fine — you were going to waste a day on it. |

## THE HIGH-VALUE PHRASE TO LOOK FOR

Many programs *specifically* name the mobile class they want:

```
*** Leaked information from Mobile (WITHOUT ROOTING) ***
*** Sensitive Information Exposure Through insecure data storage on device ***
```

Read "(without rooting)" as a **hard constraint on your PoC**, not a caveat.
Your screenshots must come from a stock device.

Conversely, a line like *"Server-side vulnerabilities discovered through mobile
applications fall under Business Platforms and APIs scope"* means a bug you find
by driving the app still pays the full web reward chart. That is permission to
treat the app as a **map to the API**.

## PER-PROGRAM PATTERNS OBSERVED

Real examples. Verify current text — programs edit their policies often.

### Pattern A — generous, mobile explicitly desired

```
Rewards (mobile, High asset value): Low 500 / Med 2000 / High 5000 / Crit 10000
Qualifying: Leaked information from Mobile (WITHOUT ROOTING)
Non-qualifying: SSL Pinning bypass, Lack of code obfuscation,
                Loss of root detection, jailbroken-only exploits,
                Android <= 7 only
In scope:   Play Store com.<vendor>.app  (High value)
```
**Read:** huge ceiling, but everything native/anti-tamper is dead. The wins are
deep links, IDOR, insecure storage on a clean device, and API authz.

### Pattern B — tight exclusions, still fine

```
Mobile applications — In scope — P1 3500-4500 / P2 1500-2500 / P3 300-500
Focus areas: Secrets leakage, mobile misconfigurations
"We are looking for 0-click or 1-click vulnerabilities."
Non-qualifying: rooted/jailbroken, ADB access, physical device access,
                install of the malware app, WebView exploitation
```
**Read:** narrow, but the exclusions *help* you. No ADB dumps, no WebView RCE
chasing, no root. Deliver a remote PoC that never touches device-local state.
The stated focus areas are free targeting — hunt exactly those.
Multi-brand scope = separate codebases = OneFixOneReward does not collapse them.

### Pattern C — behavioural rules that can get you banned

```
"Only perform tests against your own accounts."
"Only ever perform this testing against accounts you own, failure to do so
 could result in ban from the program."   <- stated 3x in one scope
"You may get suspended if you make too many fake bookings
 or not a single completed booking."
```
**Read:** read-only access to another user's data still disqualifies you.
For marketplaces, keep your account realistic — place one real order. Getting
banned wastes the whole target.

### Pattern D — one-off paid side quests worth grabbing

```
"For *.example.com we are accepting subdomain takeover reports with a fixed
 reward of 1000 EUR. To be eligible, demonstrate it with the MD5 of your
 username in a TXT file (md5(hunter_name).txt) hosted on the vulnerable FQDN.
 We won't qualify hypothetical takeovers nor simple DNS dangling records."
```
**Read:** a defined, flat, unambiguous bounty sitting next to your mobile work.
Same program, same week, no deduction. Do it early — it is the fastest
confirmed payout available and it does not compete with your mobile hours.

### Pattern E — managed platform

```
Managed: Yes   Max: $3750   Disclosure: No
"Will also accept flaw-hypothesis submissions, without penalty, and will work
 with you to develop a reasonable hypothesis into a working exploit."
```
**Read:** the platform runs triage and pays, so payout reliability beats the
headline number. The hypothesis clause is the underrated part — a near-miss
with a defensible reasoning chain gets developed instead of RTFS'd.

## WRITE-UP GATE

Before submitting, check your report against these — each maps to a real
rejection:

- [ ] PoC reproduces on a **stock, unrooted** device (if required)
- [ ] No root detection, pinning bypass, or obfuscation gap as the headline
- [ ] Not a third-party dependency CVE with no demonstrated attack vector
- [ ] Not "generic Android vulnerability"
- [ ] Not requiring physical access, MITM, or an attacker-installed app
- [ ] Discovery method explained — some programs **delay** reports without it
- [ ] PII redacted; no bulk data extracted
- [ ] Status honestly labelled (see SKILL.md HONESTY GATE)
- [ ] One vulnerability per report unless chaining is the point
- [ ] Not an ID-guessing/UUID-bruteforce claim if the program excludes that

## HONESTY GATE — CONFIRMED vs PROVEN

| Status | Means | File as |
|---|---|---|
| **MAPPED** | in manifest/dumpsys/jadx, never fired | nothing |
| **CONFIRMED REACHABLE** | resolves, or fires, but no effect shown | misconfiguration, Low |
| **IMPACT PROVEN** | you demonstrated the security effect | real severity |

**Real case.** A banking SDK inside a fintech app exported a BROWSABLE OAuth
callback reachable via `gxs://`, `gxsdev://` and `gxsstg://` — dev and staging
schemes in a production APK. Firing it with an attacker-controlled `code`
changed **no app state**, because the handler sits behind an authenticated
session and the test device was logged out.

Correct write-up: *"production app ships dev/staging OAuth callback schemes,
externally reachable, attacker-controlled `code`/`state` reach the handler."*
Wrong write-up: *"OAuth code injection / account takeover."*

An intent ignored while logged out proves **nothing** about logged-in behaviour.
Say exactly that, then get a session. Overstating here is how good hunters get
their reputation burned on a program you want to keep.

## SESSION DISCIPLINE

If the program demands a report within 24h of discovery, or has a strict
one-fix-one-reward rule, batch findings into a single coherent submission rather
than trickling near-duplicates.