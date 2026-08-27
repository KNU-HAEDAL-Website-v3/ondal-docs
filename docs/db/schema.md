# Ondal P1 DB 스키마 (확정)

> 확정일: 2026-08-19 (PM 결정) · 갱신: 2026-08-27 (제출 모델 3종 택1 결정 8 + 문제 번호 결정 9) · 기준 코드: BE `main` `b3eb5ea` (결정 8 반영 완료, 결정 9는 코드 반영 대기 - 슬라이스 2)
> P1 스키마의 단일 출처 - 변경 시 이 문서에 결정 기록 + 코드·ERD(ERDCloud) 동시 갱신
> P1 바깥(자동 채점, QnA, 공지, 출석부)은 미포함 - 스코프 제외 또는 미결 상태라 지금 그리면 추측이 됨

## 1. 확정 결정

| # | 결정 | 내용 |
|---|---|---|
| 1 | 재제출 = 이력 | 백준처럼 제출마다 행이 쌓인다. 수정·삭제 없음(append-only). 추후에도 모든 제출을 확인 가능. |
| 2 | 제출 상태 4가지 | 저장하지 않고 이력에서 계산한다. 미제출 / **제출**(초록: 마감 내 제출 있음) / **제출(추가)**(초록: 마감 내 제출 + 마감 후 재제출도 있음) / **지각**(주황: 마감 후 제출만 있음). |
| 3 | 제출 형식 = 본문 + 링크 (폐기) | **2026-08-26 PM 리뷰로 폐기 → 결정 8.** 본문(코드/zip 택1) + 링크(선택) 모델은 링크가 "첨부"인지 "제출"인지 처음 보는 사용자에게 혼란을 줌. 과제별 제출 타입 지정 폐기(모든 과제가 세 방식 상시 허용)는 유지. |
| 4 | 과제 삭제 = 경고 후 연쇄 | 운영진이 삭제 가능. "제출물 N건이 함께 삭제됩니다" 경고 → 제출 이력 + 서버의 zip 파일까지 삭제. |
| 5 | 분반 = 보관 기본 + 삭제 공존 | 학기 종료는 **보관**(기본 경로, 열람 유지 - 홈 "지난 소속"이 이 데이터를 쓴다). **영구 삭제**는 ADMIN 전용 최후 수단 - 소속·과제·제출물·파일 전부 연쇄 삭제, 분반 이름을 입력해야 확인되는 강한 경고. **users는 어떤 삭제에도 지워지지 않는다.** |
| 6 | 차시 = 과제의 선택 번호 | (2026-08-25) `assignments.session_no` 정수. 운영진 자유 입력 - 중복·건너뜀 허용, 차시 밖 과제는 NULL. 목록 기본 정렬 = 차시 오름차순(NULL 마지막) → 등록순. 차시 자체가 속성(날짜·제목)을 갖게 되는 출석부(P2) 도입 시 Session 엔티티 승격 재검토. |
| 7 | 제출 언어 저장 | (2026-08-25) `submissions.language` - 코드 제출 전용. **2026-08-26 개정: 코드 제출이면 필수** - OJ 채점(P2)의 언어 식별 대비. FILE/LINK 제출은 NULL. 화면 표시·코드 하이라이팅에도 사용. |
| 8 | 제출 형태 = 3종 택1 | (2026-08-26 PM 리뷰) 코드 / 파일 / 링크 중 하나를 골라 제출 - `submissions.type`(CODE\|FILE\|LINK) 열로 저장(조회 단순·택1 CHECK 강제 가능, 응답 type 필드와 1:1). 링크 제출은 `submission_links`(1:N, **1~5개**, `position` 순서 보존)로 다중 입력. 파일 한도 20MB → **10MB**. 기존 개발 DB는 리셋(볼륨 삭제 후 시더 재생성) - 구 모델 행 마이그레이션 없음(샘플뿐). |
| 9 | 문제 번호 = 전역 유일, 1000부터 | (2026-08-26 PM 리뷰, 상세 확정 2026-08-27) 모든 과제는 문제(설문형 포함) - `assignments.problem_no` INT **NOT NULL UNIQUE**. 채번: 폼에서 비우면 자동(현재 최대+1, 1000 시작), 직접 입력하면 그 번호(1000 이상, 중복 409). **수정 허용**(중복 409) - P1 내부용이라 오타 정정 유연성 우선, 혼란은 FE 확인 문구로 방어. 개발 DB 리셋(시더가 번호 포함 재생성). |

- 파일 저장: P1은 서버 로컬 디스크
- S3 등 전환 대비: `stored_path`에 키만 넣으면 되도록 열 설계

## 2. ERD - ERDCloud 가져오기용 SQL

- ERDCloud의 SQL 가져오기 = MySQL 문법 기준 → 아래 SQL도 MySQL 문법
- 실제 DB: PostgreSQL 16 - 타입 대응은 4절

```sql
-- =========================================================================
-- Ondal P1 DB 스키마 - 확정본 2026-08-19 · 갱신 2026-08-27 (docs/db/schema.md)
-- ERDCloud 가져오기용 MySQL 문법. 실제 DB는 PostgreSQL 16.
-- =========================================================================

CREATE TABLE users (
    id          BIGINT      NOT NULL AUTO_INCREMENT COMMENT 'PK',
    login_id    VARCHAR(50) NOT NULL COMMENT '로그인 ID - 홈페이지 계정과 매칭하는 키. 아직 로그인 안 한 부원도 이 값으로 선등록됨',
    name        VARCHAR(50) NOT NULL COMMENT '표시 이름 - 스텁 로그인에서는 login_id와 동일, 홈페이지 연동 후 실제 이름으로 갱신',
    global_role VARCHAR(20) NOT NULL COMMENT '전역 역할 ADMIN(임원단=해구르르) | MEMBER(일반 부원) - enum은 STRING 저장',
    created_at  DATETIME(6) NOT NULL COMMENT '생성 시각(UTC) - KST 변환은 프론트 몫',
    PRIMARY KEY (id),
    UNIQUE KEY uk_users_login_id (login_id)
) COMMENT='사용자 - 신원의 원본은 홈페이지, Ondal은 login_id로 매칭하는 로컬 사본. 하드 삭제 없음. 분반 삭제에도 지워지지 않는다';

CREATE TABLE cohorts (
    id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT 'PK',
    name        VARCHAR(100) NOT NULL COMMENT '분반 이름 (예: 2026-2 C언어)',
    description TEXT         NULL     COMMENT '설명 (선택)',
    status      VARCHAR(20)  NOT NULL COMMENT 'ACTIVE(진행 중) | ARCHIVED(보관 - 열람만 가능, 모든 변경은 409)',
    created_at  DATETIME(6)  NOT NULL COMMENT '생성 시각(UTC)',
    PRIMARY KEY (id)
) COMMENT='분반 - 학기·트랙 단위 반. 학기 종료는 보관(기본, 지난 소속에서 열람). ADMIN 전용 영구 삭제는 최후 수단 - 소속·과제·제출물·파일 연쇄 삭제, 분반 이름 입력 확인';

CREATE TABLE enrollments (
    id         BIGINT      NOT NULL AUTO_INCREMENT COMMENT 'PK',
    cohort_id  BIGINT      NOT NULL COMMENT 'FK → cohorts.id',
    user_id    BIGINT      NOT NULL COMMENT 'FK → users.id',
    role       VARCHAR(20) NOT NULL COMMENT '분반 안 역할 OPERATOR(교육운영진) | STUDENT(수강생). STUDENT→OPERATOR 승격만 있고 강등 없음',
    created_at DATETIME(6) NOT NULL COMMENT '소속 등록 시각(UTC)',
    PRIMARY KEY (id),
    UNIQUE KEY uk_enrollment_cohort_user (cohort_id, user_id),
    CONSTRAINT fk_enrollments_cohort FOREIGN KEY (cohort_id) REFERENCES cohorts (id),
    CONSTRAINT fk_enrollments_user   FOREIGN KEY (user_id)   REFERENCES users (id)
) COMMENT='소속 - 누가 어느 분반에서 무슨 역할인가. 한 사람은 한 분반에 역할 하나. 해제 시 하드 삭제되는 관계 테이블 - 다른 테이블이 FK로 참조하지 않는다';

CREATE TABLE assignments (
    id          BIGINT       NOT NULL AUTO_INCREMENT COMMENT 'PK',
    cohort_id   BIGINT       NOT NULL COMMENT 'FK → cohorts.id - 과제는 분반에 속한다. 조회는 항상 (id, cohort_id) 스코프(다른 반 과제면 404)',
    problem_no  INT          NOT NULL COMMENT '문제 번호 - 전역 유일, 1000부터 (결정 9). 비우면 자동(최대+1), 직접 지정 가능(1000 이상). 수정 허용, 중복은 409',
    session_no  INT          NULL     COMMENT '차시 번호(선택) - 운영진 자유 입력, 중복·건너뜀 허용. 차시 밖 과제는 NULL. 목록 기본 정렬: 차시 오름차순(NULL 마지막) → 등록순',
    title       VARCHAR(200) NOT NULL COMMENT '과제 제목',
    description TEXT         NULL     COMMENT '과제 내용 - 문제 설명·링크 포함 자유 텍스트',
    due_at      DATETIME(6)  NOT NULL COMMENT '마감 시각(UTC) - 운영진이 생성 시 설정. 마감 후 제출은 지각 표시(차단 아님). 마감을 수정하면 지각 판정도 새 마감 기준으로 다시 계산된다',
    created_at  DATETIME(6)  NOT NULL COMMENT '등록 시각(UTC)',
    PRIMARY KEY (id),
    UNIQUE KEY uk_assignments_problem_no (problem_no),
    KEY idx_assignments_cohort (cohort_id),
    CONSTRAINT fk_assignments_cohort FOREIGN KEY (cohort_id) REFERENCES cohorts (id)
) COMMENT='과제 = 문제 (결정 9 - 전역 유일 번호). 등록·수정·삭제는 운영진 이상. 보관 분반에서는 쓰기 409. 삭제는 경고(제출물 N건 함께 삭제) 후 연쇄. 제출 타입 열 없음 - 모든 과제가 코드/zip/링크를 받는다';

CREATE TABLE submissions (
    id             BIGINT        NOT NULL AUTO_INCREMENT COMMENT 'PK',
    assignment_id  BIGINT        NOT NULL COMMENT 'FK → assignments.id',
    user_id        BIGINT        NOT NULL COMMENT 'FK → users.id - enrollment이 아니라 사용자 직접 참조. 소속이 해제돼도 제출물은 남는다 (design.md 1절)',
    type           VARCHAR(10)   NOT NULL COMMENT '제출 형태 CODE(코드) | FILE(zip) | LINK(링크) - 3종 택1 (결정 8). 형태별 필수 열은 서비스 + CHECK 강제',
    code_text      TEXT          NULL COMMENT 'CODE 전용 - 붙여넣은 코드 텍스트',
    language       VARCHAR(30)   NULL COMMENT 'CODE 전용·필수 - 제출 언어(예: Python 3, C++ 17). 하이라이팅 표시 + 채점(P2) 언어 식별 (결정 7)',
    file_name      VARCHAR(255)  NULL COMMENT 'FILE 전용 - 업로드한 zip의 원본 파일명',
    stored_path    VARCHAR(500)  NULL COMMENT 'FILE 전용 - 저장 위치 키. P1은 서버 로컬 디스크, S3 전환 시에도 이 열에 키만',
    file_size      BIGINT        NULL COMMENT 'FILE 전용 - 크기(byte). 한도 10MB (결정 8). 용량 관리·삭제 경고 문구용',
    submitted_at   DATETIME(6)   NOT NULL COMMENT '제출 시각(UTC). due_at과 비교해 지각을 계산한다 - 저장하는 지각 플래그 없음',
    score          INT           NULL COMMENT '[P2 준비] 점수 - P1에서는 항상 NULL (mvp-scope 5절)',
    mentor_comment TEXT          NULL COMMENT '[P2 준비] 멘토 코멘트 - P1에서는 항상 NULL',
    PRIMARY KEY (id),
    KEY idx_submissions_assignment_user (assignment_id, user_id, submitted_at),
    KEY idx_submissions_user (user_id),
    CONSTRAINT fk_submissions_assignment FOREIGN KEY (assignment_id) REFERENCES assignments (id),
    CONSTRAINT fk_submissions_user       FOREIGN KEY (user_id)       REFERENCES users (id)
) COMMENT='제출 이력 - 재제출마다 행 추가(수정·삭제 없음). 형태는 3종 택1(type). 상태는 계산: 미제출/제출/제출(추가)/지각. 최신 제출 = submitted_at 최대 행';

CREATE TABLE submission_links (
    id            BIGINT        NOT NULL AUTO_INCREMENT COMMENT 'PK',
    submission_id BIGINT        NOT NULL COMMENT 'FK → submissions.id',
    url           VARCHAR(2048) NOT NULL COMMENT '링크 URL - GitHub·배포 주소 등',
    position      INT           NOT NULL COMMENT '입력 순서 1~5 - 표시 순서 보존 (결정 8)',
    PRIMARY KEY (id),
    KEY idx_submission_links_submission (submission_id),
    CONSTRAINT fk_submission_links_submission FOREIGN KEY (submission_id) REFERENCES submissions (id)
) COMMENT='제출 링크 - LINK 제출의 URL 1~5개. 제출과 함께 append-only, 개수·순서는 서비스 강제';
```

## 3. 계산 규칙 (열로 저장하지 않는 것)

한 학생의 한 과제 제출 상태: 제출 이력과 `due_at`만으로 계산.

```
onTime = submitted_at ≤ due_at 인 제출 존재 여부
late   = submitted_at >  due_at 인 제출 존재 여부

없음            → 미제출
onTime && !late → 제출          (초록)
onTime && late  → 제출(추가)    (초록 계열 - 마감 내 제출 후 추가 제출함)
!onTime && late → 지각          (주황)
```

- 마감(`due_at`) 수정 시 위 계산은 새 마감 기준으로 재계산 - 마감 연장으로 지각이 제출로 바뀌는 것은 의도된 동작
- 현황판의 미제출자 = 현재 `Enrollment(STUDENT)` 명단 - 제출자 - 소속 해제 학생은 명단에서 제외, 제출 데이터는 유지
- "학생의 최신 제출" = 그 (assignment, user)에서 `submitted_at`이 가장 큰 행 - `idx_submissions_assignment_user`가 이 조회를 지원

## 4. 실제 PostgreSQL 대응과 구현 메모

| ERD(MySQL 표기) | 실제 PostgreSQL (Hibernate 생성) |
|---|---|
| `BIGINT AUTO_INCREMENT` | `bigint GENERATED BY DEFAULT AS IDENTITY` |
| `DATETIME(6)` | `timestamptz(6)` - 자바 `Instant`(UTC) |
| `VARCHAR(n)` / `TEXT` / `INT` / `BIGINT` | 동일 |

구현 시 유의 사항:

- **연쇄 삭제 주체 = 서비스 (DB 아님)**
  - 이유: zip 파일이 DB 밖(디스크)에 있어 DB `ON DELETE CASCADE`만으로는 파일이 잔존
  - FK는 기본(RESTRICT)으로 유지, 서비스가 파일 → submission_links → submissions → assignments → enrollments → cohort 순서로 삭제
  - RESTRICT = 삭제 순서 누락 시 에러로 알려 주는 안전망
- **PostgreSQL은 FK 인덱스 자동 생성 없음** (MySQL과 다름) → `idx_assignments_cohort`, `idx_submissions_*`, `idx_submission_links_submission`은 엔티티에 `@Table(indexes = ...)`로 명시
- **3종 택1 정합성** (결정 8): 서비스 검증 + DB CHECK 이중 강제 - CHECK는 Flyway 전환 시 추가
  - CODE → `code_text`·`language` NOT NULL, 파일 열·링크 없음 / FILE → `stored_path` NOT NULL / LINK → 링크 1개 이상
  - 링크 1~5개·position 연속성은 서비스 규칙으로만 강제 (자식 테이블 개수 CHECK는 DB로 표현 곤란)
- **자동 채번 동시성** (결정 9): 서비스는 "현재 최대 problem_no + 1"(없으면 1000)로 채번 - 동시 등록 충돌은 `uk_assignments_problem_no`가 최후 방어(P1 규모에서 재시도 로직은 과함, 발생 시 409로 노출)
- **기존 개발 DB는 리셋** (결정 8·9): 구 모델 행 마이그레이션 없음 - 볼륨 삭제 후 시더가 새 모델로 재생성
- 제약 이름: 코드에 실제 명시된 것은 `uk_enrollment_cohort_user`뿐 - 나머지는 현재 Hibernate가 무작위 이름으로 생성, Flyway 전환 시 이 문서의 이름으로 고정
- `ddl-auto: update`는 열 삭제 미수행 → 기존 개발 DB의 잔여 열은 무시 (운영 전 Flyway로 정리)

P2·P3에서 추가 예정 (지금은 그리지 않음 - 결정 시 이 문서에 확장):

- submissions 채점 결과 열: 판정(result)·실행시간·메모리 (P2 Judge0)
- OJ 문제 확장: assignments의 `cohort_id` NULL 허용 전환 + 난이도·정답률 열 (P3 - "OJ 문제 = 분반 없는 과제" 원칙, 별도 problems 테이블 없음. 난이도는 티어 시스템의 입력값)
- 문제 태그: `tags` + 문제-태그 N:M 조인 테이블 (P3 - 관리자 큐레이션 고정 목록, 자유 입력 금지. 탐색 필터·태그별 실력 분석용, 2026-08-25 구상)
- xp_events 테이블: 경험치 획득 이력 (P3 티어 - mvp-scope 5절 구상 메모)
- attendance(출석)·Session 엔티티: 출석부 P2 이월 확정(2026-08-25) - 도입 시 `assignments.session_no`를 Session FK로 승격 재검토
- notices 테이블: 공지사항 P2 (FE 화면 구현 완료 상태)

## 5. 관련 문서

- [guide/design.md](../guide/design.md) - Cohort 슬라이스 결정 기록 (구현 완료 부분의 원본)
- [guide/1st guide(First to Cohort).md](../guide/1st%20guide%28First%20to%20Cohort%29.md) - 코드 안내서
- [permissions.md](../permissions.md) - 권한 판정 4단계
- [mvp-scope.md](../mvp-scope.md) - P1 기능 목록 (이 문서의 결정 3이 4절·5절·6절에 반영됨)
