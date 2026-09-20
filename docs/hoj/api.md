# HOJ P3 API 계약 - 채점 현황 · 사용자 페이지 · 랭킹 · 풀이 공개 · 정답 코드 · 북마크 · 실행

> 결정 13 (2026-09-20) 의 구현 계약. BE·FE 가 이 문서를 기준으로 병렬 구현한다. 응답 시각은 UTC ISO-8601, 에러는 `{code, message}` (guide/design.md 공통 규약).
> 사람 정보는 전부 `UserSummary {id, name, title}` - `loginId` 는 어디에도 내려가지 않는다 (타인 정보 비노출 원칙).

## 0. 용어

- 연습 제출 = `submissions.problem_id` 가 있는 제출 (HOJ, 분반 무관). 과제 제출 = `assignment_id` 가 있는 제출
- "푼 문제" = 그 문제에 대해 ACCEPTED 판정이 한 번이라도 있는 것 - **연습·과제 어느 쪽이든** (문제 목록의 `solved` 와 같은 규칙)
- "시도 중" = 채점된 제출은 있으나 아직 못 푼 문제
- 채점 현황 피드·다른 사람 풀이는 **연습 제출만** (PM 결정: 채점 현황은 HOJ 만)

## 1. 문제 목록·상세 확장 (기존 `GET /api/problems`, `GET /api/problems/{id}`)

`ProblemSummary` · `ProblemResponse` 에 추가:

| 필드 | 타입 | 뜻 |
|---|---|---|
| `solvedUserCount` | int | 이 문제를 푼 사람 수 (ACCEPTED 판정이 있는 사용자 수, 연습·과제 합산) |
| `submissionCount` | int | 채점된 제출 수 (`judge_results` 행 수, 연습·과제 합산) |
| `acceptedRate` | int \| null | ACCEPTED 비율(%) - 채점된 제출이 0이면 null |
| `myStatus` | `'SOLVED' \| 'ATTEMPTED' \| 'NONE'` | 나의 상태. `solved` 는 하위 호환으로 유지 |
| `bookmarked` | boolean | 내가 북마크했는지 |

`ProblemResponse` 에만 추가:

| 필드 | 타입 | 뜻 |
|---|---|---|
| `solutionLanguages` | string[] | 저장된 정답 코드의 언어 목록. **운영진 이상에게만** 값이 오고 그 밖에는 `[]` |

- 집계는 `ProblemService.findAll` 의 기존 "쿼리 3번" 자리에 group by 쿼리로 붙인다 (문제 수와 무관하게 상수 회 쿼리)
- 목록 필터(내 상태·북마크)는 FE 클라이언트 필터 - 목록은 전건 응답이므로 새 쿼리 파라미터 없음

## 2. 채점 현황 피드 - `GET /api/hoj/submissions`

- 권한 `@LoginOnly`. **연습 제출만**
- 쿼리: `problemId?` · `userId?` · `verdict?`(Verdict 7종) · `language?` · `size?`(기본 50, 최대 200) · `beforeId?`(커서 - 이 id 보다 작은 제출)
- 정렬 `id desc` (= 최신 제출 먼저)
- 응답

```json
{
  "items": [
    {
      "id": 501,
      "problem": { "id": 3, "problemNo": 2002, "title": "..." },
      "user": { "id": 7, "name": "홍길동", "title": "일반 수강생" },
      "language": "Python 3",
      "judgeStatus": "DONE",
      "verdict": "ACCEPTED",
      "passedCases": 12, "totalCases": 12,
      "maxTimeMs": 34, "maxMemoryKb": 9800,
      "submittedAt": "2026-09-20T01:02:03Z"
    }
  ],
  "nextBeforeId": 452
}
```

- `nextBeforeId` = 마지막 항목 id, 더 없으면 null. `codeText` 는 내려가지 않는다
- 채점 중(PENDING/RUNNING)은 `verdict: null`

## 3. 사용자 페이지 - `GET /api/hoj/users/{userId}`

- 권한 `@LoginOnly`. 누구나 누구의 페이지든 열람 (이름·활동만, 아이디 없음). 없는 사용자 404
- 응답

```json
{
  "user": { "id": 7, "name": "홍길동", "title": "일반 수강생" },
  "joinedAt": "2026-08-01T00:00:00Z",
  "rank": 3,
  "stats": { "solvedCount": 12, "attemptedCount": 2, "submissionCount": 40, "acceptedCount": 15, "acceptedRate": 37 },
  "languages": [ { "language": "Python 3", "count": 30 }, { "language": "C", "count": 10 } ],
  "solvedProblems": [ { "id": 3, "problemNo": 2002, "title": "...", "difficulty": 3 } ],
  "attemptedProblems": [ { "id": 9, "problemNo": 2010, "title": "...", "difficulty": 6 } ],
  "tagStats": [ { "tag": { "id": 1, "name": "구현" }, "solved": 5, "total": 20 } ],
  "activity": [ { "date": "2026-09-19", "count": 3 } ],
  "recentSubmissions": [ "2절 items 항목과 같은 모양, 최근 20건, 연습 제출만" ]
}
```

- `stats.solvedCount`·`attemptedCount`·`solvedProblems`·`attemptedProblems`·`tagStats` = 연습·과제 합산(푼 문제 규칙). `submissionCount`·`acceptedCount`·`acceptedRate`·`languages`·`recentSubmissions`·`activity` = **연습 제출만**
- `activity` = 최근 365일, KST 날짜 기준(`submitted_at at time zone 'Asia/Seoul'`), 제출 0인 날은 생략
- `rank` = 4절 랭킹의 순위(푼 문제 0개면 null)
- `solvedProblems`·`attemptedProblems` 는 `problemNo asc`, `tagStats` 는 태그 이름순(total 0 인 태그 생략)

## 4. 랭킹 - `GET /api/hoj/ranking`

- 권한 `@LoginOnly`. 쿼리 `cohortId?` (그 분반 수강생·운영진만) · `size?`(기본 100, 최대 200)
- 순위 기준: `solvedCount desc` → `lastSolvedAt asc`(같은 수면 먼저 도달한 사람) → `name asc`. 동점은 같은 순위(1,1,3). 푼 문제 0개는 제외
- 응답

```json
{
  "items": [ { "rank": 1, "user": { "id": 7, "name": "홍길동", "title": "일반 수강생" }, "solvedCount": 12, "submissionCount": 40, "lastSolvedAt": "2026-09-19T10:00:00Z" } ],
  "me": { "rank": 3, "solvedCount": 9 }
}
```

- `me` = 요청자 (푼 문제 0개면 null). `submissionCount` 는 연습 제출 수

## 5. 다른 사람 풀이 - `GET /api/problems/{id}/accepted-solutions`

- 권한 `@LoginOnly` + **요청자가 그 문제를 풀었거나 운영진 이상**. 아니면 403 `{code: "NOT_SOLVED"}` (PM 결정: 맞힌 사람만)
- 내용: 연습 제출 중 ACCEPTED 인 것을 **사용자당 최신 1건**, 요청자 본인 제외, `submittedAt desc`, 최대 50건. 쿼리 `language?`
- 응답

```json
[
  { "submissionId": 501, "user": { "id": 7, "name": "홍길동", "title": "일반 수강생" }, "language": "Python 3", "codeText": "...", "submittedAt": "...", "maxTimeMs": 34, "maxMemoryKb": 9800 }
]
```

## 6. 정답 코드(참고 풀이) - `GET/PUT /api/problems/{id}/solutions`

- 권한 둘 다 `@OperatorAnywhere` (운영진 이상). 학생에게는 존재 자체가 안 보인다 (`solutionLanguages` 도 `[]`)
- 테이블 `problem_solutions(id, problem_id, language, code_text, updated_by, updated_at)` + `uk_problem_solutions_problem_language(problem_id, language)` - V11
- 언어 값은 제출 언어와 같은 표기: `C`, `C++`, `Java`, `Python 3`, `JavaScript`, `TypeScript`
- `GET` 응답: `[ { "language": "Python 3", "codeText": "...", "updatedBy": { "id": 1, "name": "관리자", "title": "해구르르" }, "updatedAt": "..." } ]` (언어 이름순)
- `PUT` 본문: `{ "solutions": [ { "language": "Python 3", "codeText": "..." } ] }` - 전체 교체(빈 배열 = 모두 삭제). 최대 6개, 언어 중복 400, 코드 100,000자 이하. 응답 = GET 과 같음
- 문제 삭제 시 함께 삭제(서비스에서)
- **가져오기 연동**
  - 번들(`POST /api/problems/import`): `ImportProblem.solutions?: [{language, codeText}]` (선택, 최대 6) - `overwrite=true` 면 교체, 새 문제면 저장
  - 깃허브(`ProblemBankZipReader`): `solutions/sol.<ext>` 를 읽는다. 확장자 → 언어: `py→Python 3`, `c→C`, `cpp|cc→C++`, `java→Java`, `js→JavaScript`, `ts→TypeScript`. 그 밖의 확장자는 무시
  - 문제 은행 레포 `tools/build.py` 도 `solutions` 배열을 번들에 넣는다 (ondal-problems 변경)

## 7. 북마크 - `PUT/DELETE /api/problems/{id}/bookmark`

- 권한 `@LoginOnly`. 멱등. 응답 204. 테이블 `problem_bookmarks(user_id, problem_id, created_at)` pk(user_id, problem_id) - V11
- 목록·상세의 `bookmarked` 로 읽는다 (별도 GET 없음)

## 8. 내 입력으로 실행 - `POST /api/problems/{id}/run`

- 권한 `@LoginOnly` (기존 `/judge/run` 은 운영진 전용 그대로). `JudgeService.run` 재사용, DB 저장 없음
- 본문 `{ "language": "Python 3", "sourceCode": "...", "inputs": ["1 2"] }` - inputs 1~5개, 각 10,000자 이하, 코드 100,000자 이하. 문제의 허용 언어 밖이면 400 `INVALID_INPUT`
- 응답: 기존 `JudgeRunResponse` 와 같은 모양 (컴파일 출력 + 입력별 `stdout`·`stderr`·`status`·`timeMs`·`memoryKb`)
- 남용 방지: 사용자당 **분당 10회** (메모리 카운터), 초과 시 429 `{code: "TOO_MANY_REQUESTS"}`. 엔진 미연결 503 `JUDGE_UNAVAILABLE`

## 9. 에러 코드 추가

| 코드 | HTTP | 언제 |
|---|---|---|
| `NOT_SOLVED` | 403 | 5절 - 아직 못 푼 문제의 다른 사람 풀이 |
| `TOO_MANY_REQUESTS` | 429 | 8절 - 실행 한도 초과 |

## 10. FE 화면 (참고 - fe.md 없이 이 절로 갈음)

- HOJ 상단 메뉴: 문제 `/problems` · 채점 현황 `/problems/status` · 랭킹 `/problems/ranking` · 내 페이지 `/problems/users/{내 id}` · 태그 관리(관리자). 정적 경로가 `/problems/:problemId` 보다 먼저
- 문제 목록: 푼 사람·제출·정답률 열, 내 상태 필터(전체/안 푼/시도 중/해결/북마크), 북마크 별, 랜덤 문제(안 푼 문제 중)
- 문제 상세: 북마크, "다른 사람 풀이"(푼 사람·운영진만, 펼칠 때 조회, 언어 필터), 운영진 "정답 코드 보기"(언어 탭), "내 입력으로 실행"(예시 입력 1 미리 채움), 언어별 코드 템플릿, 마지막 언어 기억, Ctrl+Enter 제출, 내 기록의 "다시 편집"
- 문제 출제·수정 폼: "정답 코드" 절(언어 탭 + 편집기) - 문제 저장 뒤 `PUT /solutions`
- 채점 현황: 표 + 판정·언어 필터 + 문제 번호/이름 검색(클라이언트) + 채점 중이면 5초 자동 갱신 + "더 보기"(커서)
- 랭킹: 순위·이름·푼 문제·제출·마지막 정답, 분반 필터, 내 순위 강조
- 사용자 페이지: 헤더(이름·직책·가입·순위) · KPI 4장 · 언어 비율 · 푼 문제 격자(초록)·시도 중(빨강) · 활동 잔디(365일) · 태그 숙련도 · 최근 제출
- 마이페이지: "HOJ 내 페이지" 링크, 편집기 설정(글꼴 크기·탭 폭) - 브라우저 저장, 모든 편집기에 적용
