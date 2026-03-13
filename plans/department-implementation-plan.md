# Department 도입 구현 계획

## 배경

현재 `Church`가 사실상 데이터 격리 단위(= department 역할)로 작동하고 있음. 성실장로교회에서 신혼부부부 외에 청년부 목장이 추가되면, 같은 교회 내에서 부서별 데이터 격리가 필요함.

**변경 전**: Church → Group → GroupMember
**변경 후**: Church → Department → Group/Member

---

## 확정된 결정사항

| 항목 | 결정 |
|------|------|
| 기본 부서 | 교회 생성 시 "전체" 이름으로 자동 생성 (`is_default=true`) |
| 1 그룹 = N 부서? | 1 그룹 = 1 부서 (`group.department_id` NOT NULL) |
| 1 유저 = N 부서? | 가능 → `department_member` 중간 테이블 |
| DepartmentRole | MEMBER / LEADER / ADMIN (`department_member.role`) |
| SUPER_ADMIN 권한 | department_member 등록 없이 모든 부서 조회 가능 |
| ADMIN 권한 | 소속 부서만 |
| 가입 플로우 | 유저가 부서 선택 → 관리자가 승인 시 부서 배정 필수 (유저 선택 참고) |
| 부서 미배정 | 허용 → "부서 배정 대기 중" 안내 + 기능 제한 |
| 유저앱 홈 화면 | 소속 부서들 데이터 합산 + [부서] 태그 표시 |
| 부서 전환 UI | 불필요 — 합산 표시 |
| 메시지 검색 범위 | 내 부서 멤버만 |
| 알림 격리 | 별도 작업 불필요 (엔티티 체인으로 자연 격리) |
| 부서 삭제 | 멤버/그룹 0일 때만 가능, 기본 부서 삭제 불가 |
| 격리 대상 | 그룹, 모임, 출석, 심방, 이벤트, 생일, 멤버 검색 |
| 격리 안 함 | notification, message, prayer (엔티티 체인으로 자연 격리) |

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
  is_default BOOLEAN NOT NULL DEFAULT FALSE,
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
| `church_member_request` | `department_id CHAR(36)` + FK | nullable |

**변경 안 함**: notification, message, prayer, event (entity_type/entity_id 다형성 패턴 활용), visit (대상자 기반 자연 격리)

### 1-3. 데이터 마이그레이션

1. 각 교회에 기본 department 생성 (`is_default=TRUE`, name="전체")
2. 모든 `church_member` → 기본 department로 `department_member` 생성 (member_id FK, role: ChurchRole 매핑, SUPER_ADMIN→ADMIN)
3. 모든 `group` → 기본 department의 `department_id` 배정
4. `group.department_id` NOT NULL로 변경

### 1-4. Backend 신규 파일 (Hexagonal 패턴)

| 레이어 | 파일 |
|--------|------|
| domain/enums | `DepartmentRole.java` — MEMBER(1), LEADER(2), ADMIN(3) + hasPermissionOver() |
| domain/model | `Department.java` — AggregateRoot, churchId, name, description, isDefault |
| domain/model | `DepartmentId.java` — BaseId |
| domain/model | `DepartmentMember.java` — DomainEntity, departmentId, memberId, role, status(ACTIVE/GRADUATED) |
| domain/model | `DepartmentMemberId.java` — BaseId |
| adapter/out/persistence/entity | `DepartmentJpaEntity.java` — @SQLRestriction, church ManyToOne |
| adapter/out/persistence/entity | `DepartmentMemberJpaEntity.java` — department ManyToOne, churchMember ManyToOne |
| adapter/out/persistence/repository | `DepartmentJpaRepository.java` |
| adapter/out/persistence/repository | `DepartmentMemberJpaRepository.java` |
| adapter/out/persistence/mapper | `DepartmentPersistenceMapper.java` |
| adapter/out/persistence | `DepartmentPersistenceAdapter.java` |
| application/port/out | `DepartmentPort.java` |
| application/port/in | `DepartmentQueryUseCase.java` |
| application/port/in | `DepartmentCommandUseCase.java` |
| application/service/query | `DepartmentQueryService.java` |
| application/service/command | `DepartmentCommandService.java` — 교회 생성 시 "전체" 부서 자동 생성 |
| adapter/in/web/controller | `DepartmentController.java` |
| adapter/in/web/dto/department | CreateDepartmentRequest, UpdateDepartmentRequest, DepartmentResponse 등 |
| global/aop | `RequireDepartmentRole.java` — @RequireChurchRole 패턴 복제 |
| global/aop | `DepartmentRoleCheckAspect.java` — RoleCheckAspect 패턴 복제 |
| domain/enums | `EntityType.java`에 `DEPARTMENT` 추가 |

**Department API:**
- `GET /churches/{churchId}/departments` — 부서 목록 (가입 시에도 필요하므로 공개)
- `POST /churches/{churchId}/departments` — 부서 생성 (ADMIN+)
- `PATCH /departments/{departmentId}` — 수정
- `DELETE /departments/{departmentId}` — 삭제 (멤버/그룹 0일 때만, 기본 부서 불가)
- `GET /departments/{departmentId}/members` — 부서 멤버 목록
- `POST /departments/{departmentId}/members` — 멤버 추가
- `PATCH /departments/{departmentId}/members/{id}/role` — 역할 변경
- `DELETE /departments/{departmentId}/members/{id}` — 멤버 제거

### 1-5. 테스트

- `DepartmentCommandServiceTest`, `DepartmentQueryServiceTest`
- 기본 부서 자동 생성, 삭제 제약, DepartmentRole 권한 체크 검증

---

## Phase 2: Backend — 기존 도메인에 Department 연결

### 2-1. 도메인 모델 + JPA Entity 수정

| 도메인 모델 | JPA Entity | 변경 |
|------------|-----------|------|
| `Group.java` | `GroupJpaEntity.java` | `departmentId` / `@ManyToOne department` 추가 |
| — | — | visit: 변경 없음 (대상자 기반 자연 격리) |
| — | — | event: 변경 없음 (`entity_type='DEPARTMENT'` + `entity_id=departmentId`로 소속 구분) |
| `ChurchMemberRequest.java` | `ChurchMemberRequestJpaEntity.java` | `departmentId` / `@ManyToOne department` (nullable) |
| — | `ChurchJpaEntity.java` | `@OneToMany departments` 관계 추가 |

### 2-2. Mapper 수정

- `GroupPersistenceMapper.java` — departmentId 매핑

### 2-3. 기존 쿼리/API에 department 필터 추가

**ChurchController:**
- `GET /{churchId}/groups` → `?departmentId=` 옵셔널. 일반 유저는 소속 부서만, SUPER_ADMIN은 전체
- `GET /{churchId}/birthday-members` → 유저의 소속 부서들 기반 필터 (중복 제거)
- `GET /{churchId}/groups-with-leaders` → `?departmentId=` 옵셔널
- `POST /{churchId}/join-request` → body에 `departmentId` 추가
- 가입 승인 시 `department_member` 레코드 함께 생성

**VisitController:**
- 변경 없음 (심방 대상자의 department_member를 통해 자연 격리)

**EventController:**
- 변경 없음 (`entity_type='DEPARTMENT'` + `entity_id` 패턴으로 부서 이벤트 소속 구분)

**GroupPort / GroupPersistenceAdapter:**
- `findGroupsByMemberIdAndChurchId` → department 필터 옵션
- 그룹 생성 시 `departmentId` 필수

**HomeSummary:**
- 유저의 소속 부서들 기반 데이터 필터링

**메시지 멤버 검색:**
- 내 부서 멤버만 검색 가능

### 2-4. 부서 미배정 유저 처리

- department_member가 0개인 church_member → 그룹/모임/메시지 등 기능 제한
- 관리자에게 "부서 미배정 멤버" 알림 전송

### 2-5. 테스트 업데이트

- 기존 GroupQueryServiceTest, VisitQueryServiceTest에 department 필터 케이스 추가

---

## Phase 3: Admin (Next.js/Prisma)

### 3-1. Prisma 스키마 업데이트

`admin/prisma/schema.prisma`에 추가:
- `department` 모델, `department_member` 모델
- `group`에 `department_id` + relation
- `church`에 `departments` relation
- `member`에 `department_members` relation
- `church_member_request`에 `department_id`

### 3-2. 새 페이지: 부서 관리

- `/manage/departments/page.tsx` — 부서 목록 (멤버 수, 그룹 수, 리더)
- 부서 생성/수정/삭제 (기본 부서 삭제 불가, 멤버/그룹 0 제약)
- 멤버 배정/역할 변경/제거

### 3-3. 새 API 라우트

- `GET/POST /api/departments`
- `PATCH/DELETE /api/departments/[id]`
- `POST /api/departments/[id]/members`
- `DELETE /api/departments/[id]/members/[memberId]`
- `PATCH /api/departments/[id]/members/[memberId]/role`

### 3-4. 기존 페이지 수정 — 부서 필터 드롭다운

| 페이지 | 변경 |
|--------|------|
| Dashboard | 부서 필터 (통계, 모임) |
| Members (교적부) | 부서 필터 + 부서 뱃지. 멤버 상세에 소속 부서 목록 + 각 역할 표시 |
| Groups | 부서 필터 + 그룹 생성 시 부서 선택 필수 |
| Visits | 변경 없음 (대상자 기반 자연 격리) |
| Events | `entity_type='DEPARTMENT'` 패턴으로 부서별 이벤트 생성/조회 |
| 가입 요청 승인 | 요청 부서 표시 + 부서 배정 필수 |

### 3-5. Sidebar

`admin/src/components/Sidebar.tsx`에 "부서 관리" 메뉴 추가 (운영 관리 섹션)

---

## Phase 4: Web-app (React/Vite)

### 4-1. 타입 + API 클라이언트

`types/index.ts`에 `Department` 타입 추가

`lib/api.ts`:
- `departmentsApi.getByChurch(churchId)` 추가
- 기존 API에 `departmentId` 파라미터 추가 (getBirthdayMembers, getGroupsWithLeaders 등)

### 4-2. 가입 플로우 변경

- 교회 가입 시 부서 선택 UI 추가
- 부서 1개(기본)만 있으면 자동 선택
- `localStorage`에 유저의 소속 `departmentIds` 저장

### 4-3. 페이지 수정

| 페이지 | 변경 |
|--------|------|
| HomePage | 생일/이벤트: 소속 부서 합산 + [부서명] 태그. 중복 제거. |
| GroupListPage | 현재도 내 그룹만 보여서 변경 최소 |
| MiniCalendar | `entity_type='DEPARTMENT'` 이벤트 포함 조회 |
| 메시지 멤버 검색 | 내 부서 멤버만 |

### 4-4. 부서 미배정 상태 UX

- department_member가 0개면 "부서 배정 대기 중" 안내 화면
- 마이페이지 정도만 접근 가능
- 관리자에게 배정 요청 안내

---

## Phase 5: 엣지케이스 마무리

- 기본 부서 삭제 불가 제약
- 부서 삭제 시 멤버/그룹 0 검증
- 다중 부서 생일 중복 제거
- 부서 간 멤버 이동 (관리자 기능)
- SUPER_ADMIN 전체 부서 조회 로직 검증

---

## 검증 방법

1. **Backend**: `./gradlew test` — 신규 + 기존 테스트 통과
2. **Admin**: `npm run build` + 부서 CRUD + 필터링 수동 테스트
3. **Web-app**: `npm run build` + 가입 플로우 + 홈 화면 부서 태그 확인
4. **마이그레이션**: 로컬 DB에서 스크립트 실행 → 기존 데이터 정합성 확인
5. **통합**: 부서 생성 → 멤버 배정 → 그룹 생성 → 홈 화면 격리 검증 → 부서 삭제 제약 검증

---

## 핵심 파일 목록

### Backend — 수정 대상
- `domain/model/Group.java`, `ChurchMemberRequest.java`
- `adapter/out/persistence/entity/GroupJpaEntity.java`, `ChurchJpaEntity.java`, `ChurchMemberRequestJpaEntity.java`
- `adapter/out/persistence/GroupPersistenceAdapter.java`
- `adapter/in/web/controller/ChurchController.java`
- `domain/enums/EntityType.java`

### Backend — 참고 (패턴 복제용)
- `global/aop/RequireChurchRole.java` → RequireDepartmentRole
- `global/aop/RoleCheckAspect.java` → DepartmentRoleCheckAspect
- `domain/enums/ChurchRole.java` → DepartmentRole

### Admin — 수정 대상
- `admin/prisma/schema.prisma`
- `admin/src/components/Sidebar.tsx`
- `admin/src/app/api/groups/route.ts`, `admin/src/app/api/members/route.ts`

### Web-app — 수정 대상
- `web-app/src/lib/api.ts`
- `web-app/src/types/index.ts`
- `web-app/src/features/home/HomePage.tsx`

---

## 구현 현황 (2026-03-13)

| Phase | 상태 | 비고 |
|-------|------|------|
| Phase 1: DB + Backend 도메인 | ✅ 완료 | department, department_member 테이블 생성. 데이터 마이그레이션 완료. |
| Phase 2: Backend 기존 도메인 연결 | ✅ 완료 | Group에 departmentId 연결. ChurchController join-request에 departmentId 추가. |
| Phase 3: Admin | ✅ 완료 | Prisma 스키마, 부서 관리 페이지, 부서 CRUD API, 부서 멤버 API, 기존 페이지 부서 필터, 세션에 departmentId 추가. 테스트 144개 통과. |
| Phase 4: Web-app | ✅ 완료 | Department/DepartmentMember 타입, departmentsApi, 가입 시 부서 선택 UI (다중 부서 → 선택, 단일 부서 → 자동선택). |
| Phase 5: 엣지케이스 | ✅ 완료 | is_default 기본 부서 삭제 불가, 멤버/그룹 0 검증, SUPER_ADMIN 전체 부서 조회. |

### 실제 구현에서 원안과 달라진 점

| 항목 | 원안 | 실제 |
|------|------|------|
| `department_member` FK | `church_member_id` → church_member | `member_id` → member (조인 간소화) |
| `department_member.status` | 없음 | `ACTIVE` / `GRADUATED` 추가 (졸업 이력 보존) |
| `visit.department_id` | 추가 (NOT NULL) | **제거** — 심방 대상자 기반 자연 격리 |
| `event.department_id` | 추가 (nullable) | **제거** — `entity_type='DEPARTMENT'` + `entity_id` 다형성 패턴 활용 |
| 이벤트 부서 소속 | `department_id` 컬럼으로 필터 | `entity_type + entity_id`로 소속 구분 (CHURCH/DEPARTMENT/GROUP) |
