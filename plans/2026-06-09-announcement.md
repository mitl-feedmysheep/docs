# 공지사항 (Announcement) 기능

작성일: 2026-06-09

## 배경 및 목적

부서 관리자(admin)가 교인들에게 공지사항을 전달할 수단이 없었다. 기존 Web Push 인프라(push_subscription, WebPushAdapter)가 완성되어 있으므로, 이를 활용해 예약 발송 기반의 공지사항 기능을 구축한다.

## 결정 사항

- Event 테이블과 결합하지 않는다. Announcement는 독립 도메인.
- 즉시 발송 없음. 발송 시각(`send_at`)을 반드시 지정해야 한다.
- `is_sent = true` 상태가 된 공지사항은 수정할 수 없다.
- 공지사항 삭제 시 연결된 캘린더 이벤트는 삭제되지 않으며, 캘린더 이벤트는 별도로 삭제해야 한다.
- 주기적 발송(예배 X분 전 등)은 다음 단계로 미룬다.
- 공지사항 목록/상세는 알림 페이지와 통합하지 않고 별도 화면으로 분리한다.

## 구현 범위

### 1. Backend

#### DB 스키마

```sql
CREATE TABLE announcement (
  id              CHAR(36)       NOT NULL PRIMARY KEY,
  entity_type     VARCHAR(20)    NOT NULL,   -- 현재는 DEPARTMENT 고정, 향후 CHURCH 등 확장 가능
  entity_id       CHAR(36)       NOT NULL,
  title           VARCHAR(100)   NOT NULL,
  body            TEXT           NOT NULL,
  send_at         DATETIME       NOT NULL,
  is_sent         TINYINT(1)     NOT NULL DEFAULT 0,
  created_at      DATETIME       NOT NULL,
  updated_at      DATETIME       NOT NULL,
  deleted_at      DATETIME,
  INDEX idx_announcement_entity (entity_type, entity_id),
  INDEX idx_announcement_send_at_is_sent (send_at, is_sent)
);
```

#### 아키텍처

```
domain/model/
  Announcement.java
  AnnouncementId.java

adapter/out/persistence/
  AnnouncementJpaEntity.java
  AnnouncementJpaRepository.java
    # findTop2ByEntityTypeAndEntityIdOrderByCreatedAtDesc
    # findByEntityTypeAndEntityIdOrderByCreatedAtDesc
    # findBySendAtBeforeAndIsSentFalse(LocalDateTime now)
  AnnouncementPersistenceMapper.java
  AnnouncementPersistenceAdapter.java

application/port/in/
  query/AnnouncementQueryUseCase.java   # getRecent2, getList, getById
  command/AnnouncementCommandUseCase.java  # create, delete

application/port/out/
  AnnouncementPort.java
  PushSubscriptionPort.java             # findByMemberIds(List<MemberId>) 신규 추가 필요

application/service/
  query/AnnouncementQueryService.java
  command/AnnouncementCommandService.java

global/scheduler/
  AnnouncementPushScheduler.java
    # 기존 HourlyPrayerPushScheduler와 동일 패키지 (global/scheduler/)
    # @Scheduled(cron = "0 * * * * *")  ← 매분 실행
    # @ConditionalOnProperty("webpush.scheduler.enabled") 공유
    # send_at <= now AND is_sent=false 조회
    # → entity_type + entity_id 기준 부서 멤버 UUID 목록 조회
    #   (DepartmentMemberRepository.findByDepartmentId → member UUID 추출)
    # → PushSubscriptionPort.findByMemberIds() 로 구독 조회
    # → WebPushAdapter로 발송
    # → is_sent = true 업데이트

adapter/in/web/
  AnnouncementController.java
    GET    /announcements/recent?entityType=&entityId=   # 홈용 최신 2개
    GET    /announcements?entityType=&entityId=          # 목록 (최신순)
    GET    /announcements/{id}                           # 상세
    POST   /announcements                                # admin 전용 (생성)
    DELETE /announcements/{id}                           # admin 전용 (삭제)
  dto/announcement/
    AnnouncementResponse.java
    CreateAnnouncementRequest.java
```

#### 스케줄러 동작

```
매분 실행 (cron = "0 * * * * *")
  → send_at <= 현재시각 AND is_sent = false 인 announcement 조회
  → entity_type=DEPARTMENT 기준으로
      DepartmentMemberRepository.findByDepartmentId(entityId) → member UUID 목록
      PushSubscriptionPort.findByMemberIds(memberIds) → 구독 목록
  → WebPushAdapter로 발송
      title: announcement.title
      body:  announcement.body 앞 50자 (초과 시 "...")
      url:   /announcements/{id}
  → is_sent = true 업데이트
```

Push payload 딥링크: `/announcements/{id}` → 상세 페이지 직접 이동

#### 기존 PushSubscriptionPort 수정

`findByMemberIds(List<MemberId> memberIds): List<PushSubscription>` 추가.
`PushSubscriptionJpaRepository`에 `findByMemberIdIn(List<UUID>)` 추가.

### 2. Admin

공지사항 메뉴 신규 추가:

- **목록**: 작성된 공지사항 목록 (제목, 발송 예약 시각, 발송 여부)
- **작성 폼**:
  - 제목 (필수)
  - 본문 (필수)
  - 대상 부서: `session.departmentId` 로 자동 설정 (별도 선택 UI 없음)
  - 발송 예약 시각 (필수): 날짜 + 시 + 분 단위까지 설정 가능
  - **캘린더 이벤트 함께 생성** 체크박스 (선택):
    - 체크 시 시작일 / 종료일 입력 필드 추가 표시
    - 이벤트 생성 시 `entity_type = "DEPARTMENT"`, `entity_id = session.departmentId` 로 저장 (기존 events route 패턴 동일)
    - **공지사항을 삭제해도 캘린더 이벤트는 삭제되지 않으며, 별도로 삭제해야 한다.**

- **아키텍처**: Admin은 Prisma로 `announcement` 테이블에 직접 저장. 백엔드 API 호출 없음. 스케줄러가 `send_at` 기준으로 자동 발송.
- **캘린더 이벤트 생성** 시에도 Prisma로 `event` 테이블에 직접 저장 (동일 트랜잭션, 기존 color 자동 배정 로직 재사용).

권한: church admin 이상만 작성/삭제 가능

### 3. Web App

#### 홈 화면 변경

- 생일자 최대 노출 **3명**으로 축소 (기존 5명)
- "나의 한 주" 섹션과 생일자 섹션 **사이**에 공지사항 섹션 추가
  - 최신 2개 표시
  - "더보기" → `/announcements` 이동

#### 신규 라우트

```
/announcements        공지사항 목록 (최신순 리스트)
/announcements/:id    공지사항 상세 (게시글 형태)
```

#### 상세 페이지

- 게시글 형태 (모달 아님)
- 앱 푸시 탭 시 딥링크 도착지
- SW notificationclick 핸들러에서 `/announcements/{id}` 로 navigate

## 미포함 (다음 단계)

- 주기적 발송 (매주 예배 X분 전 등)
- 읽음 처리
