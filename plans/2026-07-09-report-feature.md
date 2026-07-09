# 버그 리포트 / 기능 요청 기능 설계

## 배경

유저가 앱을 쓰다가 버그를 발견하거나 기능을 요청하고 싶을 때 남길 수 있는 창구가 없었다. 웹앱 MY 페이지에서 내용을 적고 스크린샷을 첨부해 제출하면, 시스템 관리자(운영자 본인) 계정에게만 웹 푸시로 알려주고, 관리자가 답을 남기면 유저에게 다시 알려주는 피드백 루프를 만든다.

단순 1회성 답변이 아니라 **댓글 스레드**로 설계했다 — 리포트 내용이 애매하면 관리자가 되물을 수 있고, 유저가 다시 답할 수 있어야 하기 때문이다. 관리 기능(전체 리포트 조회 + 댓글 + 상태 변경)도 admin 대시보드가 아니라 **웹앱 안에서** 처리한다. `SYSTEM_ADMIN_MEMBER_ID` 계정으로 로그인하면 같은 화면이 "전체 리포트" 범위로 보이는 구조라, 일반 유저/관리자가 화면을 공유하고 admin(Next.js/Prisma)↔backend 간 내부 인증 문제도 생기지 않는다.

## 설계 원칙

- 리포트 유형은 enum: `BUG` / `FEATURE_REQUEST` / `QUESTION`
- 상태: `RECEIVED`(접수) / `CONFIRMED`(확인) / `IN_PROGRESS`(작업중) / `RESOLVED`(완료). **상태 변경만으로는 알림/푸시가 가지 않는다** — 관리자가 조용히 상태만 바꿀 수 있어야 한다.
- **댓글이 달릴 때만** 알림+푸시 발생. 방향은 항상 "작성자의 반대편": 관리자가 댓글 → 리포터에게, 리포터가 댓글 → 관리자에게. 알림 본문에 그 시점의 상태를 함께 실어 보낸다.
- 목록/상세 화면은 하나. 일반 유저는 자기 리포트만, `SYSTEM_ADMIN_MEMBER_ID` 계정은 전체 리포트가 보인다 (같은 API, caller에 따라 응답 범위만 다름).
- 스크린샷은 최초 제출에만 첨부 가능 (댓글은 텍스트만, 스코프 최소화).

## DB

### report (신규)

```sql
CREATE TABLE report (
  id          CHAR(36)     NOT NULL COMMENT 'PK',
  reporter_id CHAR(36)     NOT NULL COMMENT '작성자 회원 ID',
  type        VARCHAR(20)  NOT NULL COMMENT '유형 (BUG/FEATURE_REQUEST/QUESTION)',
  content     TEXT         NOT NULL COMMENT '최초 제출 내용',
  status      VARCHAR(20)  NOT NULL COMMENT '처리 상태 (RECEIVED/CONFIRMED/IN_PROGRESS/RESOLVED)',
  created_at  DATETIME(6)  NOT NULL COMMENT '생성일시',
  updated_at  DATETIME(6)  NOT NULL COMMENT '수정일시',
  deleted_at  DATETIME(6)  NULL     COMMENT '삭제일시 (soft delete)',
  PRIMARY KEY (id),
  KEY idx_reporter_created (reporter_id, created_at),
  KEY idx_status_created (status, created_at)
) COMMENT='버그 리포트 / 기능 요청';
```

### report_comment (신규)

```sql
CREATE TABLE report_comment (
  id         CHAR(36)     NOT NULL COMMENT 'PK',
  report_id  CHAR(36)     NOT NULL COMMENT '리포트 ID',
  author_id  CHAR(36)     NOT NULL COMMENT '작성자 회원 ID (리포터 또는 관리자)',
  content    TEXT         NOT NULL COMMENT '댓글 내용',
  created_at DATETIME(6)  NOT NULL COMMENT '생성일시',
  updated_at DATETIME(6)  NOT NULL COMMENT '수정일시',
  deleted_at DATETIME(6)  NULL     COMMENT '삭제일시 (soft delete)',
  PRIMARY KEY (id),
  KEY idx_report_created (report_id, created_at)
) COMMENT='리포트 댓글 스레드';
```

BaseEntity 상속, `@SQLRestriction("deleted_at is null")` 적용 (프로젝트 공통 규약).

### Media 연동

스크린샷은 `EntityType.REPORT` + `entityId = reportId` 로 기존 Media presign/complete 플로우 그대로 사용. 백엔드 Media 코드 변경 없음, enum 상수 추가만.

## Enum

- `ReportType` — `BUG`, `FEATURE_REQUEST`, `QUESTION`
- `ReportStatus` — `RECEIVED`, `CONFIRMED`, `IN_PROGRESS`, `RESOLVED`
- `EntityType` 에 `REPORT` 추가
- `NotificationType` 에 `REPORT_CREATED`(최초 제출 → 관리자), `REPORT_COMMENT_ADDED`(댓글 → 상대방) 추가

## 백엔드 변경

### 신규
- `domain/model/Report.java`, `domain/model/ReportComment.java`
- `ReportJpaEntity`, `ReportCommentJpaEntity`, 각 JpaRepository/PersistenceAdapter/Mapper
- `ReportPort`, `ReportCommentPort` (out-port)
- `ReportCommandUseCase` — `create`, `addComment`, `updateStatus`
- `ReportQueryUseCase` — `getReports(callerId, statusFilter)`, `getReportDetail(callerId, reportId)`
- `ReportCommandService`, `ReportQueryService`
- `ReportController` — `POST /reports`, `GET /reports`, `GET /reports/{id}`, `POST /reports/{id}/comments`, `PATCH /reports/{id}/status`

### 권한 규칙 (서비스 레이어에서 일관 체크)
- `create`, `addComment`(본인 리포트), `getReportDetail`(본인 리포트) — 로그인 사용자 본인
- `getReports`(전체 범위), `updateStatus`, 남의 리포트에 대한 `addComment`/`getReportDetail` — `callerId == report.system-admin-member-id` 만 허용 (아니면 403)

### 알림/푸시
- `create` → `Notification(REPORT_CREATED, receiver=adminMemberId)` 저장 + admin에게 push
- `addComment` → `Notification(REPORT_COMMENT_ADDED, receiver=상대방)` 저장 + push (작성자 본인이 곧 상대방인 셀프케이스는 생략)
- `updateStatus` → 알림/푸시 없음
- 단건 발송은 `PushNotificationSendService` 의 구독 조회+발송 로직을 참고해 memberId 하나로 좁힌 재사용 메서드로 추출

### 설정
- `application.properties`: `report.system-admin-member-id=${SYSTEM_ADMIN_MEMBER_ID:}`
- "내 정보" 응답에 `isSystemAdmin` boolean 추가 (프론트 조건부 렌더링용)

## 프론트엔드 변경 (web-app)

### api.ts
- `reportsApi.create({type, content})`
- `reportsApi.getList(statusFilter?)`
- `reportsApi.getDetail(id)`
- `reportsApi.addComment(id, content)`
- `reportsApi.updateStatus(id, status)` (관리자 전용, UI에서 `isSystemAdmin`일 때만 노출)

### 화면
- `MyPage.tsx` — "알림 설정" 카드 바로 아래 "버그 신고 · 기능 요청" 메뉴 추가
- `ReportListPage.tsx` (`/my/report`) — 일반 유저/관리자 공용. 관리자는 각 행에 작성자 이름 + 상태 필터 노출
- `ReportDetailPage.tsx` (`/my/report/:id`) — 최초 내용 + 스크린샷 + 댓글 스레드(내 글/상대 글 버블 구분) + 입력창. 관리자가 남의 리포트를 열람할 때만 상태 드롭다운 노출 (변경 시 알림 없음, 안내 문구 표시)
- `ReportCreatePage.tsx` (`/my/report/new`) — 유형 선택 + 내용 textarea + 스크린샷 첨부(선택). 제출 후 Media presign/upload/complete는 `GatheringDetailPage.handlePhotoUpload` 패턴 재사용

### 온보딩
- `onboarding-data.ts` 마지막 슬라이드로 "버그·개선사항·궁금한 점은 MY > 버그 신고·기능 요청에서" 안내 추가 (`slides/ReportSlide.tsx`)

## 화면 목업

구현 전 클릭 가능한 정적 목업으로 화면 흐름(MY 진입점 → 목록(유저/관리자) → 상세(유저/관리자) → 작성)을 먼저 확인하고 승인받았다. `docs/plans` 문서에는 별도 첨부하지 않음 — 위 "화면" 절 설명이 최종 반영본.

리포트 유형/상태를 결정하는 과정에서 계획이 두 번 바뀌었다: (1) 관리 기능을 admin 대시보드에 추가하려다 "웹앱 하나로 통일"로 변경 (관리자도 결국 웹앱에서 자기 JWT로 같은 API를 쓰는 게 더 단순함), (2) 리포트당 답변 1개(reply 필드)로 설계했다가 "댓글 스레드"로 변경 (관리자가 애매한 리포트에 되물어볼 수 있어야 함). 최종 반영본은 위 내용.

## 구현 완료 / 남은 작업

코드 변경은 모두 완료 (백엔드 컴파일+테스트 통과, 웹앱 typecheck+lint+build 통과). 아래 두 가지는 코드 변경 범위 밖이라 실행하지 않음:

1. **DB 마이그레이션** — 로컬(`local-mysql` 컨테이너, `into-the-heaven` DB)에는 위 SQL을 실행 완료. **스테이징/프로덕션 DB에는 아직 미실행** — 위 "DB" 절 SQL을 해당 환경에 직접 실행해야 함.
2. **Doppler 환경변수** — 사용자가 Doppler에 `SYSTEM_ADMIN_MEMBER_ID` 생성 완료 (값: 본인 member_id `6f33d359-bc2c-42cd-b875-12937bee989f`). 로컬 실행용으로 `backend/src/main/resources/application-local.properties` 에 `report.system-admin-member-id` 를 동일 값으로 추가함.
