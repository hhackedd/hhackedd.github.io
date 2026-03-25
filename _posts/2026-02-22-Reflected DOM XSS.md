---
title: "[Write-up] PortSwigger Academy - Reflected DOM XSS Lab"
date: 2026-02-22 00:00:00 +09:00
categories: [Web, XSS]
tag: [XSS, Write-up, PortSwigger Academy]
---

### 문제 설명
![1](../assets/img/Reflected%20DOM%20XSS/1.png)
- Reflected DOM 취약점 : 서버가 요청 데이터를 응답에 그대로 반영하고, 페이지의 JavaScript가 이를 안전하지 않게 처리하여 DOM에 삽입될 때 발생하는 취약점
- 목표 : alert() 함수가 실행되도록 payload 생성

### 취약점 분석
Lab에 접속하면 다음과 같은 초기 페이지를 확인할 수 있다. 해당 페이지에는 블로그 포스팅을 검색할 수 있는 검색창이 존재하며, 검색 결과를 통해 게시글 목록을 조회할 수 있다. 게시글을 클릭하면 댓글 기능이 포함된 포스팅 상세 페이지로 이동한다.
![2](../assets/img/Reflected%20DOM%20XSS/2.png)

먼저 입력값이 응답에 어떻게 반영되는지 확인하기 위해 검색창에 임의의 문자열을 입력하고 응답을 확인했다.  
입력값은 search 파라미터를 통해 전달되었지만 응답 페이지에는 해당 문자열이 포함되지 않았다.  
트래픽 확인 결과, /search-results 경로로 추가 요청이 발생했으며, 해당 요청의 응답 데이터에 입력값이 JSON 형태로 포함되는 것을 확인할 수 있었다.
![3](../assets/img/Reflected%20DOM%20XSS/3.png)
![4](../assets/img/Reflected%20DOM%20XSS/4.png)

응답 페이지 소스코드 확인 결과, `search('search-results')` 함수가 호출되는 것을 확인할 수 있었다.  해당 함수의 동작을 확인하기 위해 /resources/js/searchResults.js 파일을 확인했다.  
![5](../assets/img/Reflected%20DOM%20XSS/5.png)

`search()` 함수는 다음과 같이 작성되어 있었다. URL의 query string을 path에 추가해서 요청을 생성하고, 정상적으로 응답이 반환될 경우 응답 데이터(this.responseText)를 객체로 변환하고 `searchResultsObj` 변수에 저장한 뒤 `displaySearchResults()` 함수를 통해 검색 결과를 화면에 출력한다.  

```javascript
function search(path) {
    var xhr = new XMLHttpRequest();
    xhr.onreadystatechange = function() {
        if (this.readyState == 4 && this.status == 200) {
            eval('var searchResultsObj = ' + this.responseText);
            displaySearchResults(searchResultsObj);
        }
    };
    xhr.open("GET", path + window.location.search); 
    xhr.send();

    function displaySearchResults(searchResultsObj) {...}
```

하지만 응답 데이터를 객체로 변환하는 과정에서 eval() 함수를 사용하고 있었다. eval() 함수는 전달된 문자열을 그대로 JavaScript 코드로 실행하기 때문에, 응답 데이터에 임의의 코드가 포함되는 경우 악성 스크립트가 실행될 수 있는 취약점이 존재한다.    

이를 이용하여 응답 데이터에 악성 스크립트가 포함되도록 payload를 설계했다.

### 공격 과정
응답 데이터(this.responseText)에 입력값이 포함되어 JSON 객체 형태로 반환되기 때문에,  
JSON 문자열을 종료한 뒤 객체 구조를 탈출하여 alert(1) 함수를 호출하는 payload를 작성했다.  
해당 payload는 eval() 실행 과정에서 JavaScript 코드로 해석되어 실행된다.

```javascript
\"};alert(1);//
```

### 공격 결과
작성한 payload를 검색창에 입력하면 브라우저에서 alert 창이 실행되어, payload가 정상적으로 동작함을 확인할 수 있다.  
![6](../assets/img/Reflected%20DOM%20XSS/6.png)