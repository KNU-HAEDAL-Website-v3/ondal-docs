# Q&A - 테스트 계획

> 역할: 이 슬라이스의 검증 계획 - 구현 중 케이스가 늘면 이 문서만 갱신
> 명세: [api.md](api.md) · 테스트 기반 규약: [guide/design.md](../guide/design.md) 6절

## 1. 테스트 클래스·픽스처

- `qna/QuestionApiTest` - `extends ApiTestSupport`, 컨트롤러와 1:1 (21건, 2026-09-09 통과)
- 슬라이스 고유 픽스처(`createQuestion(cohortId, 세션, 제목)`)는 이 테스트 클래스의 private 헬퍼로 - 세션 주인이 작성자가 된다 (support/는 PM 파일 - guide 6절)
- 분반·소속 픽스처는 기존 공용 헬퍼 사용: `createCohort`, `enrollStudent`, `archiveCohort`, `restoreCohort`

## 2. 케이스 (역할 × 엔드포인트 대표)

- 인증·권한
  - 미로그인 → 401
  - 비소속 MEMBER GET 목록·POST → 403
  - STUDENT POST·GET 목록·단건 → 201·200
  - PUT: 작성자 → 200 / 다른 수강생·OPERATOR·ADMIN → 403 (수정은 본인만)
  - DELETE: 다른 수강생 → 403 / 작성자 → 204 / OPERATOR 남의 글 → 204 / 비소속 ADMIN → 204
  - 비소속 ADMIN GET·POST → 통과, 작성자 직책 "해구르르"
- 스코프·404
  - 교차 분반: A반 OPERATOR가 `/api/cohorts/A/questions/{B의 글}` GET·PUT·DELETE → 404, B반 글은 그대로
  - 없는 questionId → 404 / 없는 cohortId: ADMIN → 404, MEMBER → 403
- 입력 검증
  - `cohortId=abc` → 400 / title 공백·201자 → 400 / content 공백·10001자 → 400 / 깨진 JSON → 400
- 보관 분반
  - GET 목록·단건 → 200 (열람 유지), 응답의 canEdit·canDelete는 false
  - POST·PUT·DELETE → 409 COHORT_ARCHIVED / restore 후 POST → 201
- 동작 확인
  - POST 응답 Location 헤더 + 본문 id·author(name, title="일반 수강생", loginId 없음)·canEdit·canDelete true

## 3. 답변 (`qna/AnswerApiTest`, 2026-09-14 - 6건 통과, 전체 193건)

- 미로그인 401 / 비소속 GET·POST 403
- 소속 누구나 등록(201, Location, 직책, loginId 없음, canEdit·canDelete) → 목록 오래된 순, 질문자는 남의 답변 canEdit·canDelete false → 질문 단건·목록 `answerCount`
- 수정: 질문자·운영진·관리자 → 403, 작성자 → 200 / 삭제: 다른 수강생 403, 작성자·운영진·관리자 204
- 교차 질문의 답변 PUT·DELETE → 404, 없는 질문 404, 빈 내용 400
- 질문 삭제 → 답변 연쇄 삭제(리포지토리 count 0)
- 보관 분반: 목록 200(canDelete false), POST·DELETE 409, 해제 후 204
  - 작성자 직책은 분반 역할을 따른다 (OPERATOR → "교육운영진")
  - canEdit/canDelete 매트릭스: 작성자 true/true · 다른 수강생 false/false · 운영진 false/true · 관리자 false/true
  - PUT 수정은 재조회로 flush 확인 / 목록 최신순 / 빈 목록 `[]` / 삭제 후 목록에서 사라지고 단건 404

## 3. 시더

- `LocalDataSeeder`(local 전용) 확장: 샘플 분반에 질문 2건 - student1·student2 작성 (FE가 목록·작성자 배지·버튼 분기를 바로 확인)
- 테스트는 시더 비의존 - 픽스처는 각 테스트가 생성 (규약)
