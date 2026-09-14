# Q&A(질문 게시판) - 질문 글 CRUD 슬라이스 설계

> 상태: 확정 (2026-09-09 구현 완료 - ondal-BE PR #21)
> 역할: 이 슬라이스의 범위와 결정 기록 - 확정 후에는 거의 갱신하지 않으며, 결정은 BE PR 본문에도 있다
> 기준 규약: [guide/design.md](../guide/design.md) 3절·4절 - 이 문서는 그 규약에 추가되는 결정만 기록
> 스코프 결정: [decisions/6](../decisions/6-qna-%EA%B2%8C%EC%8B%9C%ED%8C%90-p1-%ED%8E%B8%EC%9E%85.md) - 왜 P1에 들어왔는가
> 스키마 원본: [db/schema.md](../db/schema.md) 결정 10 · `questions` 테이블
> 문서 구성: [api.md](api.md)(구현 계약) · [test.md](test.md)(테스트 계획) · [fe.md](fe.md)(FE 연동)

## 1. 범위

- 질문 글 CRUD API 5개 (`qna` 패키지 신설) - 명세는 [api.md](api.md)
- 답변(댓글)·알림·검색·페이징은 범위 밖 - decisions/6 재검토 조건
- 원본: 팀원의 독립 게시판 데모(BE PR #21 최초 커밋 - `CRUD/` 폴더, Thymeleaf·H2·Lombok)를 규약대로 재작성한 것 - 원본과의 차이는 PR #21 코멘트에 정리

## 2. 결정

1. **분반 스코프** - `/api/cohorts/{cohortId}/questions`. 전역 게시판 아님: 학생이 볼 수 있는 타인 정보는 자기 분반 안으로 한정(guide 4절), `@CohortRole`이 분반 소속을 판정하려면 경로에 `{cohortId}` 필요
2. **권한 3단** - 조회·등록: 분반 소속 누구나(`@CohortRole(STUDENT)`) / 수정: 작성자 본인만 / 삭제: 작성자 본인 또는 운영진 이상(ADMIN 자동 통과)
   - 운영진도 남의 글은 수정 불가 - 남의 말을 바꾸는 것은 운영이 아님. 게시판 정리(삭제)만 운영 권한
   - 어노테이션은 분반 소속까지, 본인·운영진 판정은 서비스에서 `ForbiddenException`(403) - Submission 슬라이스의 "본인 또는 운영진" 선례. 404가 아닌 이유: 같은 분반 글은 누구나 볼 수 있어 존재 비노출이 성립하지 않음
3. **`canEdit`·`canDelete` 서버 판정값** - CohortResponse.canManage 선례. 프론트가 권한 규칙을 재구현하지 않게 응답에 실어 준다. 보관 분반이면 둘 다 false (쓰기 409 규약과 정합)
4. **작성자 = `UserSummary`(id·이름·직책)** - 수강생에게 내려가는 타인 정보라 loginId·globalRole 비노출. 직책(교육운영진/일반 수강생/해구르르)은 응답 조립 시 분반 명부 1회 조회로 채운다 - 분반에서 빠진 작성자는 "일반 수강생"으로 표시(글은 남는다)
5. **목록 최신순, 페이징 없음** - guide 4절 기본값(createdAt desc). 같은 시각은 id desc로 순서 고정
6. **`updatedAt` 없음** - 기존 엔티티(Cohort·Assignment)와 동일하게 createdAt만. "수정됨" 표시가 필요해지면 그때 열 추가
7. **title 200자·content 10000자, 둘 다 필수** - Assignment title·description 한도와 동일. 내용 없는 질문은 허용하지 않음
8. **인덱스 2개 명시** - `(cohort_id, created_at)` 복합(목록 조회+정렬을 하나로) + `author_id`. PostgreSQL은 FK 인덱스 자동 생성 없음
9. **삭제 = 하드 삭제, 연쇄 없음** - 자식 테이블이 아직 없음. Answer 추가 시 서비스 연쇄(answers → question)로 확장 (schema.md 4절 규칙)
10. **시더에 샘플 2건** - student1·student2 질문. FE가 목록 정렬·작성자 배지·버튼 분기를 바로 확인
11. **답변(Answer) 편입 (2026-09-14, P2)** - decisions/6 재검토 조건(질문 글 운영 반영) 충족, PM 확정. `answers(question_id, author_id, content, created_at)` 자식 테이블
    - 답변 권한 = **분반 소속 누구나**(운영진 전용 아님 - 동료 답변 장려) / 수정 작성자만 / 삭제 작성자 또는 운영진 이상 - 질문과 같은 규칙, 사용자가 두 규칙을 외우지 않게
    - **채택·좋아요 없음** - 요구 없는 상태 모델을 만들지 않음. 필요해지면 열 추가
    - 정렬 = **오래된 순**(대화 흐름) - 질문 목록(최신순)과 다른 이유: 답변은 위에서 아래로 읽는다
    - `QuestionResponse.answerCount` - 목록에서 답 없는 질문이 눈에 띄게. 분반 단위 집계 쿼리 1회
    - 질문 삭제 시 답변은 서비스 연쇄(결정 9 의 "Answer 추가 시 서비스 연쇄" 이행). FE 삭제 경고에 답변 수 표시
    - API 4개 `/api/cohorts/{cohortId}/questions/{questionId}/answers` (#41~#44, [api.md](api.md) 5절)

## 3. 후속 작업

- [x] BE: 이슈 #20 → PR #21 머지 (2026-09-09, QuestionApiTest 21건 포함 전체 152건 통과)
- [x] FE: 질문 목록·상세·작성/수정 화면 - ondal-FE 이슈 #23 → PR #26 머지 (2026-09-14, 실 BE·mock 헤드리스 시나리오 각 18건 통과, [fe.md](fe.md) 매핑 채택). MSW mock은 시더와 동일 데이터
- [x] 답변(Answer) 슬라이스 BE - 결정 11, ondal-BE 이슈 #35 → PR #36 머지 (2026-09-14, AnswerApiTest 6건 포함 전체 193건 통과, Flyway V4)
- [ ] 답변 FE - ondal-FE 이슈 #35 → PR #36 (2026-09-14, 헤드리스 14건 mock·실 BE 통과. 서버 BE 재빌드(V4) 후 머지)
- [ ] 답변 FE - 질문 상세의 답변 영역(목록·작성·인라인 수정·삭제), 목록 "답변 N" ([fe.md](fe.md) 1절)
- ~~알림~~ - 2026-09-14 PM 확정으로 제외 (mvp-scope 5절)
