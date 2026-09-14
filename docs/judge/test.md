# 자동 채점 테스트 계획

## 1. 테스트 클래스·픽스처

- `judge/JudgeApiTest` - `JudgeController`(#47~#51)와 1:1, `extends ApiTestSupport`. 엔진은 `FakeJudgeEngine`(test 프로필 `ondal.judge.engine: fake`) - 소스의 지시 주석으로 판정을 결정(api.md 4절)
- `judge/OutputComparatorTest` - 순수 단위 테스트, 비교 규칙 고정
- `judge/Judge0EngineTest` - `MockRestServiceServer`(또는 WireMock)로 Judge0 HTTP 계약 고정: batch POST 본문·토큰 폴링·status 매핑·백오프. **실제 Judge0 는 테스트에 쓰지 않는다**(Testcontainers 로 Judge0 를 띄우는 것은 cgroup 의존 때문에 불가)
- 기존 `submission/SubmissionApiTest`·`assignment/AssignmentApiTest` 에 확장분: `judge` 필드·`judgeEnabled`·삭제 연쇄
- 비동기 워커는 테스트에서 동기 실행(`SyncTaskExecutor` 테스트 설정) - 제출 직후 결과를 바로 단언

## 2. 케이스

- 설정(#47·#48)
  - STUDENT → 403 / OPERATOR 저장 → 200, position 순 재조회 / 비공개 케이스도 #47 에는 실림
  - 케이스 51개·입력 64KB 초과·시간 상한 초과·메모리 하한 미만 → 400 / 0개 저장 → `enabled: false`, 기존 judge_results 유지
  - 보관 분반 PUT → 409 / restore 후 → 200
  - 기존 CODE 제출 2건 + `rejudge: true` → `affectedSubmissions: 2`, `rejudgeQueued: 2`, 결과가 새 케이스 기준으로 갱신 / `rejudge: false` → `rejudgeQueued: 0`, 결과 불변
- 실행(#49)
  - `expectedOutputs` 없이 → `verdict: null`, stdout 채워짐 / 있으면 케이스별 verdict / 매핑 없는 언어 → 400 / 입력 21개 → 400
- 예시(#50)
  - STUDENT 200 - 공개 케이스만, 비공개 입력 미노출 / 케이스 0개 → 빈 배열 / 비소속 → 403
- 채점 파이프라인
  - 자동 채점 과제에 CODE 제출 → 201 + `judge.status: PENDING` → (동기 워커) #20 `DONE`·`verdict: ACCEPTED`·`passedCases = totalCases`
  - 지시 주석 `WA`(2번째 케이스) → `WRONG_ANSWER`, `passedCases: 1`, `cases[1].verdict: WRONG_ANSWER`, 공개 케이스만 `actualOutput` 있음
  - `TLE`·`RE`·`CE` → 각 verdict. CE 는 `cases` 비어 있고 `compileOutput` 있음
  - FILE·LINK 제출 → `judge: null` / 케이스 없는 과제의 CODE 제출 → `judge: null`
  - #19 `judgeStatus`·`verdict`, #22 `latestVerdict` 값 확인 / 제출자 아닌 학생 #20 → 여전히 404
  - 엔진 예외(Fake 에 `// judge: ERROR`) → `status: ERROR`, `verdict: JUDGE_ERROR`, 제출 자체는 201
- 재채점(#51)
  - OPERATOR → 202 `{queued: N}` / 케이스 0개 → 409 / 보관 → 409
- 삭제 연쇄
  - 케이스·결과 있는 과제 DELETE → 204, `test_cases`·`judge_results` 0행
- 비교기 단위
  - 끝 공백·`\r\n`·마지막 빈 줄 차이 → 일치 / 중간 공백·줄 순서 차이 → 불일치 / 빈 기대 출력 vs 빈 출력 → 일치

## 3. 시더

- `LocalDataSeeder`: 1차시 과제(A+B)에 테스트케이스 3개(첫 번째 공개) + 제한 기본값. student1 의 기존 제출은 시더에서 `ACCEPTED` 결과를 직접 저장(엔진 호출 없음) - FE 가 배지·결과 상세를 바로 확인. FE mock 동일
