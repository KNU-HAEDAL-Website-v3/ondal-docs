# Q&A - API 명세

> 역할: 질문 게시판 도메인의 엔티티와 API 계약. 계약의 원본은 springdoc(OpenAPI) `Question` 태그
> 결정 배경: [design.md](design.md) · 공통 규약(에러 형식·URL 규칙): [guide/design.md](../guide/design.md) 3절·4절 · 스키마 원본: [db/schema.md](../db/schema.md)

## 1. 도메인 역할

Question(질문 글) = 분반 소속자가 자기 분반 게시판에 올리는 글을 다루는 도메인

- 수강생·운영진: 자기 분반의 질문 목록·상세 조회, 질문 등록, 자기 글 수정·삭제
- 운영진: 남의 글 삭제(게시판 정리) - 수정은 불가
- 질문은 항상 분반(Cohort) 하나에 소속 - 다른 분반의 글은 존재 자체를 비노출(404)
- 답변(댓글)은 범위 밖 - 이 슬라이스는 질문 글 CRUD까지만

## 2. 엔티티 (DB 스키마)

테이블 `questions` · 패키지 `qna` (확정 스키마: [db/schema.md](../db/schema.md) 2절)

| 필드 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | Long | PK | |
| cohort | Cohort (N:1, LAZY) | not null | 글이 속한 분반 |
| author | User (N:1, LAZY) | not null | 작성자 - 등록 시 요청자 본인으로 고정, 변경 불가 |
| title | String | not null, 최대 200자 | 질문 제목 |
| content | String | not null, 최대 10000자 | 질문 내용 - 자유 텍스트 |
| createdAt | Instant | not null | 등록 시각(UTC). 수정 시각 열 없음 (design.md 결정 6) |

## 3. API 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 23 | 질문 목록 조회 | `GET /api/cohorts/{cohortId}/questions` | 분반 소속 누구나 | 200 목록(최신순, 빈 배열 가능) | 403, 404(분반 없음) |
| 24 | 질문 상세 조회 | `GET /api/cohorts/{cohortId}/questions/{questionId}` | 분반 소속 누구나 | 200 단건 | 403, 404(분반·질문 없음, 다른 분반의 글) |
| 25 | 질문 등록 | `POST /api/cohorts/{cohortId}/questions` | 분반 소속 누구나 | 201 + Location | 400(입력 검증), 403, 404(분반 없음), 409(보관 분반) |
| 26 | 질문 수정 | `PUT /api/cohorts/{cohortId}/questions/{questionId}` | 작성자 본인 | 200 (전체 교체) | 400, 403(비소속·남의 글), 404, 409(보관 분반) |
| 27 | 질문 삭제 | `DELETE /api/cohorts/{cohortId}/questions/{questionId}` | 작성자 본인 또는 운영진 이상 | 204 | 403(비소속·남의 글인 수강생), 404, 409(보관 분반) |

- 번호는 기존 API 목록(#1~#22)에 이어 부여. 에러 응답 형식은 guide/design.md 3절 공통
- 권한 어노테이션: 5개 모두 `@CohortRole(STUDENT)` (ADMIN은 자동 통과). #26·#27의 "본인 / 운영진" 판정은 서비스에서 403
- 판정 순서(#26·#27): 보관 여부 409 → 스코프 404 → 본인·운영진 403 - 쓰기 서비스 첫 줄 `ensureActive()` 규약

**요청·응답 본문 (record DTO)**

- 응답은 다섯 엔드포인트 모두 `QuestionResponse {id, title, content, author{id, name, title}, createdAt, canEdit, canDelete}` 하나
  - `author`는 `UserSummary` - loginId·globalRole 없음. `title`은 직책 명칭 문자열(해구르르 / 교육운영진 / 일반 수강생)
  - `canEdit`: 작성자 본인 && 분반 ACTIVE / `canDelete`: (작성자 본인 || 운영진 이상) && 분반 ACTIVE - 프론트는 이 값만 보고 버튼 분기
- 등록·수정 요청은 `QuestionCreateRequest` / `QuestionUpdateRequest` - 필드·검증 동일: `title`(필수, 최대 200자), `content`(필수, 최대 10000자)

## 4. 구현 시 주의 (springdoc에 담기지 않는 내부 규약)

- 스코프 조회: 하위 id는 반드시 `findByIdAndCohortIdWithAuthor(id, cohortId)` - `findById` 단독 호출 금지, 불일치는 404(존재 비노출)
- 쓰기 3개(#25~#27)는 서비스 첫 줄 `cohort.ensureActive()` - 보관 분반이면 409 `COHORT_ARCHIVED`
- 서비스 메서드는 `(Long cohortId, ...)` - cohortId를 첫 인자로 수신
- 본인 판정 `Question.isWrittenBy(user)`, 운영진 판정 `CohortAuthorizer.isAllowed(user, cohortId, OPERATOR)` - 권한 규칙 단일 출처 재사용
- 응답 조립 `QuestionResponseAssembler` - 분반 명부(`findAllByCohortIdWithUser`) 1회 조회로 작성자 직책·요청자 역할을 함께 얻음. canDelete의 운영진 판정은 `CohortAuthorizer.canManage` 재사용
- Repository: 두 조회 모두 author fetch join(`@Query`) - 응답에 작성자가 실리므로 트랜잭션 안에서 LAZY 종결 (`WithAuthor` 이름 규약)
- 엔티티는 `User.java` 패턴(정적 팩토리 `create`/`update`, setter 없음) + 인덱스 `idx_questions_cohort_created`·`idx_questions_author`를 `@Table(indexes=...)`로 명시
- springdoc: `@Tag("Question")` + `@Operation(summary)` 최소한만. canEdit·canDelete 규칙은 `@Schema(description)`에 명시
