# RECON MODE — Copy-paste pipelines (reference)

> Moved from recon-mode/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## Copy-paste pipelines

### 1. Passive seed aggregation
```bash
# CT logs — deepest passive subdomain source
curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > ct_subs.txt
curl -s "https://api.certspotter.com/v1/issuances?domain=example.com&include_subdomains=true&expand=dns_names" | jq -r '.[].dns_names[]' | sort -u >> ct_subs.txt

# All passive sources at once
subfinder -d example.com -all -silent -o subfinder.txt
amass enum -passive -d example.com -o amass.txt -timeout 5
assetfinder --subs-only example.com > assetfinder.txt
chaos-client -d example.com -o chaos.txt -silent
github-subdomains -d example.com -t $GITHUB_TOKEN -o gh_subs.txt
cero example.com | sed 's/^*\.//' | sort -u >> cero.txt

# Passive DNS
curl -s "https://api.securitytrails.com/v1/domain/example.com/subdomains" -H "APIKEY: $ST_KEY" | jq -r '.subdomains[]' | sed 's/$/.example.com/' >> all_subs_passive.txt
curl -s "https://www.virustotal.com/api/v3/domains/example.com/subdomains?limit=40" -H "x-apikey: $VT_KEY" | jq -r '.data[].id' >> all_subs_passive.txt

# ASN / netblocks → reverse-DNS (still passive, uses public PTR)
curl -s "https://api.bgpview.io/search?query_term=Example+Inc" | jq '.data.asns'
curl -s "https://api.bgpview.io/asn/AS54321/prefixes" | jq -r '.data.ipv4_prefixes[].prefix' > netblocks.txt
prips $(cat netblocks.txt) | dnsx -ptr -resp-only -silent > rdns.txt

# Combine + dedup to seed
cat ct_subs.txt subfinder.txt amass.txt assetfinder.txt chaos.txt gh_subs.txt cero.txt \
  | sort -u | grep -i "\.example\.com$" > all_subs_passive.txt
cat ct_subs.txt subfinder.txt amass.txt assetfinder.txt chaos.txt gh_subs.txt \
  | sed 's/^\*\.//; s/\.$//' | grep -E "^[a-zA-Z0-9._-]+\.example\.com$" \
  | tr '[:upper:]' '[:lower:]' | sort -u > seed.txt
wc -l all_subs_passive.txt seed.txt
```

### 2. Subdomain enumeration (permutation + mass resolution)
```bash
# Permutations
alterx -l seed.txt -enrich -o permutations.txt          # modern, regex-based
gotator -sub seed.txt -perm permutations.txt -depth 1 -numbers 5 -mindup -adv -md > altered.txt
ripgen -d seed.txt > rip.txt
cat permutations.txt altered.txt rip.txt | sort -u > all_perm.txt   # often 200K-2M

# Validated resolvers (critical for accuracy)
wget -q https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt -O resolvers.txt
dnsvalidator -tL https://public-dns.info/nameservers.txt -threads 200 -o validated_resolvers.txt

# Mass resolve — puredns has built-in wildcard handling
puredns resolve all_perm.txt \
  --resolvers validated_resolvers.txt \
  --rate-limit 5000 \
  --wildcard-tests 30 \
  --wildcard-batch 1500000 \
  --write resolved.txt \
  --write-massdns massdns_raw.txt

# Pure brute force only when seed is thin (<50)
cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
    /usr/share/seclists/Discovery/DNS/n0kovo_subdomains_huge.txt \
    /usr/share/seclists/Discovery/DNS/dns-Jhaddix.txt | sort -u > brute.txt
puredns bruteforce brute.txt example.com --resolvers validated_resolvers.txt \
  --rate-limit 5000 --wildcard-batch 1500000 --write brute_resolved.txt

# Final combine + live probe → handoff
cat seed.txt resolved.txt brute_resolved.txt | sort -u > live_candidates.txt
httpx -l live_candidates.txt -ports 80,443,8080,8443,8000,8888,8008,8081,8181,9000,9090,3000,5000,7000,7443,8443 \
  -threads 100 -timeout 5 -retries 1 -silent -o alive.txt
```
Merge in manual patterns: `{env}-{name}`/`{name}-{env}` (env: dev,staging,stg,test,qa,uat,preprod,prod,prd,beta,sandbox,demo,internal,intra,intranet,corp,backend,api,admin), `{name}{n}`/`{name}-{n}` (n 1..20), `{region}-{name}` (us,eu,ap,uw1,ue1,euw1,euc1,apn1,aps1,jp,uk,fr,de,nl,sg,au).

### 3. DNS analysis
```bash
# All record types in parallel
dnsx -l alive.txt -a -aaaa -cname -ns -mx -txt -soa -ptr -resp -silent -o dns_records.txt

# Dangling CNAMEs → takeover candidates
dnsx -l alive.txt -cname -resp-only | tee cnames.txt
for c in $(cat cnames.txt); do dig +short $c @1.1.1.1 | grep -q '^$' && echo "DANGLING: $c"; done

# AXFR (zone transfer) on every NS
for ns in $(dig +short ns example.com); do echo "=== $ns ==="; dig @"$ns" example.com AXFR +noidnout; done | tee axfr.txt

# DNSSEC NSEC zone walk (full zone without AXFR) — always try first
ldns-walk -s 1.1.1.1 example.com

# SPF/DMARC misconfig → email spoofing primitive (P3-P2)
dig +short txt example.com | grep -i spf
dig +short txt _dmarc.example.com
# SPF too permissive (+all, ?all, ~all) + DMARC p=none + missing DKIM → spoofable

dig +short caa example.com
prips $(cat netblocks.txt) | dnsx -ptr -resp-only -silent | sort -u > rdns.txt
```
Wildcard detection: query 5 random non-existent names — same IP every time = wildcard; `for i in 1 2 3 4 5; do rand=$(openssl rand -hex 8); dig +short "$rand.example.com" @1.1.1.1; done`.

### 4. Asset discovery (alive HTTP inventory)
```bash
httpx -l all_subs.txt -ports 80,443,81,591,2082,2087,2095,2096,3000,3128,4243,4567,4711,4712,4993,5000,5104,5108,5800,6543,7000,7396,7474,8000,8001,8008,8014,8042,8060,8069,8080,8081,8083,8088,8090,8091,8118,8123,8172,8222,8243,8280,8281,8333,8443,8500,8834,8880,8888,8983,9000,9043,9060,9080,9090,9091,9200,9443,9800,9981,12443,16080,18091,18092,20720,28017 \
  -threads 100 -timeout 7 -retries 1 -follow-redirects \
  -status-code -title -tech-detect -ip -cname -tls-grab -web-server -content-length -response-time -websocket \
  -json -o httpx.jsonl
cat httpx.jsonl | jq -r .url > alive.txt

# TLS cert metadata sweep → cert reuse reveals related infra; feed SANs back into sub pool
tlsx -l resolved.txt -san -cn -ja3 -ja3s -tls-version -cipher -o tls.txt
tlsx -l resolved.txt -san -resp-only -silent | tr ',' '\n' | sort -u >> all_subs.txt

# WAF/CDN detect → informs payload strategy
wafw00f -i httpx_urls.txt -o wafw00f.txt
cdncheck -i httpx_urls.txt -o cdn.txt

# Screenshot grid
cat httpx.jsonl | jq -r .url | gowitness file -f - --threads 50 --disable-db -P screenshots/
```

### 5. JS endpoint + secret extraction
```bash
katana -list alive.txt -d 5 -jc -kf all -aff -o crawl.txt
cat crawl.txt | grep -E '\.js(\?|$)' | sort -u > js_urls.txt
cat alive.txt | gau --subs --threads 10 | grep -E '\.js(\?|$)' >> js_urls.txt
sort -u js_urls.txt -o js_urls.txt

mkdir -p js && cd js && cat ../js_urls.txt | xargs -I{} -P 20 curl -sk --max-time 15 -o "$(echo {} | md5sum | cut -d' ' -f1).js" "{}"

# Endpoint extraction (jsluice is best in 2024)
find js/ -name '*.js' | xargs -I{} jsluice urls -m '*' {} | sort -u >> endpoints_from_js.txt
find js/ -name '*.js' | xargs -I{} jsluice secrets {} > js_secrets.txt

# Secret scan
trufflehog filesystem --json js/ > js_truffle.json
rg -i 'aws_secret|aws_access|stripe_(test|live)|sk_live_|sk_test_|xoxb-|xoxp-|xapp-|pk_live_|pk_test_|api[_-]?key|client[_-]?secret|firebase|sendgrid|twilio|mailgun|github_token|ghp_|ghu_|ghs_|jwt|bearer\s+[a-z0-9]+|password\s*[:=]\s*["\x27]' js/

# Sourcemap recovery (gold mine — check for //# sourceMappingURL= in every JS)
for u in $(cat js_urls.txt); do curl -sk --max-time 10 "$u.map" -o /tmp/x.map && [ -s /tmp/x.map ] && echo "FOUND .map: $u.map"; done
npx sourcemap-explorer file.js.map   # or sourcemapper -url "...main.js.map" -output recovered/

# DOM XSS sink hunt
rg -n 'document\.write|innerHTML\s*=|outerHTML\s*=|insertAdjacentHTML|\.html\(|eval\(|setTimeout\(["\x27]\\$|setInterval\(["\x27]|Function\(|setAttribute\((?:["\x27](?:on|src|href))|location\s*=|location\.href|location\.replace|location\.assign|window\.open|postMessage' js/
# postMessage handlers
rg -n 'addEventListener\(["\x27]message["\x27]' js/ -A 10
```
Grep server-rendered SPA state for leaked objects: `__INITIAL_STATE__`, `__APOLLO_STATE__`, `__NEXT_DATA__`, `__NUXT__`. Service workers (`sw.js`) cache API URLs; internal hosts like `api-internal.corp.example.com` leak as fallbacks — each is a recon win.

### 6. Wayback / historical URL mining
```bash
echo example.com | gau --subs --threads 10 > wayback_gau.txt
echo example.com | waybackurls > wayback_wbu.txt
echo example.com | urlhunter -d 30d > wayback_uh.txt
cat wayback_*.txt | sort -u > all_historical.txt
uro < all_historical.txt > all_historical_uniq.txt   # dedup by URL template

# Interesting / leaky extensions
cat all_historical_uniq.txt | grep -E '\.(php|asp|aspx|jsp|do|action|cgi|json|xml|env|bak|old|orig|backup|swp|tmp|sql|tar|tar\.gz|zip|7z|rar|log|conf|config|cnf|ini|yml|yaml|pem|key|crt|p12|pfx|properties|rb|py|pl|sh|wsdl|wadl|swagger|openapi)(\?|$)' > sus_extensions.txt

# Params → feed content/param discovery
cat all_historical_uniq.txt | unfurl --unique keys > param_names.txt
cat all_historical_uniq.txt | qsreplace FUZZ > urls_for_fuzzing.txt

# Diff historical vs current → resurrected endpoints (still live, removed from app)
comm -23 <(sort all_historical_uniq.txt) <(sort current_crawl.txt) > only_in_historical.txt
httpx -l only_in_historical.txt -mc 200,401,403 -threads 100 -silent > resurrected.txt

# Historical JS (older = more secrets, often still hosted)
cat all_historical_uniq.txt | grep -E '\.js(\?|$)' > historical_js.txt
wget -i historical_js.txt -P historical_js/
trufflehog filesystem historical_js/ > historical_js_secrets.txt
```

### 7. Content discovery / fuzzing
```bash
# ffuf — recursive directory + extension brute
ffuf -u "https://$TARGET/FUZZ" \
  -w ~/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt \
  -mc all -fc 404,403 -ac -recursion -recursion-depth 2 \
  -e .json,.xml,.bak,.old,.zip,.tar.gz,.tar,.7z,.rar,.git,.svn,.env,.yml,.yaml,.txt,.log,.sql,.bak,.swp,.tmp,.orig,.dist,.example,.config,.conf,.ini,.cfg \
  -t 50 -timeout 7 -p 0.05 -of json -o ffuf.json

# feroxbuster — recursion built-in, deep crawl
feroxbuster -u https://$TARGET -w ~/wordlists/raft-large-directories.txt \
  -t 50 -d 5 --auto-tune --auto-bail -x .json .bak .old .zip .git .env .yml \
  -C 404 -F 403 -o ferox.txt

# Vhost brute (virtual host routing on same IP)
ffuf -u "https://$TARGET/" -H "Host: FUZZ.example.com" \
  -w ~/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
  -mc 200,301,302 -fs 0 -ac -t 50 -of csv -o vhost.csv

# Param brute
arjun -u 'https://example.com/api/data' -m POST --json --headers "Authorization: Bearer $T" --stable
x8 -u 'https://example.com/api/data' -X POST -m GET POST -w ~/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt

# Header brute (hidden auth-bypass headers)
ffuf -u 'https://example.com/admin' -H 'FUZZ: 127.0.0.1' \
  -w ~/wordlists/headers.txt -mc 200,302 -fc 403,404 -ac
# headers.txt: X-Forwarded-For, X-Real-IP, X-Originating-IP, X-Remote-IP, X-Remote-Addr,
# X-Client-IP, X-Host, X-Forwarded-Host, X-Original-URL, X-Rewrite-URL,
# X-Custom-IP-Authorization, True-Client-IP, Cluster-Client-IP, CF-Connecting-IP

# HTTP methods
for m in GET POST PUT PATCH DELETE OPTIONS HEAD TRACE CONNECT PROPFIND MKCOL COPY MOVE LOCK UNLOCK; do
  curl -sk -o /dev/null -w "$m %{http_code}\n" -X "$m" "https://$TARGET/api/admin"
done
```
Wildcard-response filtering: `-fs <size>` / `-fr 'Not Found'` / `-fl <line count>` against known 404. Never use SQLi/XSS payload lists for content discovery — WAF blocks you.

### 8. API discovery
```bash
# Docs / schemas (nuclei has templates: http/exposures/apis/)
for path in api openapi.json openapi.yaml swagger.json swagger.yaml swagger-ui swagger-ui.html api-docs api/v1 api/v2 api/v3 docs redoc graphql graphiql playground apollo wsdl actuator actuator/beans actuator/mappings actuator/env actuator/health; do
  ffuf -u "https://$TARGET/$path" -mc 200,401,403 -ac -of csv -o /tmp/r.csv
done

# kiterunner — API-aware, HEAD + scrambled methods
kr scan https://example.com -A=apiroutes-210126:20210126 -o kr_results.txt
kr brute https://example.com -w ~/wordlists/api/api-endpoints.txt -o kr_brute.txt

# GraphQL discovery + introspection
for path in graphql graphiql api/graphql v1/graphql v2/graphql query subgraph; do
  curl -sk -X POST "https://$TARGET/$path" -H 'Content-Type: application/json' -d '{"query":"{__typename}"}' -o /dev/null -w "%{http_code} https://$TARGET/$path\n"
done
# introspection query → schema.json (jq .); if disabled, brute it:
clairvoyance -o schema.json -w ~/wordlists/graphql/graphql-words.txt https://$TARGET/graphql
graphw00f -t https://$TARGET/graphql -d    # fingerprint engine

# gRPC (reflection)
grpcurl -plaintext $HOST:$PORT list
grpcurl -plaintext $HOST:$PORT list mypackage.MyService

# WebSocket endpoints
rg -oE 'wss?://[^\s"\x27<>]+' js/ alive.txt > ws_endpoints.txt
wscat -c "wss://example.com/ws" -H "Authorization: Bearer $TOKEN"

# Version detection — same path, different auth per version
for v in 1 2 3 v1 v2 v3 latest internal beta legacy old; do
  curl -sk -o /dev/null -w "$v: %{http_code}\n" "https://$TARGET/api/$v/users"
done
```
Always test all methods + trailing-slash variants (`/api/users` 401 vs `/api/users/` 200). Test `Accept: application/vnd.example.v1+json`. GraphQL aliases (`x: user(id:1) y: user(id:2)`) bypass per-field rate limits.

### 9. Port scanning & services
```bash
naabu -list netblocks.txt -p - -rate 10000 -exclude-cdn -o ports_full.txt
# high-value management/db/queue ports:
naabu -list netblocks.txt -p 9200,9300,27017,6379,5432,3306,11211,5984,1521,1433,8500,8086,5601,15672,8161,9000,3000,8081,8080,7474,9090,8888,9870,50070,2375,2376,6443,10250,4243,28017,8443,50000,5985,5986,5900,5901,5902 -rate 5000 -o highvalue_ports.txt

nmap -sV -sC -O -Pn --script=vulners,vuln,banner,http-enum,smb-enum-shares,ftp-anon,smb-vuln-* \
  -iL <(awk -F: '{print $1}' ports_full.txt | sort -u) -p $(awk -F: '{print $2}' ports_full.txt | sort -u | tr '\n' ',') -oA nmap_full

# masscan alternative (needs sudo; >50k pps risks IP ban — throttle)
sudo masscan -p1-65535 --rate 25000 -iL netblocks.txt -oG masscan.gnmap
# UDP top-100 (DNS, SNMP, NTP, IPMI)
sudo nmap -sU --top-ports 100 -sV -iL netblocks.txt -oA nmap_udp
# Passive scan — Shodan already scanned it (smap)
smap -iL netblocks.txt -oG smap.gnmap
```

### 10. Fingerprinting tech stack
```bash
httpx -l alive.txt -tech-detect -web-server -title -status-code -ip -cdn -follow-redirects -json -o tech.jsonl
cat tech.jsonl | jq -r 'select(.tech) | "\(.url)\t\(.tech | join(","))"' | sort -u
wafw00f -i alive.txt -o wafw00f.txt
nuclei -l alive.txt -t http/technologies/ -severity info -silent -o tech_nuclei.txt
retire --jspath js/ --outputformat json --outputpath retire.json   # retired JS libs = CVE candidates

# Favicon hash pivot → related infra + origin IPs
python3 -c '
import mmh3, requests, codecs, sys
for u in sys.argv[1:]:
  try:
    r = requests.get(u + "/favicon.ico", timeout=5, verify=False)
    if r.status_code == 200 and r.content:
      print(u, mmh3.hash(codecs.encode(r.content, "base64")))
  except: pass
' $(cat alive.txt)
# Then: Shodan http.favicon.hash:HASH | FOFA icon_hash="HASH" | ZoomEye iconhash:"HASH"
# Censys uses MD5 of icon: services.http.response.favicons.md5_hash
```
WAF tell-tales: `Server: cloudflare`, Akamai `X-Cache`/`AkamaiGHost`, AWS WAF `x-amzn-RequestId`, Imperva `X-Iinfo`, Sucuri `X-Sucuri-ID`, F5 `BIGipServer` cookie. CDN strips `Server` → pivot via JARM (`jarm example.com:443`; same JARM = same TLS stack). Old WAFs ignore HTTP/2 — force it with `httpx -http2`.

### 11. Shodan / Censys / FOFA queries
```bash
shodan domain example.com
shodan search 'ssl.cert.subject.cn:"example.com"'
shodan search 'ssl:"example.com" 200'
shodan search 'org:"Example Inc"'
shodan search 'hostname:.example.com'
shodan search 'http.favicon.hash:HASH'
shodan search 'http.html:"example.com" -hostname:example.com'     # origin IPs behind CDN
shodan search 'product:Jenkins hostname:example.com'
shodan search 'product:Kubernetes hostname:example.com'
shodan search 'ssl.cert.subject.cn:"example.com" -country:"US"' | grep -v 'cloudflare'   # CF bypass

censys search 'services.tls.certificates.leaf_data.subject.common_name: "example.com"'
censys search 'services.tls.certificates.leaf_data.names: "example.com"' --pages 5 -o censys_results.json
censys search 'autonomous_system.organization: "Example Inc"'
censys search 'services.http.response.html_title: "Login" and services.tls.certificates.leaf_data.subject.common_name: "example.com"'
censys search 'services.jarm.fingerprint: "07d14d16d21d21d00042d41d00041d24a458a375eef0c576d23a7bab9a9fb1"'

# FOFA queries are base64-encoded URL params
echo 'cert="example.com"' | base64 -w0    # paste into FOFA query
# domain="example.com" | cert="example.com" | icon_hash="-1234567890" | title="Example Login"
# header="example.com" | body="example.com" | ip="1.2.3.4"

jarm -i ips.txt -o jarm.txt   # group by JARM to identify shared infra
```
Default Swagger UI favicon hash 1606627656 → `shodan search 'http.favicon.hash:1606627656'` for org-wide API docs.

### 12. GitHub / GitLab / Bitbucket dorks
```bash
gitdorker -tf ~/.config/gitdorker/tokens.txt -q example.com -d ~/tools/gitdorker/Dorks/medium_dorks.txt -o gitdorker_results/ -ev 0.5
trufflehog github --org=examplecorp --json --concurrency=8 --only-verified > truffle_examplecorp.json
for u in $(cat employees_github_handles.txt); do trufflehog github --user="$u" --json --only-verified >> truffle_users.json; done

# Manual code search (GitHub API, ~5K req/h per token — rotate 5+)
for q in 'password' 'api_key' 'secret' 'jwt' 'token' 'aws_secret' 'BEGIN RSA PRIVATE KEY' 'authorization: bearer' 'Set-Cookie:' 'admin' 'staging' 'internal'; do
  curl -s -H "Authorization: token $GITHUB_TOKEN" "https://api.github.com/search/code?q=%22example.com%22+$q&per_page=100" \
    | jq -r '.items[] | "[\(.repository.full_name)] \(.html_url)"'
done > gh_code_search.txt

# Deleted/dangling blobs in history
git clone --mirror https://github.com/example/repo.git
gitleaks detect --no-git -v
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(rest)' | awk '$1=="blob" {print $2}' > all_blobs.txt

# Pastebin + Dockerhub
psbdmp search example.com
curl -s 'https://hub.docker.com/v2/search/repositories/?query=example' | jq -r '.results[].repo_name'
```
High-signal dorks (`dorks/medium_dorks.txt`): `"example.com" password`, `api_key`, `secret_key`, `client_secret`, `auth_token`, `jwt`, `aws_secret_access_key`, `AKIA`, `SLACK_TOKEN`, `stripe_api_key`, `twilio_auth_token`, `"ghp_"`, `"BEGIN RSA PRIVATE KEY"`, `filename:.env`, `filename:.npmrc _auth`, `filename:.dockercfg auth`, `filename:.git-credentials`, `filename:id_rsa`, `extension:sql`, `extension:p12`, `"mongodb://"`, `"postgres://"`, `DATABASE_URL`, `s3://`, `"blob.core.windows.net"`, `"storage.googleapis.com"`.
Live-verify high-confidence hits only: Slack → `curl -s -X POST https://slack.com/api/auth.test -H "Authorization: Bearer $token"`; AWS → `aws sts get-caller-identity` (minimum-impact PoC — never escalate beyond that).

### 13. Google / search-engine dorks
```bash
udork -t example.com -d ~/tools/uDork/dorks.txt -o results.html
```
Subdomain: `site:*.example.com -www`, `site:example.com -site:blog.example.com -site:docs.example.com`, `site:example.com inurl:staging|test|dev|internal`
Admin: `site:example.com inurl:login|admin|wp-admin|dashboard|portal|console|management`
Files: `site:example.com ext:env|log|sql|bak|zip|tar.gz`, `filetype:pdf "password"`
Source/backup: `site:example.com inurl:.git|.svn|.DS_Store|dump.sql|db.sql`
API: `site:example.com inurl:swagger|openapi|api-docs|graphql|graphiql|playground`
Errors (vuln signal): `site:example.com "Whitelabel Error Page"`, `"stack trace"`, `"PHP Parse error"`, `"java.lang.NullPointerException"`, `"sql syntax near"`
Buckets: `"example" site:s3.amazonaws.com`, `"example" inurl:storage.googleapis.com`, `"example" inurl:blob.core.windows.net`
XSS/SSRF indicators: `site:example.com inurl:redirect=|url=|returnTo=|next=|dest=|return=`, `inurl:fetch=|img=|proxy=|webhook=`
Index-of: `site:example.com intitle:"index of /"`
Pasteleaks: `site:pastebin.com example.com`, `site:gist.github.com example.com`
After ~20 queries: rotate UA + proxy + 5-15s sleep; use Bing/Yandex (different crawlers, less indexed corners).

### 14. Cloud asset discovery (AWS / GCP / Azure)
```bash
cloud_enum --keyword example --keyword examplecorp --keyword example-prod --keyword example-dev --keyword example-staging --keyword example-internal --logfile cloud_enum.log

# S3 naming permutations
for prefix in '' 'dev-' 'prod-' 'staging-' 'qa-' 'test-' 'backup-' 'data-' 'logs-' 'media-' 'static-' 'assets-' 'public-' 'private-' 'internal-' 'cdn-' 'images-' 'uploads-' 'web-' 'api-' 'app-'; do
  for suffix in '' '-dev' '-prod' '-staging' '-qa' '-test' '-backup' '-data' '-logs' '-prd'; do
    echo "${prefix}example${suffix}"; echo "${prefix}examplecorp${suffix}"
  done
done | sort -u > bucket_names.txt
s3scanner -bucket-file bucket_names.txt -enumerate-bucket-objects -threads 50 -output s3_results.json

# AWS Cognito identity pools (unauth → AssumeRole creds)
aws cognito-identity get-id --identity-pool-id us-east-1:xxxx --no-sign-request
aws cognito-identity get-credentials-for-identity --identity-id $ID --no-sign-request

# Azure blob existence via DNS
for n in example examplecorp examplestg exampledev; do
  for variant in storage data backup logs files; do
    h="${n}${variant}.blob.core.windows.net"; dig +short "$h" @1.1.1.1 | head -1 | grep -q '\.' && echo "EXISTS: $h"
  done
done
# then test anonymous listing: curl -sI "https://$h/?comp=list"

# GCP
python3 gcpbucketbrute.py -k example -u
curl -sI "https://storage.googleapis.com/$n/" | head -1

# Lambda function URLs leaked in JS/wayback/GitHub
rg -oE '[a-z0-9]+\.lambda-url\.[a-z0-9-]+\.on\.aws' js/ wayback/ -h | sort -u

# Dangling CloudFront distributions → takeover
for sub in $(cat alive.txt); do
  cname=$(dig +short cname "$sub" @1.1.1.1 | tail -1)
  if echo "$cname" | grep -q 'cloudfront.net'; then
    code=$(curl -sk -o /dev/null -w '%{http_code}' "https://$sub/")
    [ "$code" = "403" ] && curl -sk "https://$sub/" | grep -q 'Bad request' && echo "DANGLING: $sub → $cname"
  fi
done

# Azure AD tenant discovery (no auth)
curl -sI "https://login.microsoftonline.com/example.com/.well-known/openid-configuration"
```
S3 takeover signal: sub CNAMEs to `*.s3.amazonaws.com` returning `NoSuchBucket` → register that bucket. `?comp=list` on Azure blob = anonymous-read names. Lambda URL with `AuthType: NONE` = potential unauth RCE proxy.

### 15. Mobile app recon (if app in scope)
```bash
apkmd download com.example.app -o example.apk
apkleaks -f example.apk -o apkleaks.txt
apktool d -f example.apk -o ext/
rg -nIE 'https?://[a-zA-Z0-9./_?=&%-]+' ext/ | sort -u > urls_in_apk.txt
rg -nIE '(api[_-]?key|secret|token|password|jwt|firebase|aws_(access|secret)|sk_(live|test)_)["\x27`:= ]' ext/ > secrets_in_apk.txt
# Firebase misconfig
rg -oE '[a-z0-9-]+\.firebaseio\.com' ext/ | sort -u | while read fb; do curl -s "https://$fb/.json" | head -c 200; echo " <- $fb"; done
# Exported components → drozer: app.activity.info / app.provider.info / scanner.provider.injection / scanner.provider.traversal
```
App-only endpoints (`/api/v3/internal/...` with no auth assuming "only mobile hits this") are a frequent high-value finding. Insecure deep links, exported ContentProviders, `setJavaScriptEnabled(true)` + `addJavascriptInterface` WebViews, and `usesCleartextTraffic="true"` are all reportable.

### 16. Full automation pipeline (skeleton — run every 6h, diff + notify)
```bash
TARGET=$1; DATE=$(date +%Y%m%d-%H%M); OUT=$HOME/hunting/$TARGET/$DATE; mkdir -p "$OUT" && cd "$OUT"
subfinder -d "$TARGET" -all -silent | tee subs_subfinder.txt
chaos-client -d "$TARGET" -silent | tee subs_chaos.txt
github-subdomains -d "$TARGET" -t "$GITHUB_TOKEN" -o subs_github.txt
cat subs_*.txt | sort -u > subs.txt
alterx -l subs.txt -enrich -silent > permutations.txt
puredns resolve permutations.txt -r ~/.config/resolvers.txt --wildcard-batch 1500000 -q > resolved.txt
cat subs.txt resolved.txt | sort -u > all_subs.txt
httpx -l all_subs.txt -threads 100 -timeout 7 -follow-redirects -status-code -title -tech-detect -ip -cdn -tls-grab -web-server -json -o httpx.jsonl
cat httpx.jsonl | jq -r .url > alive.txt
katana -list alive.txt -d 3 -jc -kf all -silent -o crawl.txt
cat alive.txt | gau --threads 10 --subs > gau.txt
cat crawl.txt gau.txt | sort -u | uro > all_urls.txt
grep -E '\.js(\?|$)' all_urls.txt | sort -u > js_urls.txt
cat js_urls.txt | xargs -I{} -P 20 curl -sk --max-time 10 -o "js/$(echo {} | md5sum | cut -d' ' -f1).js" "{}"
trufflehog filesystem js --only-verified --json > js_secrets.json
nuclei -l alive.txt -severity medium,high,critical -t ~/nuclei-templates/ -rl 100 -timeout 10 -silent -o nuclei.txt -json -store-resp -store-resp-dir nuclei_resp/
PREV=$(ls -1 ~/hunting/$TARGET/ | sort | grep -v "^$DATE" | tail -1)
[ -n "$PREV" ] && comm -23 <(sort all_subs.txt) <(sort ~/hunting/$TARGET/$PREV/all_subs.txt) > new_subs.txt
[ -s nuclei.txt ] && notify --title "[$TARGET] nuclei findings"   # diff-before-notify; only ping on delta
```
cron: `5 */6 * * * recon_pipeline.sh example.com >> /var/log/recon-example.log 2>&1` — plus certstream listener to trigger on new CT certs, axiom for distributed scanning (`axiom-scan all_subs.txt -m httpx -p 80,443,8080,8443`). Keep `~/.nuclei-ignore.yaml` to silence always-alerting templates; throttle to avoid self-DoS/rate-limit breaches.
