# Architecture Decision Records (ADR)

> 기술 결정사항 로그입니다.
> 새로운 결정이 생길 때마다 아래에 append합니다.
> Claude는 이 파일의 결정을 존중하고 반하는 제안을 하지 않습니다.

---

## ADR-001 빌드 도구 및 기본 스택

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**:
  - Spring Boot 4.1.0 (2026.06.10 GA)
  - Java 21 (LTS, Virtual Thread 지원)
  - Gradle Kotlin DSL
  - 설정 파일: application.yml
- **이유**:
  - Java 21은 현재 LTS, Spring Boot 4.x 공식 권장
  - Gradle Kotlin DSL: 타입 안전성 + IDE 자동완성
  - Maven은 eGovFramework 경험 있으나 신규 프로젝트 트렌드는 Gradle

---

## ADR-002 JWT 라이브러리

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**: `spring-security-oauth2-jose` (nimbus-jose-jwt 내장) 사용
- **금지**: `io.jsonwebtoken:jjwt-*` 사용 금지
- **이유**:
  - Spring Boot 4.x는 Jackson 3 (패키지: tools.jackson.*) 기본 사용
  - jjwt-jackson이 Jackson 3와 호환되지 않아 런타임 오류 발생
  - nimbus-jose-jwt는 Spring Security 7.x 공식 권장, Jackson 3 충돌 없음
  - start.spring.io에서 `OAuth2 Resource Server` 선택 시 자동 포함
- **참고**: Jackson 2 기반 코드 예시는 참고하지 말 것

---

## ADR-003 테스트 DB 전략

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**: Testcontainers 사용 (H2 사용 금지)
- **이유**:
  - PostgreSQL 전용 타입 (INET, MACADDR, JSONB)은 H2에서 지원 안 됨
  - Testcontainers로 실제 PostgreSQL 컨테이너 띄워서 테스트
- **의존성**:
  ```kotlin
  testImplementation("org.testcontainers:junit-jupiter")
  testImplementation("org.testcontainers:postgresql")
  testImplementation("org.testcontainers:rabbitmq")
  ```

---

## ADR-004 WebClient 사용 방식

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**: MVC 기반 서버 유지 + WebClient만 사용
- **이유**:
  - WebClient는 spring-boot-starter-webflux 없이 사용 불가
  - WebFlux 전체 전환 없이 HTTP 클라이언트만 비동기로 사용
  - Python FastAPI 호출, MAC OUI API 호출에 사용
  - MVC + WebClient 혼용은 Spring 공식 권장 패턴
- **의존성**: `spring-boot-starter-webflux` 추가 필요

---

## ADR-005 JPA 스키마 관리 전략

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**: `ddl-auto: validate`, 스키마는 SQL로 직접 관리
- **이유**:
  - INET, MACADDR, JSONB 등 PostgreSQL 전용 타입은 Hibernate 자동 생성 불가
  - 스키마는 SQL 파일로 직접 관리, JPA는 검증만 담당
- **개발 초기**: create-drop으로 시작 → 엔티티 안정화 후 validate로 전환

---

## ADR-006 환경변수 및 시크릿 관리

- **날짜**: 2026-07-26
- **상태**: 확정
- **결정**:
  - `application.yml`: placeholder만 기록 (`${변수명}` 형식), 커밋 O
  - `.env`: 실제 값 기록, 커밋 X (.gitignore 등록 필수)
  - `.env.example`: 필요한 변수 목록만 기록, 커밋 O
  - `application-local.yml`: 로컬 전용 설정, 커밋 X
- **금지사항**:
  - 시크릿 값 하드코딩 절대 금지
  - JWT_SECRET 최소 32자 이상
  - git 히스토리에 한 번이라도 올라간 시크릿은 즉시 값 변경
- **gitignore 추가 항목**:
  ```
  .env
  .env.local
  .env.*.local
  **/application-local.yml
  **/application-local.yaml
  ```
