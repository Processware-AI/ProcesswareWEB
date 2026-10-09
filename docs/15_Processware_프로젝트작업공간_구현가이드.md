# Processware 프로젝트 작업 공간 구현가이드

버전 1.0 · 2026-10-07 · 신규 프로젝트 작업 공간 PW 기준선 · 검토용 설계

## 1. 구현 방향과 재사용 경계

하나의 Processware 배포 안에 신규 workspace 도메인을 추가한다. 표준 authoring와 현장 workspace는 별도 권한·데이터·실행 경계를 갖는다. 현재 기획의 기술 선택은 제안이며 구현 전에 저장소 코드·의존성과 시범 환경을 확인한다.

기존 무료 Full Stack FastAPI Template 기반 제안을 이어받되 새 사용자/권한·Vault·Agent 실행·규칙·이벤트·승인은 직접 구현해야 한다. 무료 템플릿만으로 이 기능이 제공되는 것은 아니다. 적용 commit·라이선스·lockfile을 구현 시작 시 확인·고정한다. [공식 저장소](https://github.com/fastapi/full-stack-fastapi-template)

| 계층 | 구현 제안 | 경계 |
|---|---|---|
| 사용자 인터페이스 | React·TypeScript 기반 대화 + 구조화 Canvas | 기존 UI 골격 재사용, 표/양식은 보조 |
| 콘텐츠 편집 | 구조화 블록·필드, Tiptap OSS 후보 | 업무 관계·승인·상태는 별도 모델. 유료 확장 필수 전제 없음 |
| 도메인 API | Python/FastAPI 서비스, 공통 command 경로 | UI·Agent·연동 모두 권한·판·검증·승인 집행 |
| 기준 데이터 | PostgreSQL의 항목·관계·판·정책·run·event | 검색/임베딩은 재생성 가능한 보조 자료 |
| 원본·첨부 | 인증된 파일/객체 저장소, hash index | DB와 일관 백업·다운로드 권한. Git은 선택 |
| 실행 | 별도 worker·내구성 queue·단계 Receipt | P1 변경 worker 1개부터, API/브라우저 수명과 분리 |
| 진행 전달 | 지속 이벤트 스트림 + 상태 조회·cursor 재접속 | 실제 sequence 표시, 화면 연결이 run 생존을 결정하지 않음 |
| 규칙 | 제한된 선언식 + 등록된 검증기 | 임의 Python/JS 실행 금지; AI 추천과 집행 분리 |
| 외부 ALM | capability manifest + adapter | 대상 판·권한별 검증; 원자적 외부 transaction 가정 금지 |

편집기 및 동시 편집 방식은 P1 시제품으로 검증한다. 초기 충돌 통제는 optimistic revision으로 구현하고 실시간 공동 편집·CRDT 도입은 실제 필요와 평가 후 결정한다. 콘텐츠 편집기를 도입해도 업무 모델 설계를 대신할 수 없다.

## 2. 제안 서비스 구조

```text
Workspace UI / Conversation UI / Approved integration
  → Domain command gateway
      → scope + expected revision + policy + rules + approval
      → DB transaction: asset revision + audit + outbox
  → Durable worker: run / event / export / deployment
      → typed tools → domain service or ALM adapter
      → step receipt + result confirmation
  → Vault DB + authenticated source storage
  → event stream / task state / review queue
```

Agent에는 DB 직접 쓰기·임의 shell·무제한 ALM 관리자 자격을 제공하지 않는다. 원문 안의 명령을 사용자 지시나 실행 권한으로 취급하지 않는다. 승인된 typed domain tools와 제한된 어댑터 작업을 제공한다.

## 3. 단계별 작업 패키지

| 단계 | 구현 순서 | 완료 게이트 |
|---|---|---|
| P1-A | 프로젝트·릴리스·조정판·Vault schema·원본·권한 기반 | PW-01~05 생성·출처·경계 시나리오 |
| P1-B | 판 검사·직접 편집·제안·규칙·승인 command | PW-07/10~12 충돌·차단·자기 검토 |
| P1-C | 대화·Harness 평가·worker·event·영향·재개 | PW-06/08/09/13/14 정책·중복·복구 |
| P1-D | 기준선·기본 export·백업 복원·실제 사이클 | PW-15 전체 시간·오류·복원 측정 |
| P2 | 다중 사용자·경쟁·병렬·예외·갱신·증적·환류 | PW-16~21 현장 개방 게이트 |
| P3 | 대상 1개 구성·배포·왕복·차이·복구 후 확장 | PW-22~24 실제 환경 확인 |

위 순서는 구현 통합 계획이며 요구사항 의존 그래프를 그대로 개발 순서로 해석하지 않는다. 공통 승인·검증·event 기반을 먼저 구축하고 기능을 연결한다.

## 4. 규칙과 상태 집행 예시

```text
rule_id: REQ-HIGH-RISK-01
applies_to: requirement
when: risk_level == high and requested_transition == submit
require: response_owner exists AND response_plan exists
on_violation: block + point_to_fields
on_unknown: block + request_clarification
authority: approved_organization_rule
evidence: template_release / decision_id
exception: not_allowed
```

위 예시는 규칙 의미 설명이며 실행 DSL 최종 문법이 아니다. server command는 현재판을 로드하고 입력 schema·역할·관계·승인을 확인한 뒤 변경과 outbox를 commit한다. Agent가 “확인했다”고 말하는 것은 검사 Receipt를 대신하지 않는다.

## 5. 이벤트·외부 도구 안전한 재처리

내부 자산 변경과 outbox 기록은 동일 DB 거래로 처리한다. worker는 이벤트 key 및 후속 action key로 중복을 제어한다. 외부 호출 전 intent, 호출 뒤 response/readback과 remote version을 기록한다.

타임아웃은 실패를 확정하는 신호가 아니다. 원격 결과가 불명이면 확인 대기 상태로 두고 지원되는 조회·idempotency 기능을 사용한다. 지원되지 않으면 수동 확인 또는 명시적 위험 결정 후 재시도한다. exactly-once 외부 실행을 일반적으로 보장한다고 표현하지 않는다.

## 6. ALM 제품별 확인 지점

2026-10-07 공식 문서 확인 기준이며 지원 선언은 실제 대상 시험 후 한다.

- Redmine은 일부 자원을 REST로 제공하며 자원별 안정성과 기능이 다르다. 프로젝트·이슈·custom field 값의 작업과 전체 workflow 정의 자동화는 별도 확인한다. [REST 자원 목록](https://www.redmine.org/projects/redmine/wiki/rest_api), [이슈 API](https://www.redmine.org/projects/redmine/wiki/rest_issues)
- Jira Cloud의 REST v3와 workflow API는 별도 권한·scope·작업 조건을 갖는다. Cloud 문서를 Data Center 지원 증거로 사용하지 않는다. [REST v3](https://developer.atlassian.com/cloud/jira/platform/rest/v3/), [workflow API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-workflows/)
- Codebeamer의 PTC 문서는 신규 개발에 Swagger API 사용을 권고한다. 설치 제품 판의 공개 API·권한·호환성·라이선스를 확인하며 내부 비공개 API를 전제로 삼지 않는다. [PTC API 사용 정책](https://support.ptc.com/help/codebeamer/r3.1/en/codebeamer/developers_guide/swagger/dg_best_practices_swagger_APIs.html)

## 7. 운영·테스트와 미결

인증·TLS·프로젝트 접근·세션 폐기·원본 다운로드·Agent 도구 권한을 실제 경로별 검사한다. 동시 수정·중복 승인·재시작·중복 이벤트·외부 타임아웃·자료 누락·해시 불일치를 평가한다. 정상 데모만으로 운영 개방하지 않는다.

시범 프로젝트의 작업 시간을 입력·생성·검토·재작업으로 나누고 비용·실패율도 기록한다. 응답 시간·동시 사용자·RPO/RTO·보존 숫자는 서버와 자료를 정한 뒤 승인한다. P1 단일 worker에서 통과한 부하 결과를 P2 다중 사용자 목표로 대체하지 않는다.

최종 미결은 모델 공급자와 처리 위치, 서버 환경, 시범 파일 형식, first ALM 제품/판, 규칙 schema, 출력 서식, 운영 목표다. 이 문서는 설치·코드 변경이나 배포 완료 보고가 아니다.
