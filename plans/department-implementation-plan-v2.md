# Department 도입 구현 계획 v2 (최종)

> 기반 문서: `department-implementation-plan.md` (v1)
> 본 문서는 v1 이후 논의된 권한 모델 재설계, 부서 전환 UI, 가입 플로우 변경 등을 반영한 최종 버전.

## 배경

현재 church → group 2단계 구조. 같은 교회 내 부서(청년부, 신혼부부부, 장년부 등)를 구분할 수 없어 이벤트, 생일, 리더 목록이 전부 섞임.

**변경 후**: church → department → group → gathering (3단계)

---

## v1 대비 변경사항

| 항목 | v1 | v2 |
|------|----|----|
| department_member FK | `church_member_id` → church_member | `member_id` → member |
| visit.department_id | 추가 (NOT NULL) | **제거** — visit_member→member→department_member로 자연 격리 |
| 기본 부서 | 교회 생성 시 "전체" 자동 생성 (`is_default=true`) | **제거** — 기본 부서 없음 |
| ChurchRole | MEMBER/LEADER/ADMIN/SUPER_ADMIN | **LEADER 제거** → MEMBER/ADMIN/SUPER_ADMIN |
| 민감정보 조회 | church LEADER+ | group LEADER+ OR dept LEADER+ OR church ADMIN+ |
| Admin 포털 접근 | church ADMIN+ | **dept LEADER+** OR church ADMIN+ |
| 심방/기도제목 | church SUPER_ADMIN 전용 | **dept ADMIN+** OR church SUPER_ADMIN |
| Web-app 홈 화면 | 소속 부서 합산 + [부서] 태그 | **단일 부서 뷰 + 부서 전환** |
| 부서 전환 UI | 불필요 | Admin: 사이드바 부서 전환, Web-app: MY 부서 전환 |
| 가입 신청 | 관리자가 승인 시 부서 배정 | **유저가 교회+부서 선택하여 신청** (부서 nullable) |
| 배포 전략 | 한번에 마이그레이션 | **expand-and-contract** (무중단) |

---

## 권한 모델 (3-tier)

### 역할 정의

**church_member.role** — 교회 전체 범위
- `SUPER_ADMIN(3)`: 담임목사 → 모든 부서 접근
- `ADMIN(2)`: 교회 운영진 (사무장 등) → 시스템 설정
- `MEMBER(1)`: 일반 교인

**department_member.role** — 부서 범위
- `ADMIN(3)`: 부서 목사 → 심방/기도제목 + 소그룹 관리 + admin 포털
- `LEADER(2)`: 부서 회장/임원 → 소그룹 관리 + admin 포털
- `MEMBER(1)`: 일반 부서원

**group_member.role** — 소그룹 범위 (기존 유지)
- `LEADER`: 셀리더 → 민감정보 조회
- `SUB_LEADER`: 서브리더
- `MEMBER`: 일반

### 권한 매트릭스

| 권한 | church SUPER_ADMIN | church ADMIN | dept ADMIN | dept LEADER | group LEADER | group SUB_LEADER | 나머지 |
|---|---|---|---|---|---|---|---|
| **모든 부서 접근** | O | - | - | - | - | - | - |
| **시스템 설정** | O | O | - | - | - | - | - |
| **Admin 포털 접근** | O | O | O | O | - | - | - |
| **심방/기도제목 관리** | O | - | O | - | - | - | - |
| **소그룹 관리** | O | - | O | O | - | - | - |
| **민감정보 조회** | O | O | O | O | O | - | - |
| **일반 기능** | O | O | O | O | O | O | O |

**범위 규칙**:
- `church SUPER_ADMIN`: 교회 내 **전체 부서** 대상 (department_member 없이)
- `church ADMIN`: 시스템 설정 가능, 부서 기능은 본인 department_member에 따라
- `dept ADMIN/LEADER`: **소속 부서만** 대상
- `group LEADER`: **소속 그룹** 멤버의 민감정보만

### 페르소나별 역할 설정

| 페르소나 | 예시 | church_member.role | department_member | group_member |
|---|---|---|---|---|
| **담임목사** | 김목사님 | SUPER_ADMIN | 없음 (전체 접근) | 없음 |
| **사무장** | 박집사님 | ADMIN | 청년부 MEMBER | 없음 |
| **부서 목사** | 이목사님 (청년부) | MEMBER | 청년부 ADMIN | 없음 |
| **부서 회장** | 최회장님 (청년부) | MEMBER | 청년부 LEADER | 1셀 MEMBER |
| **셀리더** | 정리더님 (청년부 1셀) | MEMBER | 청년부 MEMBER | 1셀 LEADER |
| **서브리더** | 한서브님 (청년부 1셀) | MEMBER | 청년부 MEMBER | 1셀 SUB_LEADER |
| **일반 교인** | 김성도님 (청년부 2셀) | MEMBER | 청년부 MEMBER | 2셀 MEMBER |
| **미배정 새가족** | 박새가족님 | MEMBER | 없음 | 없음 |
| **다중 부서 목사** | 송목사님 | MEMBER | 청년부 ADMIN, 신혼부 ADMIN | 없음 |
| **다중 부서 교인** | 윤집사님 | MEMBER | 청년부 MEMBER, 찬양팀 MEMBER | 청년1셀 LEADER, 찬양1팀 MEMBER |

---

## 확정된 결정사항

| 항목 | 결정 |
|------|------|
| 계층 | church → department → group → gathering |
| DepartmentRole | MEMBER(1) / LEADER(2) / ADMIN(3) |
| ChurchRole (변경) | MEMBER(1) / ADMIN(2) / SUPER_ADMIN(3) — LEADER 제거 |
| department_member FK | `member_id` → member |
| 기본 부서 | 없음 — 마이그레이션 시 "전체" 부서 생성 후 관리자가 재배정 |
| 졸업 | department_member.status = GRADUATED (이력 보존) |
| 격리 대상 | 그룹, 모임, 출석, 이벤트, 생일, 멤버 검색 |
| 심방 | department_id 없음 — 현재 선택 부서의 멤버 기준 필터 |
| 격리 안 함 | notification, message, prayer (엔티티 체인으로 자연 격리) |
| Admin 부서 전환 | 사이드바에 교회 전환 아래 부서 드롭다운 |
| Web-app 부서 전환 | MY에서 부서 전환 (교회 전환과 동일 패턴) |
| 부서 전환 목록 | church SUPER_ADMIN: 모든 부서, 그 외: 본인 department_member 부서만 |
| 가입 신청 | 유저가 교회+부서 선택 (부서 nullable), 해당 부서 ADMIN이 승인 |
| 가입 요청 테이블 | church_member_request 유지 (department_id NULLABLE 추가) |
| 가입 요청 노출 | dept null → church SUPER_ADMIN만, dept 있음 → 해당 dept ADMIN + church SUPER_ADMIN |
| 부서 미배정 | 허용 → "부서 배정 대기 중" + 기능 제한 |
| 부서 삭제 | 멤버/그룹 0일 때만 가능 |
| 배포 전략 | expand-and-contract (NULLABLE 추가 → 코드 배포 → NOT NULL 전환) |

---

## Phase 1: DB 스키마 + Backend Department 도메인

### 1-1. 새 테이블

**`department`**
```sql
CREATE TABLE department (
  id CHAR(36) PRIMARY KEY,
  church_id CHAR(36) NOT NULL,
  name VARCHAR(50) NOT NULL,
  description VARCHAR(200),
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at DATETIME NULL,
  FOREIGN KEY (church_id) REFERENCES church(id),
  INDEX idx_department_church_deleted (church_id, deleted_at),
  UNIQUE INDEX uk_department_church_name (church_id, name, deleted_at)
);
```

**`department_member`**
```sql
CREATE TABLE department_member (
  id CHAR(36) PRIMARY KEY,
  department_id CHAR(36) NOT NULL,
  member_id CHAR(36) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'MEMBER',
  status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted_at DATETIME NULL,
  FOREIGN KEY (department_id) REFERENCES department(id),
  FOREIGN KEY (member_id) REFERENCES member(id),
  INDEX idx_dm_department_deleted (department_id, deleted_at),
  INDEX idx_dm_member (member_id),
  UNIQUE INDEX uk_dm_dept_member (department_id, member_id, deleted_at)
);
```

### 1-2. 기존 테이블 변경

| 테이블 | 변경 | NOT NULL? |
|--------|------|-----------|
| `group` | `department_id CHAR(36)` + FK + INDEX | 마이그레이션 후 NOT NULL |
| `event` | `department_id CHAR(36)` + INDEX | nullable (교회 전체 이벤트 가능) |
| `church_member_request` | `department_id CHAR(36)` + FK | nullable (부서 모를 때) |

**변경 안 함**: visit, notification, message, prayer

### 1-3. ChurchRole 변경

```sql
-- church_member.role 값 변경: LEADER → MEMBER로 다운그레이드
-- (기존 LEADER는 department_member.role로 이전)
UPDATE church_member SET role = 'MEMBER' WHERE role = 'LEADER' AND deleted_at IS NULL;
```

### 1-4. 데이터 마이그레이션

1. 각 교회에 "전체" department 생성
2. 모든 `church_member` → "전체" department로 `department_member` 생성
   - church SUPER_ADMIN → dept ADMIN
   - church ADMIN → dept MEMBER (시스템 설정 역할이므로)
   - church LEADER → dept LEADER (기존 LEADER 권한 이전)
   - church MEMBER → dept MEMBER
3. 모든 `group` → "전체" department의 `department_id` 배정
4. `group.department_id` NOT NULL로 변경

### 1-5. Backend 신규 파일 (Hexagonal 패턴)

| 레이어 | 파일 |
|--------|------|
| domain/enums | `DepartmentRole.java` — MEMBER(1), LEADER(2), ADMIN(3) + hasPermissionOver() |
| domain/enums | `DepartmentMemberStatus.java` — ACTIVE, GRADUATED |
| domain/enums | `EntityType.java`에 `DEPARTMENT` 추가 |
| domain/enums | `ChurchRole.java` — LEADER 제거, 레벨 재조정: MEMBER(1), ADMIN(2), SUPER_ADMIN(3) |
| domain/model | `Department.java` — AggregateRoot, churchId, name, description |
| domain/model | `DepartmentId.java` — BaseId |
| domain/model | `DepartmentMember.java` — DomainEntity, departmentId, memberId, role, status |
| domain/model | `DepartmentMemberId.java` — BaseId |
| adapter/out/persistence/entity | `DepartmentJpaEntity.java` — @SQLRestriction, church ManyToOne |
| adapter/out/persistence/entity | `DepartmentMemberJpaEntity.java` — department ManyToOne, member ManyToOne |
| adapter/out/persistence/repository | `DepartmentJpaRepository.java` |
| adapter/out/persistence/repository | `DepartmentMemberJpaRepository.java` |
| adapter/out/persistence/mapper | `DepartmentPersistenceMapper.java` |
| adapter/out/persistence | `DepartmentPersistenceAdapter.java` |
| application/port/out | `DepartmentPort.java` |
| application/port/in/query | `DepartmentQueryUseCase.java` |
| application/port/in/command | `DepartmentCommandUseCase.java` |
| application/service/query | `DepartmentQueryService.java` |
| application/service/command | `DepartmentCommandService.java` |
| adapter/in/web/controller | `DepartmentController.java` |
| adapter/in/web/dto/department | CreateDepartmentRequest, UpdateDepartmentRequest, DepartmentResponse 등 |
| global/aop | `RequireDepartmentRole.java` — @RequireChurchRole 패턴 복제 |
| global/aop | `DepartmentRoleCheckAspect.java` — RoleCheckAspect 패턴 복제 |

**Department API:**
- `GET /churches/{churchId}/departments` — 부서 목록 (가입 시에도 필요하므로 공개)
- `POST /churches/{churchId}/departments` — 부서 생성 (church ADMIN+)
- `PATCH /departments/{departmentId}` — 수정 (dept ADMIN+)
- `DELETE /departments/{departmentId}` — 삭제 (church SUPER_ADMIN, 멤버/그룹 0일 때만)
- `GET /departments/{departmentId}/members` — 부서 멤버 목록 (dept LEADER+)
- `POST /departments/{departmentId}/members` — 멤버 추가 (dept ADMIN+)
- `PATCH /departments/{departmentId}/members/{id}/role` — 역할 변경 (dept ADMIN+)
- `DELETE /departments/{departmentId}/members/{id}` — 멤버 제거 (dept ADMIN+)

### 1-6. 민감정보 조회 로직 변경

기존 `ChurchQueryService.hasElevatedSearchAccess()`:
- **Before**: `church_member.role >= LEADER`
- **After**: `group_member.role = LEADER` OR `department_member.role >= LEADER` OR `church_member.role >= ADMIN`

### 1-7. 테스트

- `DepartmentCommandServiceTest`, `DepartmentQueryServiceTest`
- 삭제 제약, DepartmentRole 권한 체크 검증
- ChurchRole LEADER 제거 후 기존 테스트 업데이트

---

## Phase 2: Backend — 기존 도메인에 Department 연결

### 2-1. 도메인 모델 + JPA Entity 수정

| 도메인 모델 | JPA Entity | 변경 |
|------------|-----------|------|
| `Group.java` | `GroupJpaEntity.java` | `departmentId` / `@ManyToOne department` 추가 |
| `Event.java` | `EventJpaEntity.java` | `departmentId` / `department_id` 컬럼 (nullable) |
| `ChurchMemberRequest.java` | `ChurchMemberRequestJpaEntity.java` | `departmentId` / `@ManyToOne department` (nullable) |
| — | `ChurchJpaEntity.java` | `@OneToMany departments` 관계 추가 |

**변경 안 함**: Visit, VisitMember — department_id 없음

### 2-2. Mapper 수정

- `GroupPersistenceMapper.java` — departmentId 매핑
- `EventPersistenceAdapter.java` 내부 매핑

### 2-3. 기존 쿼리/API에 department 필터 추가

**ChurchController:**
- `GET /{churchId}/groups` → `?departmentId=` 필수 (church SUPER_ADMIN은 전체 가능)
- `GET /{churchId}/birthday-members` → 선택된 부서 멤버만
- `GET /{churchId}/groups-with-leaders` → `?departmentId=` 필수
- `POST /{churchId}/join-request` → body에 `departmentId` 추가 (nullable)
- 가입 승인 시 `church_member` + `department_member` 동시 생성

**VisitController:**
- 심방 대상자 선택 시 현재 부서 멤버 기준 필터
- visit 자체에는 department_id 없음

**EventController:**
- 이벤트 조회 시 `departmentId` 필터
- 이벤트 생성 시 `departmentId` 선택 가능 (null = 교회 전체)

**GroupPort / GroupPersistenceAdapter:**
- `findGroupsByDepartmentId` 추가
- 그룹 생성 시 `departmentId` 필수

**HomeSummary:**
- 유저의 현재 선택 부서 기반 데이터 필터링

**메시지 멤버 검색:**
- 현재 부서 멤버만 검색 가능

### 2-4. 부서 미배정 유저 처리

- department_member가 0개인 church_member → 그룹/모임/메시지 등 기능 제한
- 마이페이지만 접근 가능

### 2-5. 테스트 업데이트

- 기존 GroupQueryServiceTest에 department 필터 케이스 추가
- ChurchRole.LEADER 제거에 따른 기존 테스트 수정

---

## Phase 3: Admin (Next.js/Prisma)

### 3-1. Prisma 스키마 업데이트

`admin/prisma/schema.prisma`에 추가:
- `department` 모델, `department_member` 모델
- `group`에 `department_id` + relation
- `event`에 `department_id`
- `church`에 `departments` relation
- `member`에 `department_members` relation
- `church_member_request`에 `department_id` + department relation

### 3-2. 인증/세션 변경

**JWT payload 변경** (`admin/src/lib/auth.ts`):
- 기존: `{ memberId, memberName, churchId, churchName, role }`
- 변경: `{ memberId, memberName, churchId, churchName, churchRole, departmentId, departmentName, departmentRole }`

**로그인 플로우 변경** (`admin/src/app/api/auth/login/route.ts`):
- 기존: 이메일/비번 → 교회 선택 → 세션
- 변경: 이메일/비번 → 교회 선택 → **부서 선택** → 세션
- Admin 포털 접근 조건: `department_member.role >= LEADER` OR `church_member.role >= ADMIN`
- 교회 선택 후 접근 가능한 부서 목록 반환
  - church SUPER_ADMIN: 해당 교회 모든 부서
  - 그 외: 본인 department_member의 role >= LEADER인 부서만

**교회 전환** (`/api/auth/select-church`):
- 교회 전환 시 부서도 함께 선택 (첫 번째 부서 자동 선택 또는 부서 선택 화면)

**부서 전환** (신규: `/api/auth/select-department`):
- JWT 갱신: departmentId, departmentName, departmentRole 변경

### 3-3. Middleware 변경 (`admin/src/middleware.ts`)

- 기존 SUPER_ADMIN 전용 라우트 (`/manage/visit`, `/manage/prayer`):
  - **변경**: `departmentRole >= ADMIN` OR `churchRole === SUPER_ADMIN`
- 기존 ADMIN+ 라우트 (전체 보호 라우트):
  - **변경**: `departmentRole >= LEADER` OR `churchRole >= ADMIN`
- 시스템 라우트 (`/system/*`): 변경 없음

### 3-4. Sidebar 변경 (`admin/src/components/Sidebar.tsx`)

**상단 교회/부서 전환 영역:**
```
[성실장로교회 ▾]     ← 교회 전환 (여러 교회 시 노출)
[청년부 ▾]           ← 부서 전환 (여러 부서 시 노출)
```
- church SUPER_ADMIN: 모든 부서 목록
- 그 외: 본인 department_member (role >= LEADER) 부서만

**메뉴 구조:**
```
── 대시보드
── 교적부
── 소그룹 관리
── 멤버 관리
── 새가족 관리
── 이벤트 관리
── 가입 관리          ← 이름 변경 (기존: 교회 관리의 일부)
── 심방 관리          ← dept ADMIN+ OR church SUPER_ADMIN
── 기도제목 관리       ← dept ADMIN+ OR church SUPER_ADMIN
── 부서 설정          ← church SUPER_ADMIN 전용
[시스템]              ← system admin 전용
── 교회 관리
── 모니터링
```

### 3-5. 기존 페이지 수정 — 모두 선택된 부서 기준

**모든 API에 적용할 공통 패턴:**
```typescript
// session에서 가져오기
const { departmentId, churchRole } = session;

// 쿼리 필터
const departmentFilter = churchRole === 'SUPER_ADMIN' && !departmentId
  ? {} // 전체 (SUPER_ADMIN이 "전체" 선택 시)
  : { department_id: departmentId };
```

| 화면 | 파일 | 변경 |
|------|------|------|
| **Dashboard 통계** | `app/api/dashboard/stats/route.ts` | 선택된 부서의 그룹/멤버 통계 |
| **Dashboard 모임** | `app/api/dashboard/gatherings/route.ts` | 선택된 부서의 그룹 → 모임 |
| **Dashboard 생일** | `app/api/dashboard/birthdays/route.ts` | 선택된 부서의 department_member → member |
| **교적부** | `app/api/members/route.ts` | 선택된 부서의 department_member 필터 |
| **소그룹 조회** | `app/api/groups/route.ts` | `group.department_id` 필터 |
| **소그룹 관리** | `app/manage/group/` | 그룹 생성 시 현재 부서에 자동 배정. 멤버 배정 시 해당 부서 멤버만 |
| **기도제목** | `app/api/prayers/route.ts` | 선택된 부서의 그룹 → 모임 → 기도제목 |
| **심방** | `app/api/visits/route.ts` | 심방 대상자 선택 시 현재 부서 멤버만. visit 자체는 church 단위 |
| **이벤트** | `app/manage/event/` | 현재 부서 이벤트 + 교회 전체 이벤트(department_id=null) |
| **새가족** | `app/manage/newcomer/` | 선택된 부서의 NEWCOMER 그룹 |
| **가입 관리** | 신규 또는 기존 교회관리 변경 | department_id=null → church SUPER_ADMIN만, dept 있으면 → 해당 dept ADMIN + church SUPER_ADMIN |

### 3-6. 새 페이지: 부서 설정 (`/manage/departments`)

- church SUPER_ADMIN 전용
- 부서 목록 (멤버 수, 그룹 수)
- 부서 생성/수정/삭제 (멤버/그룹 0일 때만 삭제 가능)
- 부서 멤버 배정/역할 변경/제거
- 졸업 처리 (status → GRADUATED)

**API 라우트:**
- `GET/POST /api/departments`
- `PATCH/DELETE /api/departments/[id]`
- `POST /api/departments/[id]/members`
- `DELETE /api/departments/[id]/members/[memberId]`
- `PATCH /api/departments/[id]/members/[memberId]/role`

---

## Phase 4: Web-app (React/Vite)

### 4-1. 타입 + API 클라이언트

**`types/index.ts`:**
```typescript
interface Department {
  id: string;
  churchId: string;
  name: string;
  description?: string;
}

interface DepartmentMember {
  id: string;
  departmentId: string;
  memberId: string;
  role: 'MEMBER' | 'LEADER' | 'ADMIN';
  status: 'ACTIVE' | 'GRADUATED';
}
```

**`lib/api.ts`:**
- `departmentsApi.getByChurch(churchId)` 추가
- 기존 API에 `departmentId` 파라미터 추가

### 4-2. 부서 전환 (MY 페이지)

**`features/my/MyPage.tsx` 변경:**
```
── 내 정보 수정
── 비밀번호 변경
── 교회 전환          ← 여러 교회 시만 노출 (기존)
── 부서 전환          ← 현재 교회에서 여러 부서 시만 노출 (신규)
── 로그아웃
```

- 현재 선택된 부서를 `localStorage`에 저장 (`selectedDepartmentId`)
- 부서 전환 시 홈으로 리다이렉트

### 4-3. 가입 플로우 변경

**`features/auth/RequestChurchPage.tsx` 변경:**
1. 교회 검색 → 교회 선택 (기존)
2. **해당 교회의 부서 목록 표시** → 부서 선택 (선택 안 함 가능)
3. 가입 신청 (`churchesApi.requestRegistration(churchId, departmentId?)`)

### 4-4. 페이지별 변경 — 선택된 부서 기준 단일 뷰

| 화면 | 파일 | 변경 |
|------|------|------|
| **홈 - 요약** | `features/home/HomePage.tsx` | backend가 선택된 부서 기반 필터 |
| **홈 - 생일** | `features/home/HomePage.tsx` | 선택된 부서 멤버만 |
| **홈 - 캘린더** | `features/home/MiniCalendar.tsx` | 선택된 부서 이벤트 + 교회 전체 이벤트 |
| **홈 - 멤버 검색** | `features/home/MemberSearchModal.tsx` | 선택된 부서 멤버만 |
| **그룹 목록** | `features/group/GroupListPage.tsx` | 변경 최소 (내 그룹만 이미 표시) |
| **가입 신청** | `features/auth/RequestChurchPage.tsx` | 부서 선택 UI 추가 |
| **마이페이지** | `features/my/MyPage.tsx` | 부서 전환 메뉴 추가 |
| **기도제목** | `features/prayer/PrayerPage.tsx` | 변경 없음 (내 기도제목, 자연 격리) |
| **메시지** | `features/messages/MessagesPage.tsx` | 변경 없음 (1:1, 자연 격리) |
| **그룹/모임 상세** | 각 detail 페이지 | 변경 없음 (내부 데이터) |

### 4-5. 부서 미배정 상태 UX

- `department_member`가 0개인 유저 → "부서 배정 대기 중" 안내 화면
- 마이페이지만 접근 가능, 나머지 기능 제한
- 관리자에게 배정 요청 안내

---

## Phase 5: 엣지케이스 + 검증

1. 부서 삭제 시 멤버/그룹 0 검증
2. 부서 간 멤버 이동 (관리자 기능)
3. SUPER_ADMIN 전체 부서 조회 로직 검증
4. ChurchRole.LEADER 제거 후 기존 코드 정리
5. 다중 부서 소속 유저의 부서 전환 → 모든 화면 데이터 정합성 확인

### 검증 체크리스트

1. **마이그레이션 후**: 모든 church에 "전체" department 존재, 모든 church_member에 대응하는 department_member 존재
2. **권한**: JWT memberId → department_member 직접 조회로 권한 체크 동작
3. **Admin 격리**: 부서 전환 시 모든 화면(dashboard/교적부/소그룹/기도제목)이 해당 부서만 표시
4. **Web-app 격리**: 부서 전환 시 생일/이벤트/멤버 검색이 해당 부서 기반
5. **SUPER_ADMIN**: department_member 없이 모든 부서 전환 확인
6. **Admin 포털 접근**: dept LEADER가 로그인 가능, dept MEMBER가 로그인 불가 확인
7. **심방/기도제목**: dept ADMIN에게만 메뉴 노출 확인
8. **민감정보**: group LEADER에게만 민감정보 노출, SUB_LEADER에게는 미노출 확인
9. **가입 관리**: 부서 미선택 요청은 SUPER_ADMIN만, 부서 선택 요청은 해당 dept ADMIN도 볼 수 있는지 확인
10. **통합**: 부서 생성 → 멤버 배정 → 그룹 생성 → 모임 → 부서 전환 → 데이터 격리 확인

---

## 핵심 파일 목록

### Backend — 신규
- `domain/enums/DepartmentRole.java`, `DepartmentMemberStatus.java`
- `domain/model/Department.java`, `DepartmentId.java`, `DepartmentMember.java`, `DepartmentMemberId.java`
- `adapter/out/persistence/entity/DepartmentJpaEntity.java`, `DepartmentMemberJpaEntity.java`
- `adapter/out/persistence/repository/DepartmentJpaRepository.java`, `DepartmentMemberJpaRepository.java`
- `adapter/out/persistence/mapper/DepartmentPersistenceMapper.java`
- `adapter/out/persistence/DepartmentPersistenceAdapter.java`
- `application/port/out/DepartmentPort.java`
- `application/port/in/query/DepartmentQueryUseCase.java`, `command/DepartmentCommandUseCase.java`
- `application/service/query/DepartmentQueryService.java`, `command/DepartmentCommandService.java`
- `adapter/in/web/controller/DepartmentController.java`
- `adapter/in/web/dto/department/*`
- `global/aop/RequireDepartmentRole.java`, `DepartmentRoleCheckAspect.java`

### Backend — 수정
- `domain/enums/ChurchRole.java` — LEADER 제거
- `domain/enums/EntityType.java` — DEPARTMENT 추가
- `domain/model/Group.java`, `Event.java`, `ChurchMemberRequest.java`
- `adapter/out/persistence/entity/GroupJpaEntity.java`, `EventJpaEntity.java`, `ChurchMemberRequestJpaEntity.java`, `ChurchJpaEntity.java`
- `adapter/in/web/controller/ChurchController.java`, `VisitController.java`
- `global/aop/RoleCheckAspect.java` — LEADER 제거 반영
- `application/service/query/ChurchQueryService.java` — hasElevatedSearchAccess 변경

### Admin — 수정
- `prisma/schema.prisma`
- `src/lib/auth.ts` — JWT payload 변경
- `src/lib/roles.ts` — LEADER 제거
- `src/middleware.ts` — dept role 체크 추가
- `src/components/Sidebar.tsx` — 부서 전환 + 메뉴 변경
- `src/app/api/auth/login/route.ts` — 부서 선택 플로우
- `src/app/api/dashboard/*/route.ts` — 부서 필터
- `src/app/api/members/route.ts` — 부서 필터
- `src/app/api/groups/route.ts` — 부서 필터
- `src/app/api/prayers/route.ts` — 부서 필터
- `src/app/api/visits/route.ts` — 대상자 부서 필터
- `src/app/manage/group/` — 그룹 생성 시 부서 자동 배정
- `src/app/manage/event/` — 이벤트 부서 선택

### Admin — 신규
- `src/app/manage/departments/` — 부서 설정 페이지
- `src/app/api/departments/` — 부서 API
- `src/app/api/auth/select-department/route.ts` — 부서 전환

### Web-app — 수정
- `src/types/index.ts` — Department, DepartmentMember 타입
- `src/lib/api.ts` — departmentsApi 추가
- `src/features/my/MyPage.tsx` — 부서 전환
- `src/features/auth/RequestChurchPage.tsx` — 부서 선택
- `src/features/home/HomePage.tsx` — 부서 기반 필터

### Web-app — 신규
- `src/features/my/SelectDepartmentPage.tsx` — 부서 전환 페이지 (SelectChurchPage 패턴)
