# 7. 홈페이지 SSO - Keycloak OIDC 채택, BE 가 코드를 교환 (결정 5 일부 갱신)

- 날짜: 2026-09-13 · 상태: 확정 (BE ondal-BE 이슈 #27 / PR #30 · FE ondal-FE 이슈 #24 / PR #25 - 2026-09-14 운영 배포·브라우저 로그인 검증 완료)

## 결정

- 로그인 = **홈페이지 계정(Keycloak)** - OIDC Authorization Code + PKCE, **BE 가 confidential client 로 코드를 교환**
- Ondal 은 비밀번호를 받지 않음(인증 금지선) - 신원은 Keycloak 이 서명한 ID 토큰(`preferred_username` → `login_id`, `name` → 이름)으로만 확인
- 이후 API 인증은 **세션 그대로** (결정 5 유지) - 토큰은 로그인 순간에만 쓰고 API 인증에 쓰지 않음
- 개발·테스트는 스텁 유지 (`ondal.auth.mode=stub`), 운영은 `oidc` - prod 프로필에서 stub 이면 기동 거부

## 근거

1. **결정 5 의 재검토 조건 충족** - "홈페이지 SSO 가 토큰 발급 방식으로 확정" → 대응은 "토큰 검증 AuthService + 필요 시 세션 → 토큰 전환" 이었으나, 전환하지 않아도 됨
   - BE 가 코드를 교환하면 토큰은 서버 안에서만 존재 - 브라우저에는 세션 쿠키만
   - 단일 서버·강제 로그아웃 즉시성 등 세션의 이점(결정 5 근거 3)은 그대로 유효
2. **대안 폐기 - FE 가 토큰을 받아 BE 에 전달**
   - 토큰이 브라우저 JS 에 노출(XSS 시 탈취)·BE 는 매 요청 JWT 검증 → 세션 모델과 이중
   - Keycloak JS 어댑터 도입 = FE 학습 부하 추가
3. **Spring Security 필터체인은 여전히 미도입**
   - 필요한 것은 ID 토큰 서명 검증뿐 - `spring-security-oauth2-jose` 의 `NimbusJwtDecoder` 만 사용(JWKS 캐시·키 회전)
   - 분반 권한 모델(Enrollment.role)과 시큐리티 전역 Role 모델의 불일치(결정 5 근거 2)는 변하지 않음
4. **계정 매칭 키 = Keycloak username**
   - 운영진이 수강생을 배정할 때 사람이 아는 값이어야 함(`sub` UUID 는 불가) - 부트스트랩 SQL 도 같은 값
   - 전제: realm 의 Edit username OFF (인프라 요청서)

## 감수하는 것

- Keycloak 다운 시 신규 로그인 불가 (기존 세션은 정상) - Discovery 를 첫 사용 시 조회해 앱 기동은 IdP 와 무관
- 로그인 두 엔드포인트의 실패는 JSON 401 이 아니라 FE `/login?error=코드` 302 - 브라우저 이동이라 불가피. 나머지 API 의 401 계약은 불변
- 공용 PC 대비 로그아웃은 FE 가 `logoutUrl`(Keycloak end-session) 로 이동해야 완성 - FE 완료 (ondal-FE PR #25: 로그인 방식 `VITE_AUTH_MODE=oidc` 빌드 고정, `/login?error=` 6종 안내, `logoutUrl` 이동)
- username 변경 시 다른 사람으로 인식 - realm 설정으로 봉인, 추후 필요하면 `users.sub` 열 추가(마이그레이션) 로 전환

## 재검토 조건

- 홈페이지 회원 명부 API 연동 시(permissions.md 5절): 배정 UX 가 username 입력 → 명부 선택으로 바뀌면 매칭 키를 `sub` 로 바꾸는 편이 안전
- 모바일·서버 확장 등 결정 5 의 조건은 그대로
