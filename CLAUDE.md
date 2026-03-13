# Docs (프로젝트 문서)

프로젝트 전반의 기획, 설계, 구현 계획 문서를 모아놓는 디렉토리.

## 디렉토리 구조

```
docs/
└── plans/          # 구현 계획서
```

## plans/

대규모 기능 변경이나 신규 도메인 도입 시 구현 계획서를 보관.

| 파일 | 설명 |
|------|------|
| `department-implementation-plan.md` | Department(부서) 도입 구현 계획 — DB 스키마, Backend/Admin/Web-app 변경 사항 5단계 Phase |
| `2026-03-06-platform-monitoring-design.md` | 플랫폼 모니터링 설계 — Umami 방문 추적, 앱 활동 지표, 서버/DB 리소스, API 상태 모니터링 |
| `2026-03-06-platform-monitoring-plan.md` | 플랫폼 모니터링 구현 계획 — Umami(Docker), metrics 수집 컨테이너, Admin 대시보드 UI 구현 단계별 계획 |
| `2026-03-10-home-weekly-summary-design.md` | 홈 화면 "이번 주" 요약 카드 — 최근 7일 모임의 한주 목표/기도제목을 홈 상단에 표시하는 기능 설계 |
