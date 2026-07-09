# Web Push 알림 인프라 + 일일 기도제목 리마인더

작성일: 2026-04-21

## 배경 및 목적

홈 화면에 이번 주 기도제목이 노출되고 있지만 사용자가 자발적으로 앱에 진입하지 않아 복기가 이루어지지 않는 문제가 있었다.

대부분의 사용자가 iOS 홈화면에 PWA로 추가해 사용하고 있어 Web Push(iOS 16.4+)를 활용할 수 있었다. Web Push는 완전 무료이며 앱 전반의 알림 인프라로 재사용 가능하다.

## 구현 범위

이번 작업에서 두 가지를 동시에 구축했다:

1. **Web Push 인프라** — 구독 저장, VAPID 인증, 발송 어댑터
2. **기도제목 리마인더** — 매일 오전 9시(사용자 현지 시간 기준) 기도제목 알림

## 주요 결정 사항

### 알림 시각
- 매시간 정각에 스케줄러 실행 → 각 구독의 타임존 기준 9시인 경우만 발송
- 이유: 단일 KST 고정 대신 해외 사용자 대응을 처음부터 고려

### 타임존 처리
- `push_subscription.timezone` 컬럼에 IANA 타임존명 저장 (기본값 `Asia/Seoul`)
- 브라우저에서 `Intl.DateTimeFormat().resolvedOptions().timeZone`으로 자동 추출
- 반-시간 오프셋 타임존(인도 UTC+5:30)은 9:00~9:59 사이 발송 허용

### 알림 메시지 형식
- 제목: `오늘도, 기도로 시작해볼까요? 🙏`
- 본문: `"{기도제목 앞 18자}..."` + 2개 이상일 때 `외 N-1개`
- 딥링크: `/prayers` (기도 탭)

### 권한 요청 UX
- 온보딩: 알림 기능 소개만 (권한 요청 X)
- MY 탭: 알림 받기 토글 탭 시 브라우저 권한 다이얼로그
- `denied` 상태: 토글 off + "브라우저 설정에서 허용 필요" 안내

### 만료 구독 정리
- 410/404 응답 시 해당 구독 자동 soft delete
- 403(VAPID mismatch)도 동일 처리

## 아키텍처

### Backend

```
domain/model/
  PushSubscription.java          # 구독 도메인 모델
  PushSubscriptionId.java        # UUID wrapper

adapter/out/persistence/
  PushSubscriptionJpaEntity.java  # push_subscription 테이블
  PushSubscriptionJpaRepository   # findByMemberId, findByEndpoint, softDeleteByEndpoint
  PushSubscriptionPersistenceMapper
  PushSubscriptionPersistenceAdapter

adapter/out/webpush/
  WebPushAdapter.java             # nl.martijndwars:web-push:5.1.1 래핑

application/port/out/
  PushSubscriptionPort.java
  WebPushPort.java                # SendResult enum + PushPayload record

application/port/in/command/
  PushSubscriptionCommandUseCase.java
  dto/SubscribePushCommand.java

application/service/command/
  PushSubscriptionCommandService.java   # upsert by endpoint
  PushNotificationSendService.java      # 타임존 필터 → 기도제목 조회 → fan-out

application/service/query/
  WeeklyPrayerQueryService.java         # HomeQueryService에서 추출한 주간 기도제목 조회

adapter/in/web/
  PushSubscriptionController.java       # POST/DELETE /push/subscriptions, GET /push/vapid-public-key
  dto/push/SubscribePushRequest.java
  dto/push/UnsubscribePushRequest.java
  dto/push/VapidPublicKeyResponse.java

global/
  config/WebPushConfig.java             # @ConfigurationProperties + PushService Bean + Clock Bean
  scheduler/HourlyPrayerPushScheduler.java  # @Scheduled(cron="0 0 * * * *")
```

### DB 스키마 (push_subscription 테이블)

```sql
CREATE TABLE push_subscription (
  id            CHAR(36)        NOT NULL PRIMARY KEY,
  member_id     CHAR(36)        NOT NULL,
  endpoint      VARCHAR(1000)   NOT NULL UNIQUE,
  p256dh        VARCHAR(255),
  auth          VARCHAR(255),
  user_agent    VARCHAR(500),
  timezone      VARCHAR(64)     NOT NULL DEFAULT 'Asia/Seoul',
  created_at    DATETIME        NOT NULL,
  updated_at    DATETIME        NOT NULL,
  deleted_at    DATETIME,
  INDEX idx_push_subscription_member_id (member_id)
);
```

### Frontend

```
public/sw.js                              # push + notificationclick 핸들러 추가
src/App.tsx                               # SwNavigateHandler (SW → React router 브릿지)
src/lib/
  push.ts                                 # isSupported, getPermission, subscribe, unsubscribe
  api.ts                                  # pushApi 추가
src/features/my/components/
  NotificationToggleRow.tsx               # 3-state 토글 (default/granted/denied)
src/features/onboarding/slides/
  NotificationSlide.tsx                   # 알림 기능 소개 슬라이드
src/features/onboarding/onboarding-data.ts  # NotificationSlide OutroSlide 직전에 등록
```

## 동시성 및 성능

- 발송은 `ExecutorService(fixedThreadPool=8)`로 병렬 처리
- 타임아웃: 총 `memberCount * 10s`
- 각 send 실패는 결과 enum으로 분기 처리 (GONE/INVALID → 삭제, TRANSIENT_FAIL → 로그)

## 환경변수 (Doppler 관리)

| 변수 | 설명 |
|------|------|
| `WEBPUSH_VAPID_PUBLIC_KEY` | VAPID 공개키 |
| `WEBPUSH_VAPID_PRIVATE_KEY` | VAPID 개인키 (절대 커밋 금지) |
| `WEBPUSH_VAPID_SUBJECT` | 연락처 (mailto:...) |
| `WEBPUSH_SCHEDULER_ENABLED` | prd=true, dev=false |

키 발급: `npx web-push generate-vapid-keys`

## 한계 및 후속 과제

- **iOS 16.4+ 홈화면 추가 PWA 필수**: 미지원 환경에서는 토글 숨김
- **다중 인스턴스 배포 시 중복 발송**: 현재 단일 JVM 전제. 멀티 노드 확장 시 ShedLock 추가 필요
- **반-시간 오프셋 타임존**: 정각 기준이므로 최대 ~30분 오차 (허용 범위로 결정)

---

## 2026-07-09 업데이트: 구독 자동 복구 (silentSync) + 프롬프트 위치 수정

### 배경

브라우저 push 구독은 캐시 삭제, 브라우저 업데이트 등으로 무효화될 수 있다. 백엔드는 발송 실패(GONE/INVALID) 시 `push_subscription` 레코드를 삭제하지만, 브라우저는 구독이 살아있다고 판단해 UI에 "알림 켜짐"으로 표시된다. 유저는 알림이 안 오는 이유를 알 수 없다.

### silentSync()

앱 실행 시 push 구독을 자동으로 점검하고 복구한다.

1. `Notification.permission !== "granted"` → 리턴
2. `getSubscription()` → null이면 팝업 없이 새 구독 생성
3. endpoint와 `localStorage["push.endpoint"]` 비교 → 같으면 스킵, 다르면 백엔드 재등록

`subscribe()` 성공 시 localStorage에 endpoint 저장, `unsubscribe()` 시 삭제.
`AppShell` 마운트 시 호출 → 앱 진입마다 자동 복구. 정상 상황에서는 백엔드 호출 없음(localStorage 비교로 스킵).

### 알림 프롬프트 위치 수정

- 기존: AppShell 마운트 시 어느 페이지에서나 1.5초 후 노출
- 변경: 홈 화면(`/`) 진입 시에만 노출
- 이유: `/pending-approval` 등 가입 대기 페이지에서 프롬프트가 뜨는 것은 의도하지 않은 동작
