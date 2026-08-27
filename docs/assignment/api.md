# Assignment - API 명세

> 역할: 과제 도메인의 엔티티와 API 계약. 구현 완료 후 계약의 원본은 springdoc(OpenAPI)
> 결정 배경: [design.md](design.md) · 공통 규약(에러 형식·URL 규칙): [guide/design.md](../guide/design.md) 3절·4절 · 스키마 원본: [db/schema.md](../db/schema.md)

## 1. 도메인 역할

Assignment(과제) = 운영진이 분반에 내는 과제를 다루는 도메인

- 운영진: 자기 분반에 과제 등록·수정·삭제
- 수강생: 자기 분반의 과제 목록·상세 조회
- 과제는 항상 분반(Cohort) 하나에 소속 - 다른 분반의 과제는 존재 자체를 비노출(404)
- 과제는 차시 번호(선택)로 묶어 표시 - 차시 밖 과제도 허용
- 제출과 상태 표시(미제출/제출/지각)는 다음 슬라이스(Submission) 범위 - 이 슬라이스는 과제 CRUD까지만

## 2. 엔티티 (DB 스키마)

테이블 `assignments` · 패키지 `assignment` (확정 스키마: [db/schema.md](../db/schema.md) 2절)

| 필드 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | Long | PK | |
| cohort | Cohort (N:1, LAZY) | not null | 과제가 속한 분반 |
| problemNo | Integer | not null, 전역 유일, 1000 이상 | 문제 번호 - 비우면 자동 채번(최대+1), 직접 지정 가능. 수정 허용, 중복 409 (design.md 결정 7) |
| sessionNo | Integer | 선택 | 차시 번호 - 운영진 자유 입력(중복·건너뜀 허용), 차시 밖 과제는 NULL |
| title | String | not null, 최대 200자 | 과제 제목 |
| description | String | 선택, 최대 10000자 | 과제 내용 - 문제 링크를 포함한 자유 텍스트 |
| dueAt | Instant | not null | 마감 시각(UTC). 마감 후 제출은 차단 아닌 지각 표시. 마감 수정 시 지각 판정도 새 마감 기준으로 재계산 |
| createdAt | Instant | not null | 등록 시각(UTC) |

## 3. API 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 13 | 과제 목록 조회 | `GET /api/cohorts/{cohortId}/assignments` | 분반 소속 누구나 | 200 목록(차시 오름차순 → 등록순, 차시 없음은 마지막. 빈 배열 가능) | 403, 404(분반 없음) |
| 14 | 과제 상세 조회 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}` | 분반 소속 누구나 | 200 단건 | 403, 404(분반·과제 없음, 다른 분반의 과제) |
| 15 | 과제 등록 | `POST /api/cohorts/{cohortId}/assignments` | 운영진 이상 | 201 + Location | 400(입력 검증), 403, 404(분반 없음), 409(보관 분반·문제 번호 중복) |
| 16 | 과제 수정 | `PUT /api/cohorts/{cohortId}/assignments/{assignmentId}` | 운영진 이상 | 200 (전체 교체) | 400, 403, 404, 409(보관 분반·문제 번호 중복) |
| 17 | 과제 삭제 | `DELETE /api/cohorts/{cohortId}/assignments/{assignmentId}` | 운영진 이상 | 204 | 403, 404, 409(보관 분반) |

- 번호는 기존 API 목록(#1~#12)에 이어 부여. 에러 응답 형식은 guide/design.md 3절 공통
- 권한 어노테이션: #13·#14 = `@CohortRole(STUDENT)`, #15~#17 = `@CohortRole(OPERATOR)` (ADMIN은 자동 통과)

**요청·응답 본문 (record DTO)**

- 응답은 다섯 엔드포인트 모두 `AssignmentResponse {id, problemNo, sessionNo, title, description, dueAt, createdAt, myStatus, submissionCount}` 하나 (myStatus·submissionCount는 제출 슬라이스 확장 - submission/api.md 3절)
- 등록·수정 요청은 `AssignmentCreateRequest` / `AssignmentUpdateRequest` - 필드·검증 동일: `problemNo`(선택 - 비우면 자동 채번, 지정 시 1000 이상·중복 409 - design.md 결정 7), `sessionNo`(선택, 1 이상 정수 - design.md 결정 6), `title`(필수, 최대 200자), `description`(선택, 최대 10000자), `dueAt`(필수, 과거 시각 허용 - design.md 결정 3)
- 수정(#16)의 `problemNo`도 전체 교체 규칙 - 비우면 자동 채번이 아니라 **기존 번호 유지** (번호가 이미 있는 리소스라 "비움 = 새로 받기"가 아님. 명세 예외라 springdoc에 명시)

## 4. 구현 시 주의 (springdoc에 담기지 않는 내부 규약)

- 스코프 조회: 하위 id는 반드시 `findByIdAndCohortId(id, cohortId)`로 조회 - `findById` 단독 호출 금지, 불일치는 404(존재 비노출)
- 쓰기 3개(#15~#17)는 서비스 첫 줄 `cohort.ensureActive()` - 보관 분반이면 409 `COHORT_ARCHIVED`
- 서비스 메서드는 `(Long cohortId, ...)` - cohortId를 첫 인자로 수신
- 엔티티는 `User.java` 패턴(정적 팩토리 `create`/`update`, setter 없음) + 인덱스 `idx_assignments_cohort`를 `@Table(indexes=...)`로 명시 (PostgreSQL은 FK 인덱스 자동 생성 없음)
- cohort 로드는 `CohortRepository` 주입 - assignment → cohort 단방향 의존 (enrollment 선례)
- Repository 메서드: `findAllByCohortIdOrderBySessionNoAscCreatedAtAsc` · `findByIdAndCohortId` - fetch join 불필요(응답에 연관 없음). PostgreSQL은 ASC 정렬에서 NULL을 마지막에 두므로 차시 없는 과제가 자연히 목록 뒤로 감
- 자동 채번: 서비스가 "현재 최대 problemNo + 1"(없으면 1000) 조회 후 저장 - 동시 등록 충돌은 `uk_assignments_problem_no`가 최후 방어 (schema.md 4절). 수동 지정·수정의 중복은 저장 전 존재 검사로 409 `CONFLICT` 선반환(제약 위반 500 방지)
- springdoc: `@Tag("Assignment")` + `@Operation(summary)` 최소한만. `dueAt`의 지각 재계산 규칙은 `@Schema(description)`에 명시
