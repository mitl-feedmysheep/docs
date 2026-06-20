# 공지사항/전체푸시 이미지 첨부

## 목적

공지사항과 전체 푸시 알림에 이미지를 여러 장 첨부할 수 있게 한다. 알림(push notification)에는 이미지를 포함하지 않고, 앱 내 공지 상세 화면에서만 이미지를 표시한다.

## 구현 범위

- 이미지는 기존 Media 엔티티(entity_type=ANNOUNCEMENT)로 관리, Cloudflare R2 저장
- 공지사항과 전체 푸시(BROADCAST) 모두 동일한 이미지 첨부 기능 적용
- 어드민에서 공지 작성 후 상세 다이얼로그에서 이미지 추가/삭제
- 앱 상세 페이지에서 썸네일 가로 스크롤 → 탭 시 라이트박스 (좌우 스와이프)

## 변경 파일

### Backend
- `EntityType.java`: ANNOUNCEMENT 추가
- `AnnouncementResponse.java`: `images: List<String>` 필드 추가
- `AnnouncementController.java`: getById에서 MediaPort로 이미지 URL 조회 후 응답에 포함

### Admin
- `api/announcements/[id]/media/route.ts`: GET(목록), POST(URL 저장) 신규
- `api/announcements/[id]/media/presign/route.ts`: presigned URL 생성 신규
- `api/announcements/[id]/media/[mediaId]/route.ts`: DELETE 신규
- `announcement-manage-client.tsx`: 공지/브로드캐스트 상세 다이얼로그에 이미지 섹션 추가

### Web App
- `lib/api.ts`: AnnouncementItem에 `images?: string[]` 추가
- `AnnouncementDetailPage.tsx`: 이미지 갤러리 + 라이트박스 추가

## 이미지 업로드 흐름 (Admin)

1. 공지/브로드캐스트 생성 → ID 확보
2. 상세 다이얼로그 열기 → 이미지 섹션에서 파일 선택
3. `POST /api/announcements/{id}/media/presign` → R2 presigned URL + publicUrl 반환
4. 클라이언트에서 presigned URL로 PUT 직접 업로드
5. `POST /api/announcements/{id}/media` → publicUrl 저장, media 레코드 생성

## 앱 이미지 표시 (Web App)

- 상세 페이지 본문 아래 가로 스크롤 썸네일 열 (80×80px 정사각형)
- 탭 → 라이트박스 오버레이: 현재 이미지 전체화면, 좌우 화살표 버튼, 스와이프 지원
- 이미지 없으면 섹션 미표시
