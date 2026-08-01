# network-monitor — 프로젝트 컨텍스트

> 이 파일은 잘 변하지 않는 항목만 담습니다.
> 기술 결정사항은 decisions.md를 참조하세요.

---

## 개발자 배경

- Java 웹 개발자 5년차 (육아휴직 중, 2026-08-03 복직 예정 → 이후 사이드 프로젝트 시간 제약 커짐)
- 주력 스택: Java, MyBatis, Oracle, mysql, spring, eGovFramework
- eGovFramework: Maven 기반, 구형 Spring 컨벤션 → Spring Boot 3.x+ 패턴과 차이 있음
- 목표: SOLID 원칙 / 디자인 패턴 / 보안 설계 역량 체득 / 네트워크 학습 및 이해 + 풀스택 포트폴리오

---

## 프로젝트 개요

홈 네트워크 기기 탐색 및 시각화 웹 서비스

- MSA 구조: Spring Boot(비즈니스) + Python FastAPI(네트워크 스캔)
- 멀티 런타임, 이벤트 드리븐, 실시간 통신

---

## 확정 기술 스택

| 레이어 | 기술 | 이유 |
|---|---|---|
| Frontend | React + Vis.js | 취업시장 1위, 네트워크 그래프 시각화 |
| Reverse Proxy | Nginx | SSL 종단, CORS 일괄 처리, Rate limiting |
| Backend | Spring Boot 4.1.0 | Java 21, Gradle Kotlin DSL |
| Scanner | Python FastAPI :8001 | nmap 연동 최적, 내부 네트워크만 노출 |
| Message Queue | Redis ↔ RabbitMQ | 전략 패턴으로 yml 한 줄 전환 |
| Database | PostgreSQL | INET, MACADDR, CIDR, JSONB 타입 활용 |
| Infra | Docker Compose | 멀티 컨테이너 통합 관리 |
| 시크릿 관리 | Infisical (셀프호스팅) | `.env` 파일 대체, 원천적 커밋 방지, 인원 제한 없음 (ADR-007) |
| 이슈/작업 관리 | GitHub Issues + Plane (셀프호스팅) | Issues와 양방향 동기화, todolist 겸용 (ADR-009) |
| CI/CD | GitHub Actions | 자동 테스트 + 배포 + 시크릿 로테이션 |

---

## 핵심 아키텍처

### 보안 구조
- Nginx → Spring Boot만 라우팅 (/api, /ws)
- Python :8001은 외부 비노출 (expose만, ports 아님)
- CORS: Nginx에서 일괄 처리
- 시크릿: Infisical(셀프호스팅)로 관리, 로컬에 평문 파일 없음 (ADR-007)

### 인증/인가
- JWT — Access Token 15분 + Refresh Token 7일
- Refresh Token → Redis 저장 (TTL 활용)
- 로그아웃 → Access Token 블랙리스트 → Redis
- 역할: ADMIN (스캔 실행, 설정) / VIEWER (조회만)
- JWT 구현: nimbus-jose-jwt (OAuth2 Resource Server 스타터 내장)
  → JJWT 사용 금지 (decisions.md ADR-002 참조)

### 비동기 처리
- POST /api/scan → 즉시 200 OK + scanId 반환
- @Async + 별도 스레드풀(scanExecutor)로 Python 호출
- 스캔 진행 중 WebSocket(STOMP)으로 진행률 브로드캐스트
- 스캔 중지: REST API /api/scan/{id}/cancel

### WebSocket
- STOMP 프로토콜 (향후 양방향 확장 고려)
- 현재 용도: 서버→클라이언트 단방향
- 구독 토픽: /topic/scan/{scanId}

### 메시지 큐 전략 패턴
```java
public interface MessageQueueService {
    void publish(String topic, Object message);
    void subscribe(String topic, MessageHandler handler);
}
// RedisMessageQueueService implements MessageQueueService
// RabbitMessageQueueService implements MessageQueueService
// application.yml: mq.provider: redis | rabbitmq
```

### 스캔 결과 저장 정책
- 저장 시점: 스캔 완료 후 일괄 저장
- 기기 식별자: MAC 주소 (IP는 DHCP로 변하므로)
- 저장 흐름:
  1. Python 스캔 완료 → MQ 발행
  2. Spring MQ 소비 → @Transactional
  3. MAC으로 device 조회 → upsert
  4. scan_device 연결 기록
  5. 이전 스캔과 diff → 신규/사라진 기기 판별
- MAC OUI 조회: Spring이 WebClient 비동기 호출 (스캔과 분리)

---

## 패키지 구조 (도메인 중심)

```
io.github.canokano.networkmonitor
├── domain
│   ├── device
│   │   ├── controller/    DeviceController
│   │   ├── service/       DeviceService
│   │   ├── repository/    DeviceRepository
│   │   ├── entity/        Device
│   │   └── dto/           DeviceResponse, DeviceUpdateRequest
│   ├── scan
│   │   ├── controller/    ScanController
│   │   ├── service/       ScanService
│   │   ├── repository/    ScanHistoryRepository
│   │   ├── entity/        ScanHistory, ScanDevice
│   │   └── dto/           ScanRequest (Builder 패턴)
│   └── auth
│       ├── controller/    AuthController
│       ├── service/       AuthService
│       ├── entity/        User
│       └── dto/           LoginRequest, TokenResponse
├── infrastructure
│   ├── mq
│   │   ├── MessageQueueService.java   (인터페이스)
│   │   ├── MessageHandler.java        (함수형 인터페이스)
│   │   ├── redis/   RedisMessageQueueService
│   │   └── rabbit/  RabbitMessageQueueService
│   ├── scanner
│   │   └── ScannerClient.java         (WebClient → Python)
│   └── external
│       └── MacOuiClient.java          (OUI API WebClient)
└── common
    ├── config/
    │   ├── AsyncConfig.java
    │   ├── SecurityConfig.java
    │   ├── WebSocketConfig.java
    │   ├── RedisConfig.java
    │   └── MqConfig.java              (전략 패턴 빈 선택)
    ├── exception/
    │   ├── GlobalExceptionHandler.java
    │   └── ErrorCode.java (enum)
    └── security/
        ├── JwtProvider.java
        ├── JwtAuthFilter.java
        ├── JwtSecretResolver.java     (kid → 시크릿 매핑, 전략 패턴)
        └── UserDetailsServiceImpl.java
```

---

## DB 스키마 (핵심 4테이블)

```sql
device      : id, mac_address(MACADDR), ip_address(INET),
              hostname, vendor, os_guess, alias,
              first_seen_at, last_seen_at

scan_history: id, started_at, completed_at,
              status(RUNNING/DONE/FAILED),
              total_found, ip_range

scan_device : scan_id, device_id,
              ip_at_scan(INET), open_ports(JSONB), is_new(boolean)

users       : id, username, password, role(ADMIN/VIEWER)
```

---

## 개발 로드맵

### 1단계: 코어 기능
- [x] Docker Compose dev 환경 (PostgreSQL 18, Redis 8, RabbitMQ 4)
- [ ] Infisical 셀프호스팅 서버 구축 (Docker Compose, 별도 경로) + 프로젝트 연결 (시크릿 관리, ADR-007)
- [ ] Spring Boot 프로젝트 세팅 (도메인 중심 패키지)
- [ ] React 프로젝트 세팅 + 기본 화면
- [ ] PostgreSQL 스키마 + JPA 엔티티
- [ ] Python FastAPI 기본 nmap 스캔
- [ ] Spring @Async Python 호출
- [ ] 기본 REST API (스캔 시작, 기기 목록)

### 2단계: 인증/보안
- [ ] JWT 로그인/갱신/로그아웃 (nimbus)
- [ ] Spring Security 필터 설정
- [ ] Redis 연동 (Refresh Token, 블랙리스트)
- [ ] React 로그인 페이지 + Axios 인터셉터
- [ ] JWT_SECRET을 kid(Key ID) 기반 다중 버전 구조로 설계 (JwtSecretResolver, 전략 패턴)
- [ ] GitHub Actions로 JWT_SECRET 정기 로테이션 자동화 (Infisical API 연동)

### 3단계: 실시간/메시징
- [ ] Redis MQ 연동 (RedisImpl)
- [ ] Spring WebSocket STOMP 설정
- [ ] 스캔 진행률 실시간 브로드캐스트
- [ ] RabbitMQ 전환 실험 (RabbitImpl)

### 4단계: UI 고도화
- [ ] Vis.js 네트워크 그래프
- [ ] MAC OUI 외부 API 연동 (WebClient)
- [ ] 기기 별명 등록, 메모
- [ ] 반응형 모바일 UI

### 5단계: 품질/배포
- [ ] JUnit5 + Mockito 테스트
- [ ] Jest + RTL React 테스트
- [ ] GitHub Actions CI/CD
- [ ] Docker Compose 전체 통합
- [ ] Nginx 설정 + SSL

---

## 설계 역량 강화 목표

### SOLID 원칙
- SRP: ScanService는 스캔만, DeviceService는 기기 관리만
- OCP: MQ 구현체 추가 시 기존 코드 무변경
- DIP: MessageQueueService 인터페이스에 의존
- LSP: 구현체가 인터페이스 계약 완전 준수
- ISP: 필요한 기능만 담은 작은 인터페이스

### 디자인 패턴
- 전략(Strategy): MQ 전환, JWT 시크릿 kid 매핑
- 팩토리(Factory): 스캔 결과 객체 생성
- 옵저버(Observer): WebSocket 이벤트
- 레포지토리(Repository): JPA 추상화
- 빌더(Builder): ScanRequest 등 복잡한 객체

### Claude 활용 가이드
```
"이 코드 SOLID 원칙 위반 있어?"
"더 나은 설계 방법 있으면 알려줘"
"왜 이렇게 설계했는지 설명해줘"
"이 코드에 보안 취약점 있어?"
"공격자 관점에서 이 API 뚫릴 수 있어?"
```
