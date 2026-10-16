# AUTOMATION MODE — Environment, monitoring, notifications, templates (reference)

> Moved from automation-mode/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## VPS Setup
Specs: 2 vCPU, 4-8GB RAM, 60GB SSD, Ubuntu 22.04 LTS (DO/Linode/Hetzner/Vultr, $5-20/mo).
```bash
sudo apt update && sudo apt install -y \
  build-essential git curl wget jq python3 python3-pip golang nodejs npm \
  unzip tmux htop ripgrep fzf nmap masscan dnsutils whois \
  libpcap-dev libldns-dev
GO111MODULE=on go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install -v github.com/projectdiscovery/chaos-client/cmd/chaos@latest
go install -v github.com/projectdiscovery/notify/cmd/notify@latest
go install -v github.com/projectdiscovery/alterx/cmd/alterx@latest
go install -v github.com/lc/gau/v2/cmd/gau@latest
go install -v github.com/tomnomnom/waybackurls@latest
go install -v github.com/tomnomnom/assetfinder@latest
go install -v github.com/tomnomnom/qsreplace@latest
go install -v github.com/tomnomnom/unfurl@latest
go install -v github.com/d3mondev/puredns/v2@latest
go install github.com/owasp-amass/amass/v4/overalls/cmd/amass@latest   # or: apt install amass / download release binary
go install -v github.com/hahwul/dalfox/v2@latest
go install -v github.com/ffuf/ffuf/v2@latest
go install -v github.com/OJ/gobuster/v3@latest
go install -v github.com/sensepost/gowitness@latest
pip3 install --user trufflehog3 sqlmap arjun apkleaks dirsearch
nuclei -ut
wget https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt -O ~/.config/resolvers.txt
```
Persistence: results to `/home/hunter/results/$DATE/`, daily S3 backup, pin versions in `/opt/tools/`. Security: SSH key only, ufw allow OpenSSH, fail2ban, unattended-upgrades.

## Docker Hunting Environment
```dockerfile
FROM ubuntu:22.04
RUN apt update && apt install -y \
    git curl wget jq python3 python3-pip golang nodejs npm unzip ripgrep nmap masscan
ENV GOPATH=/root/go PATH=/root/go/bin:$PATH
RUN go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest && \
    go install github.com/projectdiscovery/httpx/cmd/httpx@latest && \
    go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest && \
    go install github.com/projectdiscovery/katana/cmd/katana@latest && \
    go install github.com/projectdiscovery/naabu/v2/cmd/naabu@latest && \
    go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest && \
    go install github.com/lc/gau/v2/cmd/gau@latest && \
    go install github.com/hahwul/dalfox/v2@latest && \
    go install github.com/ffuf/ffuf/v2@latest
RUN pip3 install sqlmap arjun trufflehog3 apkleaks dirsearch
RUN mkdir -p /root/wordlists && \
    git clone --depth 1 https://github.com/danielmiessler/SecLists /root/wordlists/SecLists
ENTRYPOINT ["/bin/bash"]
```
```bash
docker build -t hunter .
docker run -it --rm -v $PWD:/work -w /work hunter
```
```yaml
services:
  hunter:
    build: .
    volumes:
      - .:/work
      - ~/wordlists:/root/wordlists
    networks: [bbnet]
  burp:
    image: portswigger/burp-pro
    networks: [bbnet]
networks:
  bbnet:
```

## Monitoring New Assets
Certstream live-cert feed → Discord:
```python
import certstream, os, requests
TARGETS = open('targets.txt').read().split()
def cb(msg, ctx):
    if msg.get('message_type') != 'certificate_update': return
    for d in msg['data']['leaf_cert']['all_domains']:
        for t in TARGETS:
            if d.endswith('.'+t) or d == t:
                requests.post(os.environ['DISCORD_WH'], json={'content': f'NEW CERT: {d}'})
certstream.listen_for_events(cb, url='wss://certstream.calidog.io')
```
Subdomain diffing (cron):
```bash
TARGET=$1
DIR=$HOME/subwatch/$TARGET
mkdir -p $DIR
subfinder -d $TARGET -all -silent | sort -u > $DIR/now.txt
[ -f $DIR/last.txt ] || cp $DIR/now.txt $DIR/last.txt
NEW=$(comm -23 $DIR/now.txt $DIR/last.txt)
[ -n "$NEW" ] && echo "$NEW" | python3 ~/scripts/notify_discord.py "[$TARGET] new subs"
cp $DIR/now.txt $DIR/last.txt
```
Tech/asset fingerprint watch:
```bash
httpx -l alive.txt -tech-detect -title -ip -json > now.jsonl
diff <(jq -S . last.jsonl) <(jq -S . now.jsonl) | python3 notify.py "tech changed"
```
Tools: chaos.projectdiscovery.io, certstream, bbscope.

## Monitoring New Programs
Sources: HackerOne hacktivity/directory, Bugcrowd `/programs/list`, Intigriti `/programs`, YesWeHack `/program`, Immunefi `/explore`, HackenProof `/programs`, `@disclosedh1`/`@bugbounty` roundups.
```python
import requests, json, time, os
SEEN=set(open('seen.txt').read().split()) if os.path.exists('seen.txt') else set()
while True:
    r = requests.get('https://hackerone.com/programs/search?query=&sort_type=published_at&page=1', headers={'Accept':'application/json'})
    for p in r.json()['programs']:
        if p['handle'] not in SEEN:
            requests.post(os.environ['DISCORD_WH'], json={'content': f"NEW: {p['name']} https://hackerone.com/{p['handle']}"})
            SEEN.add(p['handle']); open('seen.txt','a').write(p['handle']+'\n')
    time.sleep(3600)
```
Cron: `*/30 * * * * /home/ubuntu/scripts/watch_h1.py >> /var/log/h1watch.log 2>&1`
Tools: chaos (daily scope dumps), bbscope, bbradar.

## Monitoring Scope Changes
New asset = first-mover advantage; diff scope page hourly:
```bash
#!/usr/bin/env bash
TARGET=$1
URL="https://hackerone.com/${TARGET}/policy_scopes/all_eligible/json"
DIR="$HOME/scope_watch/${TARGET}"
mkdir -p "$DIR"
curl -sk "$URL" -o "$DIR/now.json"
[ -f "$DIR/last.json" ] || cp "$DIR/now.json" "$DIR/last.json"
diff <(jq -S . "$DIR/last.json") <(jq -S . "$DIR/now.json") > "$DIR/diff.txt"
if [ -s "$DIR/diff.txt" ]; then
  cat "$DIR/diff.txt" | python3 ~/scripts/notify_discord.py "[$TARGET] scope changed"
  cp "$DIR/now.json" "$DIR/last.json"
fi
```
Cron: `*/15 * * * * /home/ubuntu/scripts/scope_watch.sh hackerone-target`. Tools: bbscope, chaos.

## Notifications (Telegram / Discord / Slack)
Telegram: `@BotFather` → `/newbot` → TOKEN; message bot then `curl https://api.telegram.org/bot<TOKEN>/getUpdates` for chat_id.
```python
import os, requests, sys
def tg(text, files=None):
    TOKEN=os.environ['TG_BOT_TOKEN']; CHAT=os.environ['TG_CHAT_ID']
    if files:
        for f in files:
            requests.post(f'https://api.telegram.org/bot{TOKEN}/sendDocument',
                          data={'chat_id':CHAT,'caption':text[:1024]},
                          files={'document':open(f,'rb')})
    else:
        requests.post(f'https://api.telegram.org/bot{TOKEN}/sendMessage',
                      data={'chat_id':CHAT,'text':text[:4000],'parse_mode':'Markdown'})
if __name__ == '__main__':
    tg(sys.stdin.read(), sys.argv[1:])
```
Discord webhook (Server settings → Integrations → Webhooks):
```python
import os, requests, sys
WH = os.environ['DISCORD_WH']
def discord(text, file=None):
    if file:
        requests.post(WH, data={'content': text[:1900]}, files={'file': open(file,'rb')})
    else:
        requests.post(WH, json={'content': text[:1900]})
discord(sys.stdin.read(), sys.argv[1] if len(sys.argv)>1 else None)
```
Slack incoming webhook:
```bash
curl -X POST -H 'Content-type: application/json' --data '{"text":"new finding"}' $SLACK_WH
```

## ffuf Automation
Content discovery wrapper:
```bash
#!/usr/bin/env bash
TARGET=$1
WL=${2:-~/wordlists/raft-large-directories.txt}
ffuf -u "https://${TARGET}/FUZZ" \
  -w "$WL" \
  -mc all -fc 404 -ac \
  -e .json,.xml,.bak,.old,.zip,.git,.env,.yml,.yaml,.txt,.log,.sql \
  -recursion -recursion-depth 2 \
  -t 50 -timeout 7 -p 0.05 \
  -of json -o "${TARGET//\//_}.json"
```
Parameter discovery: `ffuf -u "https://target/api?FUZZ=test" -w ~/wordlists/burp-parameter-names.txt -mc 200 -fs 0 -ac`
Vhost: `ffuf -u "https://${IP}/" -H "Host: FUZZ.${TARGET}" -w ~/wordlists/subdomains-top1million-110000.txt -mc 200,301,302 -fs 0 -ac`
Header bypass: `ffuf -u "https://target/admin" -H "FUZZ: 127.0.0.1" -w ~/wordlists/headers.txt -mc 200,302 -fc 403,404 -ac`

## Nuclei Custom Templates
```yaml
id: my-custom-check
info:
  name: Detect Specific Endpoint Misconfig
  author: hunter
  severity: high
  tags: misconfig,custom
http:
  - method: GET
    path:
      - "{{BaseURL}}/admin/debug"
    matchers-condition: and
    matchers:
      - type: word
        words:
          - "DEBUG"
          - "stack trace"
        condition: or
      - type: status
        status:
          - 200
```
Run: `nuclei -t custom-templates/ -l targets.txt -o findings.txt -severity medium,high,critical`
DSL functions: `to_lower, to_upper, base64, base64_decode, md5, sha256, len, contains`; e.g. `dsl: ['contains(body,"foo") && status_code == 200']`
Dynamic extractor:
```yaml
extractors:
  - type: regex
    regex:
      - 'token=([a-zA-Z0-9]+)'
    group: 1
```
Multi-step chaining:
```yaml
http:
  - raw:
      - |
        POST /login HTTP/1.1
        Host: {{Hostname}}
        ...
    extractors:
      - type: regex
        name: token
        regex: ['token=([a-zA-Z0-9]+)']
        internal: true
  - raw:
      - |
        GET /admin HTTP/1.1
        Host: {{Hostname}}
        Cookie: session={{token}}
```

## Burp Suite Automation
Extensions: Autorize, Param Miner, HTTP Request Smuggler, Active Scan++, Backslash Powered Scanner, Turbo Intruder, JSON Web Tokens, AuthMatrix, DOM Invader, Hackvertor, BurpJSLinkFinder, JS Miner, Param Auth Test, Burp Bounty, Software Vulnerability Scanner, 403 Bypasser, Logger++.

REST API (Pro; `http://127.0.0.1:1337`, config under User options → Misc):
```bash
curl -X POST 'http://127.0.0.1:1337/v0.1/scan' -d '{"urls":["https://target.com"]}' -H 'Content-Type: application/json'
```
Session handling: auto-relogin macros for long scans, CSRF token refresh, header injection (custom auth). Intruder presets: Cluster bomb (credential stuffing), Sniper (single-param fuzz), Pitchfork (paired wordlists). Turbo Intruder templates: race-single-packet-attack, examples/many-requests, ssrf-mass-test.

## Custom Tool Development
Build custom when: existing tool can't handle your edge case, need finer output control for downstream, speed bottlenecked, target has quirks (custom protocols, weird auth). Tips: I/O-bound → async (Go goroutines, asyncio, Node); CPU-bound → Rust/Go; quick → bash + `xargs -P`; pipe-friendly stdin/stdout; Unix philosophy, one thing well. Reference: tomnomnom tools (assetfinder, gf, qsreplace, anew).

Go concurrent scanner:
```go
package main
import ( "fmt"; "net/http"; "bufio"; "os"; "sync"; "time"; "crypto/tls" )

func main() {
    sem := make(chan struct{}, 50)
    var wg sync.WaitGroup
    s := bufio.NewScanner(os.Stdin)
    client := &http.Client{Timeout: 7*time.Second, Transport: &http.Transport{TLSClientConfig: &tls.Config{InsecureSkipVerify: true}}}
    for s.Scan() {
        url := s.Text()
        sem <- struct{}{}; wg.Add(1)
        go func(u string) {
            defer wg.Done(); defer func(){<-sem}()
            r, err := client.Get(u)
            if err != nil { return }
            defer r.Body.Close()
            if r.StatusCode == 200 { fmt.Println(u) }
        }(url)
    }
    wg.Wait()
}
```
Python async template:
```python
import asyncio, httpx, sys
async def probe(client, url):
    try:
        r = await client.get(url, timeout=7)
        if r.status_code == 200: print(url, flush=True)
    except: pass
async def main():
    sem = asyncio.Semaphore(50)
    async with httpx.AsyncClient(verify=False) as c:
        async def w(u):
            async with sem: await probe(c, u)
        urls = sys.stdin.read().split()
        await asyncio.gather(*(w(u) for u in urls))
asyncio.run(main())
```
