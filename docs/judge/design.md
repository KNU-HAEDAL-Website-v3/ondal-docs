# 자동 채점 (Judge0) 설계

> P2 필수 항목 (2026-09-14 PM 확정 - 백준 서비스 종료로 대체 수단 없음, mvp-scope 5절). 작성 2026-09-14.
> 이 문서 = 결정과 이유. API 는 [api.md](api.md), 화면은 [fe.md](fe.md), 테스트는 [test.md](test.md), 서버 구성은 [infra.md](infra.md)

## 1. 범위

- 목표: 운영진이 **입출력 문제를 5분 안에 출제**하고, 학생이 코드를 제출하면 **채점 결과(맞았습니다/틀렸습니다/시간 초과 등)가 자동으로** 붙는다
- 포함
  - 문제 = 기존 과제(Assignment) 그대로 + **채점 설정**(시간·메모리 제한) + **테스트케이스**(입력/기대 출력/공개 여부) - 테스트케이스가 1개 이상이면 "자동 채점 문제"
  - 채점 파이프라인: 코드 제출(#18, `type = CODE`) → 채점 큐 → Judge0 실행 → 판정 저장 → 학생·운영진 화면에 결과
  - 출제 도구: 표로 테스트케이스 편집, **정답 코드로 기대 출력 자동 채우기**, **출제 검증**(정답 코드가 모든 케이스를 통과하는지), 학생 화면 미리보기
  - 재채점: 테스트케이스·제한을 고치면 기존 제출을 다시 채점
  - 공개 케이스: 학생 과제 상세에 "예시 입력/출력"으로 표시, 채점 결과에서 실제 출력 비교 노출
- 제외 (P3 또는 미정)
  - 부분 점수·점수 자체(2026-09-14 결정: 점수 없음), 스페셜 저지(커스텀 체커)·인터랙티브 문제, 언어별 시간 보정, 표절 검사, 공개 문제 풀·리더보드(P3 OJ), 파일(zip)·링크 제출의 채점(수동 검토 유지)

## 2. 결정

1. **문제 = 과제 + 테스트케이스, 별도 problems 테이블 없음** - CLAUDE.md 원칙 1("OJ 문제 = 분반 없는 과제")의 P2 단계. `test_cases(assignment_id FK, position, input, expected_output, is_public)` + `assignments.time_limit_ms`·`memory_limit_mb`(NULL = 기본값). 테스트케이스 0개 = 지금처럼 제출만 받는 과제 → 기존 과제·제출 동작은 하나도 바뀌지 않는다
2. **채점 대상 = CODE 제출만** - FILE(zip)·LINK 는 실행 대상이 정해지지 않아 채점하지 않음. 자동 채점 문제라도 zip·링크 제출은 여전히 허용(운영진 수동 검토·코멘트) - 과제별 제출 형태 제한은 두지 않음(submission/design.md 결정 12 유지)
3. **판정은 우리 서버가, 실행만 Judge0** - Judge0 에 `expected_output` 을 보내지 않고 stdout 만 받아 BE 가 비교한다. 비교 규칙 = **각 줄 끝 공백 제거 + 마지막 빈 줄 제거 후 정확 일치**(백준 "일반" 채점과 같은 취지). 이유: 규칙을 우리가 소유(문서화·테스트 가능), 엔진을 바꿔도 판정이 같음
4. **판정 7종, 전부 통과해야 맞았습니다, 부분 점수 없음** - `ACCEPTED / WRONG_ANSWER / TIME_LIMIT / MEMORY_LIMIT / RUNTIME_ERROR / COMPILE_ERROR / JUDGE_ERROR`. 케이스는 position 순으로 전부 실행하고 **첫 실패 케이스의 종류가 대표 판정**(백준 방식). 컴파일 에러는 케이스 실행 없이 즉시. `JUDGE_ERROR` = 엔진 장애(Judge0 13 Internal Error·타임아웃·연결 실패) - 학생 잘못이 아님을 화면에 그대로 표시, 운영진 재채점으로 복구
5. **비동기 채점 + 폴링, 콜백 없음** - 제출 즉시 `judge_results` 행을 `PENDING` 으로 만들고 201 응답(제출 자체는 채점과 무관하게 성공 - 제출 기록·지각 판정 규칙 불변). BE 워커가 Judge0 batch 로 케이스를 보내고(`wait=false`) 토큰을 폴링(1초 간격, 최대 60초)해 집계. FE 는 제출 상세(#20)를 2초 간격으로 폴링(`PENDING/RUNNING` 동안만). 콜백을 쓰지 않는 이유: Judge0 → BE 역방향 URL·재시도·검증이 늘어나는데 규모(분반 30명, 문제당 케이스 10개 내외)가 작다
6. **채점 엔진은 인터페이스 뒤에** - `JudgeEngine { available(), run(language, source, inputs, limits) → RunOutcome }` 구현 `Judge0Engine`(prod) / `FakeJudgeEngine`(local·test - 지시 주석·echo) / `DisabledJudgeEngine`(**off** - 엔진 없음: 출제·저장은 되고 제출은 PENDING 대기, 엔진 연결 후 기동 시 재큐잉). 인증 `stub/oidc` 모드와 같은 패턴(`ondal.judge.engine: fake|judge0|off`, prod 기본 off - 엔진 연결 전 배포를 막지 않기 위해, prod 에서 fake 는 기동 거부). 이유: Judge0 없이 BE·FE 개발과 테스트가 돌아가야 하고, **Judge0 1.13.1 의 cgroup v1 의존(infra.md 3절)이 서버에서 풀리지 않으면 엔진만 갈아 끼울 수 있어야 한다**
7. **언어 6종은 FE 셀렉트 그대로, 매핑은 설정** - `C, C++, Java, Python 3, JavaScript, TypeScript`(SubmissionForm `LANGUAGES`) → Judge0 `language_id` 는 `application.yml` `ondal.judge.languages` 표(코드 아님). 자동 채점 문제에서 매핑 없는 언어로 제출하면 400. 과제별 허용 언어 제한은 두지 않음(P2 단순화 - C언어 반에서 파이썬으로 내면 운영진이 코멘트로 처리)
8. **제한 기본값 = 시간 2초 · 메모리 256MB, 문제별 조정, 언어별 보정 없음** - 출제 폼 기본값. Java·Python 문제는 출제자가 시간을 넉넉히 준다(폼에 안내). 서버 상한(Judge0 `MAX_CPU_TIME_LIMIT` 15초·`MAX_MEMORY_LIMIT` 512MB)을 넘는 값은 400
9. **테스트케이스 저장 = 통째 교체(PUT), 저장 시 재채점 선택** - 표를 편집하고 저장 한 번(#48). 기존 제출이 있으면 응답의 `affectedSubmissions` 로 FE 가 "제출 N건을 다시 채점할까요?" 를 묻고 `rejudge: true` 로 다시 저장 → 백그라운드 재채점. 테스트케이스 오류 정정이 실제 운영에서 가장 잦은 편집이라 별도 버튼 대신 저장 흐름에 붙임 (명시 재채점 #51 도 둠 - 엔진 장애 복구용)
10. **공개 케이스(`is_public`) = 예시 + 결과 노출** - 학생 과제 상세의 "예시 입력/출력" 절이 공개 케이스로 자동 채워진다(설명에 예시를 따로 적을 필요 없음). 채점 결과에서 공개 케이스는 입력·기대 출력·**실제 출력**까지, 비공개는 판정·시간·메모리만. 첫 케이스는 폼 기본값 공개
11. **출제 검증 도구 = 실행 API 하나(#49)** - `{language, sourceCode, inputs[], expectedOutputs?[]}` → 케이스별 `stdout·verdict·time·memory` 를 동기 응답(BE 가 Judge0 폴링, 최대 30초). "기대 출력 채우기"(expectedOutputs 없음 → stdout 을 폼에 채움)와 "출제 검증"(expectedOutputs 있음 → 통과 여부)이 같은 API. 저장은 하지 않음 - 결과를 보고 운영진이 저장(#48)
12. **`judge_results` = 제출 1건에 1행(PK = submission_id), 케이스 결과는 jsonb** - 재채점은 같은 행을 `PENDING` 으로 되돌려 덮어쓴다(채점 이력은 두지 않음 - 제출 이력이 이미 append-only). 케이스별 결과를 자식 테이블로 정규화하지 않는 이유: 항상 제출 단위로 통째 읽고 쓰며, 케이스별 질의가 없다
13. **과제 상태 배지와 판정은 별개** - 제출/제출(추가)/지각/미제출은 그대로(schema.md 결정 2), 판정은 옆 열. "틀렸습니다"도 제출이다. 홈 대시보드의 정답 수 집계는 후속(3절)
14. **큐 폭주·장애 시 제출은 절대 실패하지 않는다** - Judge0 `MAX_QUEUE_SIZE` 초과·연결 실패는 `PENDING` 유지 + 백오프 재시도(최대 10분), 그 뒤 `JUDGE_ERROR`. 제출 API 는 엔진 상태와 무관하게 201
15. **보관 분반** - 채점 설정 수정(#48)·실행(#49)·재채점(#51) 409(`ensureActive`). 이미 큐에 있는 채점은 끝까지 진행. 조회(#47·#50·결과)는 유지
16. **과제 삭제 연쇄에 채점 데이터 포함** - 순서: 파일 → `judge_results` → `submission_links` → `submissions` → `test_cases` → `assignments`(schema.md 4절 갱신). 진행 중 채점이 있으면 워커가 결과 저장 시 행 부재를 무시

## 3. 후속 작업

- [x] docs: judge/{design,api,fe,test,infra}.md (2026-09-14)
- [x] 서버: **A 채택·설치 완료 (2026-09-14)** - multipass VM `judge0`(Ubuntu 22.04, cgroup v1) 안에 Judge0 1.13.1 기동, 스모크 Accepted, `.env` 에 엔진 3키 기록 - infra.md 8절. **남은 것 = 컨테이너 재기동 1회(sudo)** 로 `engine=off → judge0` 적용
- [x] infra: ondal-BE `infra/judge0/`(compose·judge0.conf.example·README) - PR #40 머지 (2026-09-14)
- [x] BE: Flyway V6 + `judge` 슬라이스 + API #47~#51 + 응답 확장 - ondal-BE 이슈 #39 → PR #41 머지 (2026-09-14, JudgeApiTest 8·OutputComparatorTest 2·Judge0EngineTest 4 포함 전체 209건 통과). 서버 레포 pull 완료, **컨테이너 재빌드 대기(V4~V6)**
- [x] FE: 출제 섹션·학생 예시·채점 결과·현황판 판정·폴링 - ondal-FE 이슈 #41 → PR #42 (2026-09-14, 헤드리스 21건 mock·실 BE 통과). 서버 재빌드 후 #36 → #38 → #42 순서로 머지
- [x] docs: mvp-scope 5절, schema.md 결정 15, permissions 채점 행 (2026-09-14). flows 갱신은 후속
- [ ] 후속 후보: 홈 대시보드 정답 수, 언어별 시간 보정, 대량 케이스 zip 업로드, 채점 대기열 화면(운영진), P3 공개 문제 풀 전환(`cohort_id` NULL)
