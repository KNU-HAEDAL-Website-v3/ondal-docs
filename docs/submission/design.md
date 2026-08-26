# Submission(제출) - 제출 슬라이스 설계

> 상태: 확정 (2026-08-26 PM 승인 - 결정 11건)
> 역할: 이 슬라이스의 범위와 결정 기록 - 확정 후에는 거의 갱신하지 않으며, 결정은 BE PR 본문에 옮긴다
> 기준 규약: [guide/design.md](../guide/design.md) 3절·4절 - 이 문서는 그 규약에 추가되는 결정만 기록
> 스키마 원본: [db/schema.md](../db/schema.md) - submissions 테이블·상태 4종 계산·삭제 정책은 그쪽이 확정본
> 문서 구성: [api.md](api.md)(구현 계약) · [test.md](test.md)(테스트 계획) · [fe.md](fe.md)(FE 연동)

## 1. 범위

MVP 루프의 마지막 조각. 이 슬라이스가 끝나면 "운영진이 과제를 올리고, 학생이 제출하고, 운영진이 누가 냈는지 확인하는" 루프가 1회 완주된다 (mvp-scope 1절).

- 제출 API 5개 (`submission` 패키지 신설) - 제출 생성·내 이력·제출 상세·파일 다운로드·현황판. 명세는 [api.md](api.md)
- 과제 응답 확장 - `AssignmentResponse`에 상태 배지(`myStatus`)·제출 건수(`submissionCount`) 추가
- 과제 삭제 연쇄 - 기존 #17 DELETE를 파일 → submissions → assignment 순서의 서비스 연쇄 삭제로 확장 (schema.md 4절)
- 파일 저장 기반 - 서버 로컬 디스크, `stored_path`에 키만 저장 (S3 전환 대비)
- 제외: 자동 채점(P2), 멘토 코멘트·점수(P2 - 열만 존재, 항상 NULL), 분반 영구 삭제(별도 슬라이스 - guide 8절 결정 5), 분반 전체 제출 기록 페이지(결정 8)

## 2. 결정

1. **제출 = append-only 이력, 지각은 저장하지 않고 계산** *(2026-08-19 확정 사항의 구현 - schema.md 결정 1·2)* - 수정·삭제 API 없음, 재제출 무제한. 지각 판정 시각 = 서버 수신 시각(클라이언트 시계 불신 - flows 1.2절). 마감 수정 시 재계산은 계산식의 자연 결과라 별도 코드 불필요
2. **제출 요청은 항상 multipart/form-data 하나** - `request` JSON 파트(codeText·language·linkUrl) + `file` 파트(선택). 파일 없는 코드·링크 제출도 같은 형식. 엔드포인트·콘텐츠 타입이 하나면 FE 분기와 테스트가 단순해진다. 검증(본문 택1·최소 1개)은 서비스 한 곳
3. **파일 제한 = zip 단일 확장자 + 20MB** *(2026-08-26 확정 - flows 3절-3 미결 해소)* - 온프렘 디스크 보호. 확장자 검사만, 매직 바이트 검사는 P1 과함. 크기는 Spring multipart 설정 + 서비스 이중 확인. 심화반 과제가 zip 외 형식을 요구하면 화이트리스트 확장
4. **상태 enum = `NOT_SUBMITTED / SUBMITTED / SUBMITTED_EXTRA / LATE`** - schema.md 3절 계산 규칙의 서버 표현. FE는 이 값을 그대로 배지에 매핑(프론트 재계산 금지 - FE CLAUDE.md 규칙 4). 응답의 지각 여부(`late`)도 서버 판정값
5. **`AssignmentResponse`는 Assembler 도입으로 확장** - `myStatus`(viewer 의존)·`submissionCount`(OPERATOR/ADMIN만 값, STUDENT는 null - `studentCount` 선례) 추가. viewer 의존 필드가 생겨 `from(entity)`로 부족해짐 - assignment/design.md 결정 5에서 예고한 재검토의 결론. Detail 분리는 여전히 불필요(목록·단건 필드 동일)
6. **운영진·ADMIN도 제출 가능** - 제출 API는 `@CohortRole(STUDENT)`(소속 누구나 + ADMIN). 막을 이유가 없고(테스트 제출·시연) 현황판 명단은 현재 STUDENT Enrollment 기준이라 운영진 제출이 통계를 오염하지 않는다
7. **학생은 자기 제출만, 타인 제출물은 404** - 제출 상세·파일 다운로드는 본인 또는 OPERATOR/ADMIN. 불일치는 403이 아니라 404(존재 비노출 - guide 결정 11). 현황판·타인 이력은 OPERATOR 전용
8. **분반 전체 제출 기록 페이지(FE SubmissionsPage)는 P2로 이연** - 화면의 절반이 채점 결과(맞았습니다/시간/메모리) 표시라 자동 채점(P2) 종속. P1의 "내 제출 기록 확인"은 과제 상세 안의 이력 목록(#19)이 담당
9. **삭제 경고 건수 = `submissionCount`로 해결** - 별도 API 없음. FE는 과제 상세 응답의 `submissionCount`로 "제출물 N건이 함께 삭제됩니다" 확인 창 구성 (schema.md 결정 4)
10. **파일 저장 = `{upload-dir}/submissions/{UUID}.zip`** - 키를 `stored_path`에 저장, 원본 파일명은 `file_name` 열에 별도 보관(다운로드 시 Content-Disposition으로 복원). upload-dir는 application.yml 프로퍼티(`ondal.upload.dir`), 테스트는 임시 디렉터리
11. **중복 제출 방어는 FE 버튼 잠금까지만** - 서버 측 rate limit·멱등 키는 P1 미도입 (flows 3절-5의 "서버 중복 방지"는 이력 append 특성상 데이터가 깨지지 않아 위험이 낮음). 문제 발생 시 재검토

## 3. 후속 작업

- [x] BE: 이슈 #12 → `feat/12-submission-slice` 구현 → PR #13 머지 (2026-08-26, 테스트 122개 통과. 구현 중 발견: `deleteAllInBatch`는 영속성 컨텍스트 우회로 커밋 시 충돌 - PR 결정 9)
- [x] BE 후속: `StatusBoardRow.latestSubmissionId` 추가 - 이슈 #14 → PR #15 (FE 연동 중 발견한 열람 진입 키 누락)
- [x] FE: 제출 폼·내 기록·상태 배지·현황판 - 이슈 #15 → PR #16 머지 (2026-08-26). 현황판 UI는 과제 상세 안 운영진 섹션 - [fe.md](fe.md) 1절
- [x] docs: mvp-scope 6절 완료 기준 갱신 (2026-08-26)
- [ ] 스모크 테스트: 실 BE + FE로 제출 루프 1회 완주 확인 → P1 완료 선언
- [ ] 다음 슬라이스 후보: 분반 영구 삭제(연쇄 최상위) 또는 홈페이지 인증 연동
