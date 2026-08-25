# Ondal (온달)

해달 부트캠프 과제 제출·관리 플랫폼 - 자동 채점과 온라인 저지까지 한 곳에서 제공.

> 🚧 현재 개발 중 (2026 여름방학 ~ )

## 프로젝트 개요

- 목적: 부트캠프 운영의 반복 작업(과제 수합, 마감 관리, 제출 현황 파악, 채점)을 한 곳에서 처리
- **학생**: 과제 확인 → 코드·파일 제출 → 결과 확인
- **운영진**: 기수·과제 등록, 제출/미제출 현황 한눈에 파악
- **모두**: 백준처럼 상시 문제를 풀 수 있는 온라인 저지(OJ) 사용

## 개발 단계

| 단계 | 내용 | 상태 |
|------|------|------|
| P1 | 로그인, 기수/과제 관리, 제출, 현황 대시보드 | 🔨 진행 중 |
| P2 | 자동 채점 (Judge0) | ⏳ 예정 |
| P3 | 온라인 저지 모드 (상시 문제 풀 + 리더보드) | ⏳ 예정 |

## 실행 방법

> 논의 후 작성 예정

## 기술 스택

- 백엔드: Spring Boot (Java 21) + PostgreSQL 16
- 프론트엔드: React
- 채점 엔진: Judge0 CE (P2)

## 문서

처음 온 사람은 아래 순서대로 읽으면 프로젝트 전체가 잡힌다.

| 순서 | 문서 | 내용 |
|---|---|---|
| 1 | [mvp-scope.md](docs/mvp-scope.md) | P1 범위와 완료 기준 - 무엇을 만드는가 |
| 2 | [permissions.md](docs/permissions.md) | 권한 모델 - 누가 무엇을 할 수 있는가 |
| 3 | [flows-and-usecases.md](docs/flows-and-usecases.md) | 화면 흐름과 유스케이스 |
| 4 | [db/schema.md](docs/db/schema.md) | P1 DB 스키마 확정본 |
| 5 | [guide/](docs/guide/) | BE 공통 규약(design.md) + 온보딩 안내서(1st guide) |
| 6 | [enrollment/](docs/enrollment/) · [assignment/](docs/assignment/) | 도메인별 설계·API 명세 |
| 7 | [decisions/](docs/decisions/) | 의사결정 기록 - 왜 이렇게 만들었는가 |

- [기여 가이드](CONTRIBUTING.md) - 기여 방법, 문서 작성 규칙
- [행동 강령](CODE_OF_CONDUCT.md)
- [회의록](docs/meetings/)

## 라이선스

[MIT](LICENSE)
