# 푸시 알림 확장

## 개요

기존 기도제목 알림에 더해, 리딩지저스 알림 토글과 어드민 부서 전체 푸시 기능을 추가한다.

## 변경사항

### 1. 리딩지저스 알림 토글 (오전 8시)

**앱 알림 설정 페이지** (`/my/notifications`)에 "리딩지저스 매일 알림" 토글 추가.

- 부서에 활성 리딩플랜이 없으면 토글 회색(비활성, 클릭 불가) + 설명 문구 표시
- 토글 ON → `READING` PushTopic 구독
- 토글 OFF → `READING` PushTopic 해지
- 알림 내용: `{리딩플랜이름}` + `{오늘읽을분량(reading_range)}`
- 발송 시각: 사용자 timezone 기준 오전 8시

**백엔드 변경**:
- `PushTopic` enum에 `READING` 추가
- `DailyReadingPushScheduler`: 기존 부서 ACTIVE 전원 발송 → `READING` 토픽 구독자 중 해당 부서 멤버로 변경, PUSH_HOUR 7→8

### 2. 알림 허용 후 설정 페이지 이동

`NotificationPromptSheet`에서 "알림 허용하기" 성공 시 `/my/notifications`로 자동 이동.
사용자가 기도제목/리딩 알림 각각을 선택할 수 있게 한다.

### 3. 어드민 부서 전체 푸시 알림 탭

공지사항 페이지에 탭 추가: "공지사항" / "전체 푸시 알림".

**전체 푸시 알림 탭**:
- 제목 + 내용 입력
- 발송 예약 시각 (날짜 + 시간)
- 발송 이력 리스트 (제목, 예약/발송 시각, 발송 상태)
- 삭제 가능 (발송 전)

**구현**: 기존 `announcement` 테이블에 `entityType = "PUSH_BROADCAST"`, `entityId = departmentId`로 저장.
앱의 공지사항 목록은 `entityType = "DEPARTMENT"`만 조회하므로 노출 없음.
`AnnouncementPushScheduler`가 그대로 처리 (전체 부서 구독자에게 발송).

어드민 신규 API: `GET/POST /api/push-broadcasts`, `DELETE /api/push-broadcasts/[id]`

## 영향 범위

| 영역 | 파일 |
|------|------|
| Backend | `PushTopic.java`, `DailyReadingPushScheduler.java` |
| Web App | `NotificationSettingsPage.tsx`, `NotificationPromptSheet.tsx`, `AppShell.tsx` |
| Admin | `announcement-manage-client.tsx`, `app/api/push-broadcasts/route.ts`, `app/api/push-broadcasts/[id]/route.ts` |
