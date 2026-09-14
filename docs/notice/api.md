# 공지사항 - API 명세

> 역할: 공지 도메인의 엔티티와 API 계약. 계약의 원본은 springdoc(OpenAPI) `Notice` 태그
> 결정 배경: [design.md](design.md) · 공통 규약(에러 형식·URL 규칙): [guide/design.md](../guide/design.md) 3절·4절 · 스키마 원본: [db/schema.md](../db/schema.md)

## 1. 도메인 역할

Notice(공지) = 운영 측이 학생에게 알리는 글을 다루는 도메인

- 관리자: 전체 공지(모든 로그인 사용자 대상) 등록·수정·삭제, 모든 분반 공지 관리
- 운영진: 자기 분반 공지 등록·수정·삭제 (작성자 무관 - 팀 공동 관리)
- 학생·운영진: 전체 공지 + 소속 분반 공지 열람 (보관 분반 포함)
- 비소속 부원: 전체 공지만 열람

## 2. 엔티티 (DB 스키마)

테이블 `notices` · 패키지 `notice` (확정 스키마: [db/schema.md](../db/schema.md) 2절, Flyway `V2__notices.sql`)

| 필드 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | Long | PK | |
| cohort | Cohort (N:1, LAZY) | nullable | NULL = 전체 공지, 값 = 분반 공지 |
| author | User (N:1, LAZY) | not null | 작성자 - 등록 시 요청자 본인으로 고정, 변경 불가 |
| title | String | not null, 최대 200자 | 공지 제목 |
| content | String | not null, 최대 10000자 | 공지 내용 - 자유 텍스트 |
| pinned | boolean | not null, 기본 false | 필독 - 목록 최상단 고정 |
| createdAt | Instant | not null | 등록 시각(UTC). 수정 시각 열 없음 (design.md 결정 8) |

## 3. API 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 28 | 공지 목록 | `GET /api/notices` | `@LoginOnly` - 관리자 전부 / 부원 전체 + 소속 분반 | 200 목록(필독 먼저 → 최신순, 빈 배열 가능) | 401 |
| 29 | 공지 상세 | `GET /api/notices/{noticeId}` | `@LoginOnly` - 분반 공지는 소속자·관리자 | 200 단건 | 403(비소속의 분반 공지), 404 |
| 30 | 전체 공지 등록 | `POST /api/notices` | `@AdminOnly` | 201 + Location | 400(입력 검증), 403 |
| 31 | 분반 공지 등록 | `POST /api/cohorts/{cohortId}/notices` | `@CohortRole(OPERATOR)` | 201 + Location(`/api/notices/{id}`) | 400, 403, 404(분반 없음), 409(보관 분반) |
| 32 | 공지 수정 | `PUT /api/notices/{noticeId}` | `@LoginOnly` + 서비스: 전체 → 관리자 / 분반 → 운영진 이상 | 200 (전체 교체) | 400, 403, 404, 409(보관 분반) |
| 33 | 공지 삭제 | `DELETE /api/notices/{noticeId}` | 같음 | 204 | 403, 404, 409(보관 분반) |

- 번호는 기존 API 목록(#1~#27)에 이어 부여. 에러 응답 형식은 guide/design.md 3절 공통
- 판정 순서(#32·#33): 공지 404 → (분반 공지) 보관 409 → 권한 403

**요청·응답 본문 (record DTO)**

- 응답은 여섯 엔드포인트 모두 `NoticeResponse {id, title, content, pinned, cohort{id, name} | null, author{id, name, title}, createdAt, canEdit, canDelete}` 하나
  - `cohort` 가 null 이면 전체 공지. `author` 는 `UserSummary`(loginId 없음) - 전체 공지 작성자(관리자)의 직책은 해구르르, 분반 공지 작성자는 그 분반 역할 기준
  - `canEdit`·`canDelete`(같은 값): 전체 공지 → 요청자가 관리자 / 분반 공지 → 분반 ACTIVE 이고 요청자가 그 분반 운영진 이상(관리자 포함). 프론트는 이 값만 보고 버튼 분기
- 등록·수정 요청 `NoticeCreateRequest` / `NoticeUpdateRequest` - 필드·검증 동일: `title`(필수, 최대 200자), `content`(필수, 최대 10000자), `pinned`(선택, 생략 시 false)

## 4. 구현 시 주의 (springdoc에 담기지 않는 내부 규약)

- 목록 쿼리 2개: 관리자 `findAllWithAuthor()` / 부원 `findAllVisibleWithAuthor(cohortIds)` - `cohort is null or cohort.id in :cohortIds`. 소속이 없으면 서비스가 `[-1]` 로 바꿔 넘김(빈 IN 절 회피)
- 두 목록·단건 모두 author fetch join + cohort left join fetch(`@Query`) - 응답에 작성자·분반 이름이 실리므로 트랜잭션 안에서 LAZY 종결
- 상세·수정·삭제의 분반 판정은 `CohortAuthorizer.isAllowed(user, cohortId, STUDENT|OPERATOR)` 재사용 - 권한 규칙 단일 출처
- 응답 조립 `NoticeResponseAssembler` - 목록이 여러 분반에 걸치므로 `findAllByCohortIdInWithUser(cohortIds)` 1회로 작성자 직책·요청자 역할을 얻음. canEdit 의 운영진 판정은 `CohortAuthorizer.canManage` 재사용
- 쓰기 3개는 분반 공지일 때만 `cohort.ensureActive()` - 전체 공지는 분반 상태와 무관
- springdoc: `@Tag("Notice")` + `@Operation(summary)` 최소한만. canEdit·canDelete 규칙은 `@Schema(description)`에 명시
