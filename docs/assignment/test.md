# Assignment - 테스트 계획

> 역할: 이 슬라이스의 검증 계획 - 구현 중 케이스가 늘면 이 문서만 갱신
> 명세: [api.md](api.md) · 테스트 기반 규약: [guide/design.md](../guide/design.md) 6절

## 1. 테스트 클래스·픽스처

- `assignment/AssignmentApiTest` - `extends ApiTestSupport`, 컨트롤러와 1:1
- 슬라이스 고유 픽스처(`createAssignment`)는 이 테스트 클래스의 private 헬퍼로 (support/는 PM 파일 - guide 6절)
- 분반·소속 픽스처는 기존 공용 헬퍼 사용: `createCohort`, `enrollStudent`, `archiveCohort`, `restoreCohort`

## 2. 케이스 (역할 × 엔드포인트 대표)

- 인증·권한
  - 미로그인 → 401
  - 비소속 MEMBER GET 목록 → 403
  - STUDENT GET 목록·단건 → 200, POST·PUT·DELETE → 403
  - OPERATOR 전 엔드포인트 → 201·200·204
  - 비소속 ADMIN → 통과
- 스코프·404
  - **교차 분반: A반 OPERATOR가 `/api/cohorts/A/assignments/{B의 과제}` GET·PUT·DELETE → 404** (guide 6절 예고 케이스)
  - 없는 assignmentId → 404 / 없는 cohortId: ADMIN → 404, MEMBER → 403
- 입력 검증
  - `cohortId=abc` → 400 / title 공백·201자 → 400 / dueAt 누락 → 400 / 깨진 JSON → 400
  - 과거 dueAt → 성공 (design.md 결정 3)
  - sessionNo 0·음수 → 400 / sessionNo 누락 → 성공(차시 밖 과제, design.md 결정 6)
- 보관 분반
  - GET 목록·단건 → 200 (열람 유지)
  - POST·PUT·DELETE → 409 COHORT_ARCHIVED / restore 후 → 성공
- 동작 확인
  - PUT 수정은 재조회로 flush 확인
  - 목록 정렬: 차시 오름차순, 차시 없는 과제는 마지막, 같은 차시는 등록순 / 빈 목록 `[]`
  - POST 응답 Location 헤더 + 본문 id

## 3. 시더

- `LocalDataSeeder`(local 전용) 확장: 샘플 분반에 과제 3개 - 1차시(마감 지남)·2차시(마감 전)·차시 없음 1개 (FE가 그룹핑·정렬까지 바로 확인할 수 있게)
- 테스트는 시더 비의존 - 픽스처는 각 테스트가 생성 (규약)
