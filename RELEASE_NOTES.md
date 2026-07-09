# Release Notes

---

## 2026-07-09

### 웹앱 (web-app)

**푸시 알림 구독 자동 복구**
- `silentSync()` 추가: 앱 실행 시 push_subscription 만료 여부를 감지하고 자동으로 재등록
- 브라우저 endpoint와 localStorage 비교로 정상 상황에서는 백엔드 호출 없이 스킵
- `subscribe()` / `unsubscribe()` 시 localStorage endpoint 동기화

**알림 프롬프트 노출 위치 수정**
- 기존: AppShell 진입 시 어느 페이지에서나 1.5초 후 프롬프트 노출
- 변경: 홈 화면(`/`) 진입 시에만 노출
- `/pending-approval` 등 인증 대기 페이지에서 프롬프트가 뜨던 문제 수정

### 어드민 (admin)

**교회 편입 관리 접근 권한 수정**
- 기존: 로그인된 모든 관리자 접근 가능
- 변경: 부서 ADMIN 또는 교회 SUPER_ADMIN만 접근 가능
- 미들웨어 + API 4개 (목록/승인/거절/히스토리) 모두 권한 체크 추가
- 부서 ADMIN은 자기 부서에 신청된 요청만 조회/처리 가능
- 사이드바에서 권한 없는 유저에게 "교회 관리" 메뉴 숨김

### 백엔드 (backend)

**교회 편입 요청 시 부서 ADMIN 알림 발송**
- 편입 요청 생성 시 대상 ADMIN에게 인앱 알림 + 푸시 알림 동시 발송
- 부서 지정 요청 → 해당 부서 ADMIN에게 발송
- 부서 미지정 요청 → 교회 SUPER_ADMIN에게 발송
- 알림 메시지: `"OOO님이 청년부 가입을 신청했어요. 어드민에서 확인해주세요."`
- `NotificationType.JOIN_REQUEST` 추가
- 알림 발송 실패 시 예외를 삼켜 편입 요청 자체는 항상 성공 보장
