---
title: Access Control
date: 2026-03-15 01:00:00 +09:00
categories: [Web, Access Control]
tag: [Access Control]
---

## Access Control
인증된 사용자가 어떤 리소스에 접근할 수 있는지 제어하는 것

### 취약점 발생 지점
- 서버 권한 체크 누락
- 클라이언트 값 신뢰
- ID 기반 접근
- 서버와 클라이언트 간 권한 검증 불일치
- 역할/권한 설계 미흡

### 공격 유형
##### IDOR (Insecure Direct Object Reference)  
- 사용자 입력값(ID 등)으로 다른 사용자의 리소스에 직접 접근 가능

##### Horizontal Privilege Escalation  
- 같은 권한 사용자 간 다른 데이터 접근

##### Vertical Privilege Escalation  
- 일반 사용자가 관리자 권한 기능 수행

##### Access Control Bypass  
- 권한 검증 로직을 우회하여 접근 제한 무력화

### 대응 방안
- 서버에서 모든 요청 권한 검증
- 최소 권한 원칙 적용
- 클라이언트 값 신뢰 금지
- ID 기반 접근 시 서버 검증 강화
- 역할/권한 명확히 설계 (RBAC/ABAC)
- 정기적 권한 검증 및 보안 테스트 수행

### 접근 제어 모델
##### RBAC (Role-Based Access Control)  
역할(Role)에 권한을 부여하고 사용자에게 역할을 할당하는 방식
- 관리가 용이하고 실무에서 가장 널리 사용됨
- 역할이 많아질 경우 복잡해질 수 있음
- 예 : 일반 사용자, 관리자 등

##### ABAC (Attribute-Based Access Control)
사용자, 리소스, 환경 등의 속성(Attribute)을 기반으로 접근 제어 수행
- 유연하고 세밀한 제어 가능
- 구현 및 관리가 복잡함
- 예 :  
본인이 작성한 글만 수정 가능  
특정 시간/위치에서만 접근 허용

##### DAC (Discretionary Access Control)
리소스 소유자가 접근 권한을 직접 설정하는 방식
- 유연하지만 사용자 실수로 인한 보안 취약 가능
- 예 : 파일 소유자가 읽기/쓰기 권한 부여

##### MAC (Mandatory Access Control)
시스템 정책에 따라 접근을 강제적으로 제어하는 방식
- 보안성이 매우 높지만 유연성이 낮음
- 예 : 군사 등급 기반 접근 제어