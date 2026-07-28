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
- **상태 변경**: ADR-007로 대체됨 (Infisical 셀프호스팅 도입)

---

## ADR-007 시크릿 관리 도구

- **날짜**: 2026-07-28
- **상태**: 확정
- **결정**: Infisical Community Edition 셀프호스팅 도입, `.env` 파일 방식 폐기
- **이유**:
  - `.env` 파일은 `.gitignore` 설정을 사람이 놓치면 실수로 커밋될 위험이 있음 (사후 방어)
  - Infisical은 로컬에 시크릿 파일 자체가 존재하지 않아 원천 차단됨
  - Community Edition은 오픈소스(MIT)·셀프호스팅 시 사용자 수 제한이 없어, 향후 팀 규모가 커져도 도구를 바꿀 필요 없음 (SaaS형 도구의 인원 과금 모델과 다름)
  - 맥/윈도우 두 환경을 오가며 개발 중이라도, 자체 서버 하나에 붙는 방식이라 로그인만으로 동기화 가능
  - 데이터가 외부 서드파티 서버가 아닌 본인 인프라 안에만 존재
- **트레이드오프 인지**: 서버 운영(백업, 업데이트, 장애 대응)을 직접 책임져야 함. 관리자 계정 복구용 Emergency Kit 분실 시 복구 불가하므로 별도 안전한 곳에 백업 필수
- **설치 위치**: 프로젝트 저장소와 분리된 별도 경로(`C:\infra\infisical`)에서 셀프호스팅 — 여러 프로젝트가 공용으로 재사용할 인프라이므로 `network-monitor` 저장소에 포함하지 않음
- **금지**: `.env` 파일에 실제 시크릿 값 기록 (개발 초기 임시 사용 후 폐기, `.env.example`은 변수 목록 문서로만 유지). Infisical 자체의 `.env`(ENCRYPTION_KEY, AUTH_SECRET 등)도 동일하게 커밋 금지
- **실행 방식 변경**: `docker compose up -d` → Infisical CLI로 시크릿 주입 후 실행 (예: `infisical run -- docker compose up -d`)
- **참고**: JWT_SECRET 로테이션(kid 헤더 기반 다중 버전)은 2단계(인증/보안)에서 구현, GitHub Actions로 자동화 예정 — 상세는 context.md 로드맵 2단계 참조
