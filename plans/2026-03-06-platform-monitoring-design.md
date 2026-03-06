# Platform Monitoring & Analytics Design

## 목적

admin 페이지에서 플랫폼 전체 사용량을 한눈에 볼 수 있는 모니터링 페이지.
system admin(SUPER_ADMIN)만 접근 가능.

---

## 1. 유저 방문 추적 (Umami)

### 인프라
- 맥미니에 Umami Docker 컨테이너 추가 (포트 예: 3001)
- Umami 자체 DB는 별도 MySQL DB (`umami`) 또는 PostgreSQL (Umami 기본값)
- website 2개 등록: web-app, admin

### 데이터 수집
- web-app: `index.html`의 `<head>`에 Umami 스크립트 삽입
- admin: `layout.tsx`의 `<head>`에 Umami 스크립트 삽입
- 자동 수집: 페이지뷰, 방문자, 세션, referrer, 브라우저, OS, 기기

### admin에서 조회 (Umami REST API)
- `GET /api/websites/{id}/stats` - 방문자 수, 페이지뷰, 바운스율
- `GET /api/websites/{id}/active` - 현재 접속자 수
- `GET /api/websites/{id}/pageviews` - 시간별/일별 추이
- `GET /api/websites/{id}/metrics` - 페이지별, 브라우저별, 기기별 통계

### admin UI에 보여줄 지표
- 오늘/이번주/이번달 방문자 수 (web-app, admin 각각)
- 일별 방문자 추이 차트
- 현재 접속자 수
- 인기 페이지 TOP 5
- 기기/브라우저 분포

---

## 2. 앱 활동 지표 (기존 DB 조회)

### 데이터 소스: `into-the-heaven` DB 직접 쿼리 (Prisma)

### 지표
- **이번 주 모임 생성 수**: gathering 테이블 WHERE created_at BETWEEN 이번주 시작~끝
- **이번 주 기도제목 등록 수**: prayer 테이블 WHERE created_at BETWEEN 이번주 시작~끝
- **전체 기도제목 수**: prayer 테이블 COUNT (deleted_at IS NULL)
- **신규 가입자 추이**: member 테이블 GROUP BY DATE(created_at)

### admin UI
- 숫자 카드 (이번 주 모임 수, 기도제목 수, 전체 기도제목, 신규 가입자)
- 신규 가입자 일별 추이 차트

---

## 3. Backend API 모니터링

### 데이터 수집
- Spring Boot Actuator 활성화 (`/actuator/metrics`, `/actuator/health`)
- 메트릭 수집 컨테이너에서 Actuator 엔드포인트 호출

### 수집 항목
- API 평균 응답 시간 (`http.server.requests`)
- API 에러율 (5xx 응답 비율)
- 활성 스레드 수
- JVM 힙 메모리 사용량

### 저장
- `metrics` 테이블에 함께 저장 (container_name = 'backend-api')

---

## 4. 서버/DB 리소스 모니터링

### 인프라
- ops/docker-compose.yml에 `metrics-collector` 컨테이너 추가
- 5분 간격 cron으로 수집
- `into-the-heaven` DB의 `metrics` 테이블에 저장

### DB 테이블: `metrics`

```sql
CREATE TABLE metrics (
  id BIGINT AUTO_INCREMENT PRIMARY KEY COMMENT '자동 증가 PK',
  collected_at DATETIME(0) NOT NULL COMMENT '메트릭 수집 시각',
  container_name VARCHAR(50) NOT NULL COMMENT '컨테이너 이름 (backend, admin, mysql, umami 등)',
  status VARCHAR(20) NOT NULL COMMENT '컨테이너 상태 (running, stopped, restarting)',
  cpu_percent DECIMAL(5,2) COMMENT 'CPU 사용률 (%)',
  memory_mb INT COMMENT '메모리 사용량 (MB)',
  memory_percent DECIMAL(5,2) COMMENT '메모리 사용률 (%)',
  disk_percent DECIMAL(5,2) COMMENT '디스크 사용률 (%)',
  network_rx_mb DECIMAL(10,2) COMMENT '네트워크 수신량 (MB)',
  network_tx_mb DECIMAL(10,2) COMMENT '네트워크 송신량 (MB)',
  restart_count INT DEFAULT 0 COMMENT '컨테이너 재시작 횟수',
  api_avg_response_ms INT COMMENT 'API 평균 응답 시간 (ms) - backend 전용',
  api_error_rate DECIMAL(5,2) COMMENT 'API 에러율 (%) - backend 전용',
  jvm_heap_mb INT COMMENT 'JVM 힙 메모리 사용량 (MB) - backend 전용',
  active_threads INT COMMENT '활성 스레드 수 - backend 전용',
  mysql_connections INT COMMENT '현재 DB 커넥션 수 - mysql 전용',
  mysql_slow_queries INT COMMENT '슬로우 쿼리 누적 수 - mysql 전용',
  mysql_threads_running INT COMMENT '실행중인 스레드 수 - mysql 전용',

  INDEX idx_metrics_collected (collected_at DESC),
  INDEX idx_metrics_container_time (container_name, collected_at DESC)
) COMMENT='서버/DB/API 리소스 메트릭 (5분 간격 수집, 30일 보존)';
```

### 수집 대상 컨테이너
- backend (Spring Boot)
- admin (Next.js)
- mysql
- umami
- mysql-backup

### admin UI
- 컨테이너별 현재 상태 카드 (running/stopped, CPU, 메모리)
- CPU/메모리 시간별 추이 차트
- MySQL 커넥션/슬로우 쿼리 추이
- API 응답 시간/에러율 추이

### 데이터 보존 정책
- 삭제 없이 영구 보존 (5분 간격 기준 하루 288건, 월 ~8,600건 수준이라 부담 없음)

---

## 5. 장애 알림

### 트리거 조건
- 컨테이너 status가 running이 아닐 때
- 연속 2회 이상 health check 실패 시 (오탐 방지)

### 알림 방법
- Gmail SMTP (backend에 이미 설정됨)
- 메트릭 수집 스크립트에서 직접 발송 (또는 admin API 호출)

### 알림 내용
- 어떤 서비스가 다운됐는지
- 다운 감지 시각
- 마지막 정상 상태 시각

---

## 6. 접근 권한

- admin의 `/system/monitoring` 페이지로 구성
- `requireSuperAdmin()` 체크 (기존 system 페이지와 동일 패턴)

---

## 7. 구현 범위 요약

| 구성 요소 | 수정 대상 | 신규/수정 |
|-----------|----------|----------|
| Umami 컨테이너 | ops/docker-compose.yml | 신규 추가 |
| Umami 스크립트 | web-app, admin | 수정 |
| 메트릭 수집 컨테이너 | ops/docker-compose.yml | 신규 추가 |
| 메트릭 수집 스크립트 | ops/metrics-collector/ | 신규 |
| metrics 테이블 | DB (DDL) | 신규 |
| Prisma 스키마 | admin/prisma/schema.prisma | 수정 |
| Backend Actuator | backend application.properties | 수정 |
| 모니터링 API | admin/src/app/api/system/monitoring/ | 신규 |
| 모니터링 UI | admin/src/app/system/monitoring/ | 신규 |
| 장애 알림 | ops/metrics-collector/ | 신규 |
