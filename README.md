# Ondal (온달)

해달 부트캠프 과제 제출·관리 플랫폼 - 자동 채점과 온라인 저지까지 한 곳에서 제공.

> 🟢 **운영 가동 중** - P1·P2 전부 배포 완료 (2026-09-15). 테스트 주간 2026-09-19~25
>
> - Ondal: https://ondal.haedal-sos-man-in-the-mirror.com
> - HOJ: Ondal 안 `/problems` - 같은 앱의 **모드**(사이드바 "HOJ로 이동하기" → 확인 팝업 → HOJ 메뉴) - [결정 9](docs/decisions/9-hoj-ondal-%EB%82%B4%EB%B6%80-%EB%AA%A8%EB%93%9C%EB%A1%9C-%ED%86%B5%ED%95%A9.md) (별도 도메인 `oj.` 분리는 2026-09-19 철회)

## 프로젝트 개요

- 목적: 부트캠프 운영의 반복 작업(과제 수합, 마감 관리, 제출 현황 파악, 채점)을 한 곳에서 처리
- **학생**: 과제 확인 → 코드·파일 제출 → 결과 확인
- **운영진**: 기수·과제 등록, 제출/미제출 현황 한눈에 파악
- **모두**: 백준처럼 상시 문제를 풀 수 있는 온라인 저지(OJ) 사용

## 개발 단계

| 단계 | 내용 | 상태 |
|------|------|------|
| P1 | 로그인(홈페이지 Keycloak SSO), 분반·과제 관리, 제출, 현황 대시보드 | ✅ 운영 가동 |
| P2 | 자동 채점(Judge0) · 공지사항 · 출석부 · Q&A 답변 · 제출 코멘트 · 문제 라이브러리(V7)·태그·HOJ · 승인 게이트([결정 10](docs/decisions/10-%EC%8A%B9%EC%9D%B8-%EA%B2%8C%EC%9D%B4%ED%8A%B8-%EC%B2%AB-%EB%A1%9C%EA%B7%B8%EC%9D%B8%EC%9D%80-%EC%8A%B9%EC%9D%B8-%EB%8C%80%EA%B8%B0.md), 2026-09-19) · HOJ 100문제 준비([결정 11](docs/decisions/11-%EB%AC%B8%EC%A0%9C-%EC%9D%80%ED%96%89-%EB%82%9C%EC%9D%B4%EB%8F%84-%ED%97%88%EC%9A%A9-%EC%96%B8%EC%96%B4-%EB%B2%88%EB%93%A4-%EA%B0%80%EC%A0%B8%EC%98%A4%EA%B8%B0.md) - 난이도·허용 언어·번들 가져오기, 비공개 레포 ondal-problems) · 프로필 사진 = 구글 프로필 연동([결정 14](docs/decisions/14-%ED%94%84%EB%A1%9C%ED%95%84-%EC%82%AC%EC%A7%84-%ED%99%88%ED%8E%98%EC%9D%B4%EC%A7%80-%EA%B5%AC%EA%B8%80-%ED%94%84%EB%A1%9C%ED%95%84-%EC%97%B0%EB%8F%99.md), 2026-09-20) · 관리자(MAINTAINER) 전역 역할([결정 12](docs/decisions/12-%EA%B4%80%EB%A6%AC%EC%9E%90-%EC%A0%84%EC%97%AD-%EC%97%AD%ED%95%A0-%ED%95%B4%EA%B5%AC%EB%A5%B4%EB%A5%B4%EC%99%80-%EA%B0%99%EC%9D%80-%EA%B6%8C%ED%95%9C.md), 2026-09-19 - 해구르르와 같은 권한, 유지보수 팀용) | ✅ 운영 가동 (승인 게이트·HOJ은 배포 대기) |
| P3 | 리더보드·티어, 대회, 수강 신청 | ⏳ 예정 |

- 알림(디스코드·이메일)과 점수는 **두지 않기로 확정**(2026-09-14) - 코멘트만 남김
- "OJ 문제 = 분반 없는 과제" 원칙은 **폐기** - 문제(`problems`)와 배정(`assignments`)을 분리 (V7)

## 실행 방법

> 논의 후 작성 예정

## 기술 스택

- 백엔드: Spring Boot (Java 21) + PostgreSQL 16 + Flyway (V1~V7)
- 프론트엔드: React 19 + Vite + TanStack Query + Tailwind - Ondal·HOJ 는 **한 앱 안의 두 모드**(셸만 다름, 결정 9)
- 인증: 홈페이지 Keycloak OIDC (인가 코드 + PKCE, BE 가 코드 교환) - Ondal 은 비밀번호를 직접 받지 않음
- 채점 엔진: Judge0 CE - 별도 VM 에서 가동, 판정은 서버 비교기가 함

## 문서

처음 온 사람은 아래 순서대로 읽으면 프로젝트 전체가 잡힌다.

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [mvp-scope.md](docs/mvp-scope.md) | P1 범위와 완료 기준 - 무엇을 만드는가 |
| 2 | [permissions.md](docs/permissions.md) | 권한 모델 - 누가 무엇을 할 수 있는가 |
| 3 | [flows-and-usecases.md](docs/flows-and-usecases.md) | 화면 흐름과 유스케이스 |
| 4 | [db/schema.md](docs/db/schema.md) | P1 DB 스키마 확정본 |
| 5 | [guide/](docs/guide/) | BE 공통 규약(design.md) + 온보딩 안내서(1st guide) |
| 6 | [enrollment/](docs/enrollment/) · [assignment/](docs/assignment/) · [submission/](docs/submission/) · [qna/](docs/qna/) · [notice/](docs/notice/) · [attendance/](docs/attendance/) · [judge/](docs/judge/) | 도메인별 설계·API 명세 (judge = 자동 채점, 서버 구성 infra.md 포함) |
| 7 | [decisions/](docs/decisions/) | 의사결정 기록 - 왜 이렇게 만들었는가 |
| 8 | [test-week-checklist.md](docs/test-week-checklist.md) | 테스트 주간(9/19~25) 운영 서버 점검 체크리스트 - 역할별 항목, 문제 보고 형식 |
| 9 | [screens/my-page.md](docs/screens/my-page.md) | 마이페이지 화면 정의 - 기준본(v2.1)에 없는 화면의 기준 (2026-09-19) |
| 10 | [audit-fe-2026-09-20.md](docs/audit-fe-2026-09-20.md) | 프런트 전수 조사 - 원안(ui-v1) 대비 반영도 · 남은 것 (2026-09-20) |

- [기여 가이드](CONTRIBUTING.md) - 기여 방법, 문서 작성 규칙
- [행동 강령](CODE_OF_CONDUCT.md)
- [회의록](docs/meetings/)

## 라이선스

[MIT](LICENSE)
