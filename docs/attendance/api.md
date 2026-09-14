# 출석부 - API 명세

> 역할: 차시·출석 도메인의 엔티티와 API 계약. 계약의 원본은 springdoc(OpenAPI) `Session`·`Attendance` 태그
> 결정 배경: [design.md](design.md) · 공통 규약(에러 형식·URL 규칙): [guide/design.md](../guide/design.md) 3절·4절 · 스키마 원본: [db/schema.md](../db/schema.md) 결정 12

## 1. 도메인 역할

- Session(차시) = 분반의 수업 회차 - 번호·날짜·제목. 출석 기록의 단위
- Attendance(출석) = 한 차시에서 한 수강생의 판정(출석/지각/결석) - 운영진이 표시. 기록이 없으면 "미확인"
- 운영진: 차시 등록·수정·삭제, 차시 명부에 출석 표시(일괄 포함)
- 학생: 자기 기록·출석률 열람. 차시 목록 열람

## 2. 엔티티 (DB 스키마)

테이블 `sessions` · `attendances` · 패키지 `attendance` (Flyway `V3__attendance.sql`)

| 엔티티 | 필드 | 타입 | 제약 | 설명 |
|---|---|---|---|---|
| Session | id | Long | PK | |
| | cohort | Cohort (N:1, LAZY) | not null | 차시가 속한 분반 |
| | sessionNo | Integer | not null, 분반 안 유일 | 차시 번호 - 과제의 `sessionNo` 와 같은 번호 체계(FK 없음) |
| | title | String | 최대 100자, nullable | 예: "포인터와 배열" |
| | heldOn | LocalDate | not null | 수업 날짜(KST 달력일) |
| | createdAt | Instant | not null | |
| Attendance | id | Long | PK | |
| | session | Session (N:1, LAZY) | not null | |
| | user | User (N:1, LAZY) | not null | 수강생 - (session, user) 유일 |
| | status | AttendanceStatus | not null | PRESENT / LATE / ABSENT |
| | checkedAt | Instant | not null | 표시(마지막 변경) 시각 |
| | checkedBy | User (N:1, LAZY) | not null | 표시한 운영진 |

## 3. API 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 34 | 차시 목록 | `GET /api/cohorts/{cohortId}/sessions` | `@CohortRole(STUDENT)` | 200 `[SessionResponse]` (날짜 → 번호 오름차순) | 403, 404 |
| 35 | 차시 등록 | `POST /api/cohorts/{cohortId}/sessions` | `@CohortRole(OPERATOR)` | 201 + Location | 400, 403, 404, 409(번호 중복 CONFLICT / 보관) |
| 36 | 차시 수정 | `PUT /api/cohorts/{cohortId}/sessions/{sessionId}` | `@CohortRole(OPERATOR)` | 200 (전체 교체) | 400, 403, 404, 409(번호 중복 / 보관) |
| 37 | 차시 삭제 | `DELETE /api/cohorts/{cohortId}/sessions/{sessionId}` | `@CohortRole(OPERATOR)` | 204 - 출석 기록 연쇄 삭제 | 403, 404, 409(보관) |
| 38 | 차시 출석 명부 | `GET /api/cohorts/{cohortId}/sessions/{sessionId}/attendances` | `@CohortRole(OPERATOR)` | 200 `AttendanceRosterResponse` | 403, 404 |
| 39 | 출석 표시 (일괄 upsert) | `PUT /api/cohorts/{cohortId}/sessions/{sessionId}/attendances` `{records: [{loginId, status\|null}]}` | `@CohortRole(OPERATOR)` | 200 - 갱신된 명부(#38 과 같은 모양) | 400(빈 목록·값), 403, 404(차시·수강생 아님), 409(보관) |
| 40 | 내 출석 | `GET /api/cohorts/{cohortId}/attendances/me` | `@CohortRole(STUDENT)` | 200 `MyAttendanceResponse` | 403, 404 |

- 번호는 기존 API 목록(#1~#33)에 이어 부여. 에러 응답 형식은 guide/design.md 3절 공통
- 쓰기(#35~#37, #39) 판정 순서: 소속(어노테이션) → 검증 400 → 분반 404 → 보관 409 → 차시 스코프 404 → 번호 중복·수강생 아님

**요청·응답 본문 (record DTO)**

- `SessionResponse {id, sessionNo, title, heldOn, createdAt, attendanceCount}` - `attendanceCount` = 기록 수(삭제 경고용)
- `SessionCreateRequest {sessionNo?, title?, heldOn}` / `SessionUpdateRequest {sessionNo, title?, heldOn}` - 등록 시 `sessionNo` 생략은 자동 채번(최대 + 1), 1 이상. `heldOn` `yyyy-MM-dd` 필수. `title` 100자
- `AttendanceMarkRequest {records: [{loginId, status}]}` - `records` 비면 400. `status` = `PRESENT | LATE | ABSENT | null`(null 은 기록 삭제). 같은 loginId 가 여러 번이면 마지막 값
- `AttendanceRosterResponse {session: SessionResponse, summary: AttendanceStats, rows: [AttendanceRow]}`
  - `AttendanceRow {user: UserResponse, status: AttendanceStatus|null, checkedAt: Instant|null, stats: AttendanceStats}` - `user` 는 loginId 포함(운영진만 보는 응답, 표시 요청에 loginId 가 필요). `stats` 는 이 학생의 분반 전체 누계
  - `AttendanceStats {present, late, absent, unchecked, rate}` - `rate` = 출석 ÷ (출석+지각+결석) × 100 정수, 판정 0건이면 null. 명부의 `summary` 는 이 차시 기준(unchecked = 명단 − 기록)
- `MyAttendanceResponse {summary: AttendanceStats, records: [{session: SessionResponse, status: AttendanceStatus|null, checkedAt: Instant|null}]}` - 차시 최신순(날짜 → 번호 내림차순). 관리자·운영진이 부르면 기록 없이 차시만(모두 null)

## 4. 구현 시 주의 (springdoc에 담기지 않는 내부 규약)

- 차시 스코프 조회 `findByIdAndCohortId(sessionId, cohortId)` - 불일치·부재 404. 출석 기록은 `findAllBySessionIdWithUser`, 분반 누계는 `findAllByCohortIdWithUser`(session.cohort.id 경유) 1회 조회 후 userId 로 그룹
- 명부 행 = `enrollmentRepository.findAllByCohortIdWithUser(cohortId)` 의 STUDENT 만, 이름순(현황판 선례 `SubmissionService.statusBoard`)
- 표시 대상 검증: loginId → User → 그 분반 STUDENT Enrollment 존재해야 함(운영진·미소속·모르는 loginId 는 404)
- upsert: `(session, user)` 기존 기록 있으면 status·checkedAt·checkedBy 갱신, 없으면 생성, status null 이면 삭제. 트랜잭션 하나
- 차시 삭제: `attendanceRepository.deleteAllBySessionId` → `sessionRepository.delete` (서비스 연쇄, DB FK 는 RESTRICT)
- 자동 채번: `max(sessionNo)` + 1 - 동시 등록 충돌은 `uk_sessions_cohort_no` 가 최후 방어 → 409 로 노출 (과제 문제 번호와 같은 방식)
- springdoc: `@Tag("Session")`·`@Tag("Attendance")` + `@Operation(summary)` 최소한만
