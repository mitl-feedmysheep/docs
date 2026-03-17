# 설교 노트 tiptap 에디터 도입 설계

## 배경

설교 노트의 content 필드가 plain text로 저장/렌더링되고 있어, 유저가 `-`를 수동으로 사용해 bullet point를 표현하고 있다. 중첩 bullet(하위 항목)은 공백+대시로 억지 표현하는데, 렌더링 시 구분이 안 되어 가독성이 떨어진다.

## 목표

- 노션처럼 타이핑하면서 바로 bullet point가 보이는 에디터 제공
- Tab/Shift+Tab (또는 모바일 버튼)으로 중첩 bullet 지원
- 기존 plain text textarea를 tiptap 에디터로 교체

## 범위

- web-app의 CreateSermonNotePage, SermonNoteDetailPage, SermonNotesPage 수정
- 백엔드 변경 없음 (content TEXT 컬럼 그대로 사용)
- 기존 plain text 데이터 호환: 하지 않음 (추후 수동 마이그레이션)

## 설계

### 1. 저장 포맷

tiptap JSON document를 `JSON.stringify()`해서 content 컬럼에 저장.

예시 입력:
```
• 미갈의 사람을 향한 시선
  • 탄자니아 선교 같이 어깨동무
• 하나님은 어떤 분이신가
```

DB 저장 값:
```json
{
  "type": "doc",
  "content": [
    {
      "type": "bulletList",
      "content": [
        {
          "type": "listItem",
          "content": [
            { "type": "paragraph", "content": [{ "type": "text", "text": "미갈의 사람을 향한 시선" }] },
            {
              "type": "bulletList",
              "content": [
                {
                  "type": "listItem",
                  "content": [
                    { "type": "paragraph", "content": [{ "type": "text", "text": "탄자니아 선교 같이 어깨동무" }] }
                  ]
                }
              ]
            }
          ]
        },
        {
          "type": "listItem",
          "content": [
            { "type": "paragraph", "content": [{ "type": "text", "text": "하나님은 어떤 분이신가" }] }
          ]
        }
      ]
    }
  ]
}
```

### 2. 에디터 컴포넌트 (CreateSermonNotePage)

**tiptap 설정:**
- extensions: `StarterKit.configure()` (heading, bold, italic, code, blockquote 등 비활성화 — BulletList, ListItem은 StarterKit에 포함되어 있으므로 별도 추가하지 않음) + `Placeholder`
- placeholder: "설교를 들으며 느낀 점, 깨달은 점을 자유롭게 적어보세요..."

**에디터 동작:**
- 엔터 → 새 bullet item 생성
- Tab → 하위 bullet으로 들여쓰기 (최대 3단계 제한 — 모바일 32rem 뷰포트에서 깊은 중첩은 깨짐)
- Shift+Tab → 상위로 내어쓰기
- 빈 bullet에서 엔터 → bullet 해제, 일반 텍스트
- 모바일 대응: 에디터 위 소형 툴바 (들여쓰기, 내어쓰기 버튼 2개)
- 붙여넣기: 리치 텍스트 붙여넣기 시 plain text로 strip (bullet 외 서식 없음)

**canSubmit 검증:**
- 기존 `content.trim()` 대신 `editor.isEmpty`를 사용하여 빈 에디터 감지

**수정 모드 초기화:**
- API에서 데이터 로드 후 `editor.commands.setContent(JSON.parse(note.content))`로 에디터에 주입
- 에디터 생성 시 빈 상태로 시작, useEffect에서 데이터 도착 시 setContent 호출

**기존 코드 변경:**
- `<Textarea>` → `<EditorContent editor={editor}>` 교체
- content state 제거, 저장 시 `JSON.stringify(editor.getJSON())`

**스타일링:**
- 기존 textarea와 동일한 look & feel (border, padding, min-height)
- bullet 스타일: `•` (1단계), `◦` (2단계), `▪` (3단계) CSS 커스텀

### 3. 상세 보기 (SermonNoteDetailPage)

**렌더링:**
- `JSON.parse(content)` → tiptap `generateHTML(json, extensions)` → `dangerouslySetInnerHTML`
- 기존 `split(/\n{2,}/)` + `paragraphs.map()` 로직 제거
- JSON 파싱 실패 시 (기존 plain text 데이터 등) raw text를 그대로 표시하는 try/catch 처리

**스타일링:**
- 에디터와 동일한 bullet 스타일 적용
- 기존 article 영역의 line-height/font-size 유지

### 4. 목록 페이지 (SermonNotesPage)

**content 미리보기:**
- 현재 `note.content`를 직접 표시하고 있는데, JSON 저장 시 raw JSON이 노출됨
- tiptap JSON에서 text 노드만 추출하는 유틸 함수 `extractTextFromTiptapJson(json)` 작성
- 목록 카드에서 이 유틸로 plain text preview 표시

### 5. 의존성 추가

- `@tiptap/react` — React 통합
- `@tiptap/starter-kit` — 기본 extension 번들 (BulletList, ListItem 포함)
- `@tiptap/extension-placeholder` — placeholder 텍스트
- `@tiptap/html` — `generateHTML()` 함수 (상세 보기 렌더링용)

### 6. 파일 변경 목록

| 파일 | 변경 |
|------|------|
| `web-app/package.json` | tiptap 의존성 추가 |
| `web-app/src/features/sermon-notes/CreateSermonNotePage.tsx` | textarea → tiptap 에디터, 모바일 툴바 |
| `web-app/src/features/sermon-notes/SermonNoteDetailPage.tsx` | paragraphs 렌더링 → generateHTML 렌더링 |
| `web-app/src/features/sermon-notes/SermonNotesPage.tsx` | content 미리보기 → extractTextFromTiptapJson 적용 |
| `web-app/src/components/ui/sermon-note-editor.tsx` (신규) | tiptap 에디터 래퍼 컴포넌트 |
| `web-app/src/lib/tiptap-utils.ts` (신규) | extractTextFromTiptapJson, tiptap extensions 설정 공유 |
| `web-app/src/index.css` | tiptap 에디터 + bullet 스타일 |
