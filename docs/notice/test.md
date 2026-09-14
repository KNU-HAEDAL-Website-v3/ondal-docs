# 공지사항 - 테스트 계획

> 역할: 이 슬라이스의 검증 계획 - 구현 중 케이스가 늘면 이 문서만 갱신
> 명세: [api.md](api.md) · 테스트 기반 규약: [guide/design.md](../guide/design.md) 6절

## 1. 테스트 클래스·픽스처

- `notice/NoticeApiTest` - `extends ApiTestSupport`, 컨트롤러와 1:1 (10건, 2026-09-14 통과 - 전체 179건)
- 슬라이스 고유 픽스처(private 헬퍼): `createGlobal(제목, pinned)` - 관리자 세션 / `createForCohort(cohortId, 세션, 제목, pinned)` - 세션 주인이 작성자
- 분반·소속 픽스처는 공용 헬퍼: `createCohort`, `enrollStudent`, `archiveCohort`, `restoreCohort`

## 2. 케이스 (역할 × 엔드포인트 대표)

- 인증·권한
  - 미로그인 GET 목록 → 401
  - 전체 공지 POST: MEMBER → 403 / ADMIN → 201 (Location, cohort null, pinned, author.title 해구르르, loginId 없음, canEdit·canDelete true) / 비소속 부원 GET 상세 → 200, canEdit false
  - 분반 공지 POST: STUDENT·비소속 → 403 / OPERATOR → 201 (cohort id·name, author.title 교육운영진) / 비소속 ADMIN → 201
  - 목록 가시성: A반 STUDENT → 전체 + A반(필독 먼저) / 비소속 → 전체만 / ADMIN → 전부 canEdit true / A반 OPERATOR → A반 canEdit true, 전체 공지 false
  - 상세: 비소속의 분반 공지 → 403 / 소속·관리자 → 200
  - PUT·DELETE: STUDENT·비소속 → 403 / OPERATOR 자기 반 PUT → 200(pinned 반영), 전체 공지 PUT → 403 / ADMIN 둘 다 통과, 삭제 후 GET 404
- 입력 검증·404
  - title 공백 → 400 INVALID_INPUT / content 10001자 → 400 / pinned 생략 → false
  - 없는 noticeId → 404 / 없는 cohortId: ADMIN → 404, MEMBER → 403
- 보관 분반
  - GET 목록·상세 → 200 (열람 유지), canEdit false
  - POST·PUT·DELETE → 409 COHORT_ARCHIVED / restore 후 PUT → 200
