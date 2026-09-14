# 출석부 - 테스트 계획

> 역할: 이 슬라이스의 검증 계획 - 구현 중 케이스가 늘면 이 문서만 갱신
> 명세: [api.md](api.md) · 테스트 기반 규약: [guide/design.md](../guide/design.md) 6절

## 1. 테스트 클래스·픽스처

- `attendance/AttendanceApiTest` - `extends ApiTestSupport`, 컨트롤러 2개(Session·Attendance)와 1:1
- 슬라이스 고유 픽스처(private 헬퍼): `createSession(cohortId, 세션, sessionNo|null, heldOn)` / `mark(cohortId, sessionId, 세션, {loginId: status})`
- 분반·소속 픽스처는 공용 헬퍼: `createCohort`, `enrollStudent`, `archiveCohort`, `restoreCohort`

## 2. 케이스 (역할 × 엔드포인트 대표)

- 차시
  - 등록: STUDENT·비소속 → 403 / OPERATOR 번호 생략 → 201, 1 → 다음은 2 (자동 채번) / 지정 번호 중복 → 409 CONFLICT / heldOn 누락·번호 0 → 400
  - 목록: STUDENT → 200, 날짜 → 번호 오름차순 / 비소속 → 403
  - 수정: 번호·제목·날짜 교체 200, 다른 차시 번호로 바꾸면 409 / 삭제: 기록 있는 차시 → 204, 이후 명부 404, 기록 연쇄 삭제(응답 `attendanceCount` 확인)
  - 교차 분반: A반 차시 id 를 B반 경로로 → 404
- 출석
  - 명부 GET: STUDENT → 403 / OPERATOR → rows = STUDENT 만(운영진 없음, 이름순), 전부 status null, summary.unchecked = 인원, rate null
  - 표시 PUT: `[s1 PRESENT, s2 LATE]` → rows 상태 반영, summary present 1·late 1·unchecked n-2, s1.stats.rate 100·s2.stats.rate 0 / 같은 loginId 두 번 → 마지막 값 / `status: null` → 기록 삭제(미확인) / 운영진·미소속·모르는 loginId → 404 / 빈 records → 400 / 잘못된 status 문자열 → 400
  - 내 출석 GET: s1 → summary present·rate, records 차시 최신순·status / 비소속 → 403 / 두 차시 중 하나만 판정 → unchecked 1
- 보관 분반
  - GET 목록·명부·내 출석 → 200 (열람 유지) / POST·PUT·DELETE 차시, PUT 표시 → 409 COHORT_ARCHIVED / restore 후 → 정상
- 시더(local)는 테스트 대상 아님 - FE mock 과 동일 데이터 유지만 확인(수동)
