---
title: SQL Injection
date: 2026-02-14 00:00:00 +09:00
categories: [Web, SQL Injection]
tag: [SQLi]
---

## SQL Injection
입력값을 통해 SQL 쿼리를 조작하여 비정상적인 DB 동작을 유도하는 공격

### 원인
1. 입력값 검증 부족  
사용자 입력값이 필터링되지 않고 그대로 쿼리에 삽입될 때 발생

2. 문자열 결합 방식으로 SQL 작성  
사용자 입력을 문자열로 직접 결합하는 방식으로 SQL을 만들면, 입력값이 쿼리의 구조로 해석될 수 있음
```
SELECT * FROM user WHERE id = " + Input + ";
```

3. DB 에러 메시지 노출  
에러 내용을 통해 테이블 구조, 컬럼명, 쿼리 문법 등을 유추할 수 있어 SQLi 공격(특히 Error-based)의 성공 가능성을 높임

4. Prepared Statement 미사용  
*Prepared Statement* : SQL 쿼리를 미리 컴파일해 구조를 고정한 뒤, 파라미터 바인딩을 통해 입력값을 데이터로만 전달하는 방식.

5. 최소 권한 미적용  
DB 계정에 과도한 권한이 있을 경우, 공격 성공 시 피해가 기하급수적으로 커짐

### 위험도
1. 데이터 유출  
사용자 정보, 비밀번호 해시, 금융 정보 등

2. 데이터 변조 및 삭제  
UPDATE/DELETE/INSERT 등의 명령을 사용해 데이터 수정/삭제 가능

3. 관리자 권한 탈취  
계정 정보 추출, 권한 상승을 통해 DB 관리자 권한 획득 가능

4. 시스템 장악  
파일 읽기/쓰기, 시스템 명령 실행 기능을 제공하는 일부 DBMS의 경우,  
SQLi를 통해 내부 파일 접근, 서버 설정 노출, 원격 코드 실행(RCE) 가능

### 대응 방안
1. 입력값 검증  
- 화이트리스트 기반 필터링 : 허용된 값 외에는 모두 차단  
- 정규식 기반 필터링 : 사용자 ID, 이메일 등 형식이 정해져 있는 값은 정규식을 이용해 패턴 검증  
- 데이터 타입 검증 : 숫자 필드는 int, 날짜 필드는 datetime 등 타입 변환이 가능한지 체크하여 의도된 입력인지 확인  

2. Prepared Statement 사용  
- 파라미터 바인딩을 통해 SQL 쿼리 구조와 사용자 입력을 분리하고, 입력값이 SQL 구문으로 해석되지 않게 함.
```python
cursor.execute("SELECT * FROM user WHERE name=%s", (name,))   // Prepared Statement 적용
cursor.execute(f"SELECT * FROM user WHERE name='{name}'")     // Prepared Statement 미적용
```

3. 에러 메시지 노출 방지  
- 에러 메시지를 사용자에게 그대로 보여주지 않고 일반화된 메시지를 제공.

4. 최소 권한 원칙 적용
- 읽기 전용 API는 SELECT 권한만 부여
- UPDATE, DELETE, DROP 등의 명령은 필요한 계정에서만 사용

5. WAF(Web Application Firewall) 적용
- 패턴 기반 필터링

### 테스트 방법
1. Error-Based SQLi  
- 특수문자를 삽입하여 에러 발생 여부 확인
```sql
'
''
```

2. Blind SQLi  
    - Boolean-Based Blind SQLi  
    참/거짓 조건에 따른 응답 패턴 비교  
        - 페이지 내용 길이 변화  
        - 특정 요소(리스트, 표 등) 개수 변화  
        - 리다이렉트 발생 여부  
        ```
        ' AND 1=1 --
        ' AND 1=2 --
        ```

    - Time-Based Blind SQLi  
    응답 지연 발생 여부 확인
        ```
        ' AND SLEEP(5) --
        // MySQL
        '; WAITFOR DELAY '0:0:5' --
        // MSSQL
        '; SELECT pg_sleep(5) --
        // PostgreSQL
        ' AND EXISTS(SELECT 1 FROM DUAL WHERE 1=1 AND dbms_lock.sleep(5)=0) --
        // ORACLE
        ```

3. UNION SQLi  
- 컬럼 수 및 DB 구조 노출 여부 확인
    ```
    ' UNION SELECT NULL --
    ' UNION SELECT NULL, NULL --
    ' UNION SELECT 'A', NULL --
    ```

4. 자동화 도구 사용  
- SQLMap, OWASP ZAP 등