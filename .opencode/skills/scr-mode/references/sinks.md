# SCR MODE — Dangerous-function grep + per-language sinks (reference)

> Moved from scr-mode/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## 3. Cross-Language Dangerous-Function Grep

Run these greps repo-wide first; every hit is a candidate sink.
```bash
# Eval-class — arbitrary code
rg -n 'eval\(|exec\(|Function\(|setTimeout\([^,]*,|new\s+Function|compile\(|importlib|__import__\('
# Shell injection
rg -n 'system\(|popen\(|shell_exec|passthru|exec\.Command|Runtime\.getRuntime|child_process\.exec|subprocess\..*shell=True|backtick'
# Deserialization
rg -n 'pickle\.load|yaml\.load|Marshal\.load|unserialize\(|ObjectInputStream|BinaryFormatter|DataContractSerializer|TypeNameHandling|node-serialize|deserialize'
# Template injection
rg -n 'render_template_string|Twig.*sandbox.*disable|ERB\.new|eval.*template|jinja2.*Template'
# SSRF sinks
rg -n 'urlopen|requests\.get|axios\.get|http\.Get|HttpClient\.GetAsync|fetch\(|file_get_contents.*http|curl_exec'
# SQL injection (raw interpolation)
rg -n 'executeQuery\(.*\+|f"SELECT|format!.*SELECT|sprintf.*SELECT|query\(`.*\$|\.raw\(|\.exec\(.*\$'
# Path traversal
rg -n 'open\([a-zA-Z_]+\)|fs\.readFile\([a-zA-Z_]+|File\.open\([a-zA-Z_]+|os\.path\.join.*request|\.\./'
# Hardcoded secrets
rg -nE '(password|secret|token|api[_-]?key)\s*[:=]\s*["'']'
```
---

## 4. Per-Language Sink Reference

### 4.1 Python

**Dangerous functions — flag every occurrence**
```python
eval(...)                           # arbitrary code
exec(...)                           # arbitrary code
compile(...)                        # arbitrary code
os.system(...)                      # shell
subprocess.Popen(..., shell=True)   # shell injection
subprocess.call(..., shell=True)    # shell
subprocess.run(..., shell=True)     # shell
os.popen(...)                       # shell
pickle.loads(...) / pickle.load(...)      # deserialization RCE
yaml.load(...)                      # use safe_load (yaml.unsafe_load is RCE)
marshal.loads(...)                  # deserialization
__import__(user_input)              # arbitrary module
getattr(obj, user_input)            # attribute traversal
open(user_input)                    # path traversal
mark_safe(user_input)               # Django XSS
django.utils.safestring.mark_safe
jinja2.Markup(user_input)
flask.render_template_string(user_input)   # SSTI
shell=True
```
**Django sinks**
```python
Model.objects.raw(f"SELECT * FROM x WHERE id={uid}")   # SQLi
.extra(where=[user_input]) / .extra(select={'x': user_input})   # SQLi
connection.cursor().execute(f"...")                    # SQLi
{{ user_input|safe }}                                  # bypasses autoescape
{% autoescape off %}{{ user_input }}{% endautoescape %}
requests.get(user_url)                                  # SSRF (no allowlist)
HttpResponseRedirect(user_input) / redirect(request.GET['next'])  # open redirect
request.session.cycle_key() missing on login            # session fixation
@csrf_exempt                                            # check why
```
**Flask sinks**
```python
return f"<h1>Hello {name}</h1>"      # XSS
return render_template_string(name)   # SSTI
return send_file(name)                # path traversal
```
**FastAPI** — Pydantic is generally safe, but accepting `user: dict` (no schema) enables mass assignment: `User(**user).save()`.
```bash
semgrep --config=p/python --config=p/django --config=p/flask --config=p/owasp-top-ten src/
```
### 4.2 JavaScript / Node.js

**Dangerous functions**
```javascript
eval(...)                                     // code exec
Function(...)                                 // code exec
setTimeout(stringArg, ...) / setInterval(stringArg, ...)  // code exec
require(userInput)                            // arbitrary module
child_process.exec(userCmd) / execSync(userCmd)
child_process.spawn('sh', ['-c', userCmd])    // shell injection
vm.runInNewContext(userCode)                  // sandbox escape risk
fs.readFile(userPath) / fs.writeFile(userPath) // path traversal
res.sendFile(userPath)                        // path traversal
res.redirect(userInput)                       // open redirect
serialize.unserialize(input)                  // node-serialize RCE
yaml.load(input)                              // deserialization
Object.assign({}, userInput)                  // prototype pollution
_.merge(target, userInput) / _.set(obj, userPath, val)  // PP
JSON.parse + setPath                           // PP
```
**Express patterns**
```javascript
res.send(`<h1>${user.name}</h1>`);             // XSS
res.send(req.query.q);                          // reflected XSS
res.redirect(req.query.next);                   // open redirect
axios.get(req.body.url); fetch(req.body.url);   // SSRF
db.query(`SELECT * FROM u WHERE id=${id}`);     // SQLi
app.use(cors({origin: '*', credentials: true})); // dangerous combo
app.delete('/api/users/:id', (req,res)=>{});     // no auth middleware
User.update(req.body);                          // mass assignment, no allowlist
Object.assign(target, JSON.parse(req.body));    // prototype pollution
mongoose.find({$where: userInput})              // NoSQL injection
```
**Browser-side XSS sinks**
```javascript
element.innerHTML = userInput;                  // XSS
element.outerHTML = userInput;
element.insertAdjacentHTML(pos, userInput);
document.write(userInput);
$(el).html(userInput);                          // jQuery
location = userInput; location.href = userInput; // open redirect / javascript:
window.open(userInput);
eval(userInput); new Function(userInput); setTimeout(userInput);
```
```bash
semgrep --config=p/javascript --config=p/typescript --config=p/nodejsscan --config=p/expressjs src/
```
### 4.3 Java / Spring

**Dangerous functions**
```java
Runtime.getRuntime().exec(userInput)              // shell
new ProcessBuilder(userArgs).start()              // shell
ScriptEngine.eval(userInput)                      // javax.script — JS in JVM
ObjectInputStream.readObject(stream)              // deserialization RCE
new InitialContext().lookup(userInput)            // JNDI — Log4Shell-style
Class.forName(userClassName).newInstance()        // arbitrary class
DriverManager.getConnection(userUrl)              // JDBC URL injection
Statement.executeQuery("SELECT * FROM x WHERE id=" + id)   // SQLi
// XXE — factories without secureProcessing / external entities disabled:
DocumentBuilderFactory.newInstance() / SAXParserFactory / XMLInputFactory / TransformerFactory
```
**Spring patterns**
```java
// SpEL injection
@Value("#{T(java.lang.Runtime).getRuntime().exec('id')}")   // if user controls SpEL
// Path traversal
Resource resource = resourceLoader.getResource("file:" + userInput);
// SSRF
RestTemplate.getForObject(userUrl, ...);
WebClient.get().uri(userUrl).retrieve();
// Mass assignment — @RequestBody User user without @JsonIgnore on sensitive fields;
// public void update(User u) { userRepo.save(u); }  updates everything
// Spring Cloud Function CVE pattern
spring.cloud.function.routing-expression         // RCE if exposed
// Open redirect / CSRF
return "redirect:" + userInput;
@EnableWebSecurity → http.csrf().disable()       // check why
```
**Deserialization gadgets** — Apache Commons-Collections (CC1–CC11), Spring AOP (Spring1, Spring2), ROME, Hibernate, MyFaces, Click, JBoss.
```bash
semgrep --config=p/java --config=p/spring --config=p/owasp-top-ten src/
spotbugs -include findsecbugs.xml -textui src/
```
### 4.4 PHP

**Dangerous functions**
```php
eval($x)                          // code exec
assert($x)                        // <PHP 8: code exec
preg_replace('/x/e', $repl, ...)  // <PHP 7: code exec via /e
create_function('$a', $code)      // <PHP 7.2: code exec
include $userPath / require $userPath / include_once / require_once   // LFI/RFI
system($userCmd); exec($userCmd); passthru($userCmd);
shell_exec($userCmd); popen($userCmd,'r'); proc_open(...);
`$userCmd`                        // backtick = shell_exec
unserialize($userInput)           // deserialization
file_get_contents($userInput); fopen($userInput, ...); copy($userSrc, ...)   // SSRF/LFI
move_uploaded_file($file, $userDest)            // file write
$$varName = ...                   // variable variable injection
extract($_REQUEST); parse_str($userStr, $arr)  // var pollution / register_globals-like
```
**Laravel**
```php
DB::raw("...$user..."); DB::statement("DROP TABLE $table")  // SQLi
@php $x = $user @endphp            // SSTI in Blade if extended
{!! $user !!}                      // raw output → XSS
File::get($user)                   // path traversal
```
**Symfony / Twig** — `{{ user_input|raw }}` is XSS; `{{ user_input|e('html') }}` is safe.
**Type juggling (auth bypass)** — PHP < 8 loose comparisons:
```php
"0" == false; "abc" == 0;
"0e1234" == "0e5678";                 // both treated as 0
md5("240610708") == md5("QNKCDZO")   // 0e... collision
strcmp(array(), "x") → NULL == 0      // bypass
in_array("1", [1,2,3], false)         // loose
```
### 4.5 Ruby / Rails

**Dangerous methods**
```ruby
eval(input); instance_eval(input); class_eval(input); Kernel.eval(input)
Marshal.load(input)          # deserialization RCE
YAML.load(input)             # use safe_load
ERB.new(input).result        # SSTI
send(method, ...); public_send(method, ...)   # RCE if method controlled
constantize(input)           # arbitrary class
String#%(format)             # format string if format user-controlled
File.open(input); File.read(input)            # LFI
Open3.popen3(input); %x[#{input}]; `#{input}`; system(input); exec(input)
```
**Rails patterns**
```ruby
User.where("name = '#{name}'")                 # SQLi — use placeholders
User.find_by_sql("SELECT * FROM users WHERE name = '#{name}'")
User.exists?(["name = '#{name}'"])
User.create(params[:user]) / update_attributes(params[:user])  # mass assignment
redirect_to params[:return_to]                 # open redirect — validate
skip_before_action :verify_authenticity_token  # CSRF off
render file: params[:file]; render template: params[:tpl]      # path traversal
raw user_input; <%= user_input.html_safe %>    # XSS
sanitize(user_input)                           # check allow-list
params[:user].permit!                          # accepts everything
```
### 4.6 Go

**Patterns to flag**
```go
exec.Command("sh", "-c", userInput)         // shell injection
fmt.Sprintf("SELECT * FROM x WHERE id=%s", id)  // SQLi — use prepared
template.HTML(userInput); template.URL(userInput)   // bypass auto-escape → XSS
text/template instead of html/template       // no auto-escape
http.Get(userURL)                            // SSRF
http.Redirect(w, r, userInput, 302)          // open redirect
filepath.Join(base, userInput)               // ".." can escape; check filepath.Clean
os.OpenFile(userPath, ...); ioutil.ReadFile(userPath)   // path traversal
gob.Decode; encoding/asn1 custom types; reflect-based deserialization  // type confusion
```
**Safe by default** — `database/sql` with `?` placeholders; `html/template` auto-escapes; use `crypto/rand` (never `math/rand`) for tokens.
**Races** — unsynchronized goroutine writes (`go func(){ sharedMap[k]=v }()`), TOCTOU (`if !exists(file){ create(file) }`).
```bash
gosec ./...; staticcheck ./...; nilaway ./...
semgrep --config=p/go ./...
```
### 4.7 Rust

**Patterns**
- `unsafe { ... }` — review every block; common UB source.
- `std::mem::transmute` — type pun; UB if invalid.
- `Box::from_raw` / `Box::into_raw` — manual lifetime management.
- `unwrap()` / `expect()` in HTTP handlers — DoS via panic.
- Integer overflow in `usize` arithmetic (silent in release) — use `checked_add` / `saturating_add`.
- `format!` / `println!` with a user-controlled *format string* — use `"{}", input`, never `input` as the format.
- `process::Command::new("sh").arg("-c").arg(user)` — shell injection (rare in idiomatic code).
- `serde_json` from untrusted source — type confusion via tagged enums.

**Frameworks** — Actix-Web `web::Json<T>`, Axum, Rocket `Form<T>`: check deserialization size limits. SQLx `query!` is compile-time checked; Diesel `sql_query` raw is dangerous.
```bash
cargo clippy -- -D warnings
cargo audit
cargo geiger    # find unsafe usage
semgrep --config=p/rust src/
```
### 4.8 .NET / C#

**Dangerous APIs**
```csharp
Process.Start(userCmd, userArgs)            // shell
new SqlCommand($"SELECT * FROM x WHERE id={id}", conn)  // SQLi — use parameters
db.Database.ExecuteSqlRaw($"...{user}...")  // EF raw SQL
XmlDocument.LoadXml(userXml)                // XXE if XmlResolver not null
XmlReaderSettings { DtdProcessing = Parse } // XXE
new BinaryFormatter().Deserialize(stream)   // serialization RCE
JsonConvert.DeserializeObject(json, new JsonSerializerSettings { TypeNameHandling = All })  // RCE via $type
DataContractSerializer with user-controlled KnownTypes
SoapFormatter, NetDataContractSerializer, LosFormatter, ObjectStateFormatter   // RCE-prone
Activator.CreateInstance(Type.GetType(userTypeName))   // arbitrary class
Assembly.Load(userBytes)                   // arbitrary code
HttpClient.GetAsync(userUrl)               // SSRF
File.ReadAllText(userPath); File.WriteAllText(userPath, ...)   // LFI
@Html.Raw(userInput)                       // bypass anti-XSS
```
**ASP.NET patterns** — `await TryUpdateModelAsync(user)` without explicit fields = mass assignment; controller without `[Authorize]`; `return Redirect(userInput)` → use `LocalRedirect`.
```bash
dotnet tool install --global security-scan && security-scan src.csproj
semgrep --config=p/csharp src/
```
### 4.9 C / C++

**Memory-corruption sinks**
```c
strcpy, strcat, sprintf, gets             // unbounded
strncpy without explicit null-termination // off-by-one
memcpy with attacker-controlled size      // overflow
malloc(user_size) without check           // 0 or huge
alloca(user_size)                         // stack overflow
realloc — handle NULL return
free(p); ... use(p);                      // UAF; double-free
sscanf without size limit
fopen("rb"); read N bytes into stack buffer
integer overflow in size calc: a*b+c
printf(user)                              // format string: info leak / RCE
TOCTOU on file ops
```
**Tooling** — ASan (`-fsanitize=address`), UBSan (`-fsanitize=undefined`), LeakSanitizer, Valgrind, AFL++/libFuzzer fuzzing, Coverity/SonarQube, `semgrep --config=p/c`.

### 4.10 Solidity / Smart Contracts

**Vulnerability classes**
- **Reentrancy** — external call before state update. Mitigation: checks-effects-interactions, ReentrancyGuard.
- **Integer overflow/underflow** — silent wrap pre-0.8. Use SafeMath or 0.8+.
- **Access control** — missing `onlyOwner` / role check on critical functions.
- **tx.origin auth** — phishing risk; use `msg.sender`.
- **Unchecked low-level calls** — `(bool ok,) = addr.call(...)` without checking `ok`.
- **Front-running / MEV** — order-dependent ops on public mempool.
- **block.timestamp / block.number as randomness** — miner-manipulable.
- **delegatecall to untrusted contract** — full storage takeover.
- **Self-destruct from arbitrary caller** — drain funds.
- **Price oracle manipulation** — flash loan + Uniswap spot price.
- **Signature replay** — missing nonce or chain-id in EIP-712.
- **Initialization** — `initialize()` callable by anyone in proxy patterns.
- **Storage collision** in upgradeable proxies.
- **Phishing approval** — unlimited ERC20 approve to malicious contract.
```bash
slither contracts/
myth analyze contracts/Token.sol
echidna-test contracts/Token.sol --contract Token
forge test --gas-report
```
---
