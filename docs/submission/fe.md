# Submission - FE 연동 메모

> 역할: FE 작업자용 - 제출 API와 화면의 매핑, 표시 규칙
> 명세: [api.md](api.md) - 계약의 원본은 BE 구현 후 springdoc · FE 공통 규칙: ondal-FE `CLAUDE.md`

## 1. 화면 매핑 (기준: FE PR #16 구현 + 2026-08-27 제출 모델 개정 - 개정분은 슬라이스 1에서 구현)

- 과제 상세(`AssignmentDetailPage`)의 제출란 placeholder → 제출 섹션 3단 (폼 → 내 기록 → 현황판)
  - 제출 폼: 탭 = 코드 붙여넣기(+ 언어 셀렉트 **필수**) / zip 업로드(**10MB**) / **링크(+ 버튼으로 1~5개)** - 택1(`type`). 탭 밖 공통 링크 필드 폐지 (design.md 결정 12·13)
  - 제출 = #18 multipart(`request` JSON 파트 + `file` 파트). 선택한 탭의 필수값이 다 차야 버튼 활성(서버 400의 사전 차단), zip·10MB·링크 개수도 클라이언트 선검사
  - 내 제출 기록 = #19 **표**: 순번 · 제출 형태 · 지각 배지 · 제출 시각 - 채점 결과 열은 P2 자리 예약 (design.md 결정 15). 코드 확인은 행 펼침 → #20, 파일은 #21 다운로드 anchor
- 과제 목록·상세의 상태 배지 → `AssignmentResponse.myStatus` 그대로 매핑 (assignment/fe.md 2절의 보류 해제)
- 현황판 = **과제 상세 안 운영진 섹션** (#22) - 상태/횟수/최근 제출 표
  - 열람 = `latestSubmissionId`로 최신 제출(대표) 펼침 (#20) - 전체 이력 열람은 P2
  - 보관 분반에서도 열람 유지: 진입 판정은 `canManage`(ACTIVE 전용)가 아니라 역할(ADMIN 또는 `myRole == OPERATOR`)
  - 홈 `OperatorDashboard`는 분반 단위 집계(제출률·출석률) 화면이라 P1 API로 채울 수 없음 - 목데이터 유지, P2에서 집계 API와 함께
- 과제 삭제 확인 창: "제출물 N건이 함께 삭제됩니다" - N = `AssignmentResponse.submissionCount`
- `SubmissionsPage`(분반 전체 제출 기록)는 P2 이연 - 채점 결과 열이 본체 (design.md 결정 8). 라우트·사이드바 메뉴 미노출, 학생 홈 "더보기"는 과제 목록으로 연결

## 2. 표시 규칙

- 상태 배지 색: `SUBMITTED`·`SUBMITTED_EXTRA` = 초록, `LATE` = 주황, `NOT_SUBMITTED` = 회색 (schema.md 결정 2)
- 제출 형태 표기: `CODE` = 코드 제출, `FILE` = 파일 업로드, `LINK` = 링크 제출 - 서버 `type` 그대로 매핑, 아이콘·라벨은 FE 재량
- 코드 작성·열람 = CodeMirror 6 (2026-08-27, design.md 결정 17): 폼 코드 탭은 편집 모드, 기록 펼침·현황판 열람은 read-only + 복사 버튼. `language` 값 → 언어 확장 매핑(C/C++→cpp, Java, Python 3, JavaScript/TypeScript), 매핑 없는 값은 하이라이팅 없이 표시
- `myStatus`·`late`는 서버 판정값 그대로 - dueAt과 submittedAt으로 프론트 재계산 금지 (FE CLAUDE.md 규칙 4)
- 시각 표시: `submittedAt`·`lastSubmittedAt`(UTC) → KST 변환. "10분 전" 상대 표시는 FE 가공 허용
- 마감 후에도 제출 버튼 활성 - "지각 제출로 기록됩니다" 확인 안내 후 진행 (flows UC-S4 A1)
- 제출 실패(400·401)·세션 만료 시 작성 코드·첨부·링크 입력값 화면 보존 (FE CLAUDE.md 필수 규칙)
- 처리 중 제출 버튼 잠금 - 중복 제출 방지는 FE 몫 (design.md 결정 11)
- 보관 분반: `status == ARCHIVED`면 제출 폼 사전 비활성, 이력·현황판 열람은 유지

## 3. P1 범위 밖 디자인 요소

- 채점 결과(맞았습니다/틀렸습니다·실행 시간·메모리)는 자동 채점(P2) - `SubmissionsPage` 목데이터의 `result`·`time`·`memory`·`tone` 열이 여기 해당
- 현황판의 출석률(`attendance`)은 출석부(P2)
- 멘토 코멘트·점수 표시는 P2 - API 응답에 필드 자체가 없음
