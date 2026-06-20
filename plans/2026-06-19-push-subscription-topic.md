# Push Subscription Topic 분리 설계

## 배경

기존에는 push_subscription row 존재 = 모든 알림 동의 (일괄). 향후 알림 종류가 늘어날 것을 고려해 topic별 동의를 분리한다.

## 설계 원칙

- `push_subscription` 존재 = 공지 알림 동의 (기본값, 항상 받음)
- `push_subscription_topic` row 존재 = 해당 topic 추가 opt-in
- 두 테이블 모두 hard delete (디바이스/동의 정보라 이력 불필요, 예외적으로 기본 정책과 다름)

## DB

### push_subscription (기존, hard delete로 변경)

기존 BaseEntity 상속 유지하되, 삭제 시 hard delete (repository.delete()) 사용.

### push_subscription_topic (신규)

```sql
CREATE TABLE push_subscription_topic (
  id         CHAR(36)     NOT NULL COMMENT 'PK',
  member_id  CHAR(36)     NOT NULL COMMENT '회원 ID',
  topic      VARCHAR(50)  NOT NULL COMMENT '알림 종류 (PRAYER 등)',
  created_at DATETIME(6)  NOT NULL COMMENT '생성일시',
  PRIMARY KEY (id),
  UNIQUE KEY uq_member_topic (member_id, topic),
  KEY idx_member_id (member_id)
) COMMENT='회원별 알림 topic 동의';
```

BaseEntity 미사용 (deleted_at 없음). hard delete.

### PushTopic enum

```
PRAYER      - 매일 오전 9시 기도제목 알림
```

향후 추가 예정: GATHERING, VISIT 등

## 백엔드 변경

### 신규
- `PushTopic` enum (domain/enums)
- `PushSubscriptionTopicJpaEntity`
- `PushSubscriptionTopicJpaRepository`
- `PushSubscriptionTopicPort` (out-port)
- `PushSubscriptionTopicPersistenceAdapter`
- `PushSubscriptionTopicCommandUseCase` (in-port)
- `PushSubscriptionTopicCommandService`
- `POST /push/topics/{topic}` — topic 구독
- `DELETE /push/topics/{topic}` — topic 구독 해제

### 변경
- `PushSubscriptionPersistenceAdapter.deleteByEndpoint()` → hard delete
- `HourlyPrayerPushScheduler` → PRAYER topic 구독자만 발송

## 프론트엔드 변경

### NotificationToggleRow → 두 줄로 분리
1. "알림 동의" 토글 — push_subscription 생성/삭제
2. "기도제목 매일 알림" 토글 — PRAYER topic 구독/해제 (알림 동의 ON일 때만 표시)

### api.ts
- `pushApi.subscribeTopic(topic)` — POST /push/topics/{topic}
- `pushApi.unsubscribeTopic(topic)` — DELETE /push/topics/{topic}
- `pushApi.getTopics()` — GET /push/topics (구독 중인 topic 목록)

## 발송 로직 변경

```
공지 발송: push_subscription 전체 → 발송
기도제목 발송: push_subscription_topic(PRAYER) → member_id 목록 → push_subscription JOIN → 발송
```
