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
  - 개발이 맥 중심으로 이뤄지고 윈도우는 보조적으로만 사용될 예정이라, 단일 서버(맥)에 붙는 방식으로 충분
  - 데이터가 외부 서드파티 서버가 아닌 본인 인프라 안에만 존재
- **트레이드오프 인지**: 서버 운영(백업, 업데이트, 장애 대응)을 직접 책임져야 함. 관리자 계정 복구용 Emergency Kit 분실 시 복구 불가하므로 별도 안전한 곳에 백업 필수
- **설치 위치**: 프로젝트 저장소와 분리된 맥북 내 별도 경로(`~/infra/infisical`)에서 셀프호스팅 — 여러 프로젝트가 공용으로 재사용할 인프라이므로 `network-monitor` 저장소에 포함하지 않음
- **윈도우 접속 방식**: 별도 VPN/터널링 도구 없이, 같은 LAN 내에서 맥의 로컬 IP(예: `http://192.168.x.x:8080`)로 직접 접속. 맥/윈도우가 서로 다른 네트워크에 있는 원격 접속 상황이 생기면 그때 Tailscale 등 재검토
- **금지**: `.env` 파일에 실제 시크릿 값 기록 (개발 초기 임시 사용 후 폐기, `.env.example`은 변수 목록 문서로만 유지). Infisical 자체의 `.env`(ENCRYPTION_KEY, AUTH_SECRET 등)도 동일하게 커밋 금지
- **실행 방식 변경**: `docker compose up -d` → Infisical CLI로 시크릿 주입 후 실행 (예: `infisical run -- docker compose up -d`)
- **참고**: JWT_SECRET 로테이션(kid 헤더 기반 다중 버전)은 2단계(인증/보안)에서 구현, GitHub Actions로 자동화 예정 — 상세는 context.md 로드맵 2단계 참조


## ADR-008 Infisical CLI-서버 버전 고정 정책

- **날짜**: 2026-08-01
- **상태**: 확정
- **배경**: self-hosted Infisical 서버(`infisical/infisical:latest-postgres`)와
  Homebrew로 설치한 최신 CLI 간 API 버전 불일치로 `infisical run` 실행 시
  `404 Route not found` 에러 발생. CLI가 `/api/v4/secrets` 경로로 요청했으나,
  self-hosted 서버 이미지에는 해당 API 버전이 아직 반영되지 않은 상태였음
  (Infisical은 Cloud에 신규 API를 먼저 배포하고 self-hosted 이미지에는
  추후 반영하는 것으로 추정).
- **진단 과정**:
  - 서버(`docker compose pull` + `docker rmi` 후 재다운로드)를 최신으로
    맞춰도 동일 에러 재현 → 서버가 오래된 게 원인이 아님을 확인
  - CLI를 `npm install -g @infisical/cli@0.31.0`으로 구버전 고정 →
    구버전 API 경로(`v3`)로 요청하면서 정상 연결 확인
- **결정**: CLI 버전을 `0.31.0`으로 고정. `brew upgrade infisical`로
  임의 업그레이드하지 않는다.
- **추가로 발견된 부수 원인**: Infisical UI에 표시되는 환경 이름("Development")과
  실제 내부 슬러그(`dev`)가 다름. CLI 명령에는 반드시 슬러그(`--env=dev`)를
  사용해야 하며, `--env=development`처럼 표시 이름을 쓰면 연결은 성공해도
  시크릿을 0개 반환한다.
- **향후 규칙**: self-hosted Infisical 서버 이미지를 업그레이드할 경우,
  CLI 버전도 반드시 같은 시점에 함께 맞춰서 업그레이드한다
  (서버만 올리거나 CLI만 올리는 것 금지 — 오늘과 동일한 문제 재발 위험).
- **참고**: 문제 진단 시 `docker logs infisical-backend`로 서버가 실제로
  받은 요청과 응답 상태 코드를 직접 확인하는 것이, CLI가 출력하는
  뭉뚱그려진 에러 메시지보다 훨씬 정확한 원인 파악 방법이었음.