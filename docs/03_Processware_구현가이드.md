# Processware 무료 템플릿 적용·구현가이드

버전 0.6 · 2026-10-07 · 기획서/요구사항정의서 0.6 대응

## 1. 권장 구현 결정

| 영역 | 제안 | 이유·경계 |
|---|---|---|
| 기반 템플릿 | Full Stack FastAPI Template | 인증·Python API·React UI·PostgreSQL·Compose 배포 골격 재사용 |
| 대화 업무 제어 | Conversation Orchestrator + typed domain tools | 목표·맥락·계획·위임·중간 지시·설명 담당; 실제 권한·승인·발행은 서버 검사 |
| 프런트엔드 | 템플릿 React/TypeScript/Vite/shadcn UI 유지 | 전문 목록·검토·매핑 화면에 적합; 새 Next.js 서비스 추가 불필요 |
| API | FastAPI + 템플릿 ORM/migration 유지 | 기존 Python vault_rules와 쉽게 통합 |
| 업무 DB | PostgreSQL | run·gate·업무 상태·revision 색인; 서버 내 파일에 DB를 중복 배치하지 않음 |
| worker | 별도 Python 서비스 + DB durable queue | 브라우저/API 수명과 분리; 초기 서버당 변경 worker 1개 |
| 파일 | 서버 persistent volume + private Git | 원본/파생물/초안/승인 Vault 경로 분리 |
| 검색 | PostgreSQL 전문/부분 검색부터 | 한글 검색 품질 실측 후 필요하면 별도 검색 도구; vector 필수 아님 |
| 진행 표시 | SSE + 상태 조회 fallback | 이벤트 sequence·cursor로 재접속; WebSocket부터 도입할 필요 없음 |
| 엔진 | HarnessAdapter로 기존 하네스 우선 검증 | SDK·독립 엔진 교체 지점을 고정; 하네스 지침은 실행 코드로 간주하지 않음 |
| 운영 | TLS proxy·API/UI·worker·Postgres, 백업 작업 | 서버 OS·도메인·접속 경로는 D-01 확정 필요 |

무료 템플릿 기능·라이선스 확인 출처: [공식 README](https://github.com/fastapi/full-stack-fastapi-template), [MIT LICENSE](https://github.com/fastapi/full-stack-fastapi-template/blob/master/LICENSE). 기반 commit과 lockfile은 구현 시작 시 고정한다. 현재 template branch를 추적하면서 무조건 최신 버전으로 바꾸지 않는다.

## 2. 템플릿에서 유지·변경·제거할 항목

| 처리 | 항목 |
|---|---|
| 유지 | 로그인 골격, API schema/client 생성, 기본 ORM/migration, TypeScript 라우팅, sidebar·폼·테이블·대화상자, 기본 테스트·Compose |
| 변경 | 단일 관리자 계정, 모든 객체 조직 scope, 인증 파일 다운로드, 서버 키 관리, durable worker와 승인·revision 모델 |
| 제거/비활성화 | 공개 회원가입, 사용자 초대, 샘플 Items CRUD와 샘플 데이터, 필요 없는 메일 복구·데모 도구·외부 배포 서비스 설정 |
| 새로 구현 | 기본 Agent Workspace·대화/Task 기억·계획·CommandProposal/Receipt·대화 승인·source viewer/locator, 원문 대조, 요건 판별·상충, WI/TMP 편집, 추적성 matrix, 백필 매핑, run/gate, publish 복구, 점검·개정 |

메일 복구 기능을 제거하면 서버 운영자의 안전한 계정 재설정 절차를 제공한다. 샘플 JWT 흐름을 유지할 경우 브라우저 token 저장·만료·폐기·CSRF/CORS 정책을 실제로 점검한다. 템플릿에 인증 코드가 있다는 사실만으로 운영 보안을 완료 처리하지 않는다.

Shadcn Admin은 [공식 README](https://github.com/satnaing/shadcn-admin)와 [LICENSE](https://github.com/satnaing/shadcn-admin/blob/main/LICENSE)를 확인한 뒤 필요한 data table·bulk action·filter 패턴만 옮길 수 있다. 두 프로젝트의 전체 라우터·인증 provider·빌드 환경을 합치지 않는다.

## 3. 저장소 배치 제안

현재 repo의 하네스와 Vault를 유지하며 웹 모듈을 추가한다. 실제 경로는 저장소 AGENTS/운영 정책과 검토해 확정한다. 아래는 신규 설계 예시다.

```text
processware/
  .claude/                  # 기존 command/agent 지침 보존
  tools/vault_rules/        # 결정적 규칙 모듈 재사용·강화
  vault/                    # 기존 기준/템플릿; 운영 조직 저장소와 구분
  sources/ inputs/           # 기존 자료; 운영 import 원천으로만 읽기
  publish/                  # 기존 연동; S3
  app/
    backend/
      api/                  # typed endpoints, auth, organization guard
      domain/               # source, requirement, asset, record, assessment
      services/             # review, trace, publish, backup
      adapters/             # harness, parser, storage, git, provider
      workers/              # DB queue, lease, checkpoint, recovery
      migrations/ tests/
    frontend/
      features/             # screens by domain
      components/           # shared viewer, table, diff, gate
      api/                  # generated schema client
    deploy/                 # Compose, volumes, proxy, backup/restore
  docs/webapp/               # 승인한 기획·요구사항·결정 기록
```

운영 데이터는 개발 checkout에 직접 쓰지 않고 별도 서버 volume으로 둔다.

```text
data/
  organizations/{uuid}/
    sources/{source_uuid}/{revision_uuid}/original
    extracted/{revision_uuid}/
    work/runs/{run_uuid}/
    library/                  # 승인 Vault, private Git
    exports/{export_uuid}/
  backups/
```

upload의 client 파일명은 표시용 metadata다. 물리 경로는 서버 UUID로 생성한다. 원본 저장소·다른 조직 폴더·credential은 worker의 일반 쓰기 권한에서 제외한다. 원문 바이너리·키·민감 증적은 공개 GitHub repo로 push하지 않는다.

## 4. HarnessAdapter 계약

어댑터는 다음 기능을 제공한다. 특정 SDK의 현재 메서드명과 동일하다고 가정하지 않는다.

| 함수 개념 | 입력 | 출력·보장 |
|---|---|---|
| validate_capabilities | engine config·업무 kind | 승인 재개·취소·tool 권한 지원 여부 |
| start | RunSpec·input snapshot·workdir·allowlist | engine_session_id·정규화 이벤트 스트림 |
| resume | engine_session_id·checkpoint·gate decision | 승인 대상 hash 검사 후 후속 실행 |
| cancel | engine_session_id·run generation | 취소 요청·늦은 출력 수신 차단 정책 |
| inspect | engine_session_id | 실제 상태·복구 가능 단계 |
| collect_outputs | manifest·workdir | 파일 목록·hash·역할·요건/자산 ID·QA 결과 |

RunSpec은 kind, 조직, baseline, 입력 revision, options, actor, budget, prompt/engine/schema versions를 담는다. 지침은 저장소 판에서 로드하고 파일·명령·network 도구는 server policy로 제한한다. LLM 출력의 tool 명령이나 승인 표시를 서버의 실제 승인 결정으로 해석하지 않는다.

### 기존 지침과 어댑터 매핑

| kind | 기존 정의 | 웹 입력 | 정규화 출력·필수 gate |
|---|---|---|---|
| ingest | `process-ingest.md` | source revision·category·edition·OCR·delta | 구조·requirements·source_map·QA / 원문요건 확정 |
| plan | `process-plan.md` | 승인 적용요건·흐름·As-Is·base baseline | 자산 drafts·steps·MAT·QA / 흐름·발행 승인 |
| do | `process-do.md` | WI/TMP revision·입력 | 단계 trace·REC draft / 수행 검토·승인 |
| backfill | `process-backfill.md` | evidence revisions·scope·OCR | mapping·groups·field payload·REC / 매핑·필드 확정 |
| check | `process-check.md` | 기준선·증적 snapshot·period·self mode | assessment·finding·NCR draft / 결과 확정 |
| audit | `process-audit.md` | 외부 요건 판·자산 기준선·strictness | coverage·GAP·export snapshot / 결과 확정 |
| act | `process-act.md` | 개선 항목·영향·승인 계획 | RCA·revision plan·As-Is·새 plan run / 범위·개정 승인 |

실제 command가 허용하지 않는 새 option을 이름만 만들어 전달하지 않는다. command/agent 지침에 필요한 웹 실행 mode를 추가하거나 어댑터에서 격리·출력 수집을 구현한 뒤 기능 계약을 검증한다.

### S0 통과·실패 분기

통과 조건은 실제 실행에서 구조화 상태/산출물 수집, 비대화형 사용자 승인 정지·재개, 프로세스 재시작 복구, 제한 경로 쓰기, 외부 push 억제, 취소 후 발행 방지다. 7개 명령 전체를 한 번에 포팅하기 전 ingest→plan 대표 흐름으로 검증한다.

command 자체가 사람이 터미널에서 대답해야만 진행되거나 승인 재개를 안전하게 분리할 수 없으면 S0 실패다. 이 경우 지침을 유지하고 SDK 기반 실행 adapter로 control·gate·checkpoint를 구현한다. 같은 초기 화면에 수동 CLI 복사 버튼만 넣은 상태는 사용자가 원하는 웹 업무 구현 완료가 아니다.

## 5. 기존 규칙 코드 통합 시 보강

확인한 `tools/vault_rules/validator.py`는 PyYAML이 없으면 frontmatter 검사를 수행하지 않는다. 웹 앱은 시작 시 필수 의존성과 규칙 스키마를 검증하고 누락 시 작성/발행 기능을 차단한다. 기존 validate success를 완전한 의미 QA success로 표시하지 않는다.

번호는 폴더 스캔만으로 계산하면 여러 요청이 같은 번호를 받을 수 있다. 서버 번호 예약 테이블과 조직 잠금을 추가하고 unique 제약을 둔다. 기존 번호 형식을 임의 재작성하기보다 import 시 format_version과 old/new ID mapping을 먼저 확인한다.

문서 경로는 README의 예시와 실제 tree가 다를 수 있다. 명령 config에서 canonical storage role→실제 path를 매핑한다. `02_표준`/`02_적용요건`, `_state.yaml` 위치, act queue 경로, MAT-010 존재 여부 등은 import report에 포함한다. 코드와 현재 승인 기준이 충돌하면 자동 rename하지 않고 explicit migration으로 처리한다.

## 6. 구현 백로그와 완료 증거

| 작업 | 구현 내용 | FR/NFR 연결 | 완료 증거 |
|---|---|---|---|
| B0-1 저장소 조사 | path registry·branch 목록·규칙 스키마·민감파일 exclusion | OPS-01, AST-03 | import preview, 실제 경로·충돌 보고 |
| B0-2 대화·엔진 spike | 목표→계획→typed command→adapter start/resume/cancel·allowlist·실제 지침 2개 | CON-01~09, RUN-01~06 | 실제 mutation·대화 승인·복구·도구 정책 기록 |
| B0-3 대화 Workspace | context binding·task memory·근거/diff 카드·통합 events·steering | CON-02/05/07~10 | 실제 상태와 메시지 일치·재접속·조직/판 격리 |
| B1 기반 앱 | template commit 고정·단일 로그인·org guard·DB migration·TLS 배포 | ORG, NFR-01/13 | 실제 서버 로그인·미인증 차단 |
| B2 source | upload·rights·revision·parser·locator·품질 report | SRC-01~03 | 권한 차단·포맷별 위치 검증 |
| B3 requirement | 구조·의무 분류·적용/제외·상충·흐름 UI | REQ-01~04 | 대표 자료의 승인 요건 패키지 |
| B3-S 표준 통합·추가 | 적용 구성·관계·의무 배분·증분 계획·표준별 coverage·회귀 QA | STD-01~09 | AT-29~36과 두 모드의 실제 기준선 발행 |
| BH Harness 개선 | 피드백·scope·후보·suite·sandbox·release·activation·관측/복귀 | HAR-01~09, NFR-22~23 | AT-37~45의 실제 신규/기존 run 버전·격리·실패 후보 차단 |
| B4 asset | generator·draft editor·WI steps·rule QA·limited retry | AST-01~06 | 초안과 규칙·의미 QA 결과 분리 |
| B5 publish/trace | snapshot gate·private Git·manifest recovery·matrix | AST-07~08, TRC-01~02 | stale 승인/부분 장애 테스트와 기준선 |
| B6 운영 | backup/restore·audit·external change·export·disk policy | OPS-02~05, EXP-01 | 새 서버 복원 및 제한 원문 제외 |
| B7 OCR/backfill | 파일별 추출·후보·group·field source·legacy REC | SRC-04, BFL-01~05 | 20개 대표 증적 검토·중복 없음 |
| B8 최소 Do | 승인 WI/TMP·단계 입력·반려·REC 승인 | DO-01~02 | 실제 수행 기록과 백필 구분 |
| B9 Check/GAP | 자체점검·4단계 판정·coverage·finding 확정 | CHK, GAP, EXP-02 | 백필/미평가 분리 보고서 |
| B10 Act | 개선 큐·RCA 가설·영향·계획 승인·Plan 재실행·효과 확인 | TRC-03, ACT-01~03 | 폐쇄 루프 1건 전체 링크 |
| BH2 코드 개선 | 규칙/parser/tool/adapter 코드 격리 변경·계약·회귀·호환성·배포 | HAR-10 | S2 AT-46·migration/복귀 증거 |
| BF 프로젝트/현장 | project overlay·DOCX/XLSX 배포·adapter 계약·회수/정비·증적 bundle·지적 자산 공개 | FLD-01~10, NFR-24~25 | AT-47~56의 실제 파일 round-trip·hash/provenance·공개 projection |
| B11 확장 | KPI·HWPX/legacy 변환·Redmine·provider 교체 | KPI/INT/FMT, D-03 | 별도 범위·인수 기록 |

기능마다 domain schema→API→worker/adapter→UI→인수 시나리오 순으로 세로로 구현한다. API·화면부터 모두 mock으로 만든 뒤 엔진을 마지막에 붙이면 가장 큰 불확실성을 뒤로 미루게 된다. 초기 수직 검증은 원천 1개·자산 1개로 시작한다. S1 인수 전에는 복수 표준 통합과 기존 기준선에 추가 적용을 모두 실제 처리해야 한다.

## 7. 테스트 전략

결정적 규칙: 번호 충돌, MIME/경로 검사, schema, scope, 권한, snapshot hash, 날짜/미확인 값, coverage 계산, deduplication을 고정 입력으로 검증한다.

엔진 contract: start·gate·resume·cancel·checkpoint·outputs를 mock adapter로 안정적으로 검증하고 같은 계약을 실제 adapter로 확인한다. mock 통과와 실제 생성 품질을 별도 결과로 남긴다.

업무 품질: 허용된 대표 원문과 증적의 수동 정답을 만든다. 요건 추출 recall/precision, WI 후보 top-3 정확도, 필드 추출 오류·미확인 보존을 실측한다. 유사도 75는 확률 75%가 아니다.

E2E: 요구사항정의서 AT-01~56에서 필수 흐름을 실행한다. 특히 gate 직후 hash 변경, worker 종료, Git commit/DB 사이 장애, 백필 재실행, 취소 후 늦은 결과를 테스트한다. 전체 페이지 단순 렌더 테스트를 많이 만드는 것보다 이 업무 불변 조건을 검증한다.

운영 인수: 깨끗한 서버 복원, 데이터 volume 유지, 이전 앱 버전 rollback, TLS 만료·backup 실패·디스크 부족 표시를 확인한다. 한글 보고서는 긴 표·페이지 나눔·원문 권한 필터를 실제 파일로 확인한다.

## 8. 서버 운영·배포 인수 체크

1. 서버 OS·자원·도메인/VPN·접속 제한·TLS를 확정한다.
2. DB와 내부 worker는 외부 포트를 열지 않고 서비스 계정을 최소 권한으로 구성한다.
3. 업로드·library·work·backup volume을 구분하고 영속성·디스크 한도를 설정한다.
4. 초기 계정과 서버 secrets를 생성하며 샘플 key/password를 제거한다.
5. 내부 Git과 원격 GitHub를 구분하고 push는 기본 비활성화한다.
6. parser/OCR/tool capability 검사를 실행하고 미지원 기능을 UI에 표시한다.
7. 데이터 migration과 백업·복원 훈련을 완료한다.
8. representative S1/S2 AT 결과와 사용자 매뉴얼로 인수한다.

이번 문서 작성은 실제 서버 접근·웹 앱 구현·GitHub 변경·배포를 수행하지 않는다. 구현 단계에서는 이 항목을 기술 설계와 운영 설정으로 구체화한다.

## 9. 개발 지시문 예시

다음 문장은 이 문서들을 저장소에 연결한 뒤 개발 요청에 사용할 수 있다.

> Processware 기획서와 요구사항정의서 0.4을 기준으로 개인 서버용 1인 웹 앱을 구현한다. 기본 진입점은 대화 중심 Agent Workspace다. 사용자가 목표·자료·수정·판단을 전달하면 Agent가 계획·전문 agent 위임·실행을 진행하고 근거·diff·결과를 패널에서 확인한다. 대화 지시는 typed command와 실제 receipt로 연결하고 대화 승인은 동일 gate 정책을 적용한다. 먼저 저장소 지침과 실제 트리·branch·vault_rules를 확인하고, Full Stack FastAPI Template의 특정 commit을 기반으로 인증·조직·원천 자료와 durable Run/HITL 구조를 만든다. S0에서 실제 ingest/plan 실행·승인 재개·취소·파일 격리를 검증한 뒤 S1의 다중 표준 통합·기존 기준선에 추가 적용과 원천→요건→승인 자산을 완성한다. 공통 업무를 재사용하되 고유 의무·원 요건 판을 보존하고 표준별 커버리지·기존 의무 회귀·무변경 판 재사용을 검증한다. 완료한 FR/NFR/AT ID와 증거를 기록하고 미지원·미검증을 구분한다. S1 이후 S2 백필·최소 Do·자체 점검·GAP·개정까지 순서대로 구현한다. 원본 불변, revision별 추적성, 승인 snapshot, legacy_evidence, 자체 점검 구분, 자료 AI 권한을 모든 단계에서 지킨다.

## 10. 최종 결정이 필요한 항목

기획서 D-01~09을 적용한다. 지금 확정할 핵심은 서버 환경·첫 표준 및 대표 자료·하네스 연결 방식·AI 권한과 예산이다. 무료 템플릿은 권장안이며 사용자의 기술 선호나 S0 결과가 바뀌면 같은 요구사항을 유지한 채 기반만 교체한다.

## 11. 대화 중심 구현 계약

메시지 → context resolver → TaskBrief/TaskPlan → CommandProposal → 서버 schema/권한/version 검사 → domain tool/Run → 실제 Receipt/Event → 근거·결과 카드 → TaskMemory 순으로 연결한다. 전문 하네스 실행과 별도로 대화 orchestrator를 두되 승인·발행·실제 상태의 권위를 모델 응답에 맡기지 않는다.

도구 계약에는 읽기 조회, 요건 분할/병합·적용 판단, 흐름 편집, 자산 초안 수정, 백필 그룹/필드 수정, run 시작/중간 지시/취소, gate 결정, 개선 계획·후속 점검이 포함된다. 임의 SQL·셸·자유 경로 쓰기 도구 대신 scope와 revision을 검사하는 업무 명령을 제공한다. 결과는 affected_revision_ids·event/receipt_id·status·error·관련 gate로 반환한다.

Task memory는 대화 로그 복사나 자유 텍스트 요약만으로 구현하지 않는다. 목표·제약·조직·baseline·선택 revision·결정·허용 범위·미해결 질문·plan/run/gate refs를 구조화하고 원 메시지를 참조한다. pending 질문에 응답이 없다는 이유로 승인을 추정하지 않는다.

첫 개발 증거는 자연어 목표 하나가 실제 자료 분석·요건 수정·자산 초안·대화 승인·발행까지 이어지는 수직 흐름이다. 대화 UI를 마지막 부가기능으로 붙이지 않는다. S2는 AT-22에서 백필과 개정의 대화 제어를 검증한다.

## 12. 구체적인 대화 UI 구현

UI·UX 상세명세 0.4과 FR-UX-01~16을 적용한다. 기반 앱의 table/Items 홈을 목표 입력·진행/판단 대기 업무 홈으로 바꾸고, ConversationTimeline·Composer·ContextChip·Plan/Question/Progress/Decision/Outcome card registry·ArtifactCanvas·ReviewInbox·TaskMemoryPanel을 구현한다.

선택 문단은 SelectionRef로 message/context에 연결하고 카드 action은 검증된 domain command만 호출한다. message·card instance·UI state와 actual task/run/gate를 구분한다. 알 수 없는 renderer는 읽기 fallback이며 모델 HTML을 실행하지 않는다.

UI 후보 [AI Elements](https://github.com/vercel/ai-elements)의 [Apache-2.0 LICENSE](https://github.com/vercel/ai-elements/blob/main/LICENSE), dependencies, 실제 import/runtime를 고정한다. 필요한 presentation component만 가져오고 특정 hosting/Node orchestration 의존을 기본으로 추가하지 않는다. AI SDK transport를 도입하면 선택 버전의 event adapter를 검증한다.

메시지 stream과 worker 취소를 분리한다. client disconnect는 run 종료가 아니며 server cancel endpoint와 generation guard가 필요하다. 초안 undo는 domain revision/receipt 기반으로 구현하고 메시지 목록 slice를 승인본 rollback으로 사용하지 않는다. AT-23~28로 IME·context·card·selection·scroll·취소·복구·폭별 동작을 검증한다.


## 13. S1 핵심: 다중 표준 통합·추가 적용 구현

1. **입력·모드:** TaskBrief에 integrate_new/augment_baseline, StandardProfileRevision, 표준 판·범위, base_baseline을 고정한다. 기준선 선택이 없으면 추가 적용을 실행하지 않는다. 기존 ingest/plan run과 전문 Agent를 조합한다.
2. **관계·설계:** CrossStandardRelationRevision과 ObligationAllocation을 원 요건 revision에 연결한다. equivalent/overlap/contains/unique/conflict/unknown의 방향·조건·추가 의무를 보존한다. 관계 후보는 AI가 제안하고 관계 검토·충족 근거는 별도 확정한다.
3. **통합:** 공통 흐름·자산을 설계하고 고유 조건·단계·필드를 함께 배분한다. 원 요건을 통합 텍스트로 덮어쓰거나 source 추적 링크를 삭제하지 않는다.
4. **추가:** ExtensionPlanRevision은 재사용/보완/신규/상충/판단 대기 항목을 가진다. 실제 content 수정과 link-only를 구분하고 무변경 자산은 같은 판을 참조한다. 생성 도구의 write allowlist는 승인된 변경 대상을 기준으로 제한한다.
5. **QA·회귀:** 결정적 검사로 원 의무 누락·판 참조·ID/hash·표준별 분모/분자·대상 목록·멱등성을 검증한다. 의미 검토로 공통 활동이 각 의무를 충족하는지 판단하고 검토 근거를 저장한다. 유사도만으로 자동 합격시키지 않는다.
6. **발행:** profile/relation/allocation/extension plan/link-only를 ChangeSet manifest·hash에 포함한다. gate에서 표준별 검증·기존 의무 회귀·diff를 보여 주고 base와 승인 snapshot을 재검사한다. 자산·구성·링크는 같은 기준선으로 활성화한다.
7. **지속 관리:** 표준 추가·제거·판 교체는 새 구성 판이다. 남는 표준이 참조하는 공통 자산을 자동 삭제하지 않는다. S1의 기본 영향·회귀와 S2 전체 증적·평가 영향 분석을 구분한다.

필수 인수: AT-29~36. 합성 정답 자료로 동등/중첩/포함/고유/상충/불확실과 무변경·link-only·부분 수정·신규 생성·기존 의무 퇴행을 검증하고, 자료 이용이 허용된 실제 복수 기준으로 두 모드의 end-to-end 증거를 남긴다. mock 대화·텍스트 요약만으로 인수하지 않는다.

구현 모듈 제안: domain/standard_profiles, cross_standard_relations, obligation_allocations, extension_plans; services/integration_design, standard_coverage, integration_qa. 이는 논리 기능 분리이며 별도 서버·마이크로서비스를 추가해야 한다는 의미는 아니다.


## 14. Harness 검증·개선 구현 계약

S1에서 FeedbackCase→ScopePolicy→Candidate→EvaluationSuite/Run→ReleaseGate→Release/Activation→Observation/Rollback까지 실제 상태로 연결한다. 지침·템플릿·지원 설정을 먼저 다루고, 새 실행 규칙·tool/adapter 코드는 S2 별도 코드 artifact 배포 경로로 확장한다. 기존 7개 업무 run kind는 유지하고 `task_kind=harness_improvement`와 별도 EvaluationRun으로 개선 업무를 제어한다.

release 파일은 불변 디렉터리/저장소에 두고 run 시작 시 공통 release·overlay·표준 조건을 해결한 snapshot을 격리 작업 공간에 materialize한다. 파일명만 release라고 표시하는 것으로 인수하지 않는다. 실제 사용 지침·template·config·코드 ref/hash가 snapshot과 일치하고 재개 시 다시 활성 버전을 읽지 않아야 한다. 기존 엔진이 이를 지원하는지 S0에서 검증하고 필요하면 adapter를 보완한다.

평가 runner는 별도 쓰기 공간·운영 읽기 정책·외부 전송 차단·예산을 갖는다. 실제 상태 mutation 도구를 mock 출력으로 바꾸거나 평가 전용 저장소에 연결하고 production credential을 주지 않는다. 기준/후보 조건과 suite 판을 고정한다. 결정적 검사와 의미 rubric·검토를 분리하고 필수 회귀 실패를 평균 점수로 상쇄하지 않는다.

후보 수정·scope/suite/rubric/base 변경은 evaluation-validity와 gate를 invalidated로 만든다. 활성 포인터 전환은 트랜잭션/잠금·expected pointer·idempotency key·ActivationEvent로 처리한다. 장애 후 실제 포인터와 event를 재조정하여 완료 여부를 보고한다. 지침 후보가 권한·승인·발행 보호 코드를 수정할 수 없도록 allowlist를 구성한다.

관측은 버전별 실제 실패·편집·QA·비용의 조건과 한계를 기록한다. rollback은 호환 release의 신규 작업용 활성화이며 자산 rollback·DB rewind와 분리한다. S2 코드 변경은 isolated checkout/artifact·unit/contract/integration/security 회귀·dependency/schema/migration·복귀 계획을 검증하고 release manifest에 배포 artifact ref를 연결한다.

인수 대상: HAR-01~10·NFR-22~23·AT-37~46. 기본 사례는 조직별 WI 역할 누락 개선과 기존 표준 통합·추가·백필 unknown·stale 승인·권한 보호 회귀다. 실제 서버에서 개선 전후 비교·활성화·기존 run 재개·신규 run 적용·호환 복귀를 확인한다. 원격 모델 비결정성 때문에 이전 출력의 완전 재생성을 보증하지 않고 입력/구성·실행·판정 근거를 재현 참조로 남긴다.


## 15. 프로젝트 배포·현장 정비·증적/지적 자산 구현

관리 plane은 초기 전문가 계정 1명으로 운용한다. field consumption은 배포 파일/외부 시스템부터 제공하며 모든 구성원의 내부 구축 계정을 선행 조건으로 만들지 않는다. ProjectProfile/AssetSet/Deployment, Submission/Binding/NormalizedEvidence/Clarification, EvidencePackage, Case/KnowledgeAsset을 별도 revision으로 관리한다.

DOCX/XLSX 빈 템플릿 변환은 필드 schema·ID/판·사용 안내·meta표식을 넣고 실제 사용자 편집 round-trip을 시험한다. 스프레드시트 수식 주입·매크로·비정상 구조를 실행하지 않는다. PDF/HTML 절차는 읽기용이며 기재 사실을 만드는 템플릿과 구분한다. 원 binary·추출·정비본·표시 변환·보정은 같은 파일로 덮어쓰지 않는다.

adapter 공통 계약은 capability·recipient scope·remote refs·receipt·idempotency·부분 실패·수집을 정의한다. 개별 실제 adapter가 없으면 파일 배포 경로로 S2 인수하고 연동 미설정으로 표시한다. 기존 Redmine S3 범위와 다중 현장 계정/포털 확장을 구분한다. 현장 관리자 계정 공유를 요구하지 않는다.

증적 exporter는 목적/프로젝트/기간/기준/원 배포 판·원본/정비/provenance·사후 확인/검토·한계·권한 필터의 snapshot을 고정한다. 패키지 생성과 적합 판정을 구분한다. 재사용 exporter/catalog는 별도 권한·정보 제거·범용성 검토를 적용하며 예시와 원 증적을 분리한다. 이 사례의 Harness 개선 연결에는 HAR scope/평가/release 계약을 적용한다.

S2 인수: 실제 절차·빈 DOCX/XLSX 배포→현장 임의 작성→원본 회수→메타 소실/구판/필드 변형 확인→정비·사후 사실 확인→증적 index/PDF/HTML/허용 ZIP→범용 회사 사례→표준/Harness 개선 후보를 이어 검증한다. 자동 서명/승인/날짜 생성·권한 없는 제공·원본 변경·타 프로젝트 공개는 0건이어야 한다.
