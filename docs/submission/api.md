# Submission - API 명세

> 역할: 제출 도메인의 엔티티와 API 계약. 구현 완료 후 계약의 원본은 springdoc(OpenAPI)
> 결정 배경: [design.md](design.md) · 공통 규약(에러 형식·URL 규칙): [guide/design.md](../guide/design.md) 3절·4절 · 스키마 원본: [db/schema.md](../db/schema.md)

## 1. 도메인 역할

Submission(제출) = 학생이 과제에 내는 제출물과 그 이력을 다루는 도메인

- 학생: 자기 분반 과제에 제출(재제출 무제한, 이력 append-only)·자기 이력 열람
- 운영진: 과제별 현황판(제출/미제출/지각)·모든 제출물 열람·파일 다운로드
- 제출물은 항상 과제(Assignment) 하나에 소속 - 타인 제출물·다른 분반 제출물은 404(존재 비노출)
- 제출 형식 = 본문(코드 텍스트 또는 zip, 택1) + 링크(선택) - 최소 1개 필요 (schema.md 결정 3)
- 상태 4종(미제출/제출/제출(추가)/지각)은 저장하지 않고 이력에서 계산 (schema.md 3절)

## 2. 엔티티 (DB 스키마)

테이블 `submissions` · 패키지 `submission` (확정 스키마: [db/schema.md](../db/schema.md) 2절)

| 필드 | 타입 | 제약 | 설명 |
|---|---|---|---|
| id | Long | PK | |
| assignment | Assignment (N:1, LAZY) | not null | 제출 대상 과제 |
| user | User (N:1, LAZY) | not null | 제출자 - Enrollment이 아닌 사용자 직접 참조. 소속 해제돼도 제출물은 남는다 (guide 8절 결정 9) |
| codeText | String | 선택, 최대 100000자 | 본문·코드 - 붙여넣은 코드. 본문은 코드/파일 중 택1(서비스 강제) |
| language | String | 선택, 최대 30자 | 본문·코드 - 제출 언어(예: Python 3). 표시·하이라이팅용, 채점 무관 |
| fileName | String | 선택, 최대 255자 | 본문·파일 - 업로드한 zip의 원본 파일명 |
| storedPath | String | 선택, 최대 500자 | 본문·파일 - 저장 키 `submissions/{UUID}.zip`. P1은 로컬 디스크, S3 전환 시에도 키만 |
| fileSize | Long | 선택 | 본문·파일 - 크기(byte) |
| linkUrl | String | 선택, 최대 2048자 | 링크 - GitHub·배포 URL 등 |
| submittedAt | Instant | not null | 제출 시각(UTC) = 서버 수신 시각. dueAt과 비교해 지각 계산 - 지각 플래그 열 없음 |
| score · mentorComment | Integer · String | P2 준비 | P1에서는 항상 NULL - DTO에 미노출 |

- 수정·삭제 없음(append-only) - 도메인 메서드는 `create`뿐
- 상태 enum `SubmissionStatus {NOT_SUBMITTED, SUBMITTED, SUBMITTED_EXTRA, LATE}` - 엔티티 열이 아니라 계산 결과 (schema.md 3절: onTime·late 존재 여부 조합)

## 3. API 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 18 | 제출 생성 | `POST /api/cohorts/{cohortId}/assignments/{assignmentId}/submissions` | 분반 소속 누구나 | 201 + Location | 400(형식 검증·파일 제한), 403, 404(분반·과제), 409(보관 분반) |
| 19 | 내 제출 이력 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/submissions/my` | 분반 소속 누구나 | 200 목록(최신순, 빈 배열 가능) | 403, 404 |
| 20 | 제출 상세 열람 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/submissions/{submissionId}` | 본인 또는 운영진 이상 | 200 단건 | 403, 404(타인 제출물·다른 분반 체인 불일치 포함) |
| 21 | 제출 파일 다운로드 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/submissions/{submissionId}/file` | 본인 또는 운영진 이상 | 200 zip 바이너리 + Content-Disposition(원본 파일명) | 403, 404(파일 없는 제출 포함) |
| 22 | 제출 현황판 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/status-board` | 운영진 이상 | 200 학생별 행 목록 | 403, 404 |

- 번호는 기존 API 목록(#1~#17)에 이어 부여. 에러 응답 형식은 guide/design.md 3절 공통
- 권한 어노테이션: #18~#21 = `@CohortRole(STUDENT)`, #22 = `@CohortRole(OPERATOR)` (ADMIN은 자동 통과)
- #20·#21의 "본인 또는 운영진"은 어노테이션이 아니라 서비스 판정 - 학생이 타인 submissionId 접근 시 404 (guide 결정 11 존재 비노출)
- 지각 허용: 마감 후에도 #18은 성공 - 차단 없음, 응답의 `late`로 표시 (flows 3절-1 확정)

**기존 API 확장 (같은 슬라이스에서 변경)**

- #13 목록·#14 상세: `AssignmentResponse`에 `myStatus`(요청자의 상태 4종 - 비소속 ADMIN은 null), `submissionCount`(제출 이력 총 건수 - OPERATOR/ADMIN만 값, STUDENT는 null) 추가. 조립은 신설 `AssignmentResponseAssembler` (design.md 결정 5)
- #17 과제 삭제: 서비스가 파일 → submissions → assignment 순서로 연쇄 삭제 (schema.md 4절). FE 경고 건수는 `submissionCount` 사용 (design.md 결정 9)

**요청·응답 본문 (record DTO)**

- #18 요청 = `multipart/form-data` 고정 (design.md 결정 2)
  - `request` 파트(JSON): `SubmissionCreateRequest {codeText?(최대 100000자), language?(최대 30자), linkUrl?(최대 2048자)}`
  - `file` 파트(선택): zip 1개, 최대 20MB
  - 형식 검증(서비스): 본문(codeText/file) 둘 다 있으면 400 · 본문·링크 모두 없으면 400 · zip 외 확장자 400 · language는 codeText 있을 때만 의미(파일 제출에 실리면 400)
- `SubmissionResponse {id, user: UserSummary, codeText, language, fileName, fileSize, linkUrl, submittedAt, late}` - #18·#20 응답. `late` = `submittedAt > dueAt` 서버 판정값
- `SubmissionSummary {id, language, fileName, fileSize, linkUrl, submittedAt, late}` - #19 목록 응답. codeText 제외(이력 20건 × 코드 전문 수신 방지) - 코드 확인은 #20
- `StatusBoardRow {user: UserSummary, status, submissionCount, lastSubmittedAt, latestSubmissionId}` - #22 응답. 행 = 현재 STUDENT Enrollment 명단(운영진 먼저 아님 - 이름순), 소속 해제 학생은 제외·데이터는 유지 (schema.md 3절). `latestSubmissionId`(제출 없으면 null) = 운영진이 #20·#21로 진입하는 열람 키 *(2026-08-26 추가, BE PR #15 - FE 연동 중 발견한 계약 누락. "최신 제출 = 대표")*

## 4. 구현 시 주의 (springdoc에 담기지 않는 내부 규약)

- 스코프 조회는 체인 전부: assignment는 `findByIdAndCohortId`, submission은 `findByIdAndAssignmentId` - 손자 리소스라 두 단계 모두 필수 (guide 4절)
- #18은 서비스 첫 줄 `cohort.ensureActive()` - 보관 분반 409. #19~#22는 조회라 보관 분반에서도 200 (열람 유지)
- 파일 저장 순서: 검증 → 디스크 저장 → DB insert. DB 실패 시 저장한 파일 삭제 시도(고아 파일 방지). 삭제 연쇄는 역순 - 파일 먼저 지워야 RESTRICT 안전망이 성립 (schema.md 4절)
- 저장 루트는 `ondal.upload.dir` 프로퍼티 - 테스트는 `@TempDir` 주입, local 기본값 `./uploads`
- #21은 `ResponseEntity<Resource>` 반환 - "ResponseEntity 금지" 규약(guide 4절)의 명시적 예외(바이너리 + Content-Disposition 헤더 필요). 파일명은 RFC 5987 인코딩(한글 파일명)
- 상태 계산은 `SubmissionStatus.of(onTime, late)` 정적 메서드 한 곳 - Assembler(#13·#14)와 현황판(#22)이 공용
- #22 쿼리: 과제의 전체 제출 1회 조회 후 서비스에서 사용자별 그룹핑 (P1 규모, N+1 금지). `idx_submissions_assignment_user` 활용
- multipart 한도는 application.yml `spring.servlet.multipart.max-file-size: 20MB` + 서비스 이중 확인 - Spring 한도 초과는 400 INVALID_INPUT으로 변환(전역 핸들러에 `MaxUploadSizeExceededException` 추가)
- springdoc: `@Tag("Submission")` + `@Operation(summary)` 최소한만. `late`·`myStatus` 판정 규칙은 `@Schema(description)`에 명시
