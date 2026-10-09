# ProcesswareWEB

Processware 웹 플랫폼의 기획·요구사항·설계 및 제품 기획 대화 기록.

표준 프로세스 자산을 전문가가 구축·검증하고, 승인된 템플릿을 프로젝트 작업 공간에 적용하여 사람과 AI Agent가 산출물·실행 규칙·증적을 함께 관리하는 플랫폼을 기획한다.

## 문서 보기

- [전체 문서 목록](docs/INDEX.md)
- [기존 표준 자산 구축 기획서](docs/01_Processware_기획서.md) · [요구사항정의서](docs/02_Processware_요구사항정의서.md)
- [신규 프로젝트 작업 공간 기획서](docs/12_Processware_프로젝트작업공간_신규기획서.md) · [상세 요구사항](docs/14_Processware_프로젝트작업공간_요구사항정의서.md)
- [제품 기획 대화 전체](conversation/transcript.md) · [결정사항·미결 과제](conversation/decisions-and-next-steps.md)
- [문서 전체 ZIP](downloads/Processware_전체문서_2026-10-09.zip)

HTML 통합본은 `docs/00_Processware_기획_요구사항_통합.html`과 `docs/23_Processware_프로젝트작업공간_기획_요구사항_통합.html`이다. GitHub에서는 HTML 소스가 표시될 수 있으므로 ZIP을 풀거나 저장소를 내려받아 브라우저에서 연다. 자동 사이트 배포는 구성하지 않았다.

## 구성

```text
docs/             작성한 문서 29개 + 문서 색인
conversation/     사용자·AI 대화(Markdown/JSON), 결정사항 및 미결 과제
downloads/        작성 문서 전체 ZIP
upload-manifest.json  파일 크기·SHA-256·대화 내보내기 범위
```

기존 00~11 문서군과 신규 12~23 문서군은 별도 기준선을 유지한다. 현재는 기획·설계 단계이며 실제 서비스 구현·배포·ALM 검증 완료를 뜻하지 않는다.

대화 기록은 최초 요청부터 2026-10-09 업로드 요청까지의 사용자 메시지·AI 답변·진행 메시지다. 자동 환경·UI 데이터와 내부 지시·추론·도구 로그는 제외하고 로컬 문서 경로를 상대 링크로 정리했다. 문서는 원본 파일과 동일하게 보존했다. 과거 대화의 제안과 최종 기획을 혼동하지 않도록 결정사항 요약을 함께 제공한다.

원천 프로젝트: [Processware-AI/processware](https://github.com/Processware-AI/processware)
