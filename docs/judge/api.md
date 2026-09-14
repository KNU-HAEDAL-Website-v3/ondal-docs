# 자동 채점 API 명세

> 결정과 이유는 [design.md](design.md). 번호는 기존 목록(#1~#46)에 이어 #47~#51. 에러 형식은 guide/design.md 3절 공통

## 1. 엔드포인트

| # | 기능 | 메서드 · 경로 | 권한 | 성공 | 주요 실패 |
|---|---|---|---|---|---|
| 47 | 채점 설정·테스트케이스 조회 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/judge` | 운영진 이상 | 200 `JudgeConfigResponse` (비공개 케이스 포함) | 403, 404 |
| 48 | 채점 설정·테스트케이스 저장 (통째 교체) | `PUT /api/cohorts/{cohortId}/assignments/{assignmentId}/judge` | 운영진 이상 | 200 `JudgeConfigResponse` (+ `affectedSubmissions`, `rejudgeQueued`) | 400(제한 범위·케이스 수·빈 입력), 403, 404, 409(보관 분반) |
| 49 | 출제 도구 실행 - 기대 출력 채우기·출제 검증 | `POST /api/cohorts/{cohortId}/assignments/{assignmentId}/judge/run` | 운영진 이상 | 200 `JudgeRunResponse` (동기, 최대 30초) | 400(언어 매핑 없음·입력 개수), 403, 404, 409(보관 분반), 503(엔진 사용 불가 - `JUDGE_UNAVAILABLE`) |
| 50 | 공개 케이스(예시) 조회 | `GET /api/cohorts/{cohortId}/assignments/{assignmentId}/judge/samples` | 분반 소속 누구나 | 200 `JudgeSamplesResponse` | 403, 404 |
| 51 | 재채점 (이 과제의 코드 제출 전부) | `POST /api/cohorts/{cohortId}/assignments/{assignmentId}/judge/rejudge` | 운영진 이상 | 202 `{queued}` | 403, 404, 409(보관 분반 · 테스트케이스 0개) |

- 권한 어노테이션: #47·#48·#49·#51 = `@CohortRole(OPERATOR)`, #50 = `@CohortRole(STUDENT)` (ADMIN 자동 통과)
- 채점 결과 조회 API 는 따로 없다 - 제출 상세(#20)·내 이력(#19)·현황판(#22)·제출 생성(#18) 응답에 실린다(2절). FE 는 #20 을 폴링
- `judge/run`(#49)은 저장하지 않는다 - 순수 실행. 남용 방지: 운영진 이상 + 요청당 입력 최대 20개 + 소스 100000자

## 2. 기존 API 확장

- #13 목록·#14 상세 `AssignmentResponse` + `judgeEnabled: boolean`(테스트케이스 1개 이상) - 목록 행 "자동 채점" 배지, 상세의 예시 절 표시 여부
- #18 생성·#20 상세 `SubmissionResponse` + `judge: JudgeResult | null` - CODE 제출이고 자동 채점 문제면 값(직후는 `status: PENDING`), 그 외 null
- #19 이력 `SubmissionSummary` + `judgeStatus: PENDING|RUNNING|DONE|ERROR | null`, `verdict: Verdict | null` - 표의 "채점 결과" 열
- #22 현황판 `StatusBoardRow` + `latestVerdict: Verdict | null`, `latestJudgeStatus` - 최신 제출의 판정
- #17 과제 삭제 연쇄에 `judge_results`·`test_cases` 포함 (design.md 결정 16)

## 3. 요청·응답 본문 (record DTO)

- `JudgeConfigResponse {enabled, engineAvailable, timeLimitMs, memoryLimitMb, defaultTimeLimitMs: 2000, defaultMemoryLimitMb: 256, maxTimeLimitMs, maxMemoryLimitMb, maxTestCases, languages: string[], testCases: TestCaseResponse[], affectedSubmissions, rejudgeQueued}`
  - `engineAvailable` = 엔진 연결 여부 - false(off) 면 FE 는 "저장은 되지만 채점은 엔진 연결 후" 배너, 실행(#49)은 503. `languages` = 지원 언어(FE 셀렉트 문자열)
  - `testCases[] = {id, position, input, expectedOutput, isPublic}` - position 순. 기본값·상한은 폼 안내용(서버 설정을 그대로 내려 FE 하드코딩 방지)
  - `affectedSubmissions` = 이 과제의 CODE 제출 건수(재채점 대상) - #47·#48 모두. `rejudgeQueued` = #48 에서 실제로 큐에 넣은 건수(요청 `rejudge=false` 면 0)
- #48 요청 `JudgeConfigRequest {timeLimitMs?(null = 기본), memoryLimitMb?(null = 기본), testCases: [{input, expectedOutput, isPublic}], rejudge: boolean}`
  - 검증(400): 케이스 0~50개(0개 = 자동 채점 해제 - 기존 `judge_results` 는 남김), `input`·`expectedOutput` 각 최대 64KB(빈 문자열 허용 - 입력 없는 문제), 제한은 100ms~`MAX`·16MB~`MAX`
  - 통째 교체 - 기존 행 삭제 후 재삽입(id 는 바뀔 수 있음, FE 는 id 에 의존 금지)
- #49 요청 `JudgeRunRequest {language, sourceCode, inputs: string[], expectedOutputs?: string[] | null, timeLimitMs?, memoryLimitMb?}` → `JudgeRunResponse {compileOutput | null, runs: [{index, stdout, stderr, verdict | null, timeMs, memoryKb}]}`
  - `expectedOutputs` 가 있으면 `verdict` 를 채움(비교 규칙 = design.md 결정 3), 없으면 null(기대 출력 채우기 용도)
  - 제한을 안 주면 폼의 현재 값이 아니라 기본값 - FE 가 폼 값을 명시해서 보낸다
- `JudgeSamplesResponse {enabled, timeLimitMs, memoryLimitMb, languages: string[], samples: [{position, input, expectedOutput}]}` - 공개 케이스만, 학생 과제 상세 예시 절
- `JudgeResult {status: PENDING|RUNNING|DONE|ERROR, verdict: Verdict | null, passedCases, totalCases, maxTimeMs | null, maxMemoryKb | null, compileOutput | null, cases: JudgeCaseResult[], judgedAt | null}`
  - `Verdict = ACCEPTED | WRONG_ANSWER | TIME_LIMIT | MEMORY_LIMIT | RUNTIME_ERROR | COMPILE_ERROR | JUDGE_ERROR` - `status = DONE` 일 때만 값, `ERROR` 면 `JUDGE_ERROR`
  - `cases[] = {position, verdict, timeMs, memoryKb, isPublic, input?, expectedOutput?, actualOutput?}` - 공개 케이스만 세 텍스트가 실림(각 4KB 로 잘라 `truncated: true`), 비공개는 null. **운영진에게도 같은 규칙**(비공개 입력을 보려면 #47)
  - `compileOutput` 은 8KB 로 잘라 보냄
- 언어 문자열은 FE 셀렉트 값 그대로(`"C" | "C++" | "Java" | "Python 3" | "JavaScript" | "TypeScript"`) - 서버 `ondal.judge.languages` 표에 없는 값은 400 `INVALID_INPUT`("이 언어는 자동 채점을 지원하지 않습니다")

## 4. 구현 시 주의 (springdoc 밖 규약)

- 패키지 `judge` 수직 슬라이스: `entity(TestCase, JudgeResult)` → `repository` → `engine(JudgeEngine, Judge0Engine, FakeJudgeEngine)` → `service(JudgeConfigService, JudgeService, OutputComparator, JudgeWorker)` → `controller(JudgeController)` → `dto`
- 제출 생성(#18) 훅: `SubmissionService.create` 끝에서 `judgeService.enqueueIfJudged(submission)` - CODE 이고 과제에 케이스가 있으면 `judge_results(PENDING)` 저장 후 이벤트 발행(`@TransactionalEventListener(AFTER_COMMIT)` → 워커). 커밋 전에 워커가 돌지 않게 하는 이유: 제출 행이 보이기 전에 채점이 끝나는 경합 방지
- 워커: `@Async` 풀(스레드 2, 큐 무제한) 1건 = 제출 1건 - Judge0 batch POST(케이스 수만큼, 최대 `MAX_SUBMISSION_BATCH_SIZE` 20 → 케이스 50개면 3회 나눔) → `RUNNING` → 1초 폴링 → 집계 → `DONE`. 연결 실패·큐 초과(HTTP 503/422)는 지수 백오프(2s → 4s → ... 최대 10분) 후 `ERROR`. 서버 재시작 시 `PENDING/RUNNING` 잔여는 기동 후 재큐잉(`ApplicationReadyEvent`)
- 비교기 `OutputComparator.matches(expected, actual)`: `\r\n → \n`, 각 줄 `stripTrailing()`, 끝의 빈 줄 제거, 그 뒤 `equals`. 단위 테스트로 규칙 고정
- Judge0 요청 필드: `source_code`(base64), `language_id`, `stdin`(base64), `cpu_time_limit`(초, = timeLimitMs/1000), `cpu_extra_time 0.5`, `wall_time_limit = cpu × 2 + 1`(최소 1초 - Judge0 1.13 보안 규칙), `memory_limit`(KB), `max_processes_and_or_threads 60`, `enable_network false`, `redirect_stderr_to_stdout false`, `max_file_size 1024`. `expected_output` 은 보내지 않음. 응답 `fields=token,status,stdout,stderr,compile_output,time,memory,exit_code,message`, `base64_encoded=true`
- Judge0 status → verdict: 3 Accepted·4 Wrong Answer → **둘 다 우리 비교기로 재판정**(expected 를 안 보내므로 4 는 나오지 않음) / 5 → TIME_LIMIT / 6 → COMPILE_ERROR / 7~12 → RUNTIME_ERROR(단, 메모리 초과는 Judge0 가 SIGSEGV·NZEC 로 보고할 수 있어 `memory >= limit` 이면 MEMORY_LIMIT 로 보정) / 13·14·타임아웃 → JUDGE_ERROR
- 시간·메모리 집계: 케이스 최대값. Judge0 `time` 은 초(float) → ms 반올림, `memory` 는 KB
- 인증: 모든 Judge0 요청에 `X-Auth-Token`(`ondal.judge.judge0.token`, prod 는 .env). `engine=judge0` 인데 url·token 이 비면 기동 거부. prod 기본 `engine=off`(DisabledJudgeEngine) - 워커는 `available()=false` 면 PENDING 을 건드리지 않고, 엔진 연결 뒤 기동 시 재큐잉(ApplicationReadyEvent)
- 워커 트랜잭션: AFTER_COMMIT 리스너 안에서는 기본 전파(REQUIRED)가 끝난 트랜잭션에 참여해 커밋되지 않으므로 `TransactionTemplate(REQUIRES_NEW)` 로 RUNNING·DONE 저장. 엔진 호출은 트랜잭션 밖
- 저장 케이스 결과(`case_results` jsonb) = `[{position, verdict, timeMs, memoryKb, actualOutput(4KB), truncated}]` - 공개 여부·입력·기대 출력은 응답 조립 때 test_cases 에서 position 으로 찾는다(케이스 교체 뒤 결과는 재채점 전까지 stale 일 수 있음)
- FakeJudgeEngine(local·test): 소스에 `// judge: TLE` 같은 지시 주석이 있으면 그 판정, 없으면 stdin 을 그대로 echo 한 것을 stdout 으로(= 기대 출력을 입력과 같게 두면 ACCEPTED). 테스트에서 케이스별 판정을 결정적으로 만드는 장치
- 삭제 연쇄: `AssignmentService.delete` → `judgeService.deleteAllOf(assignmentId)`(judge_results → test_cases) 를 제출 삭제 앞뒤에 순서대로(design.md 결정 16)
- springdoc: `@Tag("Judge")`, 판정·비교 규칙은 `@Schema(description)` 에 명시
