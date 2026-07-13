# KST/UTC 날짜·시간 처리 공용화

## 배경

admin 공지사항 발송예약에서 15:00으로 설정한 시각이 다음날 00:00으로 저장되는 버그가 발견됨. 원인은 프론트에서 `YYYY-MM-DDTHH:mm:00` 형태의 타임존 정보 없는 문자열을 만들어 서버로 보내고, 서버(`new Date(sendAt)`)가 이를 서버 프로세스의 기본 타임존(UTC)으로 해석해버린 것. 저장 시 KST→UTC 변환이 누락된 상태에서, 화면 표시 시점의 정상적인 UTC→KST 변환(`getHours()` 등)이 그 오차를 그대로 노출한 것.

이 버그는 공지사항 하나에 국한된 게 아니라 같은 패턴(날짜/시/분 폼 입력 → 타임존 없는 문자열 → 서버에서 `new Date()` 파싱)이 여러 기능에 반복적으로 존재했음. DB/서버를 통째로 KST로 바꾸는 방안도 검토했으나 기각: 문제의 근본 원인은 "UTC를 써서"가 아니라 경계(boundary)에서 명시적 변환이 누락된 것이었고, KST로 전환해도 동일한 클래스의 버그가 반대 방향으로 재발할 위험(서버 인프라 타임존 가정, 로그/모니터링 도구와의 불일치, 향후 해외 유저 대응 시 재마이그레이션 필요)이 더 크다고 판단.

## 결정

UTC 저장은 유지하고, "KST 입력 → UTC 저장" / "UTC 저장값 → KST 표시" 변환을 공용 유틸 함수로 강제. 앞으로 만드는 기능도 이 유틸을 거치도록 함.

## 구현

### admin (`admin/src/lib/datetime.ts`, 신규)
- `buildKstIso(date, hour, minute)` — 폼 입력을 `+09:00` 오프셋이 명시된 ISO 문자열로 변환 (저장용)
- `formatKstDateTime` / `formatKstDate` / `formatKstTime` — 저장된 UTC 인스턴트를 KST 벽시계 시각으로 표시 (`Intl.DateTimeFormat` + `timeZone: "Asia/Seoul"` 사용, 실행 환경 타임존 무관)
- `todayKstDateString()` / `kstDayRangeUtc(dateStr)` — 서버사이드 "오늘" 계산용

### 실제 버그였던 곳 (수정 완료)
- `bulletin-manage-client.tsx` — 주보 발송예약 시각 빌드에 오프셋 누락 (공지사항과 동일한 버그, 별도 발견)
- `visit-manage-client.tsx` — 심방 시작/종료 시각(`toLocalDateTimeString`) 빌드에 오프셋 누락
- `backend/.../AnnouncementPushScheduler.java` — `LocalDateTime.now()`가 명시적 zone 없이 JVM 기본 타임존에 의존. `hibernate.jdbc.time_zone=UTC` 설정으로 `send_at`은 UTC 인스턴트 digit이 그대로 저장되므로, 비교 기준도 `LocalDateTime.now(ZoneOffset.UTC)`로 명시 (KST로 하면 9시간 일찍 발송되는 반대 방향 버그가 생김 — 코드에 주석으로 이유 남김)

### 일관성 정리 (버그는 아니었지만 같은 유틸로 통일)
- `announcement-manage-client.tsx` — 기존 `formatDatetime`(로컬 getter 기반)을 `formatKstDateTime`으로 교체, `sendAt` 빌드도 `buildKstIso`로 통일
- `bulletin-manage-client.tsx`, `visit-manage-client.tsx` — 표시부 포맷 함수들도 공용 유틸로 교체
- `app/api/departments/[id]/reading-plan/progress/route.ts`, `.../weekly-group-progress/route.ts` — "오늘"/"이번 주" 계산이 서버 프로세스 로컬 타임존(`new Date()` + `setHours`)에 의존하던 것을 `todayKstDateString()`/`kstDayRangeUtc()`로 교체. 백엔드의 `ReadingPlanQueryService`가 `LocalDate.now(KST)`를 쓰는 것과 동일한 관례로 정렬. 기존에는 KST 00:00~09:00 사이에 "오늘" 판정이 하루 어긋날 수 있는 좁은 시간대 버그가 있었음

## 손대지 않은 것 (다음에 필요하면)

- `backend AnnouncementController.java`의 `POST /announcements` (`CreateAnnouncementRequest.sendAt`이 naive `LocalDateTime`) — web-app/admin 어디서도 호출하지 않는 것으로 확인된 미사용 엔드포인트. 같은 버그 클래스를 갖고 있지만 죽은 코드라 이번엔 방치. 나중에 실제로 쓰기 시작하면 타입을 `OffsetDateTime`으로 바꿀 것
- admin 대시보드 내비게이션 위젯, 연도 필터 드롭다운 등 클라이언트 전용 `new Date()` 호출들 — 정확성 문제 없음(브라우저 로컬 타임존 = 사실상 항상 KST), 일관성 차원의 리팩터는 보류
- `usage-report/route.ts`의 수동 +9시간 계산 — 현재는 정상 동작하나 취약한 패턴. 다음에 이 파일을 만질 일이 있으면 공용 유틸로 교체
- 백엔드의 zone 없는 `LocalDate.now()` 호출들(`GroupPersistenceAdapter`/`DepartmentPersistenceAdapter`/`ChurchPersistenceAdapter`의 `currentYear`, `EducationCommandService.completedDate`) — 12/31~1/1 KST 자정 경계에서만 영향, 저위험이라 보류
