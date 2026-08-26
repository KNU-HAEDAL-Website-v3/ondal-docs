# Submission - 테스트 계획

> 역할: 이 슬라이스의 검증 계획 - 구현 중 케이스가 늘면 이 문서만 갱신
> 명세: [api.md](api.md) · 테스트 기반 규약: [guide/design.md](../guide/design.md) 6절

## 1. 테스트 클래스·픽스처

- `submission/SubmissionApiTest` - `extends ApiTestSupport`, 컨트롤러와 1:1
- 기존 `assignment/AssignmentApiTest`에 확장분 케이스 추가 - `myStatus`·`submissionCount`·삭제 연쇄
- 슬라이스 고유 픽스처(`submitCode`, `submitFile`, `submitLink`)는 SubmissionApiTest의 private 헬퍼로 (support/는 PM 파일)
- 파일 저장 루트는 `@TempDir` - 테스트 간 파일 잔존 없음, DatabaseCleaner와 별개로 디렉터리 정리 확인
- multipart 요청: `MockMvc.multipart()` + `request` JSON 파트 + `file` 파트(`MockMultipartFile`)

## 2. 케이스 (역할 × 엔드포인트 대표)

- 인증·권한
  - 미로그인 → 401 / 비소속 MEMBER → 403
  - STUDENT: 제출·내 이력·자기 제출 상세·자기 파일 다운로드 → 성공
  - STUDENT가 타인 submissionId 상세·파일 → **404** (403 아님 - 존재 비노출)
  - STUDENT 현황판 → 403 / OPERATOR·비소속 ADMIN 현황판·타인 제출 상세 → 200
  - OPERATOR 본인 제출 → 201 (design.md 결정 6 - 현황판 명단에는 미포함 확인)
- 스코프·404 (손자 체인)
  - 교차 분반: A반 경로로 B반 과제의 제출 접근 → 404
  - 교차 과제: 과제 1 경로로 과제 2의 submissionId → 404
  - 없는 submissionId·assignmentId → 404
- 제출 형식 검증 (400)
  - codeText + file 동시 → 400 / 본문·링크 모두 없음 → 400
  - 링크만 → 201 / 코드만 → 201 / 파일만 → 201 / 코드+링크 → 201
  - zip 외 확장자(.exe, .txt) → 400 / 20MB 초과 → 400 INVALID_INPUT / language가 파일 제출에 실림 → 400
  - codeText 100001자 → 400 / language 31자 → 400 / linkUrl 2049자 → 400
- 지각·상태 계산
  - 마감 후 제출 → 201 + `late: true` (차단 없음) / 마감 전 → `late: false`
  - myStatus: 무제출 → NOT_SUBMITTED / 마감 내만 → SUBMITTED / 마감 내+후 → SUBMITTED_EXTRA / 마감 후만 → LATE
  - 마감 수정 후 재조회 → 상태 재계산 (LATE → SUBMITTED로 변화 확인)
  - AssignmentResponse: STUDENT는 `submissionCount: null`·자기 myStatus / OPERATOR는 건수 값 / 비소속 ADMIN은 `myStatus: null`
- 현황판
  - 행 = STUDENT 명단 전체(미제출자 포함) / 운영진 제출은 행 없음
  - 상태·submissionCount·lastSubmittedAt 값 검증 / 소속 해제 학생은 행 제외
- 재제출·이력
  - 같은 학생 3회 제출 → 이력 3건 최신순, 이력 목록에 codeText 없음
  - 내 이력에 타인 제출 미포함
- 파일
  - 업로드 후 다운로드 → 바이트 일치 + Content-Disposition 원본 파일명(한글 포함)
  - 코드 제출의 파일 다운로드 → 404
- 보관 분반
  - 제출 → 409 COHORT_ARCHIVED / 이력·상세·다운로드·현황판 → 200 (열람 유지) / restore 후 제출 → 201
- 삭제 연쇄 (AssignmentApiTest 확장)
  - 제출물 있는 과제 DELETE → 204, submissions 행·디스크 파일 삭제 확인
  - 제출물 없는 과제 DELETE → 204 (기존 동작 유지)

## 3. 시더

- `LocalDataSeeder`(local 전용) 확장: 샘플 과제에 제출 시나리오 4종 - 마감 내 제출 1명·지각 1명·마감 내+추가 1명·미제출 1명 (FE가 배지 4종·현황판을 바로 확인)
- 코드·링크 제출만 시딩 - zip은 디스크 파일이 필요해 시더 부적합
- 테스트는 시더 비의존 - 픽스처는 각 테스트가 생성 (규약)
