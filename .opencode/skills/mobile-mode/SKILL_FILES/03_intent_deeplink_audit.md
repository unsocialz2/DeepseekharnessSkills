# 03 — Intent & Deep-Link Audit

The highest-yield mobile class on most programs, and the one most often
botched. This is the procedure, with the traps that produce fake results.

## PRINCIPLE: AUDIT BEFORE YOU FIRE

Three separate acts. Keep them apart in your notes and in the report.

| Act | Command | Touches the app? | Cost |
|---|---|---|---|
| **MAPPED** | read manifest / dumpsys | no | free |
| **RESOLVES** | `cmd package query-activities` | no | free |
| **FIRES** | `am start` / `am broadcast` | yes, maybe a backend | has consequences |

Never skip to act three. Resolution is free and non-destructive; firing a money
route against someone's production service is not.

## STEP 1 — GET THE FILTERS

```bash
adb shell dumpsys package <pkg> > pkg.txt
```

### THE `Authority:` TRAP — read this once

`dumpsys` renders an intent-filter host as:

```
Authority: "ovo.id": -1
```

**Never as `Host:`.** A parser grepping for `Host:` returns an empty list and
concludes the filters are unhosted — which produces a dramatic and completely
wrong "unrestricted deep link" theory. Verify your parser against raw output
before you build anything on it.

Correct extraction:

```bash
grep -E 'Scheme:|Authority:|Path|PatternMatcher|Category: "android.intent.category.(BROWSABLE|DEFAULT)"|Action:' pkg.txt
```

## STEP 2 — PARSE IT CORRECTLY

Regexes over flattened `dumpsys` break constantly. Parse line-oriented and
group by the `filter <hash>` header:

```python
import re
lines = open('pkg.txt', encoding='utf-8', errors='replace').read().splitlines()

# Real dumpsys line format is:  <leading-hash> <pkg>/<activity> filter <hash>
hdr = re.compile(r'[\w.]+/([\w.$]+)\s+filter\s+[0-9a-f]+')
idx = [i for i, l in enumerate(lines) if hdr.search(l)]

groups = []
for n, i in enumerate(idx):
    end  = idx[n+1] if n+1 < len(idx) else len(lines)
    act  = hdr.search(lines[i]).group(1)
    body = '\n'.join(lines[i:end])
    sch  = sorted(set(re.findall(r'Scheme:\s*"([^"]+)"', body)))
    auth = sorted(set(re.findall(r'Authority:\s*"([^"]+)"', body)))   # Authority, not Host
    path = sorted(set(re.findall(
        r'Path(?:Prefix|Pattern|Suffix)?:\s*"PatternMatcher\{(\w+): ([^}]+)\}"', body)))
    if not sch or 'android.intent.action.VIEW' not in body:
        continue
    groups.append((act, sch, auth, path, 'BROWSABLE' in body))

seen, uniq = set(), []
for g in groups:
    k = (g[0], tuple(g[1]), tuple(g[2]), tuple(g[3]))
    if k not in seen:
        seen.add(k); uniq.append(g)

print(f"{'ACTIVITY':<52} {'SCHEME':<22} {'AUTHORITY':<16} PATH")
for act, sch, auth, path, br in uniq:
    print(f"{act:<52} {','.join(sch):<22} {(','.join(auth) or '-'):<16} "
          f"{'; '.join(f'{t}:{v}' for t, v in path) or '(any)'}")
```

The header regex must tolerate the leading hash and the `<pkg>/<activity>`
slash — `(?:pkg\.)?([\w.$]+)\s+filter` matches nothing and yields zero groups,
which again looks like "no deep links exist" rather than a parser bug. Verified
against real output: 93 filter blocks → 38 distinct VIEW groups.

`BROWSABLE` matters: it is what makes a link reachable from a **web page**.
Almost every deep link has it. It is not itself a finding — but it tells you the
attack is remote and requires zero installs.

## STEP 3 — COMPOSE THE URI CORRECTLY

Every wrong form below produces a silent "no match" that looks like a
non-vulnerable app.

Filter: `scheme=gxsdev`, `authority=ovo.id`, `path=/oauth-callback`

| URI | Result | Why |
|---|---|---|
| `gxsdev://oauth-callback?code=X` | ✗ | `oauth-callback` parses as the **authority**; path is empty |
| `gxsdev:///oauth-callback?code=X` | ✗ | authority is empty, filter requires `ovo.id` |
| `gxsdev://ovo.id/oauth-callback?code=X` | ✓ | scheme + authority + literal path |

Rules:
- With `authority` declared, the authority **must appear** in the URI.
- With `path` declared, the path **must appear** as a path, not the authority.
- `<data>` with a scheme and **no** path matches **any** path.
- Pattern kinds: `LITERAL` exact · `PREFIX` · `GLOB` (wildcards) · `SUFFIX`.
- Multiple schemes in one `<data>` element match any of them.

## STEP 4 — RESOLVE, DON'T GUESS

```bash
adb shell cmd package query-activities -a android.intent.action.VIEW \
  -d "gxsdev://ovo.id/oauth-callback?code=TEST&state=xyz"
```
```
      packageName=<pkg>
        name=<pkg>.loyalty.app.Ovo
```

If that returns nothing, the URI form is wrong — **not** the app being secure.
Fix the URI before drawing any conclusion.

## STEP 5 — TRIAGE THE ROUTES

In fintech apps the "authority" is a **route name**. Sort by what the route
does, not alphabetically.

| Route name | Class | Priority |
|---|---|---|
| `jsbridgewebcontainer`, `webview` | JS bridge reachable from a link | **highest** |
| `transfer`, `withdrawal`, `topup`, `paybill`, `paylater`, `checkout` | money movement | **highest** |
| `kyc`, `verifyemail`, `scan` | identity / verification bypass | high |
| `login`, `session`, `auth` | session fixation, OAuth confusion | high |
| `upgrade`, `promo`, `promo-detail`, `stamp`, `cashback` | business logic, entitlement | medium |
| `inbox`, `notifications`, `history` | PII disclosure | medium |
| partner names (`indomaret`, `superbank`, …) | third-party integration | medium |

Runners-up worth checking: `http` accepted alongside `https` (cleartext route
into the app), `/gkyc/` path prefix on an external domain (web-driven identity
verification), and path-prefix filters with no path (`PREFIX:/deals` matches
`/deals-anything`).

## STEP 6 — WHAT MAKES IT A FINDING

An exported browsable deep link is **normal and not a bug**. It becomes a
finding only when the handler trusts attacker input:

- **WebView load** — attacker URL reaches a `WebView` with JS on, or a
  `addJavascriptInterface` bridge. XSS → native bridge → token theft.
- **OAuth callback** — accepts `code`/`state` without validation, no PKCE
  binding, no state check, or exchanges the code against a weak backend.
- **Session injection** — `ovo://login?session_id=X` makes the app adopt an
  attacker session (session fixation).
- **Money movement** — route pre-fills amount/recipient and the API trusts
  client-supplied values.
- **KYC bypass** — identity flow skippable or completable without the
  verification step.

## STEP 7 — DEV/STAGING LEFTOVERS

Dev/staging schemes and endpoints in a production APK are a **real, cheap
misconfiguration**:

```
scheme: prod, proddev, prodstg, gxs, gxsdev, gxsstg, app-dev, app-uat
host:   api.staging.vendor.com   internal-dev.vendor.com
```

File it as: *production app ships non-production deep-link schemes/endpoints,
externally reachable.* That is honest and usually Low.

It is **not** an ATO by itself. To claim code injection you must show the
handler consuming the value — which needs an authenticated session.

## SIDE CHANNEL — STATIC ROUTE MINING

Filter names come from the code too, so a no-device pass still finds routes:

```bash
grep -rniE "deep.?link|deeplink|app.?link|intent|uri|scheme" java_out/ \
  | grep -iE "host|route|path|url" | head -60
grep -rhoE "[a-z0-9_]+://[a-z0-9_./-]+" java_out/ \
  | sed -E 's#^[a-z0-9_]+://##' | sort -u | head -40
```

## REPORTING ANCHORS

Always capture: `dumpsys package` excerpt with the exact filter, the
`query-activities` resolution output, the app's reaction (screenshot or logcat),
and an explicit statement of what remains unproven.

State the trust boundary explicitly, e.g. *"any web page can trigger this route
with attacker-controlled `code`; the handler requires an authenticated session,
which we did not hold, so we do not claim impact beyond reachability."*

That framing survives triage. An inflated one does not.