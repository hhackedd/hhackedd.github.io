---
title: File Upload
date: 2026-03-07 01:00:00 +09:00
categories: [Web, File Upload]
tag: [File Upload]
---

## File Upload Vulnerability
파일 업로드 기능의 검증 부족으로 인해 악성 파일 업로드 및 실행이 가능한 취약점

### 원인
1. 업로드 파일 확장자 검증 부족

    - 실행 가능한 스크립트 파일 업로드 가능

    ```
    .php
    .jsp
    .asp
    .exe
    .sh
    ```

    - 확장자 우회 검증 부재

    ```
    shell.php.jpg
    shell.jpg.php
    shell.phP
    ```

2. MIME 타입 / Content-Type 신뢰
    - 실제 파일 내용을 서버 측에서 검사하지 않음
    - 웹 서버가 브라우저에서 전송하는 Content-Type 헤더만 신뢰하는 경우 발생  

    ```
    MIME 타입 : 데이터 형식 표준
    Content-Type : HTTP 프로토콜에서 전송되는 MIME 타입 헤더
    ```

3. 업로드 위치 제한 부재
    - 업로드 파일을 웹 루트 내부에 저장하는 경우, 업로드된 파일이 웹을 통해 직접 접근 및 실행될 수 있음

    ```
    /var/www/html/uploads/shell.php
    > http://example.com/uploads/shell.php
    ```

4. 파일 이름 검증 부족
- 경로 조작 가능
- 기존 파일 덮어쓰기 가능

5. 파일 내용 검증 부족
- 파일 확장자가 정상적이어도 내부에 악성 코드가 포함될 수 있음  
- 예 : 이미지 파일 내부에 스크립트를 삽입하는 Polyglot 파일

6. 파일 크기 제한 부재
- 업로드 파일 크기 제한이 없는 경우, 대용량 파일을 업로드하여 서버 자원을 고갈시킬 수 있음

7. 웹 서버 실행 설정 미흡
- 업로드 디렉터리에서 스크립트 실행 허용

### 위험도
1. 원격 코드 실행 (RCE)

2. 서버 장악 및 내부망 침투
- 서버 계정 탈취
- 시스템 명령 실행
- 백도어 설치
- 내부망 이동 (Lateral movement)

3. 데이터 유출 및 계정 탈취
- DB 접근
- 계정 정보·세션 쿠키 탈취
- 민감 정보 유출

4. 서버 자원 고갈 (DoS)

5. 악성 코드 배포
- 업로드 서버가 악성 파일 호스팅 서버로 악용될 수 있음

### 대응 방안
1. 확장자 화이트리스트 적용
    - 허용된 확장자만 업로드 가능하도록 제한

    ```
    jpg
    jpeg
    png
    gif
    pdf
    ...
    ```

2. MIME 타입 및 Magic Number 검사
    - 서버에서 직접 파일의 Magic Number(파일 시그니처)를 검사하여 실제 파일 형식 검증

    ```
    PNG : 89 50 4E 47
    JPEG : FF D8 FF
    ...
    ```

3. 웹 루트 내부 저장 금지

    ```
    /var/www/html/uploads/  (X)
    /var/data/uploads/      (O)
    ```

4. 업로드 디렉터리 실행 권한 제한

    ```
    # httpd.conf
    <Directory /var/www/html/uploads>
        php_admin_flag engine off
    </Directory>

    # .htaccess
    php_flag engine off
    ```

5. 파일 이름 랜덤화
- 서버에서 임의 UUID로 파일 이름 변경

6. 파일 크기 제한

7. 이미지 재인코딩
- 이미지/문서 파일의 숨겨진 악성 코드 제거 가능

8. 업로드 파일 다운로드 시 실행 방지
- 파일이 브라우저에서 실행되지 않고 다운로드 되도록 처리

    ```
    Content-Disposition: attachment
    ```

9. 악성 파일 검사
- Anti-virus
- Malware scanner
- WAF 정책 적용

### 테스트 방법
1. 확장자 우회 테스트

    ```
    shell.php
    shell.php.jpg
    shell.jpg.php
    shell.phP
    shell.php%00.jpg (Null 문자 우회)
    ```

2. MIME / Content-Type 검증 테스트
- 업로드 요청에서 Content-Type 헤더를 변조하여 파일 업로드 시도

    ```
    Content-Type: image/jpeg

    실제 파일 내용
    <?php system($_GET['cmd']); ?>
    ```

3. 업로드 경로 / URL 접근 테스트
- 업로드 디렉터리 직접 접근 가능 여부
- 파일 실행 여부

    ```
    /uploads/image.png
    /uploads/shell.php
    ```

4. 파일 내용(Content) 검사 테스트
- 이미지 헤더를 일부 포함한 악성 파일 업로드 시도
- 서버가 정상 이미지로 인식하는지 테스트

5. 파일 크기 제한 테스트
- 대용량 파일 업로드 시도
- 업로드 제한 존재 여부
- 서버 응답 지연 / 서버 오류 발생 여부