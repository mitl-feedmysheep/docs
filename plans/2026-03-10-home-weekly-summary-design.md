# 홈 화면 "이번 주" 요약 카드 기능

## Context

현재 홈 화면은 인사말, 공지사항, 생일, 캘린더로 구성되어 있고 소그룹 활동 관련 콘텐츠가 없다. 유저가 속한 그룹들의 최신 모임에서 한주 목표와 기도제목을 모아 홈 화면 상단에 보여주면 앱의 실용성이 크게 향상된다.

## 디자인 결정사항

- **데이터 범위**: 유저의 모든 그룹에서 각 그룹의 최신 모임의 goal/prayers 수집, null 제외
- **UI 위치**: 인사말 subtitle 자리를 대체 (데이터 없으면 기존 인사말 폴백)
- **UI 형태**: 한 카드에 "목표"와 "기도제목" 섹션을 불릿포인트로 표시, 그룹명 배지 포함
- **API 전략**: 백엔드 전용 엔드포인트 `GET /members/me/home-summary` 신규 생성 (2 쿼리로 해결)
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

### Step 3: 백엔드 — GatheringPort 확장 + Repository 쿼리

**수정**: `application/port/out/GatheringPort.java`
```java
// 추가
List<Gathering> findLatestGatheringsByGroupIds(List<UUID> groupIds);
```

**수정**: `adapter/out/persistence/repository/GatheringJpaRepository.java`
- JPQL 서브쿼리로 그룹별 최신(date 기준) gathering 조회
- `@EntityGraph`로 gatheringMembers, groupMember, member, prayers 일괄 로딩

```java
@Query("SELECT g FROM GatheringJpaEntity g " +
       "WHERE g.id IN (" +
       "  SELECT g2.id FROM GatheringJpaEntity g2 " +
       "  WHERE g2.group.id IN :groupIds " +
       "  AND g2.date = (SELECT MAX(g3.date) FROM GatheringJpaEntity g3 WHERE g3.group.id = g2.group.id)" +
       ")")
@EntityGraph(attributePaths = {
    "group",
    "gatheringMembers",
    "gatheringMembers.groupMember",
    "gatheringMembers.groupMember.member",
    "gatheringMembers.prayers"
})
List<GatheringJpaEntity> findLatestByGroupIds(@Param("groupIds") List<UUID> groupIds);
```

**수정**: `adapter/out/persistence/GatheringPersistenceAdapter.java`
- `findLatestGatheringsByGroupIds` 구현 (mapper 통해 도메인 변환)

### Step 4: 백엔드 — HomeQueryService

새 파일: `application/service/query/HomeQueryService.java`

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class HomeQueryService implements HomeQueryUseCase {

    private final GroupPort groupPort;
    private final GatheringPort gatheringPort;

    @Override
    public HomeSummaryResponse getHomeSummary(MemberId memberId) {
        // 1. 소속 그룹 조회 (1 쿼리)
        List<Group> groups = groupPort.findGroupsByMemberId(memberId.getValue());
        if (groups.isEmpty()) return HomeSummaryResponse.builder().build();

        // 2. 그룹별 최신 모임 조회 (1 쿼리)
        List<UUID> groupIds = groups.stream().map(g -> g.getId().getValue()).toList();
        Map<UUID, String> groupNameMap = groups.stream()
                .collect(Collectors.toMap(g -> g.getId().getValue(), Group::getName));
        List<Gathering> latestGatherings = gatheringPort.findLatestGatheringsByGroupIds(groupIds);

        // 3. 같은 날짜 tie → startedAt 최신으로 선택
        Map<UUID, Gathering> latestPerGroup = latestGatherings.stream()
                .collect(Collectors.toMap(
                    g -> g.getGroup().getId().getValue(), g -> g,
                    (g1, g2) -> g1.getStartedAt().isAfter(g2.getStartedAt()) ? g1 : g2));

        // 4. 각 모임에서 해당 memberId의 GatheringMember → goal/prayers 수집
        List<HomeGoalResponse> goals = new ArrayList<>();
        List<HomePrayerResponse> prayers = new ArrayList<>();

        for (var entry : latestPerGroup.entrySet()) {
            String groupName = groupNameMap.get(entry.getKey());
            entry.getValue().getGatheringMembers().stream()
                .filter(gm -> gm.getGroupMember() != null
                        && gm.getGroupMember().getMember() != null
                        && gm.getGroupMember().getMember().getId().getValue().equals(memberId.getValue()))
                .findFirst()
                .ifPresent(gm -> {
                    if (gm.getGoal() != null && !gm.getGoal().isBlank())
                        goals.add(HomeGoalResponse.builder().groupName(groupName).goal(gm.getGoal()).build());
                    if (gm.getPrayers() != null)
                        gm.getPrayers().stream()
                            .filter(p -> p.getPrayerRequest() != null && !p.getPrayerRequest().isBlank())
                            .forEach(p -> prayers.add(HomePrayerResponse.builder()
                                .groupName(groupName).prayerRequest(p.getPrayerRequest())
                                .description(p.getDescription()).isAnswered(p.isAnswered()).build()));
                });
        }

        return HomeSummaryResponse.builder().goals(goals).prayers(prayers).build();
    }
}
```

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
- shadcn/ui의 `Card`, `CardHeader`, `CardTitle`, `CardContent` 사용
- `Badge variant="secondary"` 로 그룹명 표시
- 응답된 기도제목(`isAnswered: true`) 필터링
- 목표/기도제목 중 한쪽만 있으면 해당 섹션만 표시
- 둘 다 있으면 `Separator`로 구분
- 모두 비어있으면 `null` 반환
- lucide-react `Target` 아이콘 사용

### Step 9: 프론트엔드 — HomePage 수정

**수정**: `web-app/src/features/home/HomePage.tsx`

- `homeSummary` state 추가 (`useState<HomeSummary | null>(null)`)
- `useEffect` 내 `membersApi.getHomeSummary()` 호출 (에러 시 null, try/catch)
- 인사말 subtitle (`<p>오늘도 하나님의 은혜...`) 자리를 조건부 렌더링:
  - 데이터 있으면 → `<WeeklySummaryCard />`
  - 없으면 → 기존 인사말 텍스트
- 인사말 제목("~님, 반가워요!")과 액션 버튼(알림/메시지)은 항상 유지

---

## 인덱스 확인

`gathering` 테이블에 `(group_id, date)` 복합 인덱스 존재 여부 확인 필요:

```sql
SHOW INDEX FROM gathering WHERE Column_name IN ('group_id', 'date');
```

없으면 추가 권장: `CREATE INDEX idx_gathering_group_id_date ON gathering(group_id, date);`

## 검증 방법

1. **백엔드**: Swagger에서 `GET /members/me/home-summary` 호출, 인증 토큰으로 테스트
2. **프론트엔드**: `npm run dev` 후 홈 화면 확인
   - 모임에 goal/prayer 데이터가 있는 계정: 카드 표시 확인
   - 데이터 없는 계정: 인사말 폴백 확인
3. **DB 직접 확인**: 해당 유저의 최신 모임 데이터 존재 여부 쿼리

## 수정 파일 요약

| # | 파일 | 변경 |
|---|------|------|
| 1 | `backend/.../dto/home/HomeSummaryResponse.java` | **신규** |
| 2 | `backend/.../dto/home/HomeGoalResponse.java` | **신규** |
| 3 | `backend/.../dto/home/HomePrayerResponse.java` | **신규** |
| 4 | `backend/.../port/in/query/HomeQueryUseCase.java` | **신규** |
| 5 | `backend/.../service/query/HomeQueryService.java` | **신규** |
| 6 | `backend/.../port/out/GatheringPort.java` | 메서드 추가 |
| 7 | `backend/.../repository/GatheringJpaRepository.java` | 쿼리 추가 |
| 8 | `backend/.../GatheringPersistenceAdapter.java` | 구현 추가 |
| 9 | `backend/.../controller/MemberController.java` | 엔드포인트 추가 |
| 10 | `web-app/src/types/index.ts` | 타입 추가 |
| 11 | `web-app/src/lib/api.ts` | API 함수 추가 |
| 12 | `web-app/src/features/home/WeeklySummaryCard.tsx` | **신규** |
| 13 | `web-app/src/features/home/HomePage.tsx` | 조건부 렌더링 |
