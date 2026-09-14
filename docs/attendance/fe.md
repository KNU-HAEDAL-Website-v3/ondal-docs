# 출석부 - FE 연동 메모

> 역할: FE 작업자용 - 차시·출석 API 와 화면의 매핑, 표시 규칙
> 명세: [api.md](api.md) - 계약의 원본은 springdoc `Session`·`Attendance` 태그 · FE 공통 규칙: ondal-FE `CLAUDE.md`
> 상태: 견본 화면(피그마 28:836 학생 / 28:1013 운영진, `StudentAttendanceView`·`OperatorAttendanceView`)을 실 API 로 교체 - 2026-09-14 진행

## 1. 화면 매핑

- 진입 `/attendance` (사이드바 "출석") - 역할 판정은 기존대로: 관리자이거나 `canManage` 분반이 있으면 운영진 뷰, 아니면 학생 뷰
- **학생 뷰 - 출석 현황**: 분반 선택(소속 분반, 기본 첫 ACTIVE) → `GET /api/cohorts/{id}/attendances/me`
  - 출석률 링 = `summary.rate`(null 이면 "-"), 카드 3개 = present / late / absent, 표 = `records` (날짜 `heldOn`·요일(FE 계산)·차시 `N차시 · 제목`·상태 배지). 상태 배지: 출석 초록 / 지각 노랑 / 결석 빨강 / 미확인 회색(`status: null`)
  - 견본의 "월 이동" 버튼 제거 - 페이징 없이 전체 차시
- **운영진 뷰 - 출결 관리**: 분반 선택(관리자 = 진행 중 분반 전부 / 운영진 = `canManage` 분반) → 차시 선택(`GET sessions`, 기본 = 가장 최근 날짜) → `GET .../sessions/{id}/attendances`
  - 요약 카드 = `summary` (전체 수강생 = rows 수, 출석·지각·결석, 출석률 = 이 차시 `summary.rate`)
  - 표 = `rows`: 이름 · 아이디 · 상태(셀렉트: 미확인/출석/지각/결석 → 바꾸면 즉시 `PUT records:[{loginId, status}]`) · 표시 시각(`checkedAt`, KST) · 누계 출석률(`stats.rate` 막대)
  - "일괄 출석 처리" = 미확인 전원을 PRESENT 로 한 번에 `PUT` (확인 창) / "차시 추가" = 인라인 폼(번호 자동·날짜·제목) / 차시 수정·삭제(삭제 경고에 `attendanceCount`)
  - 견본의 기수·반 필터, 출석 시간, 출석부 다운로드, 체크박스·페이지네이션 제거 - 대응하는 서버 개념 없음 (design.md 1절)
- 보관 분반: 학생·운영진 뷰 모두 열람만 - 셀렉트·버튼 비활성 + 안내

## 2. 표시 규칙

- 출석률·상태·요약은 **서버 값 그대로**(CLAUDE.md 규칙 4) - 프론트에서 비율 재계산·지각 환산 금지. 요일만 `heldOn` 으로 FE 가 계산
- `heldOn` 은 날짜 문자열(`yyyy-MM-dd`, KST 달력일) - 시간대 변환 없이 그대로 표시. `checkedAt` 은 UTC → KST
- 에러: 401 재로그인 / 403 홈(비소속·학생의 명부 URL) / 404 안내 / 409 COHORT_ARCHIVED·번호 중복은 안내만

## 3. API 클라이언트

- `src/api/sessions.ts`(차시 CRUD) · `src/api/attendances.ts`(명부·표시·내 출석) - `questions.ts` 패턴. 타입은 `types.ts`(BE DTO 미러)
- 표시 성공 응답이 갱신 명부라 `setQueryData` 로 바로 반영(수강생 배정 선례)
- MSW: 차시 4개 + 출석 3개 핸들러, 시드 = BE `LocalDataSeeder.seedAttendance` 와 동일(차시 2 + 기록 5)

## 4. 범위 밖

- 셀프 체크인, 출석 시간 자동 기록, 엑셀 내보내기, 지각 환산, 사유 메모, 알림 (design.md 1절)
