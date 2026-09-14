# 공지사항 - FE 연동 메모

> 역할: FE 작업자용 - 공지 API와 화면의 매핑, 표시 규칙
> 명세: [api.md](api.md) - 계약의 원본은 springdoc `Notice` 태그 · FE 공통 규칙: ondal-FE `CLAUDE.md`
> 상태: 견본 화면(피그마 2:37234, `StudentNoticesView`·`OperatorNoticesView`)을 실 API 로 교체 - 2026-09-14 진행

## 1. 화면 매핑

- 목록 `/notices` (역할 통합 한 화면): `GET /api/notices` 그대로 - 서버가 가시성·정렬(필독 먼저)을 정함. 행 = 필독 배지 · 제목 · 대상(`cohort?.name ?? '전체 공지'`) · 작성자 이름 · KST 시각
  - "공지 작성" 버튼: 관리자이거나 `canManage` 분반이 하나라도 있으면 표시 (홈의 운영진 뷰 판정과 동일)
  - 대상 필터(전체 / 전체 공지만 / 분반별)는 클라이언트 필터 - 목록이 이미 내가 볼 수 있는 것만이라 안전
  - 견본의 "유형·상태·기간" 필터, 상태 열(게시 중/예약/숨김), 페이지네이션은 제거 - 대응하는 서버 개념 없음 (design.md 1절)
- 상세 `/notices/:noticeId`: 제목·필독 배지·대상·작성자 배지·시각·본문(줄바꿈 보존). 수정·삭제 버튼은 **`canEdit`·`canDelete` 로만 분기**
- 작성 `/notices/new?cohort=` · 수정 `/notices/:noticeId/edit`: title(200)·content(10000, 여러 줄)·필독 체크. 작성 폼의 대상 선택 = 관리자: "전체 공지" + 진행 중 분반 전체(`GET /api/cohorts`) / 운영진: `canManage` 인 소속 분반. 대상에 따라 `POST /api/notices` 또는 `POST /api/cohorts/{id}/notices`. 수정 폼은 대상 고정(변경 API 없음)
- 삭제: 확인 창 후 204 → 목록으로

## 2. 표시 규칙

- `author.title` 은 서버 문자열 그대로 배지 (Q&A 와 동일 규칙), `createdAt`(UTC) → KST
- 에러: 401 → 재로그인(작성 내용 임시 저장 복원 - `lib/draft`), 403 → 홈(비소속의 분반 공지 상세, 권한 없는 edit URL), 404 → 안내, 409 COHORT_ARCHIVED → 안내만

## 3. API 클라이언트

- `src/api/notices.ts` 신설 - `questions.ts` 패턴. 타입 `NoticeResponse`·`NoticePayload` 는 `types.ts`(BE DTO 미러)
- MSW: 핸들러 6개, 시드 2건 = BE `LocalDataSeeder.seedNotices` 와 동일(전체 공지 필독 1 + 분반 공지 1)

## 4. 범위 밖

- 예약 게시·숨김·댓글·읽음 확인·알림 (design.md 1절)
