# 8. HOJ 를 Ondal 에서 떼어 별도 앱으로 분리

- 날짜: 2026-09-15 · 갱신 2026-09-18 (도메인 `oj.` 확정, 배포 경로 Pages 로 정리) · 상태: 확정 (1단계 반영 완료, 2단계 도메인 연결 대기)
- 결정자: PM
- 관련: [db/schema.md](../db/schema.md) 결정 16(V7 문제 라이브러리) · [judge/design.md](../judge/design.md) 결정 17 · [permissions.md](../permissions.md)

## 결정

- **Ondal 과 HOJ 는 서로 다른 앱** - 화면·주소를 나눈다
  - Ondal (`ondal.…`): 부트캠프 **운영** - 분반·과제·출석·공지·Q&A
  - HOJ (`oj.…`): **문제 은행** - 문제 풀이·출제·태그. 추후 자체 프로그래밍 대회. 호스트명은 `hoj` 대신 짧은 **`oj`** (2026-09-18 PM) - `?app=hoj`·`VITE_APP=hoj`·`HOJ_URL` 같은 **키 이름은 그대로**, 주소만 `oj.haedal-sos-man-in-the-mirror.com`
- **백엔드는 하나** - 채점·사용자·세션을 두 벌로 만들지 않는다 (BE CLAUDE.md 원칙 1)
- **코드베이스도 하나** - `VITE_APP` 이 어느 앱으로 빌드할지 고른다. 페이지 컴포넌트는 두 앱이 공유

## 왜

- 성격이 다르다: Ondal 은 "마감이 있는 과제를 낸다", HOJ 는 "문제를 골라 푼다"
  - 한 사이드바에 섞으면 분반이 없는 사람에게 과제 메뉴가, 과제만 하는 사람에게 문제 메뉴가 계속 보임
- 대회를 붙일 자리가 필요하다 - Ondal 사이드바에 대회를 끼워 넣는 모양이 되면 운영 메뉴가 계속 불어난다
- 그렇다고 서버를 나누면 사용자·세션·채점기를 두 벌 관리해야 한다 - **얻는 것 없이 운영 비용만 2배**

## 어떻게

- 경로 체계를 **일부러 같게** 둔다 (`/problems/:id` 는 두 앱에서 같은 주소)
  - 같은 페이지 컴포넌트를 공유하므로 내부 링크가 양쪽에서 그대로 동작
  - 분리 배포 전후로 링크를 고칠 일이 없고, 예전 링크를 그대로 HOJ 로 넘길 수 있다
- 로그인 복귀: BE 가 `?app=` **키만** 받고 실제 주소는 서버 설정(`frontend-urls`)에서 꺼낸다 - 오픈 리다이렉트 차단
- 전환 스위치는 FE `VITE_HOJ_URL` **하나**
  - 비어 있으면: 문제·태그 화면이 Ondal 안에 그대로 (로컬·mock 개발용)
  - 값이 있으면: Ondal 의 `/problems/*`·`/admin/tags` 를 HOJ 로 넘기고, 사이드바에서 태그 관리 메뉴를 빼고, HOJ 링크를 새 탭으로 연다

## 대안과 기각 사유

| 안 | 기각 사유 |
|---|---|
| Ondal 안에 탭으로 유지 | 성격이 달라 메뉴가 계속 섞임. 대회가 붙을 자리가 없음 |
| 레포·백엔드까지 분리 | 사용자·세션·채점기 2벌. 문제 하나를 과제로 배정하는 흐름이 서버 간 연동이 됨 |
| Ondal 라우트를 지우고 HOJ 로만 접근 | 그동안 공유된 `/problems/12` 링크·북마크가 전부 404 |

## 진행 상태

- ✅ 1단계 - 코드 분리: `lib/apps.ts` 스위치 · `hojRoutes.tsx` · `HojShell` · `dist-hoj` 빌드(`npm run build:hoj`) · BE 다중 오리진 복귀(CORS·`frontend-urls.hoj`)
- ✅ 1.5단계 - 전환 스위치: `HojRedirect` 로 예전 주소를 경로 그대로 HOJ 로 넘김 (지금은 스위치 OFF 라 운영 동작 변화 없음)
- ⬜ 2단계 - 배포 (2026-09-18 정리): HOJ 는 Pages 프로젝트 `haedal-hoj-fe` 로 이미 배포된다 - FE `deploy.yml` 의 `deploy-hoj` 잡이 main push 마다 `npm run build:hoj` → `dist-hoj` 를 올림 (2026-09-16 첫 배포, https://haedal-hoj-fe.pages.dev). 남은 순서:
  1. 대시보드에서 Pages 프로젝트에 커스텀 도메인 **`oj.haedal-sos-man-in-the-mirror.com`** 연결 (org owner 권한 불필요). pages.dev 주소는 세션 쿠키 same-site 밖이라 로그인이 안 되므로 이 단계가 필수
  2. 서버 `.env` 에 `HOJ_URL=https://oj.…`, `CORS_ORIGINS` 에 `https://oj.…` 반영 후 BE 재기동 (BE 기본값도 `oj.` 로 맞춤 - ondal-BE PR)
  3. FE `.env.production` 에 `VITE_HOJ_URL=https://oj.…` 한 줄 추가 → Ondal 의 `/problems/*`·`/admin/tags` 가 HOJ 로 넘어감 (ondal-FE 드래프트 PR - 1·2 와 운영 Ondal 배포 파이프라인 복구 뒤 머지)
  - ※ 순서를 뒤집어 도메인 없이 3 부터 넣으면 `/problems` 가 없는 주소로 넘어감
  - 경위: 2026-09-16 FE #62 가 "Pages 프로젝트 생성이 막혔다"고 보고 Worker `ondal-hoj` 로 바꿨으나 오판이었다 - 같은 날 #61 의 첫 실행이 `haedal-hoj-fe` 를 생성·배포하는 데 성공한 기록이 있고, Worker 배포는 레포 토큰에 Workers 권한이 없어 실패만 반복했다 → FE #64 에서 Pages 로 되돌림

## 미결

- HOJ 문제 조회를 **비로그인에게 열지 여부** - 당분간 로그인 필수 유지(2026-09-15 PM). 대회·홍보 필요 시 재검토
- 두 앱의 탭 제목 분리 (`index.html` 이 공용)
