# 9. HOJ 는 Ondal 안의 모드 - 별도 앱 분리(결정 8) 철회

- 날짜: 2026-09-19 · 상태: 확정 · 결정자: PM
- 대체: [8-hoj-별도-앱-분리](8-hoj-%EB%B3%84%EB%8F%84-%EC%95%B1-%EB%B6%84%EB%A6%AC.md) (철회)
- 관련: FE ondal-FE PR #71(모드 전환) · #70(디자인 경계 정리) · BE 변경 없음 · [permissions.md](../permissions.md) "문제 라이브러리(HOJ)·태그" 절은 그대로

## 결정

- **HOJ는 Ondal 과 같은 앱·같은 주소 안의 모드** - 별도 빌드(`VITE_APP=hoj`)·별도 도메인(`oj.`)·별도 Pages 프로젝트를 두지 않는다
- 두 모드는 **메뉴(셸)만 다르다**
  - Ondal 모드(`AppShell`, 좌측 사이드바): 홈 · 출석 · 과제 · 내 수업 · Q&A · 공지사항 · 분반 관리(관리자)
  - HOJ 모드(`HojShell`, 상단 가로 메뉴): 문제 · 태그 관리(관리자) · (추후) 대회
  - 경로로 갈린다: `/problems/*`·`/admin/tags` = HOJ 모드, 나머지 = Ondal 모드 (`lib/appSwitch.ts` `modeOf`)
- 서로 오가는 버튼
  - Ondal 사이드바 아래쪽 **"HOJ로 이동하기"** ↔ HOJ 상단 바 **"Ondal로 이동하기"**
  - 누르면 **"~로 이동할까요?" 확인 팝업(네 / 아니요)** - 메뉴가 통째로 바뀌므로 실수로 넘어가는 것을 막는다 (`AppSwitchButton`)
  - 이동 후에는 그 모드에서 **마지막에 보던 화면**으로 - 탭 단위(sessionStorage) 기억, 없으면 홈(`/` · `/problems`)
- HOJ 모드에 있는 동안 탭 제목만 "HOJ - 해달 온라인 저지" 로 바뀜, 로고·워드마크는 청록(`--hoj-brand`)으로 Ondal(남보라)과 구분
- 백엔드는 그대로 하나. `frontend-urls.hoj`·다중 CORS 오리진(BE #47·#49)은 무해하므로 두되, 서버 `.env` 의 `HOJ_URL`·`CORS_ORIGINS` 의 `oj.` 항목은 정리해도 됨 (선택)

## 왜 결정 8 을 뒤집는가

- 결정 8 의 근거 "한 사이드바에 섞이면 메뉴가 불어난다"는 **셸을 나누는 것만으로 해결**됨 - 주소·배포까지 나눌 필요는 없었음
- 도메인 분리의 비용이 컸다
  - Cloudflare 계정이 둘이라(Pages ↔ 존) 커스텀 도메인이 CNAME 경로로만 가능했고 진행이 막힘
  - 스위치를 먼저 켜 운영 링크가 죽는 사고(FE #65 → #67 되돌림)가 이미 한 번 남
  - pages.dev 주소는 세션 쿠키 밖이라 로그인이 안 되어 검증조차 도메인 뒤에 묶임
- 같은 앱이면 세션·쿠키·CORS 고민이 없고, HOJ ↔ Ondal 이동이 SPA 라우팅이라 즉시이며, 과제 상세의 "HOJ 에서 이 문제 보기" 같은 링크가 전부 내부 링크로 남는다
- 대회가 붙을 자리는 HOJ 모드의 상단 메뉴 한 줄 - 별도 앱이 아니어도 확보됨

## 정리한 것 (FE #71)

- 삭제: `hojRoutes.tsx` · `components/HojRedirect.tsx` · `lib/apps.ts` · `VITE_APP`/`VITE_HOJ_URL`/`VITE_ONDAL_URL` · `build:hoj`/`dev:hoj` · `.env.hoj`·`.env.hoj-mock` · `deploy.yml` 의 `deploy-hoj` 잡 · 로그인 `?app=hoj`
- 추가: `lib/appSwitch.ts`(모드 판정·마지막 화면 기억) · `components/AppSwitchButton.tsx`(확인 팝업, shadcn AlertDialog) · `routes.tsx` 에서 셸 두 갈래
- FE #69(스위치 재적용 드래프트)는 닫음
- 사용자 몫(선택): Pages 프로젝트 `haedal-hoj-fe` 삭제, 서버 `.env` 의 `oj.` 항목 정리. **oj. CNAME 은 추가하지 않는다**

## 미결

- HOJ 문제 조회를 비로그인에게 열지 여부 - 로그인 필수 유지 (결정 8 의 미결 승계)
