# Q&A - FE 연동 메모

> 역할: FE 작업자용 - 질문 게시판 API와 화면의 매핑, 표시 규칙
> 명세: [api.md](api.md) - 계약의 원본은 springdoc `Question` 태그 · FE 공통 규칙: ondal-FE `CLAUDE.md`
> 상태: 구현 완료 (2026-09-14, ondal-FE 이슈 #23 → PR #26 머지·운영 배포) - 와이어프레임 없이 아래 매핑을 그대로 채택. 확정 구성 원문은 FE 이슈 #23 코멘트

## 1. 화면 매핑 (확정 - PR #26)

- 경로: `/cohorts/:cohortId/questions`(목록) · `…/new`(등록) · `…/:questionId`(상세) · `…/:questionId/edit`(수정) - 분반 하위 경로 (과제의 `?cohort=` 방식은 사이드바 진입용이라 미사용)
- 작성 내용 임시 저장: 폼 입력을 sessionStorage(탭 단위, 분반/질문 단위 키)에 보관 → 세션 만료(401) 후 재로그인 복귀 시 복원, 저장 성공·취소 시 삭제 (2절 "401 → 재로그인(작성 내용 보존)" 의 구현)
- 진입: 분반 페이지(`CohortPage`)에 "Q&A" 섹션 - 과제 섹션과 같은 층위, "질문 보기" 버튼
- 질문 목록(#23): 제목 · 작성자 이름 + 직책 배지 · 작성 시각(KST). "질문하기" 버튼은 분반 소속이면 항상 표시(등록 권한 = 소속 누구나), 보관 분반이면 비활성
- 질문 상세(#24): 제목·내용·작성자·시각 + 수정/삭제 버튼 - **`canEdit`·`canDelete`로만 분기** (프론트에서 loginId 비교·역할 판정 금지)
- 작성 폼(#25) · 수정 폼(#26): title(최대 200)·content(최대 10000, 여러 줄). 과제 등록 폼(`AssignmentFormPage`) 패턴 복제. 수정 폼은 상세 응답으로 프리필
- 삭제(#27): 확인 창 후 204 → 목록으로. 운영진이 남의 글을 지울 때는 "작성자의 글을 삭제합니다" 문구 권장

## 2. 표시 규칙

- `author.title`은 서버가 정한 문자열(해구르르 / 교육운영진 / 일반 수강생) 그대로 배지 표시 - 분반 카드의 운영진 표시와 동일 규칙
- `createdAt`(UTC) → KST 변환 표시 (FE CLAUDE.md 규칙 4). 수정 시각 없음 - "수정됨" 표시 불가
- 목록은 서버 정렬(최신순) 그대로, 페이징 없음(전체 수신) - 검색·필터는 FE 자유
- 에러: 401 → 재로그인(작성 내용 보존), 403 → 홈(ApiErrorView 공통), 404 → 안내, 409 COHORT_ARCHIVED → 홈으로 보내지 않고 안내만

## 3. API 클라이언트

- `src/api/questions.ts` 신설 - `assignments.ts` 패턴(요청 함수 + React Query 훅). 타입은 `types.ts`에 `QuestionResponse`·`QuestionRequest` 추가(BE DTO 미러)
- MSW: `src/mocks/handlers.ts`에 5개 핸들러, `data.ts` 시드는 BE `LocalDataSeeder`의 질문 2건과 동일하게 유지

## 4. P1 범위 밖

- 답변(댓글)·알림·검색·페이징 - decisions/6 재검토 조건
