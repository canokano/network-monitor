# Claude 세션 시작 가이드

## 매 세션 시작 시 복붙

```
아래 파일 읽고 프로젝트 파악해줘:
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/context.md
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/decisions.md
```

---

## 작업 유형별 추가 파일

특정 작업 시 관련 파일 추가로 붙여넣기:

```
# DB/엔티티 작업 시
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/context.md
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/decisions.md
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/schema.md

# API 작업 시
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/context.md
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/decisions.md
https://raw.githubusercontent.com/canokano/network-monitor/main/docs/api-spec.md
```

---

## 결정사항 발생 시 프로세스

```
1. Claude와 대화 중 기술 결정 확정
        ↓
2. "decisions.md에 ADR 추가해줘" 요청
        ↓
3. Claude가 ADR 내용 작성해줌
        ↓
4. 로컬 decisions.md에 복붙 후 git push
        ↓
5. 다음 세션부터 자동 반영
```

---

## 파일별 역할

| 파일 | 역할 | 수정 빈도 |
|---|---|---|
| context.md | 스택, 아키텍처, 패키지 구조 | 낮음 |
| decisions.md | 기술 결정 로그 (ADR) | 결정 시마다 |
| session-guide.md | Claude 세션 시작 템플릿 | 거의 없음 |
| schema.md | DB 스키마 상세 | DB 작업 시 |
| api-spec.md | API 명세 | API 작업 시 |

---

## Claude에게 자주 쓰는 요청

```
"이 코드 SOLID 원칙 위반 있어?"
"더 나은 설계 방법 있으면 알려줘"
"왜 이렇게 설계했는지 설명해줘"
"decisions.md에 ADR 추가해줘"
"이 코드에 보안 취약점 있어?"
"공격자 관점에서 이 API 뚫릴 수 있어?"
```
