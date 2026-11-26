# 푸시 알림 발송·클릭·구독 이력 수집 + 어드민 분석

## 배경

DAU/WAU/MAU가 기대보다 낮고, 매일 9시 기도 알림도 잘 눌리지 않는다는 문제의식에서 출발.
알림을 개선하기 전에 **현재 기준값(baseline)** 을 먼저 쌓아야 개선 효과를 비교할 수 있다.

현재 상태 (2026-11-26, local-mysql 기준 조회):

| 지표 | 수치 |
|---|---|
| 전체 회원 | 425명 (ACTIVE 부서 회원 409명) |
| 푸시 권한 허용 회원 | 57명 (기기 69대) |
| 기도 알림(PRAYER) ON | 22명 (~5%) |
| 리딩 알림(READING) ON | 23명 |
| 푸시 허용했지만 어떤 토픽도 안 켬 | 27명 |

문제점:
- 발송 결과는 로그로만 남고 DB에 없음 → 몇 명에게 갔는지, 몇 명이 눌렀는지 알 수 없음
- `push_subscription_topic`은 OFF 시 hard delete → "켰다가 끈 사람"을 알 수 없음
  - `deleted_at` 컬럼이 없고 `uq_member_topic (member_id, topic)` unique 제약이 있어 soft delete로 단순 전환 불가
  - → 테이블은 "현재 상태" 전용으로 유지하고, **이력은 별도 테이블**에 쌓기로 결정

## 범위

- **포함**
  - 모든 푸시 발송 건을 `push_delivery_log`에 기록 (발송 결과 + 클릭)
  - 토픽 ON/OFF 이력을 `push_topic_history`에 기록
  - 클릭 수집 API + 서비스워커 연동
  - 어드민 `/system/monitoring`에 "푸시 알림" 탭
  - 일간/주간 사용 리포트 메일에 푸시 섹션 추가
  - 버그 수정: 기기 1대 구독 해지 시 회원의 모든 토픽이 삭제되는 문제
- **제외**
  - 기도 알림 문구/내용 개편 → `2026-11-26-my-prayer-and-prayer-push.md`
  - 과거 데이터 백필 (기록은 배포 시점부터)

## 설계

### 1. DB

```sql
CREATE TABLE push_delivery_log (
  id               CHAR(36)     NOT NULL COMMENT 'PK (푸시 payload에 deliveryId로 포함되어 클릭 추적에 사용)',
  member_id        CHAR(36)     NOT NULL COMMENT '수신 회원 ID',
  subscription_id  CHAR(36)     NULL     COMMENT '발송 대상 push_subscription ID (구독 삭제 후에도 이력 유지를 위해 FK 없음)',
  push_type        VARCHAR(50)  NOT NULL COMMENT '알림 종류 (DAILY_PRAYER/DAILY_READING/ANNOUNCEMENT/BROADCAST/GATHERING_COMMENT/REPORT_COMMENT/CHURCH_REQUEST 등)',
  template_key     VARCHAR(50)  NULL     COMMENT '사용한 문구 템플릿 키 (문구 로테이션/AB 비교용, 예: PRAYER_A_1)',
  entity_type      VARCHAR(30)  NULL     COMMENT '알림이 가리키는 엔티티 종류 (예: PRAYER, ANNOUNCEMENT)',
  entity_id        CHAR(36)     NULL     COMMENT '알림이 가리키는 엔티티 ID (예: 노출한 기도제목 ID)',
  send_result      VARCHAR(20)  NOT NULL COMMENT '발송 결과 (SUCCESS/GONE/INVALID/TRANSIENT_FAIL)',
  sent_at          DATETIME(6)  NOT NULL COMMENT '발송 시각',
  clicked_at       DATETIME(6)  NULL     COMMENT '최초 클릭 시각 (클릭 안 하면 NULL)',
  created_at       DATETIME(6)  NOT NULL COMMENT '생성일시',
  updated_at       DATETIME(6)  NOT NULL COMMENT '수정일시',
  deleted_at       DATETIME(6)  NULL     COMMENT '삭제일시 (soft delete)',
  PRIMARY KEY (id),
  KEY idx_pdl_type_sent (push_type, sent_at),
  KEY idx_pdl_member_type_sent (member_id, push_type, sent_at)
) COMMENT='푸시 알림 발송/클릭 로그 (기기 단위 1건)';

CREATE TABLE push_topic_history (
  id          CHAR(36)     NOT NULL COMMENT 'PK',
  member_id   CHAR(36)     NOT NULL COMMENT '회원 ID',
  topic       VARCHAR(50)  NOT NULL COMMENT '푸시 토픽 (PRAYER/READING)',
  action      VARCHAR(20)  NOT NULL COMMENT '변경 종류 (SUBSCRIBE/UNSUBSCRIBE)',
  source      VARCHAR(30)  NOT NULL COMMENT '변경 원인 (USER_TOGGLE: 설정 토글, ALL_DEVICES_REMOVED: 마지막 기기 해지/만료)',
  created_at  DATETIME(6)  NOT NULL COMMENT '변경 시각',
  updated_at  DATETIME(6)  NOT NULL COMMENT '수정일시',
  deleted_at  DATETIME(6)  NULL     COMMENT '삭제일시 (soft delete)',
  PRIMARY KEY (id),
  KEY idx_pth_topic_created (topic, created_at),
  KEY idx_pth_member_topic_created (member_id, topic, created_at)
) COMMENT='푸시 토픽 구독 ON/OFF 이력';
```

인덱스 근거:
- 어드민/리포트: 기간 + 종류별 집계 → `(push_type, sent_at)`, `(topic, created_at)`
- 문구 로테이션(다음 문서): "이 회원에게 최근 보낸 DAILY_PRAYER" → `(member_id, push_type, sent_at)`
- 클릭 업데이트는 PK 조회

기존 `push_subscription_topic`의 `findMemberIdByTopic`(topic 단독 조건)은 현재 `member_id` 인덱스만 있어 풀스캔. 데이터 소량이지만 이번에 인덱스 추가:

```sql
ALTER TABLE push_subscription_topic ADD INDEX idx_pst_topic (topic);
```

### 2. 백엔드

**발송 경로 일원화** — `webPushPort.send`를 직접 호출하던 곳 6군데:
`PushNotificationSendService`, `DailyReadingPushScheduler`, `AnnouncementPushScheduler`, `GatheringCommandService`, `ReportCommandService`, `ChurchCommandService`

→ `PushDispatchService`(`send` / `sendToSubscriptions` / `sendToMembers` / `sendToMember`) 하나로 모았다.
`PushMessage(type, title, body, url, templateKey, entityType, entityId)`로 종류·문구·대상 엔티티를 함께 넘긴다.
1. `push_delivery_log` 행 ID 선발급
2. payload에 `deliveryId`, `type` 포함해 발송
3. 결과(`send_result`) 저장, GONE/INVALID면 기존처럼 구독 삭제
- `PushPayload` record에 `deliveryId`, `type` 필드 추가
- 로그 저장 실패가 발송을 막지 않도록 예외는 삼키고 warn 로그

**클릭 수집 API**
- `POST /push/deliveries/{deliveryId}/click` (인증 불필요 — 서비스워커에서 토큰 접근 불가. deliveryId가 추측 불가한 UUID라 남용 위험 낮음)
- `clicked_at IS NULL`일 때만 갱신 (최초 클릭만), 없는 ID는 204로 무시

**토픽 이력**
- `PushSubscriptionTopicCommandService.subscribe/unsubscribe`에서 실제 상태가 바뀐 경우에만 `push_topic_history` 기록 (이미 ON인데 subscribe 재호출 등은 기록 안 함)

**버그 수정: 기기 1대 해지 시 전체 토픽 삭제**
- 현재 `PushSubscriptionCommandService.unsubscribe`가 endpoint 삭제 후 `deleteAllByMemberId`로 회원 토픽 전부 삭제 → 폰+PC 사용자가 PC에서만 해지해도 폰 알림까지 꺼짐
- 수정: endpoint 삭제 후 **남은 기기가 0대일 때만** 토픽 삭제 (+ `ALL_DEVICES_REMOVED` 이력 기록)
- 발송 중 GONE/INVALID로 마지막 기기가 지워지는 경우는 토픽을 유지 (재설치/재허용 시 설정 복원). 이력에도 남기지 않음

**admin(Next.js) 직접 발송** — `admin/src/lib/webpush.ts`의 `sendPushToMembers`(목회자 코멘트, `ADMIN_COMMENT`)도 같은 규격으로 `push_delivery_log`를 기록하고 payload에 `deliveryId`/`type`/`pd`를 넣는다. backend `PushType`에도 `ADMIN_COMMENT`를 추가해 값 집합을 맞췄다.

### 3. 웹앱 (서비스워커 + 앱 측 보조 수집)

현재 구조:
- `public/sw.js`는 Vite가 처리하지 않는 정적 파일 → `import.meta.env.VITE_API_BASE_URL` 사용 불가
- 등록: `index.html` 인라인 스크립트에서 `register('/sw.js')`
- `tag`를 URL로 추론 (`/` → `mitl-daily-prayer`, `/reading*` → `mitl-daily-reading`, 그 외 `mitl-notification`)
- `install`에서 `skipWaiting()`, `activate`에서 `clients.claim()` → 새 SW는 다음 앱 실행 시 즉시 활성화
- 백엔드 CORS는 모든 origin 허용 → SW에서 API 도메인으로 cross-origin fetch 가능

**3-1. API base URL 전달**
- `index.html`에서 `register('/sw.js?api=%VITE_API_BASE_URL%')` — Vite가 HTML의 `%VITE_*%`를 빌드 시 치환 (신규 env 불필요)
- SW: `?api=` 값을 읽어 절대 주소(`https://...`)와 같은 origin 상대 경로(`/api`, 현재 `.env` 값) 모두 지원. 치환되지 않은 값이면 SW 측 수집은 건너뛰고 앱 측 fallback에 맡긴다
- 스크립트 URL이 바뀌면 같은 scope의 registration이 업데이트되며 push 구독은 registration에 묶여 유지됨 → **배포 전 실기기(iOS PWA, Android Chrome)에서 업데이트 후 알림 수신 유지 확인 필수**

**3-2. `push` 핸들러**
- notification `data`에 `{ url, deliveryId, type }` 보관
- `tag`를 `type` 기반으로: `DAILY_PRAYER` → `mitl-daily-prayer`, `DAILY_READING` → `mitl-daily-reading`, 그 외 → `mitl-notification` (기존 동작 유지)
  - 기도 알림 딥링크가 `/` → `/prayers?...`로 바뀌므로 URL 추론 방식은 더 이상 동작하지 않음

**3-3. `notificationclick` 핸들러**
- 기존 이동 로직(열린 창 focus + postMessage, 없으면 openWindow) 유지
- 병렬로 `fetch(API_BASE + '/push/deliveries/' + deliveryId + '/click', { method: 'POST', keepalive: true }).catch(() => {})`를 `waitUntil`에 포함
- 클릭 수집 실패가 화면 이동을 막지 않도록 `Promise.allSettled`

**3-4. 앱 측 보조 수집 (fallback)**
- 서버가 payload `url`에 `pd={deliveryId}` 쿼리를 붙여 보냄 (예: `/prayers?highlight=xxx&pd=yyy`)
- 앱(`App.tsx`의 `SwNavigateHandler` 근처)에서 URL에 `pd`가 있으면 클릭 API 호출 후 `replace`로 쿼리 제거
- 서버는 최초 클릭만 반영(멱등) → SW와 앱 양쪽에서 호출돼도 중복 집계 없음
- 목적: **구 SW를 쓰는 사용자**(배포 후 아직 앱을 안 열어 SW 미갱신)도 구 SW가 `data.url`을 그대로 열기 때문에 클릭이 수집됨. iOS에서 SW fetch가 누락되는 경우도 보완

**3-5. 배포 전환기 동작**

| 상황 | 화면 이동 | 클릭 수집 | tag |
|---|---|---|---|
| 새 SW | 정상 | SW + 앱 | type 기반 |
| 구 SW (미갱신) | 정상 (`data.url` 사용) | 앱 fallback만 | 기도 알림이 `mitl-notification`으로 분류 (전날 알림 대체 안 됨, 기능 문제 없음) |

### 4. 어드민 — `/system/monitoring` "푸시 알림" 탭

system admin 전용 (기존 모니터링 페이지 권한 그대로). Prisma로 직접 집계.

- 상단 지표 카드 (기간 선택: 7일/30일)
  - 토픽별 현재 구독자 수 (PRAYER / READING) + 기간 내 순증감
  - 종류별 발송 회원 수 / 클릭 회원 수 / 클릭률
- 차트: 일별 클릭률 추이 (종류별 라인)
- 표: 종류 × 부서 클릭률, 문구 템플릿(`template_key`)별 클릭률
- 이탈: 기간 내 SUBSCRIBE 후 7일 이내 UNSUBSCRIBE 비율
- 클릭률 정의: **회원·일(KST) 단위** — `SUCCESS로 받은 (회원, 날짜) 중 클릭한 (회원, 날짜)의 비율`. 같은 날 같은 종류 알림을 여러 기기·여러 번 받아도 1건
- 클릭 시간대(KST) 분포 추가
- 구현: `admin/src/lib/push-analytics.ts`(raw SQL, `CONVERT_TZ`로 KST 변환), `push-analytics-shared.ts`(클라이언트 공용 라벨/타입), `GET /api/system/monitoring/push?days=7|30|90`
- 화면: `/system/monitoring`을 "시스템" / "푸시 알림" 탭으로 분리 (`monitoring-tabs.tsx`, `push-analytics-client.tsx`)

### 5. 사용 리포트 메일

`/api/system/monitoring/usage-report`에 "푸시 알림" 섹션 추가
- 종류별 클릭/발송(회원·일) / 클릭률, 전일(전주) 대비 %p
- 푸시 섹션 조회가 실패해도(예: 테이블 미생성) 나머지 리포트는 그대로 발송
- PRAYER/READING 구독자 수 + ON/OFF 건수

## 테스트

- 백엔드 (Mockito + AssertJ, @Nested + @DisplayName)
  - `PushDispatchService`: 결과별 로그 저장, GONE 시 구독 삭제, 로그 저장 실패해도 발송 결과 반환
  - 클릭 API: 최초 클릭만 반영, 없는 ID 무시
  - 토픽 이력: 상태 변경 시에만 기록
  - 기기 해지: 남은 기기 있으면 토픽 유지 / 0대면 삭제 + 이력
- 어드민: 클릭률 집계 함수 단위 테스트

## 배포 시 체크

- DDL 수동 실행 (위 CREATE 2건 + `push_subscription_topic` 인덱스) — **backend 배포 전에 실행**. 테이블이 없으면 발송 로그 저장만 실패하고(warn 로그) 발송은 정상 진행되지만, 토픽 ON/OFF는 이력 저장 실패로 트랜잭션이 롤백되어 토글이 실패한다
- admin: `prisma/schema.prisma`에 `push_subscription_topic` / `push_delivery_log` / `push_topic_history` 모델 추가 (DB 변경 없음, `prisma generate`만)
- 신규 환경변수 없음 (sw.js API base 주입 방식에 따라 변경 시 안내)
