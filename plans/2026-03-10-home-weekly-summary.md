# 홈 화면 "이번 주" 요약 카드 기능

## Context

현재 홈 화면은 인사말, 공지사항, 생일, 캘린더로 구성되어 있고 소그룹 활동 관련 콘텐츠가 없다. 유저가 속한 그룹들의 최근 7일 이내 모임에서 한주 목표와 기도제목을 모아 홈 화면 상단에 보여준다.

## 디자인 결정사항

- **데이터 범위**: 유저의 모든 그룹에서 **지난 7일 이내** 모임의 goal/prayers 수집, null 제외
- **UI 위치**: 인사말 subtitle 자리를 대체 (데이터 없으면 기존 인사말 폴백)
- **UI 형태**: 한 카드에 "목표"와 "기도제목" 섹션을 불릿포인트로 표시, 그룹명 배지 포함
- **API 전략**: 백엔드 전용 엔드포인트 `GET /members/me/home-summary` (2 쿼리)
- **쿼리 전략**: EntityGraph 대신 **해당 유저의 gathering_member만 직접 조회** (over-fetching 방지)
- **응답된 기도제목**: `isAnswered: true`인 것은 프론트에서 필터링하여 미표시

### UI 목업

```
┌──────────────────────────────┐
│  🎯 이번 주                       │
│──────────────────────────────│
│  목표                            │
│  • 매일 QT 30분 [청년셀]       │
│  • 전도 1회 [새가족부]          │
│                                │
│  기도제목                        │
│  • 취직 준비 [청년셀]          │
│  • 건강 회복 [청년셀]          │
│  • 신앙 성장 [새가족부]       │
└──────────────────────────────┘
```

---

## 구현 계획

### Step 1: 백엔드 — Response DTO 생성

새 파일 3개 (base: `backend/src/main/java/mitl/IntoTheHeaven/adapter/in/web/dto/home/`):

**HomeSummaryResponse.java**
```java
@Getter @Builder
public class HomeSummaryResponse {
    @Builder.Default
    private final List<HomeGoalResponse> goals = new ArrayList<>();
    @Builder.Default
    private final List<HomePrayerResponse> prayers = new ArrayList<>();
}
```

**HomeGoalResponse.java**
```java
@Getter @Builder
public class HomeGoalResponse {
    private final String groupName;
    private final String goal;
}
```

**HomePrayerResponse.java**
```java
@Getter @Builder
public class HomePrayerResponse {
    private final String groupName;
    private final String prayerRequest;
    private final String description;
    private final boolean isAnswered;
}
```

### Step 2: 백엔드 — UseCase 인터페이스

새 파일: `application/port/in/query/HomeQueryUseCase.java`

```java
public interface HomeQueryUseCase {
    HomeSummaryResponse getHomeSummary(MemberId memberId);
}
```

### Step 3: 백엔드 — Port + Repository (직접 조회 방식)

#### 3a. 새 DTO: `HomeSummaryData`

새 파일: `application/port/out/HomeSummaryData.java`

gathering_member 조인 쿼리의 projection 결과를 담는 DTO:
```java
@Getter @AllArgsConstructor
public class HomeSummaryData {
    private final UUID gatheringMemberId;
    private final String goal;
    private final LocalDate gatheringDate;
    private final String groupName;
}
```

#### 3b. GatheringPort 확장

**수정**: `application/port/out/GatheringPort.java`
```java
// 추가: 해당 멤버의 지난 7일 이내 모임 데이터 조회
List<HomeSummaryData> findRecentGatheringMemberData(UUID memberId, LocalDate since);
```

#### 3c. Repository 쿼리

**수정**: `adapter/out/persistence/repository/GatheringJpaRepository.java`

```sql
-- 쿼리 1: 해당 유저의 지난 7일 모임의 gathering_member 데이터
SELECT gm.id, gm.goal, g.date, grp.name
FROM gathering_member gm
JOIN gathering g ON gm.gathering_id = g.id
JOIN group_member grpm ON gm.group_member_id = grpm.id
JOIN `group` grp ON g.group_id = grp.id
WHERE grpm.member_id = :memberId
  AND g.date >= :since
  AND g.deleted_at IS NULL AND gm.deleted_at IS NULL AND grpm.deleted_at IS NULL;
```

JPQL `new` 생성자 표현식 또는 interface projection 사용.

#### 3d. PrayerPort 확장

**수정**: `application/port/out/PrayerPort.java`
```java
// 추가
List<Prayer> findByGatheringMemberIds(List<UUID> gatheringMemberIds);
```

**수정**: `adapter/out/persistence/repository/PrayerJpaRepository.java`
```sql
-- 쿼리 2: gathering_member_id 목록으로 기도제목 조회
SELECT p FROM PrayerJpaEntity p
WHERE p.gatheringMemberId IN :gatheringMemberIds AND p.deletedAt IS NULL
```

#### 3e. Persistence Adapter 구현

**수정**: `adapter/out/persistence/GatheringPersistenceAdapter.java` — `findRecentGatheringMemberData` 구현
**수정**: `adapter/out/persistence/PrayerPersistenceAdapter.java` — `findByGatheringMemberIds` 구현

### Step 4: 백엔드 — HomeQueryService

새 파일: `application/service/query/HomeQueryService.java`

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class HomeQueryService implements HomeQueryUseCase {
    private final GatheringPort gatheringPort;
    private final PrayerPort prayerPort;

    @Override
    public HomeSummaryResponse getHomeSummary(MemberId memberId) {
        LocalDate since = LocalDate.now().minusDays(7);

        // 쿼리 1: 해당 유저의 최근 7일 gathering_member 데이터
        List<HomeSummaryData> data = gatheringPort.findRecentGatheringMemberData(
            memberId.getValue(), since);
        if (data.isEmpty()) return HomeSummaryResponse.builder().build();

        // goals 수집 (null/blank 제외)
        List<HomeGoalResponse> goals = data.stream()
            .filter(d -> d.getGoal() != null && !d.getGoal().isBlank())
            .map(d -> HomeGoalResponse.builder()
                .groupName(d.getGroupName()).goal(d.getGoal()).build())
            .toList();

        // 쿼리 2: 기도제목 조회
        List<UUID> gmIds = data.stream().map(HomeSummaryData::getGatheringMemberId).toList();
        Map<UUID, String> gmIdToGroupName = data.stream()
            .collect(Collectors.toMap(HomeSummaryData::getGatheringMemberId, HomeSummaryData::getGroupName));

        List<Prayer> prayers = prayerPort.findByGatheringMemberIds(gmIds);
        List<HomePrayerResponse> prayerResponses = prayers.stream()
            .filter(p -> p.getPrayerRequest() != null && !p.getPrayerRequest().isBlank())
            .map(p -> HomePrayerResponse.builder()
                .groupName(gmIdToGroupName.get(p.getGatheringMemberId().getValue()))
                .prayerRequest(p.getPrayerRequest())
                .description(p.getDescription())
                .isAnswered(p.isAnswered())
                .build())
            .toList();

        return HomeSummaryResponse.builder().goals(goals).prayers(prayerResponses).build();
    }
}
```

총 **2 쿼리**, 해당 유저 데이터만 조회.

### Step 5: 백엔드 — MemberController에 엔드포인트 추가

**수정**: `adapter/in/web/controller/MemberController.java`
- `HomeQueryUseCase` 주입
- 엔드포인트 추가:

```java
@Operation(summary = "Get Home Summary")
@GetMapping("/me/home-summary")
public ResponseEntity<HomeSummaryResponse> getHomeSummary(@AuthenticationPrincipal String memberId) {
    HomeSummaryResponse response = homeQueryUseCase.getHomeSummary(MemberId.from(UUID.fromString(memberId)));
    return ResponseEntity.ok(response);
}
```

### Step 6: 프론트엔드 — 타입 추가

**수정**: `web-app/src/types/index.ts`

```typescript
export interface HomeSummaryGoal {
  groupName: string;
  goal: string;
}

export interface HomeSummaryPrayer {
  groupName: string;
  prayerRequest: string;
  description: string;
  isAnswered: boolean;
}

export interface HomeSummary {
  goals: HomeSummaryGoal[];
  prayers: HomeSummaryPrayer[];
}
```

### Step 7: 프론트엔드 — API 함수 추가

**수정**: `web-app/src/lib/api.ts`
- `membersApi`에 추가:

```typescript
getHomeSummary: () => authedFetch<HomeSummary>("/members/me/home-summary"),
```

### Step 8: 프론트엔드 — WeeklySummaryCard 컴포넌트

새 파일: `web-app/src/features/home/WeeklySummaryCard.tsx`

- Props: `{ goals: HomeSummaryGoal[], prayers: HomeSummaryPrayer[] }`
- shadcn/ui `Card`, `CardHeader`, `CardTitle`, `CardContent`, `Badge`, `Separator` 사용
- lucide-react `Target` 아이콘
- 응답된 기도제목(`isAnswered: true`) 프론트에서 필터링
- 목표/기도제목 중 한쪽만 있으면 해당 섹션만 표시, 둘 다 있으면 Separator 구분
- 모두 비어있으면 null 반환

### Step 9: 프론트엔드 — HomePage 수정

**수정**: `web-app/src/features/home/HomePage.tsx`
- `homeSummary` state 추가 (`useState<HomeSummary | null>(null)`)
- `useEffect` load 함수 내 `membersApi.getHomeSummary()` 호출 (에러 시 null, try/catch)
- 119-121번 줄의 인사말 subtitle을 조건부 렌더링:
  - 데이터 있으면 → `<WeeklySummaryCard />`
  - 없으면 → 기존 `<p>오늘도 하나님의 은혜 안에서 좋은 하루 되세요</p>`
- 인사말 제목("~님, 반가워요!")과 액션 버튼은 항상 유지

---

## 인덱스 확인

```sql
SHOW INDEX FROM gathering WHERE Column_name IN ('group_id', 'date');
SHOW INDEX FROM gathering_member WHERE Column_name IN ('group_member_id', 'gathering_id');
SHOW INDEX FROM prayer WHERE Column_name = 'gathering_member_id';
```

## 검증 방법

1. **백엔드**: Swagger에서 `GET /members/me/home-summary` 호출, 인증 토큰으로 테스트
2. **프론트엔드**: `npm run dev` 후 홈 화면 확인
   - 지난 7일 내 모임에 goal/prayer 있는 계정: 카드 표시
   - 데이터 없는 계정: 인사말 폴백
3. **DB**: `SELECT * FROM gathering WHERE date >= CURDATE() - INTERVAL 7 DAY` 로 대상 모임 확인

## 수정 파일 요약

| # | 파일 | 변경 |
|---|------|------|
| 1 | `backend/.../dto/home/HomeSummaryResponse.java` | **신규** |
| 2 | `backend/.../dto/home/HomeGoalResponse.java` | **신규** |
| 3 | `backend/.../dto/home/HomePrayerResponse.java` | **신규** |
| 4 | `backend/.../port/in/query/HomeQueryUseCase.java` | **신규** |
| 5 | `backend/.../port/out/HomeSummaryData.java` | **신규** |
| 6 | `backend/.../service/query/HomeQueryService.java` | **신규** |
| 7 | `backend/.../port/out/GatheringPort.java` | 메서드 추가 |
| 8 | `backend/.../port/out/PrayerPort.java` | 메서드 추가 |
| 9 | `backend/.../repository/GatheringJpaRepository.java` | 쿼리 추가 |
| 10 | `backend/.../repository/PrayerJpaRepository.java` | 쿼리 추가 |
| 11 | `backend/.../GatheringPersistenceAdapter.java` | 구현 추가 |
| 12 | `backend/.../PrayerPersistenceAdapter.java` | 구현 추가 |
| 13 | `backend/.../controller/MemberController.java` | 엔드포인트 추가 |
| 14 | `web-app/src/types/index.ts` | 타입 추가 |
| 15 | `web-app/src/lib/api.ts` | API 함수 추가 |
| 16 | `web-app/src/features/home/WeeklySummaryCard.tsx` | **신규** |
| 17 | `web-app/src/features/home/HomePage.tsx` | 조건부 렌더링 |
