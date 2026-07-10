# 일간/주간 사용 리포트 이메일

## 배경

서비스가 얼마나, 어떻게 활용되고 있는지 매일/매주 자동으로 이메일로 받아보고 싶다는 요청. 이미 admin 대시보드(`/system/monitoring`)에서 Umami 통계를 조회하는 코드가 있고, 장애 알림 메일 인프라(`ops/metrics-collector`가 cron으로 admin API를 호출 → nodemailer로 발송)도 있어서 그 두 가지를 재사용해 구현한다.

## 조사 결과

- Umami `stats` 엔드포인트는 `comparison` 필드로 직전 동일 길이 기간과의 비교를 자동으로 준다 (일간 리포트면 전날 대비, 주간 리포트면 전주 대비를 별도 계산 없이 받을 수 있음).
- 기존 admin 대시보드 코드는 인기 페이지 조회에 `type=url`을 쓰는데, 지금 붙어있는 Umami 버전에서는 `type=path`가 맞는 파라미터였다 (`type=url`은 항상 400). 이번 리포트는 `type=path`로 구현하고, 대시보드 쪽 기존 버그는 범위 밖이라 손대지 않는다.
- 컨테이너(`metrics-collector`) 타임존은 UTC. KST 09:00 = UTC 00:00, KST(월) 12:00 = UTC(월) 03:00 — 날짜가 안 바뀌므로 cron 표현식이 단순하다.

## 범위

- **포함**: webApp/admin 각각 방문자수·페이지뷰·세션수·이탈률·평균 체류시간, 신규가입자수, (webApp만) 인기 페이지 Top5·디바이스 비율
- **제외**: 모임/기도제목 생성 수 (사용자가 명시적으로 불필요하다고 함), 교회/부서별 breakdown

## 설계

### admin: `POST /api/system/monitoring/usage-report`

- 기존 `/api/system/monitoring/alert`와 동일하게 `X-Alert-Secret` 헤더로 인증 (세션 인증 아님 — 서버 간 호출).
- body: `{ "period": "daily" | "weekly" }`
  - `daily`: 어제 00:00~24:00 (KST)
  - `weekly`: 지난 월요일 00:00 ~ 이번 월요일 00:00 (KST), 즉 직전 한 주
- Umami `stats`/`metrics` 호출은 기존 `/api/system/monitoring` route의 패턴 재사용 (webApp/admin 사이트 ID, `UMAMI_API_URL`/`UMAMI_API_TOKEN`)
- 신규가입자수는 `prisma.member.count({ where: { deleted_at: null, created_at: { gte, lt } } })`
- 발송은 기존 alert route와 동일한 nodemailer(Gmail SMTP) transporter, 수신자도 동일 `ALERT_EMAIL_TO` 재사용 (신규 env 불필요)
- 메일 제목: `[IntoTheHeaven] 일간 사용 리포트 (YYYY-MM-DD)` / `[IntoTheHeaven] 주간 사용 리포트 (YYYY-MM-DD ~ YYYY-MM-DD)`

### ops: `metrics-collector/report.sh` (신규)

- 인자로 `daily`/`weekly`를 받아 admin의 usage-report API를 호출하는 얇은 트리거 스크립트 (healthcheck.sh와 동일하게 `host.docker.internal:3000` 사용)
- cron (UTC 기준, docker-compose.yml에 추가):
  - `0 0 * * * report.sh daily` → 매일 KST 09:00
  - `0 3 * * 1 report.sh weekly` → 매주 월 KST 12:00

## 안 한 것

- 교회/부서별 활용도 breakdown (요청 범위 밖, 필요해지면 별도 논의)
- 리포트용 별도 수신 이메일 주소 (기존 alert 수신자와 동일하게 감)
