# Platform Monitoring Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** admin 페이지에 플랫폼 모니터링 대시보드를 추가하여 유저 방문, 앱 활동, 서버/DB 리소스, API 상태를 한눈에 볼 수 있게 한다.

**Architecture:** Umami(Docker)로 유저 방문 추적, ops의 메트릭 수집 컨테이너로 서버/DB/API 리소스를 5분마다 수집하여 into-the-heaven DB의 metrics 테이블에 저장. admin Next.js에서 Umami API + Prisma 쿼리로 데이터를 조회하여 모니터링 UI에 렌더링.

**Tech Stack:** Umami (Docker), Alpine + shell script (metrics collector), Spring Boot Actuator, Next.js API Routes, Prisma, recharts (차트)

---

## Task 1: Umami Docker 컨테이너 셋업

**Files:**
- Modify: `ops/docker-compose.yml`
- Modify: `ops/.env.example`

**Step 1: ops/docker-compose.yml에 Umami 서비스 추가**

```yaml
# ops/docker-compose.yml 에 아래 서비스들 추가

  umami-db:
    image: postgres:16-alpine
    container_name: umami-db
    restart: unless-stopped
    env_file: .env
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: ${UMAMI_DB_PASSWORD}
    volumes:
      - umami-db-data:/var/lib/postgresql/data

  umami:
    image: ghcr.io/umami-software/umami:postgresql-latest
    container_name: umami
    restart: unless-stopped
    ports:
      - "3002:3000"
    environment:
      DATABASE_URL: postgresql://umami:${UMAMI_DB_PASSWORD}@umami-db:5432/umami
      DISABLE_TELEMETRY: 1
    depends_on:
      - umami-db

# volumes 섹션에 추가
volumes:
  umami-db-data:
```

> Note: Umami는 PostgreSQL을 기본 지원. 기존 MySQL과 별개로 가벼운 PostgreSQL 컨테이너를 띄움.

**Step 2: .env.example 업데이트**

```
# 기존 내용 유지하고 아래 추가
UMAMI_DB_PASSWORD=
```

**Step 3: 사용자에게 안내**

> "ops/.env 파일에 UMAMI_DB_PASSWORD를 설정해주세요. 그 후 `docker compose up -d umami-db umami`로 Umami를 시작하세요. 브라우저에서 http://맥미니IP:3002 접속 후 기본 계정(admin/umami)으로 로그인하여:
> 1. 비밀번호 변경
> 2. website 2개 추가 (web-app, admin)
> 3. 각 website의 tracking code(website ID)와 API 토큰을 메모해주세요."

**Step 4: Commit**

```bash
git add ops/docker-compose.yml ops/.env.example
git commit -m "feat: add Umami analytics Docker setup"
```

---

## Task 2: Umami 스크립트 삽입 (web-app, admin)

**Files:**
- Modify: `web-app/index.html`
- Modify: `admin/src/app/layout.tsx`
- Modify: `admin/.env.example` (또는 .env.local 안내)

**Step 1: web-app/index.html에 Umami 스크립트 추가**

`<head>` 태그 안, `<!-- Font -->` 줄 바로 위에 추가:

```html
    <!-- Analytics -->
    <script defer src="%VITE_UMAMI_URL%/script.js" data-website-id="%VITE_UMAMI_WEBSITE_ID%"></script>
```

> Note: Vite는 `%ENV_VAR%` 형식이 아니라 빌드 시 치환이 안 됨. 대안으로 하드코딩하거나, `index.html`에 직접 URL을 넣어야 함. 실제로는 배포 환경의 Umami URL을 직접 기입:

```html
    <!-- Analytics -->
    <script defer src="https://umami.intotheheaven.app/script.js" data-website-id="WEB_APP_WEBSITE_ID"></script>
```

> 사용자에게: "Umami에서 web-app website를 추가한 후 발급된 website ID로 `data-website-id` 값을 교체해주세요. Umami URL도 실제 도메인 또는 IP:3002로 변경해주세요."

**Step 2: admin/src/app/layout.tsx에 Umami 스크립트 추가**

`<head>` 태그 안, `<meta name="theme-color" ...>` 줄 아래에 추가:

```tsx
        <script
          defer
          src={`${process.env.NEXT_PUBLIC_UMAMI_URL}/script.js`}
          data-website-id={process.env.NEXT_PUBLIC_UMAMI_WEBSITE_ID}
        />
```

**Step 3: 사용자에게 환경변수 안내**

> "admin의 .env.local (또는 Doppler)에 아래 환경변수를 추가해주세요:
> - `NEXT_PUBLIC_UMAMI_URL` = Umami 서버 URL (예: http://맥미니IP:3002)
> - `NEXT_PUBLIC_UMAMI_WEBSITE_ID` = Umami에서 admin website 추가 후 발급된 ID"

**Step 4: Commit**

```bash
git add web-app/index.html admin/src/app/layout.tsx
git commit -m "feat: add Umami tracking script to web-app and admin"
```

---

## Task 3: metrics 테이블 생성 + Prisma 스키마

**Files:**
- DDL: SQL 실행 (사용자에게 전달)
- Modify: `admin/prisma/schema.prisma`

**Step 1: DDL 작성 (사용자에게 전달)**

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
) COMMENT='서버/DB/API 리소스 메트릭 (5분 간격 수집, 영구 보존)';
```

> "이 DDL을 into-the-heaven DB에 실행해주세요."

**Step 2: Prisma 스키마에 metrics 모델 추가**

`admin/prisma/schema.prisma` 파일 맨 아래에 추가:

```prisma
/// 서버/DB/API 리소스 메트릭 (5분 간격 수집, 영구 보존)
model metrics {
  id                  BigInt    @id @default(autoincrement())
  collected_at        DateTime  @db.DateTime(0)
  container_name      String    @db.VarChar(50)
  status              String    @db.VarChar(20)
  cpu_percent         Decimal?  @db.Decimal(5, 2)
  memory_mb           Int?
  memory_percent      Decimal?  @db.Decimal(5, 2)
  disk_percent        Decimal?  @db.Decimal(5, 2)
  network_rx_mb       Decimal?  @db.Decimal(10, 2)
  network_tx_mb       Decimal?  @db.Decimal(10, 2)
  restart_count       Int?      @default(0)
  api_avg_response_ms Int?
  api_error_rate      Decimal?  @db.Decimal(5, 2)
  jvm_heap_mb         Int?
  active_threads      Int?
  mysql_connections   Int?
  mysql_slow_queries  Int?
  mysql_threads_running Int?

  @@index([collected_at(sort: Desc)], map: "idx_metrics_collected")
  @@index([container_name, collected_at(sort: Desc)], map: "idx_metrics_container_time")
}
```

**Step 3: Prisma Client 생성**

Run: `cd admin && npx prisma generate`

**Step 4: Commit**

```bash
git add admin/prisma/schema.prisma
git commit -m "feat: add metrics table to Prisma schema"
```

---

## Task 4: Spring Boot Actuator 활성화

**Files:**
- Modify: `backend/build.gradle`
- Modify: `backend/src/main/resources/application.properties`
- Modify (if exists): Spring Security 설정에서 actuator 경로 허용

**Step 1: build.gradle에 Actuator 의존성 추가**

`dependencies` 블록 안에 추가:

```gradle
    // Actuator (monitoring)
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

**Step 2: application.properties에 Actuator 설정 추가**

파일 맨 아래에 추가:

```properties
# === Actuator (Monitoring) ===
management.endpoints.web.exposure.include=health,metrics
management.endpoints.web.base-path=/actuator
management.endpoint.health.show-details=always
```

**Step 3: Spring Security에서 /actuator/** 경로 허용**

참고: Security 설정 파일을 확인하여 `/actuator/**` 경로를 permitAll로 추가해야 함. 기존 SecurityConfig 파일을 찾아서 수정.

```java
// SecurityFilterChain 내 .requestMatchers("/actuator/**").permitAll() 추가
```

> 실행 시 확인할 파일: `backend/src/main/java/mitl/IntoTheHeaven/global/` 아래 security 관련 설정

**Step 4: 빌드 확인**

Run: `cd backend && ./gradlew compileJava`
Expected: BUILD SUCCESSFUL

**Step 5: Commit**

```bash
git add backend/build.gradle backend/src/main/resources/application.properties
# + security config 파일 (수정한 경우)
git commit -m "feat: enable Spring Boot Actuator for monitoring"
```

> "Doppler에 Actuator 관련 환경변수 변경은 없습니다. 다만 backend 재배포가 필요합니다."

---

## Task 5: 메트릭 수집 컨테이너 (ops)

**Files:**
- Modify: `ops/docker-compose.yml`
- Create: `ops/metrics-collector/collect.sh`
- Modify: `ops/.env.example`

**Step 1: collect.sh 작성**

```bash
#!/bin/sh
# 메트릭 수집 스크립트 - 5분마다 실행
# Docker 컨테이너 리소스 + MySQL 상태 + Backend Actuator 수집 → DB 저장

COLLECTED_AT=$(date '+%Y-%m-%d %H:%M:%S')
DISK_PERCENT=$(df / | tail -1 | awk '{print $5}' | tr -d '%')

# 대상 컨테이너 목록
CONTAINERS="into-the-heaven-backend into-the-heaven-admin mysql umami mysql-backup"

for CONTAINER in $CONTAINERS; do
  # Docker stats (1회 스냅샷)
  STATS=$(docker stats "$CONTAINER" --no-stream --format "{{.CPUPerc}}|{{.MemUsage}}|{{.NetIO}}" 2>/dev/null)

  if [ -z "$STATS" ]; then
    # 컨테이너가 없거나 중지됨
    STATUS="stopped"
    CPU=0
    MEM_MB=0
    MEM_PERCENT=0
    NET_RX=0
    NET_TX=0
    RESTART=0
  else
    STATUS="running"
    CPU=$(echo "$STATS" | cut -d'|' -f1 | tr -d '%')
    MEM_RAW=$(echo "$STATS" | cut -d'|' -f2)
    MEM_USED=$(echo "$MEM_RAW" | cut -d'/' -f1 | xargs)
    # MiB -> MB 변환
    MEM_MB=$(echo "$MEM_USED" | sed 's/[^0-9.]//g' | awk '{printf "%d", $1}')
    MEM_TOTAL=$(echo "$MEM_RAW" | cut -d'/' -f2 | xargs | sed 's/[^0-9.]//g')
    if [ "$MEM_TOTAL" != "0" ] && [ -n "$MEM_TOTAL" ]; then
      MEM_PERCENT=$(awk "BEGIN {printf \"%.2f\", ($MEM_MB/$MEM_TOTAL)*100}")
    else
      MEM_PERCENT=0
    fi
    NET_RAW=$(echo "$STATS" | cut -d'|' -f3)
    NET_RX=$(echo "$NET_RAW" | cut -d'/' -f1 | xargs | sed 's/[^0-9.]//g')
    NET_TX=$(echo "$NET_RAW" | cut -d'/' -f2 | xargs | sed 's/[^0-9.]//g')
    RESTART=$(docker inspect "$CONTAINER" --format='{{.RestartCount}}' 2>/dev/null || echo 0)
  fi

  # Backend Actuator 메트릭 (backend 컨테이너만)
  API_AVG_MS="NULL"
  API_ERROR_RATE="NULL"
  JVM_HEAP="NULL"
  THREADS="NULL"

  if [ "$CONTAINER" = "into-the-heaven-backend" ] && [ "$STATUS" = "running" ]; then
    # 응답 시간
    API_AVG_MS=$(wget -qO- "http://into-the-heaven-backend:8080/actuator/metrics/http.server.requests" 2>/dev/null \
      | sed -n 's/.*"TOTAL_TIME".*"value":\([0-9.]*\).*/\1/p' \
      | awk '{printf "%d", $1 * 1000}' 2>/dev/null || echo "NULL")
    [ -z "$API_AVG_MS" ] && API_AVG_MS="NULL"

    # JVM 힙
    JVM_HEAP=$(wget -qO- "http://into-the-heaven-backend:8080/actuator/metrics/jvm.memory.used?tag=area:heap" 2>/dev/null \
      | sed -n 's/.*"value":\([0-9.]*\).*/\1/p' \
      | awk '{printf "%d", $1 / 1048576}' 2>/dev/null || echo "NULL")
    [ -z "$JVM_HEAP" ] && JVM_HEAP="NULL"

    # 활성 스레드
    THREADS=$(wget -qO- "http://into-the-heaven-backend:8080/actuator/metrics/jvm.threads.live" 2>/dev/null \
      | sed -n 's/.*"value":\([0-9.]*\).*/\1/p' \
      | awk '{printf "%d", $1}' 2>/dev/null || echo "NULL")
    [ -z "$THREADS" ] && THREADS="NULL"
  fi

  # MySQL 메트릭 (mysql 컨테이너만)
  MYSQL_CONN="NULL"
  MYSQL_SLOW="NULL"
  MYSQL_THREADS="NULL"

  if [ "$CONTAINER" = "mysql" ] && [ "$STATUS" = "running" ]; then
    MYSQL_STATUS=$(mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -p"$DB_PASSWORD" -N -e \
      "SHOW GLOBAL STATUS WHERE Variable_name IN ('Threads_connected','Slow_queries','Threads_running')" 2>/dev/null)
    MYSQL_CONN=$(echo "$MYSQL_STATUS" | grep Threads_connected | awk '{print $2}')
    MYSQL_SLOW=$(echo "$MYSQL_STATUS" | grep Slow_queries | awk '{print $2}')
    MYSQL_THREADS=$(echo "$MYSQL_STATUS" | grep Threads_running | awk '{print $2}')
    [ -z "$MYSQL_CONN" ] && MYSQL_CONN="NULL"
    [ -z "$MYSQL_SLOW" ] && MYSQL_SLOW="NULL"
    [ -z "$MYSQL_THREADS" ] && MYSQL_THREADS="NULL"
  fi

  # DB INSERT
  mysql -h "$DB_HOST" -P "$DB_PORT" -u "$DB_USER" -p"$DB_PASSWORD" "$DB_NAME" -e \
    "INSERT INTO metrics (collected_at, container_name, status, cpu_percent, memory_mb, memory_percent, disk_percent, network_rx_mb, network_tx_mb, restart_count, api_avg_response_ms, api_error_rate, jvm_heap_mb, active_threads, mysql_connections, mysql_slow_queries, mysql_threads_running)
     VALUES ('$COLLECTED_AT', '$CONTAINER', '$STATUS', $CPU, $MEM_MB, $MEM_PERCENT, $DISK_PERCENT, $NET_RX, $NET_TX, $RESTART, $API_AVG_MS, NULL, $JVM_HEAP, $THREADS, $MYSQL_CONN, $MYSQL_SLOW, $MYSQL_THREADS);" 2>/dev/null
done

# === 장애 알림 ===
# 중지된 컨테이너 확인
DOWN_LIST=""
for CONTAINER in $CONTAINERS; do
  IS_RUNNING=$(docker inspect -f '{{.State.Running}}' "$CONTAINER" 2>/dev/null)
  if [ "$IS_RUNNING" != "true" ]; then
    DOWN_LIST="$DOWN_LIST $CONTAINER"
  fi
done

# 다운된 서비스가 있고, 이전 5분에도 다운이었으면 (연속 2회) 알림 발송
if [ -n "$DOWN_LIST" ]; then
  ALERT_FLAG="/tmp/metrics_alert_sent"
  if [ -f "$ALERT_FLAG" ]; then
    # 이미 이전에 감지됨 → 알림 발송
    PREV_DOWN=$(cat "$ALERT_FLAG")
    # 교집합 확인 (연속 다운인 서비스만)
    CONSECUTIVE=""
    for svc in $DOWN_LIST; do
      echo "$PREV_DOWN" | grep -q "$svc" && CONSECUTIVE="$CONSECUTIVE $svc"
    done
    if [ -n "$CONSECUTIVE" ]; then
      # 알림 발송 (admin API 호출)
      wget -qO- --post-data="{\"services\":\"$CONSECUTIVE\",\"detectedAt\":\"$COLLECTED_AT\"}" \
        --header="Content-Type: application/json" \
        --header="X-Alert-Secret: $ALERT_SECRET" \
        "http://${ADMIN_HOST}:3001/api/system/monitoring/alert" 2>/dev/null || true
    fi
  fi
  echo "$DOWN_LIST" > "$ALERT_FLAG"
else
  rm -f /tmp/metrics_alert_sent
fi
```

**Step 2: docker-compose.yml에 metrics-collector 추가**

```yaml
  metrics-collector:
    image: alpine:3.19
    container_name: metrics-collector
    restart: unless-stopped
    env_file: .env
    entrypoint: /bin/sh
    command:
      - -c
      - |
        apk add --no-cache mysql-client docker-cli
        chmod +x /scripts/collect.sh
        echo "*/5 * * * * /scripts/collect.sh" | crontab -
        crond -f
    volumes:
      - ./metrics-collector/collect.sh:/scripts/collect.sh
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DB_NAME=into-the-heaven
      - ADMIN_HOST=into-the-heaven-admin
```

**Step 3: .env.example 업데이트**

```
# 기존 내용 + 아래 추가
ALERT_SECRET=
```

> "ops/.env에 ALERT_SECRET 값을 설정해주세요 (장애 알림 API 인증용, 아무 랜덤 문자열). admin의 .env.local에도 동일한 ALERT_SECRET을 추가해야 합니다."

**Step 4: Commit**

```bash
git add ops/docker-compose.yml ops/metrics-collector/collect.sh ops/.env.example
git commit -m "feat: add metrics collector container with alert support"
```

---

## Task 6: 장애 알림 API (admin)

**Files:**
- Create: `admin/src/app/api/system/monitoring/alert/route.ts`

**Step 1: 알림 API 구현**

이 API는 metrics-collector에서 호출하여 Gmail로 알림 이메일을 발송.

```typescript
// admin/src/app/api/system/monitoring/alert/route.ts
import { NextRequest, NextResponse } from "next/server";
import nodemailer from "nodemailer";

export async function POST(request: NextRequest) {
  try {
    // ALERT_SECRET 검증
    const secret = request.headers.get("X-Alert-Secret");
    if (secret !== process.env.ALERT_SECRET) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const body = await request.json();
    const { services, detectedAt } = body;

    // Gmail SMTP로 알림 발송
    const transporter = nodemailer.createTransport({
      host: "smtp.gmail.com",
      port: 587,
      secure: false,
      auth: {
        user: process.env.ALERT_EMAIL_USER,
        pass: process.env.ALERT_EMAIL_PASSWORD,
      },
    });

    await transporter.sendMail({
      from: process.env.ALERT_EMAIL_USER,
      to: process.env.ALERT_EMAIL_TO,
      subject: `[IntoTheHeaven] 서비스 장애 감지: ${services}`,
      html: `
        <h2>서비스 장애 알림</h2>
        <p><strong>다운된 서비스:</strong> ${services}</p>
        <p><strong>감지 시각:</strong> ${detectedAt}</p>
        <p>맥미니 서버를 확인해주세요.</p>
      `,
    });

    return NextResponse.json({ success: true });
  } catch (error) {
    console.error("Alert sending failed:", error);
    return NextResponse.json({ error: "알림 발송 실패" }, { status: 500 });
  }
}
```

**Step 2: nodemailer 설치**

Run: `cd admin && npm install nodemailer && npm install -D @types/nodemailer`

**Step 3: 환경변수 안내**

> "admin의 .env.local (또는 Doppler)에 아래 환경변수를 추가해주세요:
> - `ALERT_SECRET` = ops/.env의 ALERT_SECRET과 동일한 값
> - `ALERT_EMAIL_USER` = Gmail 주소
> - `ALERT_EMAIL_PASSWORD` = Gmail 앱 비밀번호
> - `ALERT_EMAIL_TO` = 알림 받을 이메일 주소"

**Step 4: Commit**

```bash
git add admin/src/app/api/system/monitoring/alert/route.ts admin/package.json admin/package-lock.json
git commit -m "feat: add service down alert API with Gmail notification"
```

---

## Task 7: 모니터링 API 라우트 (admin)

**Files:**
- Create: `admin/src/app/api/system/monitoring/route.ts`

**Step 1: 통합 모니터링 API 구현**

```typescript
// admin/src/app/api/system/monitoring/route.ts
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/db";
import { requireSuperAdmin } from "@/lib/require-super-admin";

/**
 * GET /api/system/monitoring
 * 모니터링 대시보드 전체 데이터 조회
 * Query: range (1h | 6h | 24h | 7d | 30d, default: 24h)
 */
export async function GET(request: NextRequest) {
  const auth = await requireSuperAdmin();
  if (!auth.ok) return auth.response;

  try {
    const { searchParams } = new URL(request.url);
    const range = searchParams.get("range") || "24h";

    // 시간 범위 계산
    const now = new Date();
    const rangeMap: Record<string, number> = {
      "1h": 1 * 60 * 60 * 1000,
      "6h": 6 * 60 * 60 * 1000,
      "24h": 24 * 60 * 60 * 1000,
      "7d": 7 * 24 * 60 * 60 * 1000,
      "30d": 30 * 24 * 60 * 60 * 1000,
    };
    const since = new Date(now.getTime() - (rangeMap[range] || rangeMap["24h"]));

    // 1. 최신 메트릭 (컨테이너별 현재 상태)
    const latestMetrics = await prisma.$queryRaw`
      SELECT m.* FROM metrics m
      INNER JOIN (
        SELECT container_name, MAX(collected_at) as max_at
        FROM metrics
        GROUP BY container_name
      ) latest ON m.container_name = latest.container_name AND m.collected_at = latest.max_at
    ` as Array<Record<string, unknown>>;

    // 2. 시간별 메트릭 추이
    const metricsHistory = await prisma.metrics.findMany({
      where: { collected_at: { gte: since } },
      orderBy: { collected_at: "asc" },
    });

    // 3. 앱 활동 지표
    const weekStart = new Date(now);
    weekStart.setDate(now.getDate() - now.getDay());
    weekStart.setHours(0, 0, 0, 0);

    const [weeklyGatherings, weeklyPrayers, totalPrayers, recentSignups] = await Promise.all([
      // 이번 주 모임 생성 수
      prisma.gathering.count({
        where: {
          deleted_at: null,
          created_at: { gte: weekStart },
        },
      }),
      // 이번 주 기도제목 등록 수
      prisma.prayer.count({
        where: {
          deleted_at: null,
          created_at: { gte: weekStart },
        },
      }),
      // 전체 기도제목 수
      prisma.prayer.count({
        where: { deleted_at: null },
      }),
      // 최근 30일 신규 가입자 (일별)
      prisma.$queryRaw`
        SELECT DATE(created_at) as date, COUNT(*) as count
        FROM member
        WHERE deleted_at IS NULL
          AND created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        GROUP BY DATE(created_at)
        ORDER BY date ASC
      ` as Promise<Array<{ date: string; count: number }>>,
    ]);

    // 4. Umami 데이터 (별도 fetch)
    let umamiData = null;
    const umamiUrl = process.env.UMAMI_API_URL;
    const umamiToken = process.env.UMAMI_API_TOKEN;
    const umamiWebAppId = process.env.UMAMI_WEBAPP_WEBSITE_ID;
    const umamiAdminId = process.env.UMAMI_ADMIN_WEBSITE_ID;

    if (umamiUrl && umamiToken) {
      const headers = { Authorization: `Bearer ${umamiToken}`, "Content-Type": "application/json" };
      const startAt = since.getTime();
      const endAt = now.getTime();

      try {
        const [webAppStats, adminStats, webAppActive, adminActive] = await Promise.all([
          umamiWebAppId
            ? fetch(`${umamiUrl}/api/websites/${umamiWebAppId}/stats?startAt=${startAt}&endAt=${endAt}`, { headers }).then(r => r.json())
            : null,
          umamiAdminId
            ? fetch(`${umamiUrl}/api/websites/${umamiAdminId}/stats?startAt=${startAt}&endAt=${endAt}`, { headers }).then(r => r.json())
            : null,
          umamiWebAppId
            ? fetch(`${umamiUrl}/api/websites/${umamiWebAppId}/active`, { headers }).then(r => r.json())
            : null,
          umamiAdminId
            ? fetch(`${umamiUrl}/api/websites/${umamiAdminId}/active`, { headers }).then(r => r.json())
            : null,
        ]);

        umamiData = {
          webApp: { stats: webAppStats, active: webAppActive },
          admin: { stats: adminStats, active: adminActive },
        };
      } catch (e) {
        console.error("Umami API error:", e);
      }
    }

    return NextResponse.json({
      success: true,
      data: {
        containers: latestMetrics,
        history: metricsHistory,
        activity: {
          weeklyGatherings,
          weeklyPrayers,
          totalPrayers,
          recentSignups,
        },
        umami: umamiData,
      },
    });
  } catch (error) {
    console.error("Monitoring API error:", error);
    return NextResponse.json({ error: "모니터링 데이터 조회 실패" }, { status: 500 });
  }
}
```

**Step 2: 환경변수 안내**

> "admin의 .env.local (또는 Doppler)에 아래 환경변수를 추가해주세요:
> - `UMAMI_API_URL` = Umami 서버 내부 URL (예: http://umami:3000 또는 http://맥미니IP:3002)
> - `UMAMI_API_TOKEN` = Umami API 토큰 (Umami 설정에서 생성)
> - `UMAMI_WEBAPP_WEBSITE_ID` = web-app website ID
> - `UMAMI_ADMIN_WEBSITE_ID` = admin website ID"

**Step 3: Commit**

```bash
git add admin/src/app/api/system/monitoring/route.ts
git commit -m "feat: add monitoring API with metrics, activity, and Umami data"
```

---

## Task 8: 모니터링 UI 페이지 (admin)

**Files:**
- Create: `admin/src/app/system/monitoring/page.tsx`
- Create: `admin/src/app/system/monitoring/monitoring-client.tsx`
- Modify: `admin/src/components/sidebar.tsx` (사이드바에 메뉴 추가)

**Step 1: recharts 설치**

Run: `cd admin && npm install recharts`

**Step 2: 사이드바에 모니터링 메뉴 추가**

`admin/src/components/sidebar.tsx` 의 System Admin Menu 섹션에 모니터링 링크 추가.
기존 `{/* System Admin Menu */}` 블록 안, "교회 관리" 링크 아래에 추가:

```tsx
// import에 Activity 아이콘 추가
import { ..., Activity } from "lucide-react";

// System Admin Menu 블록 안, 교회 관리 Link 아래에 추가:
          <Link
            href="/system/monitoring"
            onClick={onNavigate}
            className={cn(
              "flex items-center gap-3 rounded-lg px-3 py-2.5 text-sm font-medium transition-colors",
              pathname.startsWith("/system/monitoring")
                ? "bg-indigo-600 text-white"
                : "text-slate-300 hover:bg-slate-800 hover:text-white"
            )}
          >
            <Activity className="h-5 w-5" />
            모니터링
          </Link>
```

**Step 3: page.tsx 생성**

```tsx
// admin/src/app/system/monitoring/page.tsx
import { AdminLayout } from "@/components/admin-layout";
import { MonitoringClient } from "./monitoring-client";

export default function MonitoringPage() {
  return (
    <AdminLayout>
      <MonitoringClient />
    </AdminLayout>
  );
}
```

**Step 4: monitoring-client.tsx 생성**

이 파일이 가장 크므로 주요 섹션별로 구현:

- 헤더 + 시간 범위 셀렉터
- 서비스 상태 카드 (컨테이너별 running/stopped, CPU, 메모리)
- 앱 활동 지표 카드 (모임, 기도제목, 가입자)
- Umami 방문자 카드 (web-app, admin 각각)
- CPU/메모리 추이 차트 (recharts LineChart)
- API 응답시간/에러율 차트
- MySQL 커넥션/슬로우 쿼리 차트
- 신규 가입자 추이 차트

> 이 파일은 크기가 크므로 구현 시 실제 코드를 작성. 기존 dashboard-client.tsx의 패턴 (Card, CardHeader, CardContent, Loader2, Select 등)을 따름.

**Step 5: Commit**

```bash
git add admin/src/app/system/monitoring/ admin/src/components/sidebar.tsx admin/package.json admin/package-lock.json
git commit -m "feat: add monitoring dashboard page with charts and status cards"
```

---

## Task 순서 요약

| Task | 내용 | 의존성 |
|------|------|--------|
| 1 | Umami Docker 셋업 | 없음 |
| 2 | Umami 스크립트 삽입 | Task 1 (website ID 필요) |
| 3 | metrics 테이블 + Prisma | 없음 |
| 4 | Backend Actuator | 없음 |
| 5 | 메트릭 수집 컨테이너 | Task 3, 4 |
| 6 | 장애 알림 API | 없음 |
| 7 | 모니터링 API | Task 3 |
| 8 | 모니터링 UI | Task 7 |

**병렬 가능:** Task 1, 3, 4, 6은 독립적이라 동시 진행 가능.
