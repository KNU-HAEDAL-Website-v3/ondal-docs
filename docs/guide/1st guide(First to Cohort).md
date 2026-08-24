# 1st guide - 스프링 기초부터 첫 도메인(Cohort)까지

> 대상: Ondal 백엔드 코드를 처음 읽는 사람 - Spring·JPA 사전 지식 불필요. 기술 면접에서 "왜 이렇게 짰나"까지 답할 수 있는 깊이를 목표로 함
> 구성: **1부(1~9장) 기초** - 웹·서블릿·스프링·JPA·세션·테스트의 바닥 / **2부(10~22장) Cohort 슬라이스** - 그 바닥 위에서 `cohort`·`enrollment` 코드가 실제로 움직이는 방식
> 짝 문서: [design.md](design.md) = "왜 이렇게 정했나"(결정 기록) / 이 문서 = "기초 + 코드가 어떻게 움직이나"(안내서) / 수강생 배정 결정: [enrollment/design.md](../enrollment/design.md)
> 기준 코드: BE `main` `2e8e774` (분반 PR #2 · 패키지 세분화 PR #3 · 수강생 배정 PR #5 · 문서 윤문 PR #7 머지, 2026-08-22). 테스트 69개
> 이어지는 안내서: 2nd guide(과제 슬라이스)부터는 1부 기초를 반복하지 않고 이 문서의 장 번호를 참조함

---

## 0. 읽는 순서

| 장 | 내용 | 비고 |
|---|---|---|
| **1부. 기초** | | 스프링을 처음 보면 1장부터, 경험자는 9장 질문으로 자가 점검 후 모르는 장만 |
| 1장 | 웹 백엔드가 하는 일 - HTTP·REST·JSON·무상태 | 모든 장의 공통 언어 |
| 2장 | 자바 웹 서버의 뼈대 - 서블릿·Tomcat·DispatcherServlet | 요청이 컨트롤러에 닿기까지 |
| 3장 | 스프링 핵심 - IoC 컨테이너·빈·의존성 주입 | 코드의 생성자들이 하는 일 |
| 4장 | 프로젝트 뼈대 - Gradle·스타터·application.yml·docker-compose | 파일 한 줄씩 |
| 5장 | 스프링 MVC 사용법 - 컨트롤러 어노테이션·Jackson·WebConfig | 컨트롤러가 읽히기 시작하는 장 |
| 6장 | JPA·Hibernate·Spring Data JPA - 엔티티·영속성 컨텍스트·트랜잭션·리포지토리 | 가장 긴 장. 6.3·6.5가 핵심 |
| 7장 | 인증·인가의 기초 - 세션·쿠키·보안 용어·Spring Security를 보류한 이유 | 401/403/404 구분 |
| 8장 | 테스트 기초 - JUnit 5·MockMvc·Testcontainers | 테스트 코드를 읽기 위한 준비 |
| 9장 | 1부 점검 - 면접 예상 질문 | 답 위치를 장·절로 표기 |
| **2부. Cohort 슬라이스** | | 기존 "Cohort 코드 안내서"를 1부 위에 다시 놓은 것 |
| 10장 | 무엇을 만들었나 - 수직 슬라이스 | |
| 11장 | 데이터 - 사람·분반·소속 (엔티티 코드 읽기) | |
| 12장 | 요청 하나의 여행 - **여기까지 읽으면 뼈대 완성** | |
| 13장 | 권한 장치 상세 | 최고 위험 영역 |
| 14장 | 소속 다루기 - `EnrollmentService` (수강생 배정 포함) | |
| 15장 | 로그인 - `auth/` | |
| 16장 | 설정·CORS·시더·springdoc | |
| 17장 | 예외 처리 - `common/error` | |
| 18장 | API 한 장 (#1~#12) | |
| 19장 | 테스트 - 69개가 도는 방식 | |
| 20장 | 다음 도메인을 만들 때 - 복제 절차 | 2nd guide의 출발점 |
| 21장 | 파일 지도 | 코드를 열 때 참조 |
| 22장 | 전체 요약 + 2부 면접 질문 | 완독 후 기억 점검 |

---

# 1부. 기초

## 1. 웹 백엔드가 하는 일 - HTTP·REST·JSON·무상태

### 1.1 클라이언트와 서버

- 클라이언트: 요청을 보내는 쪽 - Ondal에서는 브라우저에서 도는 프런트엔드(React, 로컬 `localhost:5173`, 배포 Cloudflare Pages)
- 서버: 요청을 받아 처리하고 응답을 돌려주는 쪽 - Ondal 백엔드(Spring Boot, 로컬 `localhost:8080`)
- 백엔드의 일 = **요청을 받아 → 규칙을 적용하고 → DB를 읽고 쓰고 → 응답을 만든다**. 이 문서 전체가 이 네 단계의 세부
- 둘 사이의 약속(계약) = HTTP 위의 API. 계약의 기준 문서는 서버가 자동 생성하는 Swagger UI(16장)

### 1.2 HTTP 요청과 응답의 생김새

- 요청 = **메서드 + 경로(+쿼리) + 헤더 + 본문(선택)**
- 응답 = **상태코드 + 헤더 + 본문(선택)**
- 로그인 요청·응답의 실제 모습:

```http
POST /api/auth/login HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{"loginId":"admin"}
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Set-Cookie: JSESSIONID=3F2A...; Path=/; HttpOnly; SameSite=Lax

{"id":1,"loginId":"admin","name":"관리자","globalRole":"ADMIN"}
```

- 이어지는 요청은 받은 쿠키를 다시 실어 보냄 - 이것이 "로그인 상태"의 정체(1.6절, 7장):

```http
GET /api/auth/me HTTP/1.1
Host: localhost:8080
Cookie: JSESSIONID=3F2A...
```

- 용어
  - 헤더: 메타데이터. `Content-Type`(본문 형식), `Cookie`/`Set-Cookie`(쿠키), `Location`(생성된 자원 위치), `Origin`(요청 출처)
  - 본문(body): 실제 데이터. Ondal은 요청·응답 모두 JSON
  - 쿼리 파라미터: 경로 뒤 `?key=value` - 필터·옵션용. 예: `GET /api/cohorts?status=ARCHIVED`
  - 경로 변수: 경로 안의 식별자. 예: `/api/cohorts/3`의 `3`

### 1.3 HTTP 메서드의 의미와 멱등성

| 메서드 | 의미 | 안전(읽기만) | 멱등 | Ondal에서의 쓰임 |
|---|---|---|---|---|
| GET | 조회 | O | O | 목록·상세·명부·내 정보 |
| POST | 생성 또는 "행위" | X | X (원칙) | 분반 생성, 로그인/로그아웃, 수강생 일괄 배정, `archive`/`restore` 같은 상태 전이(Ondal은 멱등으로 설계) |
| PUT | 전체 교체 | X | O | 분반 수정(이름·설명 전부 보냄), 운영진 지정 |
| PATCH | 부분 수정 | X | 경우에 따라 | Ondal 미사용 |
| DELETE | 삭제 | X | O | 운영진 해제, 수강생 제외 |
| OPTIONS | 서버가 허용하는 것 질의 | O | O | 브라우저의 CORS 사전 요청(1.7절) |

- **멱등(idempotent)**: 같은 요청을 여러 번 보내도 결과(서버 상태)가 한 번 보낸 것과 같음
  - 예: `POST /api/cohorts/3/archive`를 두 번 눌러도 상태는 ARCHIVED 한 번 - 네트워크 재시도에 안전
  - Ondal 규약(design.md 4절): 상태 전이 = `POST /{id}/{동사}` 멱등, 수정 = PUT 전체 교체, 삭제 = DELETE 204(대상 없으면 404)

### 1.4 상태코드 - 숫자 한 자리가 뜻하는 것

| 코드 | 뜻 | Ondal의 에러 코드(`code`) | 언제 |
|---|---|---|---|
| 200 OK | 성공 | - | 조회·수정·멱등 전이 |
| 201 Created | 생성됨 | - | `POST /api/cohorts` (+ `Location` 헤더) |
| 204 No Content | 성공, 본문 없음 | - | DELETE 성공 |
| 400 Bad Request | 요청 형식·값이 잘못됨 | `INVALID_INPUT` | 빈 이름, 51자 loginId, 깨진 JSON, `/api/cohorts/abc` |
| 401 Unauthorized | **미인증**(이름과 달리 "로그인 안 됨") | `UNAUTHENTICATED` | 세션 없음·만료·삭제된 사용자 |
| 403 Forbidden | 인증은 됐으나 권한 없음 | `FORBIDDEN` | 일반 부원이 분반 생성, 남의 반 조회 |
| 404 Not Found | 자원 없음 (또는 **존재를 숨김**) | `NOT_FOUND` | 없는 분반 id, 다른 반의 하위 자원, 없는 경로 |
| 405 Method Not Allowed | 그 경로에 그 메서드 없음 | `METHOD_NOT_ALLOWED` | `DELETE /api/cohorts` |
| 409 Conflict | 현재 상태와 충돌 | `CONFLICT`, `COHORT_ARCHIVED` | 이미 운영진인 사람을 수강생 배정, 보관 분반 변경 |
| 500 Internal Server Error | 서버 버그 | `INTERNAL_ERROR` | 처리되지 않은 예외(원인은 로그에만) |

- 1xx 정보 / 2xx 성공 / 3xx 리다이렉트 / 4xx 클라이언트 잘못 / 5xx 서버 잘못
- 401 vs 403 vs 404의 구분 = 7.1절. Ondal 프런트 규칙: **403만 홈으로**, 401은 로그인으로, 404는 안내 페이지

### 1.5 REST와 자원 URL

- REST: "자원(resource)을 URL로 식별하고, 메서드로 행위를 표현"하는 API 설계 스타일
- URL 설계 관례
  - 자원은 **명사·복수형**: `/api/cohorts`, `/api/cohorts/{cohortId}/students`
  - 계층(소유)은 경로 중첩으로: 분반 3의 수강생 `s1` → `/api/cohorts/3/students/s1`
  - 필터·옵션은 쿼리로: `?status=ARCHIVED`
  - 동사는 피하되, 상태 전이처럼 자원 모델로 표현이 어색하면 `POST /{id}/{동사}` 허용(`archive`, `restore`)
- Ondal의 추가 규칙: 분반에 속한 자원은 **반드시** `/api/cohorts/{cohortId}/...` 아래 - 권한 판정이 이 경로 변수를 읽기 때문(13장)

### 1.6 무상태(stateless)와 "로그인 상태"

- HTTP는 무상태: 요청 하나하나가 독립적. 서버는 이전 요청을 기억하지 않음
- 그런데 "로그인한 사용자"는 기억해야 함 → 두 가지 방법
  - **세션**: 서버가 사용자별 기억 공간(세션)을 만들고, 그 열쇠(세션 id)를 쿠키로 브라우저에 줌. 이후 요청마다 쿠키가 실려 옴 → Ondal 채택
  - **토큰(JWT 등)**: 서버가 서명한 증표를 주고, 브라우저가 헤더에 실어 보냄. 서버는 기억 없이 서명만 검증
- 세부 비교와 선택 이유: 7.2~7.3절

### 1.7 JSON과 직렬화

- JSON: `{"key": value}` 텍스트 형식. 자바 객체와 1:1은 아니므로 변환이 필요
  - **직렬화**: 자바 객체 → JSON 문자열 (응답)
  - **역직렬화**: JSON 문자열 → 자바 객체 (요청 본문)
- 변환 라이브러리 = **Jackson**(Spring Boot 기본). 규칙은 5.4절
- 요청에 `Content-Type: application/json`이 없거나 본문이 JSON이 아니면 서버는 본문을 읽지 못함 → 400

### 1.8 출처(origin)와 CORS 미리보기

- 출처 = **스킴 + 호스트 + 포트**. `http://localhost:5173`과 `http://localhost:8080`은 **다른 출처**
- 브라우저의 기본 규칙(동일 출처 정책): 다른 출처의 응답을 스크립트가 읽지 못하게 차단
- 서버가 "이 출처는 허용"이라고 응답 헤더로 알려 주는 절차 = **CORS**. 일부 요청은 본 요청 전에 브라우저가 `OPTIONS`로 먼저 물어봄 = **사전 요청(preflight)**
- Ondal의 CORS 설정과 원리: 5.6절·7.6절. 인터셉터가 사전 요청을 통과시키는 코드가 있는 이유도 거기에

---

## 2. 자바 웹 서버의 뼈대 - 서블릿·Tomcat·DispatcherServlet

### 2.1 서블릿(Servlet)

- 자바에서 HTTP 요청을 처리하는 표준 규격(Jakarta Servlet). "요청 객체와 응답 객체를 받아 처리하는 자바 클래스"
- 핵심 타입 - 우리 코드에도 그대로 등장
  - `HttpServletRequest`: 메서드·경로·헤더·본문·세션 접근. `AuthInterceptor`, `AuthController`가 사용
  - `HttpServletResponse`: 상태코드·헤더·본문 기록
  - `HttpSession`: 서버 측 세션. `request.getSession(false)`로 꺼냄(7.2절)
- 패키지가 `jakarta.servlet.*`인 이유: Java EE → Jakarta EE로 이관되며 `javax.*`가 `jakarta.*`로 바뀜(Spring 6 / Boot 3 이후)

### 2.2 서블릿 컨테이너 Tomcat과 요청당 스레드

- 서블릿은 혼자 못 돎. 소켓을 열고 HTTP를 파싱해 서블릿을 호출하는 프로그램 = **서블릿 컨테이너** = Tomcat
- Spring Boot는 Tomcat을 **라이브러리로 jar 안에 내장** → `java -jar` 한 줄로 웹서버 기동 (4.7절)
- 동작 모델: 연결 수신 → HTTP 파싱 → **스레드 풀에서 스레드 하나를 꺼내 요청 하나를 끝까지 처리** → 응답
  - 의미 1: 동시에 들어온 요청은 서로 다른 스레드에서 **같은 빈 객체**(싱글톤, 3.5절)를 공유 → 빈에 요청별 상태를 두면 안 됨
  - 의미 2: 트랜잭션·영속성 컨텍스트는 "현재 스레드"에 묶임(ThreadLocal) - 6.5절

### 2.3 Filter와 Interceptor

| | Filter | HandlerInterceptor |
|---|---|---|
| 소속 | 서블릿 표준 | 스프링 MVC |
| 위치 | DispatcherServlet **앞** | DispatcherServlet **안**, 컨트롤러 직전·직후 |
| 아는 것 | 요청·응답만 | + 어떤 컨트롤러 메서드가 처리할지(`HandlerMethod`) |
| 예외 처리 | `@RestControllerAdvice`에 닿지 않음 | 닿음 → 같은 `{code, message}` 응답 |
| Ondal | 직접 작성 없음(Boot 기본만) | `AuthInterceptor`, `AuthorizationInterceptor` |

- Ondal이 인터셉터를 고른 이유: 권한 판정에 "이 핸들러에 어떤 어노테이션이 붙었나"가 필요 → 핸들러를 아는 층이어야 함. 예외도 한 곳(17장)으로 모임

### 2.4 DispatcherServlet - 요청이 컨트롤러에 닿기까지

- 스프링 MVC의 유일한 서블릿. 모든 요청의 단일 진입점(프런트 컨트롤러 패턴)
- 내부 흐름 - 2부 12장의 "요청 하나의 여행"이 이 그림을 그대로 따라감:

```
Tomcat ─ Filter 체인 ─▶ DispatcherServlet.doDispatch(request, response)
  │
  ├─ (1) HandlerMapping.getHandler(request)
  │       → HandlerExecutionChain { handler = HandlerMethod(컨트롤러 메서드), interceptors[] }
  │         경로 변수는 이 시점에 request attribute(URI_TEMPLATE_VARIABLES)에 문자열로 저장됨
  │         CORS 요청이면 CorsInterceptor가 체인 맨 앞에 붙고, 사전 요청이면 handler가 PreFlightHandler로 바뀜
  │
  ├─ (2) interceptors[i].preHandle() 등록 순서대로
  │       Ondal: AuthInterceptor(로그인?) → AuthorizationInterceptor(권한?)
  │       false 반환 또는 예외 → 컨트롤러 미실행
  │
  ├─ (3) HandlerAdapter.handle()
  │       ├─ ArgumentResolver 들이 파라미터를 채움: @PathVariable, @RequestParam, @RequestBody(+@Valid), @LoginUser
  │       ├─ 컨트롤러 메서드 실행 → 서비스 → 리포지토리 → DB
  │       └─ ReturnValueHandler → HttpMessageConverter(Jackson) → 응답 본문 기록
  │
  ├─ (4) interceptors.postHandle() 역순 → afterCompletion()
  │
  └─ 어느 단계든 예외 발생 시: HandlerExceptionResolver 체인
        → ExceptionHandlerExceptionResolver → @RestControllerAdvice 의 @ExceptionHandler (GlobalExceptionHandler)
```

- 이 그림에서 기억할 것 세 가지
  1. 인터셉터는 **파라미터 바인딩보다 먼저** 돈다 → `AuthorizationInterceptor`가 `{cohortId}`를 문자열로 직접 읽고 숫자 변환 실패를 스스로 400 처리하는 이유(13장)
  2. 인터셉터·리졸버·컨트롤러·서비스 **어디서 던진 예외든** 같은 예외 핸들러로 모임
  3. 사전 요청(OPTIONS)도 인터셉터 체인을 탄다 → 인터셉터 첫 줄의 `CorsUtils.isPreFlightRequest` 검사

### 2.5 "MVC"의 M·V·C와 REST API

- Model(데이터)·View(화면)·Controller(요청 처리)의 분리 패턴
- 서버가 HTML을 그리던 시절: 컨트롤러가 View 이름을 반환 → 템플릿 렌더링
- REST API 서버(Ondal): View가 없음. 컨트롤러 반환값을 **메시지 컨버터가 JSON으로 직렬화**해 본문에 기록 - `@RestController`가 그 선언(5.1절)

---

## 3. 스프링 핵심 - IoC 컨테이너·빈·의존성 주입

### 3.1 스프링과 스프링 부트

- 스프링 프레임워크: 자바 객체(POJO)들을 **컨테이너가 생성·연결·관리**하게 해 주는 프레임워크. 핵심 = IoC 컨테이너(`ApplicationContext`)
- 스프링 부트: 그 위에 **자동 설정 + 내장 서버 + 스타터 의존성**을 얹어 설정 없이 바로 뜨게 한 층. 스프링 없는 부트는 없음
- Ondal 채택 근거: [decisions/1](../decisions/1-백엔드-spring-채택.md)

### 3.2 IoC(제어의 역전)와 DI(의존성 주입)

- IoC: 객체를 "누가 만들고 연결하느냐"의 제어권이 내 코드가 아니라 컨테이너에 있음
  - 예: `CohortController`는 `new CohortService(...)`를 하지 않음. 컨테이너가 `CohortService` 빈을 만들어 생성자에 넣어 줌
- DI: IoC를 실현하는 방법 - 의존 객체를 밖에서 넣어 줌
- 주입 방식 3가지: 생성자 주입 / 세터 주입 / 필드 주입(`@Autowired` 필드)
- Ondal 메인 코드는 **전부 생성자 주입**. 생성자가 하나면 `@Autowired` 생략 가능(Spring 4.3+) → 메인 코드에 `@Autowired`가 한 곳도 없음

```java
// CohortService.java - 생성자 하나 = 이 클래스가 의존하는 것 전부
public CohortService(CohortRepository cohortRepository,
                     EnrollmentService enrollmentService,
                     CohortResponseAssembler assembler) { ... }
```

### 3.3 생성자 주입을 고른 이유 (면접 단골)

- 필드를 `final`로 둘 수 있음 → 불변, 주입 누락 시 컴파일 에러
- 필수 의존성이 생성자 시그니처에 드러남 → 스프링 없이 `new`로 조립해 단위 테스트 가능
- 순환 의존을 **기동 시점에** 발견(필드 주입은 실행 중에야 드러남)
- 예외적으로 필드 주입을 쓴 곳과 이유
  - 테스트 클래스의 `@Autowired` 필드(`ApiTestSupport`): 테스트 인스턴스는 JUnit이 만들므로 생성자 주입이 어색 → 관례상 허용
  - `DatabaseCleaner`의 `@PersistenceContext EntityManager`: JPA 표준 어노테이션. 트랜잭션에 따라 바뀌는 **EntityManager 프록시**를 주입받는 방식

### 3.4 빈(Bean)과 등록 방법·스테레오타입

- 빈 = 컨테이너가 생성·관리하는 객체
- 등록 방법
  1. **컴포넌트 스캔**: `@Component` 계열이 붙은 클래스를 찾아 자동 등록 - Ondal 메인 코드 전부
  2. **`@Bean` 메서드**: `@Configuration` 클래스 안에서 메서드가 반환한 객체를 등록 - 내가 어노테이션을 못 붙이는 외부 클래스용. Ondal에서는 테스트의 `PostgresContainerConfig`(`PostgreSQLContainer` 빈) 하나
- 스테레오타입 어노테이션 = "이 클래스는 빈이며 역할은 X"라는 이름표

| 어노테이션 | 의미 | Ondal에서 |
|---|---|---|
| `@Component` | 범용 빈 | 인터셉터 2개, `LoginUserArgumentResolver`, `CohortAuthorizer`, `CohortResponseAssembler`, `AuthorizationMappingValidator`, `LocalDataSeeder`, 테스트 `DatabaseCleaner`·`LoginHelper` |
| `@Service` | 비즈니스 계층 표시 - 기능 차이 없음 | `UserService`, `CohortService`, `EnrollmentService`, `StubAuthService` |
| `@Repository` | 영속 계층 + **예외 변환**(JPA 예외 → 스프링 `DataAccessException`) | 직접 안 붙임. Spring Data 인터페이스는 자동 등록·자동 변환 |
| `@Controller` / `@RestController` | 웹 핸들러. `@RestController` = `@Controller` + `@ResponseBody` | 컨트롤러 4개 |
| `@Configuration` | 설정 클래스 | `WebConfig` |
| `@RestControllerAdvice` | 전역 예외 처리 빈 | `GlobalExceptionHandler` |

- 면접: `@Service`와 `@Component`의 차이 → 기능 없음. `@Service`는 `@Component`를 메타 어노테이션으로 가진 이름표. `@Repository`만 예외 변환이라는 실제 기능이 있음

### 3.5 빈 스코프 - 싱글톤과 "상태를 갖지 않는다"

- 기본 스코프 = **싱글톤**: 컨테이너당 인스턴스 하나. 모든 요청(스레드)이 공유(2.2절)
- 따라서 빈의 필드에는 **의존성만**. 요청별 데이터(세션, 사용자)는 필드에 저장하지 않고 메서드 인자·`request`에서 매번 꺼냄 - `AuthInterceptor`가 그렇게 생긴 이유
- 다른 스코프(prototype, request, session)도 있으나 Ondal은 전부 싱글톤

### 3.6 `@SpringBootApplication`과 기동 순서

```java
@SpringBootApplication
public class OndalApplication {
    public static void main(String[] args) {
        SpringApplication.run(OndalApplication.class, args);
    }
}
```

- `@SpringBootApplication` = 세 어노테이션의 합성
  - `@SpringBootConfiguration`(= `@Configuration`): 이 클래스가 설정 클래스
  - `@EnableAutoConfiguration`: 자동 설정 켜기(3.7절)
  - `@ComponentScan`: **이 클래스가 있는 패키지(`kr.haedal.ondal`)와 하위**를 스캔
- → 모든 코드가 `kr.haedal.ondal.*` 아래에 있는 이유. 다른 패키지에 `@Service`를 만들면 스캔 대상이 아님(빈이 안 됨)
- `SpringApplication.run`의 순서
  1. 환경(Environment) 준비 - `application.yml` 읽기, 프로필 결정
  2. `ApplicationContext` 생성
  3. 빈 정의 로딩 - 컴포넌트 스캔 + 자동 설정
  4. 싱글톤 빈 전부 생성 - 의존 그래프 순서대로 생성자 호출
  5. 내장 Tomcat 기동 - 포트 열림
  6. `CommandLineRunner` / `ApplicationRunner` 호출

### 3.7 자동 설정(Auto-configuration)

- 클래스패스를 보고 "있으면 켜는" 조건부 설정
  - `@ConditionalOnClass`: 예) Hibernate가 있으면 JPA 설정 로드
  - `@ConditionalOnMissingBean`: 내가 같은 타입 빈을 만들면 자동 설정은 **물러남**
- Ondal이 한 줄도 안 썼는데 생긴 빈: `DataSource`(HikariCP), `EntityManagerFactory`, `JpaTransactionManager`, `DispatcherServlet`, Jackson `ObjectMapper`, `RequestMappingHandlerMapping`, 테스트의 `MockMvc`
- `WebConfig implements WebMvcConfigurer` = 자동 설정을 **끄지 않고 덧붙이는** 공식 확장점. `@EnableWebMvc`를 붙이면 Boot의 MVC 자동 설정이 통째로 꺼지므로 쓰지 않음(5.6절)

### 3.8 빈 생명주기와 Ondal이 쓰는 훅

- 기동 중 순서
  1. 생성자 호출(= 의존성 주입)
  2. `@PostConstruct` / `InitializingBean` - Ondal 미사용
  3. **모든 싱글톤 생성 완료** → `SmartInitializingSingleton.afterSingletonsInstantiated()` → `AuthorizationMappingValidator`
     - 이 시점인 이유: 컨트롤러 빈이 **전부** 만들어져 `RequestMappingHandlerMapping`에 매핑이 다 등록된 뒤라야 "모든 핸들러"를 검사 가능. 생성자·`@PostConstruct`는 너무 이름
     - 여기서 예외 → 컨텍스트 초기화 실패 → **부팅 실패**. "권한 어노테이션 누락 = 배포 불가"의 구현
  4. 컨텍스트 완료, Tomcat 포트 열림
  5. `CommandLineRunner.run()` → `LocalDataSeeder`. 검증(3)이 실패하면 시더(5)는 돌지 않음
  6. 종료 시 `@PreDestroy` / `DisposableBean` - 미사용

### 3.9 프로필(Profile)

- 같은 코드를 환경별로 다르게 띄우는 장치
- `spring.profiles.active: local`(application.yml) → 아무 지정 없이 실행하면 local
- `@Profile("local")`이 붙은 `LocalDataSeeder` → local에서만 **빈으로 등록**. 테스트(`@ActiveProfiles("test")`)에서는 "실행이 안 되는" 것이 아니라 **존재하지 않음**
- 운영은 `--spring.profiles.active=prod` 또는 환경변수 `SPRING_PROFILES_ACTIVE=prod`로 덮어씀

### 3.10 외부 설정 주입 - `@Value`, 설정 우선순위, `@Qualifier`

- `@Value("${ondal.cors.allowed-origins:http://localhost:5173}") String[] allowedOrigins` (`WebConfig`)
  - `${키:기본값}` - 키가 없으면 콜론 뒤 값
  - `"a,b,c"` 콤마 문자열 → `String[]` 자동 변환
- 설정 출처 우선순위(높음 → 낮음): 커맨드라인 인자 > OS 환경변수 > `application-{profile}.yml` > `application.yml` 프로필 문서 > 공통 문서
  - 운영에서 `ONDAL_CORS_ALLOWED_ORIGINS=https://...pages.dev` 환경변수로 덮으면 됨(대소문자·하이픈에 관대한 relaxed binding)
- `@Qualifier("requestMappingHandlerMapping")` (`AuthorizationMappingValidator`)
  - 같은 타입 빈이 여럿일 때 **이름으로** 고름. springdoc·actuator 같은 라이브러리가 `HandlerMapping` 빈을 추가할 수 있어 "우리 컨트롤러 매핑" 빈을 못 박은 것
  - 면접: 같은 타입 빈이 둘이면? → `NoUniqueBeanDefinitionException`으로 기동 실패. 해결 = `@Qualifier`, `@Primary`, 파라미터 이름 일치, `@Profile`로 하나만 등록

### 3.11 인터페이스에 의존하기 - `AuthService` / `StubAuthService`

- `AuthController`는 **인터페이스** `AuthService`를 주입받음. 구현체 `StubAuthService`가 `@Service`라 컨테이너가 골라 넣음
- 홈페이지 연동 `HomepageAuthService`로 바꿀 때 컨트롤러는 한 글자도 안 바뀜 = DI의 존재 이유(전략 교체, OCP)
- 구현체가 둘 다 빈이면 3.10절의 모호성 문제 → `@Profile`로 환경별 하나만, 또는 `@Primary`

### 3.12 순환 의존

- A → B → A 생성자 주입은 **기동 실패**(Boot 2.6+ 기본 금지)
- Ondal에서 실제로 마주친 지점: `CohortService`가 `EnrollmentService`를 쓰고, `EnrollmentService`도 분반 응답(`CohortResponse`)을 만들어야 함 → 서로 주입하면 순환
- 해결: 응답 조립을 `CohortResponseAssembler`로 **빼서** 둘 다 그것을 주입. 의존 그래프가 `CohortService → EnrollmentService → Assembler`, `CohortService → Assembler`의 DAG가 됨
- 면접 답: 순환은 `@Lazy`로 덮지 말고 **책임을 분리해 제3 컴포넌트로 추출**하는 것이 정석

### 3.13 프록시 - `@Transactional`이 "마법처럼" 동작하는 원리 미리보기

- 스프링은 일부 빈을 **원본 대신 프록시(대리 객체)**로 컨테이너에 등록
  - `@Transactional`이 붙은 `CohortService` → 실제로 주입되는 것은 `CohortService`를 상속한 프록시 클래스(CGLIB)
  - 프록시의 메서드 = "트랜잭션 시작 → 원본 메서드 호출 → 커밋/롤백"
- 이것이 AOP(관점 지향 프로그래밍)의 스프링식 구현. 공통 관심사(트랜잭션·로깅)를 원본 코드 수정 없이 둘러쌈
- 함정(6.5절에서 다시): 프록시를 거쳐야 동작하므로 **같은 클래스 안에서 자기 메서드를 호출하면 적용되지 않음**, `private` 메서드에도 적용되지 않음
- Ondal에서 프록시인 것: `@Transactional` 서비스 3개, Spring Data 리포지토리(인터페이스의 구현체 자체가 프록시), `@PersistenceContext`의 `EntityManager`

---

## 4. 프로젝트 뼈대 - Gradle·스타터·application.yml·docker-compose

### 4.1 `build.gradle` - 플러그인과 JDK

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '4.1.0'
    id 'io.spring.dependency-management' version '1.1.7'
}
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }
```

- `java`: 컴파일·테스트·jar 생성
- `org.springframework.boot`: `bootRun`(실행), `bootJar`(실행 가능 fat jar), `developmentOnly` 구성 제공
- `io.spring.dependency-management`: **Boot BOM**(호환 버전 표)을 적용 → 아래 의존성에 버전을 안 적어도 됨
- 면접: 스타터에 버전이 없는 이유 → Boot 버전(4.1.0)이 정하는 BOM이 Spring Framework·Hibernate·Jackson·Tomcat 등 수백 개의 **서로 호환되는 버전 조합**을 관리. 반대로 `springdoc`은 BOM 밖이라 `3.1.0`을 직접 명시(주석에 이유 기재)
- 툴체인 21: Gradle이 JDK 21로 컴파일·실행(없으면 내려받음). Ondal 코드가 쓰는 Java 17~21 기능: `record`(DTO 전부), `instanceof` 패턴 매칭(`handler instanceof HandlerMethod handlerMethod`), `Stream.toList()`, `List.of`/`Map.of`, `var`(테스트)

### 4.2 의존성 구성(configuration)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:3.1.0'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'
    runtimeOnly 'org.postgresql:postgresql'
    testImplementation 'org.springframework.boot:spring-boot-starter-data-jpa-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-validation-test'
    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
    testImplementation 'org.springframework.boot:spring-boot-testcontainers'
    testImplementation 'org.testcontainers:testcontainers-postgresql'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}
tasks.named('test') { useJUnitPlatform() }
```

| 구성 | 컴파일 시 | 런타임 | 비고 |
|---|---|---|---|
| `implementation` | O | O | 일반 의존성 |
| `runtimeOnly` | X | O | PostgreSQL 드라이버 - 코드는 JDBC 인터페이스(`java.sql.*`)만 쓰고 드라이버 클래스를 직접 참조하지 않음 → 컴파일에 불필요. 드라이버 교체가 코드에 영향 없음을 구조로 보장 |
| `developmentOnly` | O | 로컬만 | `bootJar`에 **미포함** - devtools가 운영 jar에 들어가지 않게 |
| `testImplementation` / `testRuntimeOnly` | 테스트에만 | | |

### 4.3 각 스타터가 끌고 오는 것

- `spring-boot-starter-data-jpa`: Spring Data JPA + Hibernate(JPA 구현체) + `spring-orm`·`spring-tx`(트랜잭션) + `spring-jdbc` + **HikariCP**(커넥션 풀, Boot 기본)
- `spring-boot-starter-validation`: Jakarta Bean Validation API + Hibernate Validator(구현체). `@NotBlank`·`@Size`·`@Valid`의 출처. Hibernate ORM과 이름만 같은 별개 프로젝트
- `spring-boot-starter-webmvc`: Spring MVC + **내장 Tomcat** + Jackson 3(JSON). Boot 4에서 `-starter-web`이 `-starter-webmvc`로 개명
- `springdoc-openapi-starter-webmvc-ui`: 컨트롤러 어노테이션을 읽어 OpenAPI 문서 생성 + Swagger UI(`/swagger-ui/index.html`, `/v3/api-docs`)
- `spring-boot-devtools`: 클래스 변경 시 자동 재시작. 운영 제외
- `org.postgresql:postgresql`: JDBC 드라이버
- 테스트
  - `-*-test` 스타터: JUnit 5 + AssertJ + Mockito + Hamcrest + MockMvc + JsonPath 묶음(Boot 4에서 모듈별로 분리됨)
  - `spring-boot-testcontainers`: `@ServiceConnection`(8.5절)
  - `testcontainers-postgresql`: Testcontainers 2.x 아티팩트 - 1.x의 `postgresql`과 이름·패키지가 다름(주석에 경고)
  - `junit-platform-launcher`: 최신 Gradle이 테스트 실행기를 번들하지 않아 명시
- `useJUnitPlatform()`: JUnit 5(Jupiter)로 테스트 실행

### 4.4 `settings.gradle`

- `rootProject.name = 'ondal'` → 산출물 `ondal-0.0.1-SNAPSHOT.jar`. 단일 모듈 프로젝트

### 4.5 `application.yml` 한 줄씩

- 구조: `---`로 나뉜 **다중 문서**. 첫 문서 = 공통, 둘째 문서 = `spring.config.activate.on-profile: local`일 때만 적용

```yaml
spring:
  profiles:
    active: local
```
- 기본 프로필. 테스트는 `@ActiveProfiles("test")`로 덮어씀

```yaml
  jpa:
    hibernate:
      ddl-auto: update
```
- Hibernate가 엔티티 매핑을 보고 스키마를 어떻게 할지

| 값 | 동작 |
|---|---|
| `none` | 아무것도 안 함 |
| `validate` | 엔티티와 테이블이 맞는지 검사만, 다르면 기동 실패 |
| `update` | 없는 테이블·컬럼 **추가만**. 삭제·타입 변경·제약 변경 없음 |
| `create` | 기동 시 drop 후 생성 |
| `create-drop` | 종료 시에도 drop |

- `update`는 개발 편의용. 운영 금지 이유: 컬럼 삭제·이름 변경을 반영 못 하고 실수가 데이터 손실로 직결 → "운영 전 Flyway 전환"이 알려진 부채로 기록됨([db/schema.md](../db/schema.md) 4절). 면접 정답: 운영은 `validate` + 마이그레이션 도구(Flyway/Liquibase)

```yaml
    open-in-view: false
```
- **OSIV(Open Session In View)**. `true`(Boot 기본, 경고 로그 출력)면 영속성 컨텍스트를 **요청이 끝날 때까지** 열어 둠 → 컨트롤러·직렬화 중에도 지연 로딩 가능, 대신 DB 커넥션을 응답 끝까지 점유
- `false`면 트랜잭션(=서비스 메서드)이 끝나면 닫힘 → 컨트롤러에서 `enrollment.getUser().getName()` 같은 지연 로딩 접근 = `LazyInitializationException`
- 이 한 줄이 "**서비스가 DTO를 완성해서 돌려준다**" 규약의 근거(6.4절, 12.3절). 주석 그대로: 켜 두면 N+1을 눈치 못 챈 채 굴러감

```yaml
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: debug
```
- 실행 SQL을 줄바꿈 정렬해 로그 출력. 바인딩 값까지 보려면 `org.hibernate.orm.jdbc.bind: trace` 추가

```yaml
  jackson:
    time-zone: UTC
```
- `Instant`가 `"2026-08-23T04:00:00Z"` 같은 ISO-8601 문자열로 나감. 저장 UTC·표시 KST는 프런트 책임(docs 결정)

```yaml
server:
  servlet:
    session:
      timeout: 12h
      cookie:
        same-site: lax
        http-only: true
```
- `timeout`: 마지막 접근 후 12시간 비활성이면 세션 만료. 세션은 **Tomcat 메모리**에 존재 → 서버 재시작 시 전원 로그아웃, 인스턴스 2대면 공유 불가(정석은 Spring Session + Redis, P1은 감수 - [decisions/5](../decisions/5-세션-인증-채택-spring-security-보류.md))
- `same-site: lax`: 다른 사이트에서 시작된 요청 중 **최상위 GET 이동에만** 쿠키가 실리고 `<form method=POST>`·iframe·cross-site fetch에는 안 실림 → CSRF 1차 방어(7.4절)
- `http-only`: JS `document.cookie`로 접근 불가 → XSS로 세션 탈취 완화
- 운영(HTTPS)에서는 `secure: true` 추가 필요

```yaml
---
spring:
  config:
    activate:
      on-profile: local
  datasource:
    url: jdbc:postgresql://localhost:5432/ondal
    username: ondal
    password: ondal-local-password
```
- local 전용 데이터소스. docker-compose 값과 1:1
- **test 프로필은 datasource 설정 자체가 없음** - Testcontainers `@ServiceConnection`이 컨테이너를 띄운 뒤 url/username/password를 런타임에 주입(8.5절)
- 진짜 비밀은 `.gitignore`된 `application-local.yml`에 - 프로필별 파일이 같은 프로필의 yml 문서보다 우선(3.10절)

### 4.6 `docker-compose.yml`

- `postgres:16` 컨테이너 1개. `POSTGRES_DB/USER/PASSWORD` 환경변수로 최초 기동 시 DB·계정 생성
- `5432:5432` 포트 포워딩 → `localhost:5432`로 접속. 로컬에 다른 PostgreSQL이 떠 있으면 충돌(README 트러블슈팅)
- named volume `ondal-db-data` → `docker compose down` 후에도 데이터 유지. 초기화는 `down -v`

### 4.7 `OndalApplication` - main 하나로 웹서버가 뜨는 이유와 실행 순서

- 전통 방식: WAR를 빌드해 외부 Tomcat에 배포. Boot: **Tomcat을 라이브러리로 jar 안에 넣고** `main`에서 기동 → `java -jar ondal.jar`. 컨테이너·클라우드 배포에 맞는 모델
- README 실행 순서 = 의존 순서: `docker compose up -d`(DB) → `./gradlew bootRun`(local 프로필 → 시더 → 8080) → Swagger UI
- curl 흐름 `-c`(응답의 Set-Cookie 저장) → `-b`(요청에 쿠키 실어 보냄) = 브라우저의 세션 쿠키 동작을 손으로 흉내 낸 것

---

## 5. 스프링 MVC 사용법 - 컨트롤러 어노테이션·Jackson·WebConfig

### 5.1 컨트롤러 어노테이션 한 표

| 어노테이션 | 위치 | 의미 | Ondal 예 |
|---|---|---|---|
| `@RestController` | 클래스 | 빈 + 모든 메서드 반환값을 본문(JSON)으로 | 컨트롤러 4개 |
| `@RequestMapping("/api/cohorts")` | 클래스 | 공통 경로 prefix | `CohortController`, `AuthController`. `EnrollmentController`는 prefix가 제각각이라 메서드에 전체 경로 |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` | 메서드 | 메서드 + 경로 매핑 | `@GetMapping("/{cohortId}")` |
| `@PathVariable` | 파라미터 | 경로 변수 바인딩(이름 일치) | `@PathVariable Long cohortId` |
| `@RequestParam` | 파라미터 | 쿼리 파라미터. `defaultValue`, `required` | `@RequestParam(defaultValue = "ACTIVE") CohortStatus status` |
| `@RequestBody` | 파라미터 | 본문 JSON → 객체(역직렬화) | `@RequestBody @Valid CohortCreateRequest request` |
| `@Valid` | 파라미터 | Bean Validation 실행 | 위와 같이 |
| `@ResponseStatus(HttpStatus.NO_CONTENT)` | 메서드 | 고정 상태코드 | `void removeStudent(...)` → 204 |
| `ResponseEntity<T>` | 반환 타입 | 상태코드·헤더를 직접 지정 | `ResponseEntity.created(uri).body(dto)` → 201 + Location |
| `@Tag`, `@Operation`, `@Schema` | 클래스/메서드/필드 | springdoc 문서용 - 동작에는 영향 없음 | 전 컨트롤러·DTO |

- Ondal 반환 규약(design.md 4절): 200은 DTO 직접 반환, 201은 `ResponseEntity.created`, 204는 `void` + `@ResponseStatus`. 그 외 `ResponseEntity` 금지 - 컨트롤러 모양을 균일하게

### 5.2 파라미터 바인딩과 타입 변환

- 경로 변수·쿼리 파라미터는 항상 **문자열**로 도착 → 파라미터 타입으로 변환(ConversionService)
  - `"3"` → `Long`, `"ARCHIVED"` → `CohortStatus`(enum 이름 일치)
  - 변환 실패(`/api/cohorts/abc`, `?status=FOO`) → `MethodArgumentTypeMismatchException` → 17장 핸들러가 400 `INVALID_INPUT`
- 예외: `@CohortRole` 경로의 `{cohortId}`는 인터셉터가 바인딩보다 **먼저** 문자열을 읽어 직접 400 처리(2.4절, 13.4절)

### 5.3 요청 본문 → DTO → 검증

- `@RequestBody`: `Content-Type: application/json` 본문을 Jackson이 record 생성자로 역직렬화
  - JSON이 깨졌거나 타입이 안 맞음 → `HttpMessageNotReadableException` → 400
  - JSON에 없는 필드는 `null`(record 컴포넌트가 `null`로 들어옴) → 그래서 `CohortCreateRequest.operatorLoginIdsOrEmpty()` 같은 방어가 있음
- `@Valid`: 역직렬화 **후** Bean Validation 실행
  - 검증 어노테이션은 **요청 DTO 필드에만** 단다(Ondal 규약). 컨트롤러 파라미터에 직접 `@NotBlank`를 달지 않음
  - 실패 → `MethodArgumentNotValidException` → 400, 첫 번째 필드 오류를 `"필드명: 메시지"`로
- 자주 쓰는 제약: `@NotBlank`(null·빈 문자열·공백만 불가), `@NotEmpty`(컬렉션 비어 있으면 불가), `@Size(max=)`, 컬렉션 원소 검증 `List<@NotBlank @Size(max = 50) String>`
- Bean Validation을 거치지 않는 입력(경로 변수 `{loginId}`)은 서비스가 한 번 더 막음 - `UserService.findOrCreateMember`의 길이 검사(14.4절)

### 5.4 응답 - Jackson 직렬화 규칙

| 자바 | JSON | Ondal 예 |
|---|---|---|
| `record` | 컴포넌트 이름 = 필드 | `CohortResponse` → `{"id":..,"name":..}` |
| `enum` | 상수 이름 문자열 | `"status":"ACTIVE"`, `"myRole":"STUDENT"` |
| `enum` + `@JsonValue` 메서드 | 그 메서드 반환값 | `RoleTitle` → `"title":"교육운영진"` |
| `Instant` | ISO-8601 UTC 문자열 | `"createdAt":"2026-08-18T02:11:03.412Z"` |
| `null` 필드 | `null`로 포함 | `"studentCount":null` (학생에게) |
| `List` | 배열 | `"operators":[...]` |
| `Map` | 객체 | `HealthController`의 `Map.of("status","UP")` → `{"status":"UP"}` |
| `void` + 204 | 본문 없음 | DELETE 응답 |

- Jackson 3: 패키지가 `tools.jackson.databind`(테스트의 `ObjectMapper` import), 어노테이션은 `com.fasterxml.jackson.annotation` 유지(`RoleTitle`의 `@JsonValue` import 주석)

### 5.5 `HealthController` - 가장 작은 컨트롤러

```java
@RestController
public class HealthController {
    @GetMapping("/api/health")
    public Map<String, String> health() { return Map.of("status", "UP"); }
}
```
- 배포·모니터링용 생존 확인. `AuthPaths.PUBLIC`에 있어 로그인 면제 + 권한 어노테이션 면제(기동 검증기가 공개 경로는 건너뜀)
- 이 세 줄에 5.1~5.4절이 전부 들어 있음: 빈 등록, 경로 매핑, 반환값 → JSON

### 5.6 `WebMvcConfigurer`와 `WebConfig` - 세 가지 등록

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    addCorsMappings(...)        // (1) CORS
    addInterceptors(...)        // (2) 인터셉터 2개, 순서·경로·제외
    addArgumentResolvers(...)   // (3) @LoginUser 리졸버
}
```

- (1) CORS
  - `registry.addMapping("/api/**").allowedOrigins(allowedOrigins).allowedMethods("*").allowedHeaders("*").allowCredentials(true)`
  - `allowCredentials(true)`가 핵심: 세션 **쿠키**가 오가려면 자격증명 허용 + **와일드카드가 아닌 명시적 origin** 필요(브라우저 규칙). 그래서 origin을 설정값으로 받음
  - 운영에서 리버스 프록시로 같은 도메인 아래 두면 CORS 자체가 불필요
  - 원리·사전 요청: 7.6절
- (2) 인터셉터
  - 등록 순서 = 실행 순서: `AuthInterceptor`(로그인?) → `AuthorizationInterceptor`(권한?)
  - 같은 경로 패턴 `/api/**`, 같은 제외 목록 `AuthPaths.PUBLIC`. 제외 목록은 **한 곳에서만** 관리 → 새 API를 추가하면 아무 설정 없이 자동으로 로그인 필수(default-deny)
- (3) 아규먼트 리졸버: `LoginUserArgumentResolver`를 등록해야 `@LoginUser User me`가 채워짐
- `@EnableWebMvc`를 붙이지 않는 이유: Boot의 MVC 자동 설정(Jackson 컨버터, 정적 자원 등)이 통째로 꺼짐. `WebMvcConfigurer` 구현만으로 "덧붙이기"가 가능

### 5.7 인터셉터·리졸버 인터페이스

- `HandlerInterceptor`
  - `preHandle(request, response, handler)`: 컨트롤러 전. `true` 반환 → 계속, `false` 또는 예외 → 중단. `handler`는 보통 `HandlerMethod`(컨트롤러 메서드 정보)
  - `postHandle`, `afterCompletion`: 후처리. Ondal 미사용
- `HandlerMethodArgumentResolver`
  - `supportsParameter(parameter)`: 이 파라미터를 내가 채울지 판단 - `@LoginUser`가 붙고 타입이 `User`인지
  - `resolveArgument(...)`: 값을 만들어 반환 - 세션 id로 DB에서 `User` 조회
  - 등록된 리졸버들을 순서대로 물어 첫 번째 "지원" 리졸버가 채움

### 5.8 커스텀 어노테이션 만들기 - `@LoginUser`, `@LoginOnly`류의 정체

```java
@Target(ElementType.PARAMETER)        // 붙일 수 있는 위치 - 파라미터
@Retention(RetentionPolicy.RUNTIME)   // 런타임까지 유지 - 리플렉션으로 읽어야 하므로 필수
public @interface LoginUser { }
```

- 어노테이션 = 코드에 붙이는 **메타데이터**. 그 자체는 아무 동작도 없음. 누군가(리졸버·인터셉터·검증기)가 리플렉션으로 읽어 동작을 결정
- `@Retention(RUNTIME)`이 없으면 컴파일 후 사라져 읽을 수 없음
- `@Target`으로 위치 제한: `@LoginUser`는 파라미터, `@LoginOnly`/`@AdminOnly`/`@CohortRole`은 메서드·클래스
- 속성: `@CohortRole`의 `EnrollmentRole value();` → `@CohortRole(EnrollmentRole.OPERATOR)`처럼 값 전달
- 읽는 쪽: `parameter.hasParameterAnnotation(LoginUser.class)`, `AnnotatedElementUtils.findMergedAnnotation(element, type)`(메타 어노테이션·상속까지 찾는 스프링 유틸 - 13.2절)

---

## 6. JPA·Hibernate·Spring Data JPA - 엔티티·영속성 컨텍스트·트랜잭션·리포지토리

### 6.1 ORM과 세 개의 층

- 문제: 자바는 객체(참조·상속·컬렉션), DB는 테이블(행·FK·조인) → 패러다임 불일치. JDBC로 SQL을 직접 쓰면 객체와 행 사이의 변환 코드가 반복됨
- ORM(Object-Relational Mapping): 객체와 테이블의 대응을 선언하면 SQL 생성·변환을 라이브러리가 대행
- 세 층의 역할

| 층 | 정체 | Ondal에서 보이는 것 |
|---|---|---|
| JPA | 자바 표준 **인터페이스·어노테이션 명세** | `jakarta.persistence.*` - `@Entity`, `@Id`, `EntityManager` |
| Hibernate | JPA의 **구현체** - 실제 SQL 생성·캐시·프록시 | SQL 로그, `ddl-auto`, `LazyInitializationException` |
| Spring Data JPA | 리포지토리 **추상화** - 인터페이스만 쓰면 구현 생성 | `JpaRepository`, 쿼리 메서드, `@Query` |

- 면접: "JPA와 Hibernate의 차이" → 명세 vs 구현. "Spring Data JPA는" → JPA 위의 편의 계층, 리포지토리 구현을 런타임에 만들어 줌

### 6.2 엔티티 매핑 어노테이션 - `User`, `Cohort`, `Enrollment`에 쓰인 것 전부

| 어노테이션 | 뜻 | Ondal 사용 |
|---|---|---|
| `@Entity` | 이 클래스 = 테이블 1개 | `User`, `Cohort`, `Enrollment` |
| `@Table(name=, uniqueConstraints=)` | 테이블 이름·제약 | `users`(`user`는 PostgreSQL 예약어), `cohorts`, `enrollments` + `uk_enrollment_cohort_user(cohort_id, user_id)` |
| `@Id` | 기본키 | 전부 `Long id` |
| `@GeneratedValue(strategy = IDENTITY)` | DB가 채번(auto increment / identity 컬럼) | 전부 |
| `@Column(nullable, unique, length, updatable, columnDefinition)` | 컬럼 제약 | `loginId` unique·50, `createdAt` `updatable=false`, `description` `columnDefinition="text"` |
| `@Enumerated(EnumType.STRING)` | enum을 **이름 문자열**로 저장 | `globalRole`, `status`, `role` 전부 STRING |
| `@ManyToOne(fetch = LAZY, optional = false)` | N:1 연관. 지연 로딩, not null | `Enrollment.cohort`, `Enrollment.user` |
| `@JoinColumn(name = "cohort_id", nullable = false)` | FK 컬럼 이름 | 위와 같이 |

- `IDENTITY` 전략의 의미: id를 DB가 INSERT 시점에 만들므로 **`persist` 즉시 INSERT SQL 실행**(쓰기 지연 불가). 그래서 `cohortRepository.save(Cohort.create(...))`가 반환한 객체에 id가 바로 들어 있고, `CohortService.create`가 그 id로 `enrollmentService.assign(cohort.getId(), ...)`를 이어서 호출 가능
- `EnumType.ORDINAL`(순서 번호 저장)을 금지하는 이유: enum 상수 순서를 바꾸거나 중간에 추가하면 기존 데이터의 의미가 통째로 깨짐
- `optional = false`: Hibernate가 "항상 있다"를 알아 inner join을 쓰고 프록시 처리도 단순해짐
- 시각은 `Instant`(UTC) → PostgreSQL `timestamptz`. 저장은 UTC, 표시는 프런트가 KST로
- 기본 생성자가 `protected`인 이유
  - JPA 스펙: 엔티티는 인자 없는 생성자 필수 - Hibernate가 리플렉션으로 객체를 만들고, 지연 로딩 프록시가 **이 클래스를 상속**해야 함(`private`이면 상속 불가)
  - `public`으로 열어 두면 `new Cohort()` 같은 불완전 객체 생성이 가능 → `protected`로 "JPA만 쓰라"고 표시
- Ondal 엔티티 규약(이후 도메인 동일): `protected` 기본 생성자 + `private` 전체 생성자(`createdAt = Instant.now()`) + 정적 팩토리 `create(...)` + **setter 없음**(상태 변경은 의미 있는 도메인 메서드로만) + enum STRING + `Instant`. Lombok 미사용

### 6.3 EntityManager와 영속성 컨텍스트 (핵심)

- `EntityManager`: JPA의 중심 API. 엔티티를 저장(`persist`)·조회(`find`)·삭제(`remove`)·쿼리. Spring Data 리포지토리가 내부에서 이것을 사용
- **영속성 컨텍스트**: EntityManager가 관리하는 "엔티티 보관소". 트랜잭션 하나당 하나가 열리고 닫힘(스프링 기본 설정). 다음 네 기능이 여기서 나옴
  1. **1차 캐시**: 같은 컨텍스트 안에서 같은 id를 두 번 조회하면 두 번째는 DB를 안 침. 같은 id면 **같은 자바 객체**(동일성 보장)
  2. **쓰기 지연**: INSERT/UPDATE/DELETE SQL을 모아 두었다가 **플러시** 시점에 한꺼번에 전송 (단, IDENTITY의 INSERT는 즉시)
  3. **더티 체킹(변경 감지)**: 조회 시점의 스냅샷을 보관 → 플러시 때 현재 값과 비교 → 달라진 엔티티에 대해 UPDATE SQL 자동 생성. `CohortService.update`에 `save()` 호출이 **없는** 이유
  4. **지연 로딩**: 연관 엔티티를 필요할 때 DB에서 읽음(6.4절)
- 플러시(flush) 시점: 트랜잭션 커밋 직전 / JPQL 쿼리 실행 직전(쿼리 결과 일관성) / `flush()` 명시 호출. 플러시 = SQL 전송이지 커밋이 아님
- 엔티티의 네 상태

| 상태 | 뜻 | Ondal 예 |
|---|---|---|
| 비영속(new) | 컨텍스트와 무관한 새 객체 | `Cohort.create(...)` 직후 |
| 영속(managed) | 컨텍스트가 관리 - 더티 체킹 대상 | `save()` 후, `findById()` 결과 |
| 준영속(detached) | 관리되다 컨텍스트가 닫힘 | 트랜잭션 종료 후 컨트롤러로 나간 엔티티 - 지연 로딩 불가 |
| 삭제(removed) | 삭제 예약 | `enrollmentRepository.delete(enrollment)` 후 플러시 전 |

- 트랜잭션이 끝나면 영속성 컨텍스트가 닫힘 = OSIV `false`의 의미(4.5절). 컨트롤러가 받는 엔티티는 **준영속** → 지연 로딩 불가 → 서비스가 DTO를 완성해 반환

### 6.4 연관관계·지연 로딩·프록시·N+1

- `@ManyToOne(fetch = LAZY)`: `Enrollment`를 조회해도 `cohort`·`user` 자리에는 **프록시**(Hibernate가 만든 서브클래스 객체)만 들어 있음. 실제 필드에 접근하는 순간 SELECT 실행(초기화)
  - 예외: 프록시의 `getId()`는 id를 이미 알고 있어 DB를 치지 않음 - `CohortResponseAssembler`의 `e.getCohort().getId()` 주석이 그 뜻
  - 초기화 시점에 영속성 컨텍스트가 닫혀 있으면 `LazyInitializationException`
- `EAGER`(즉시 로딩)를 안 쓰는 이유: 매 조회마다 연관을 무조건 같이 읽어 제어가 불가능하고 N+1의 온상. **기본은 LAZY, 필요한 조회에서만 fetch join**
- 단방향만 쓰는 이유: `Cohort` 안에 `List<Enrollment>` 같은 컬렉션을 두지 않음(11.4절). 양방향은 조회 한 번에 컬렉션이 딸려 오거나 순환 참조·직렬화 문제를 처음부터 유발. 소속이 필요하면 `EnrollmentRepository`로 별도 조회
- **N+1 문제**: 분반 N개를 읽고(쿼리 1) 분반마다 소속을 지연 로딩하면 쿼리 N번 추가 → 총 N+1
  - 해법: (a) **fetch join**(`join fetch e.user`) - 한 쿼리로 연관까지 (b) **IN 조회** - id 목록으로 한 번에 (c) `@BatchSize` (d) DTO 프로젝션
  - Ondal 적용: `findAllByCohortIdInWithUser(cohortIds)` - 분반 N개의 소속을 **쿼리 1번**으로 읽고 자바에서 `groupingBy`(12.4절)

### 6.5 트랜잭션과 `@Transactional` (핵심)

- 트랜잭션: "전부 성공 아니면 전부 취소"의 작업 단위(ACID 중 원자성). DB 커넥션 위에서 `BEGIN` ~ `COMMIT`/`ROLLBACK`
- 스프링의 `@Transactional` 동작(3.13절 프록시)
  1. 컨테이너가 서비스를 **프록시**로 감싸 주입
  2. 프록시 메서드 진입 → `TransactionInterceptor` → `JpaTransactionManager`가 커넥션 확보·`autocommit=false`·영속성 컨텍스트 시작, 현재 **스레드**에 묶음(ThreadLocal)
  3. 원본 메서드 실행
  4. 정상 종료 → 플러시 → 커밋 / 예외 → 롤백
- 전파(propagation) 기본값 `REQUIRED`: 이미 트랜잭션이 있으면 **합류**, 없으면 새로 시작
  - `CohortService.create`(트랜잭션 A) 안에서 `enrollmentService.assign(...)`을 호출 → 같은 트랜잭션 A에 합류 → 운영진 지정이 409로 실패하면 **분반 INSERT도 함께 롤백**(주석 그대로)
- 롤백 규칙: **`RuntimeException`·`Error` → 롤백, checked 예외 → 커밋**(기본). Ondal 예외 6종이 전부 `RuntimeException`을 상속한 이유
- `readOnly = true`의 효과: Hibernate 플러시 모드 MANUAL(더티 체킹 스킵 → 성능), 드라이버에 읽기 전용 힌트, 읽기 복제본 라우팅 가능. Ondal 규약: 클래스에 `@Transactional`, 조회 메서드만 `readOnly = true`로 덮어씀(메서드 설정이 클래스 설정보다 우선)
- 주의점(면접 단골)
  - `public` 메서드에만 적용(프록시 기반)
  - **자기 호출(self-invocation)**: 같은 클래스의 메서드가 `this.otherMethod()`를 부르면 프록시를 거치지 않아 `@Transactional` 설정이 무시됨. Ondal에서 `assignStudents → assign`, `create → requireCohort`는 이미 같은 트랜잭션 안이라 문제 없음
  - 트랜잭션 범위 = 영속성 컨텍스트 범위(기본) → 서비스 밖은 준영속
- `CohortResponseAssembler`가 자체 `@Transactional`이 없는 이유: 항상 서비스 트랜잭션 **안에서** 호출된다는 전제(합류). 밖에서 단독 호출하면 LAZY가 터짐 - 주석에 명시

### 6.6 Spring Data JPA 리포지토리

```java
public interface CohortRepository extends JpaRepository<Cohort, Long> {
    List<Cohort> findAllByStatusOrderByCreatedAtDesc(CohortStatus status);
}
```

- 인터페이스만 선언 → 기동 시 Spring Data가 **구현체(프록시)를 만들어 빈으로 등록**. `@Repository` 불필요
- `JpaRepository<T, ID>`가 주는 것: `save`, `findById`(Optional), `findAll`, `delete`, `count`, `existsById` 등
  - `save()`: id가 `null`이면 `persist`(INSERT), 아니면 `merge`. 영속 상태 엔티티는 `save` 없이도 더티 체킹으로 반영됨
  - `findById` → `Optional<T>`: 없으면 빈 Optional → Ondal은 `orElseThrow(() -> new NotFoundException(...))`로 404
- **쿼리 메서드 파생**: 메서드 이름을 파싱해 JPQL 생성
  - 형식: `findBy` + 속성 + 조건(`And`/`Or`/`In`/`Between`...) + `OrderBy` + 속성 + `Asc`/`Desc`
  - 중첩 속성: `findByCohortIdAndUserId` → `e.cohort.id = ? and e.user.id = ?` - 연관 엔티티의 id 비교는 **조인 없이 FK 컬럼으로** 처리(권한 판정용으로 가볍다는 주석의 근거)
  - 반환 타입: `Optional<T>`(0~1건), `List<T>`, `long count...`, `boolean exists...`
  - 파싱 실패 = **기동 실패**. Ondal 규약의 근거: fetch join 조회 이름 `...WithUser`를 `@Query` 없이 쓰면 Spring Data가 `With`를 속성으로 해석하려다 기동 실패 → `WithXxx`에는 반드시 `@Query`
- **`@Query` JPQL**: 테이블이 아니라 **엔티티·필드 기준**의 객체지향 쿼리. `:userId` 이름 바인딩 + `@Param`
  - `select e from Enrollment e join fetch e.cohort where e.user.id = :userId` - `join fetch`가 지연 로딩 연관을 한 번에 채움
  - `order by e.role asc` - enum은 STRING 저장이라 **알파벳순**(`OPERATOR` < `STUDENT`) → "운영진 먼저"가 우연히 맞음. 의도한 순서인지 주석으로 밝히는 규약
- 네이티브 쿼리: `entityManager.createNativeQuery("TRUNCATE TABLE ...")` - 테스트의 `DatabaseCleaner`. 메타모델(`getMetamodel().getEntities()`)로 엔티티 목록을 얻어 테이블 이름을 만듦
- 리포지토리 계층에 두지 않는 것: 정렬·필터의 복합 규칙(`findMyCohorts`의 ACTIVE 우선 정렬은 서비스에서 `Comparator`로) - 쿼리가 복잡해지는 것보다 자바에서 읽기 쉬운 쪽 선택

### 6.7 스키마 관리 - `ddl-auto`에서 Flyway로

- 현재: `ddl-auto: update`가 엔티티를 보고 테이블을 맞춤(4.5절). 제약 이름은 `uk_enrollment_cohort_user`만 코드에 명시, 나머지는 Hibernate 임의 이름
- 운영 전: Flyway 마이그레이션(버전 붙은 SQL 파일을 순서대로 적용, 이력 테이블로 추적) + `ddl-auto: validate`. 이유·계획: [db/schema.md](../db/schema.md) 4절
- 면접: "운영에서 스키마 변경은 어떻게?" → 마이그레이션 도구로 SQL을 버전 관리, 앱은 검증만

---

## 7. 인증·인가의 기초 - 세션·쿠키·보안 용어·Spring Security를 보류한 이유

### 7.1 인증과 인가, 그리고 401·403·404

| 용어 | 질문 | 실패 시 | Ondal 구현 위치 |
|---|---|---|---|
| 인증(Authentication) | **누구냐** | 401 `UNAUTHENTICATED` | `AuthInterceptor`, `LoginUserArgumentResolver` |
| 인가(Authorization) | **해도 되냐** | 403 `FORBIDDEN` | `AuthorizationInterceptor` + `CohortAuthorizer` |
| 존재 비노출 | 남의 것인지조차 알려 주지 않음 | 404 `NOT_FOUND` | 서비스의 스코프 조회(`findByIdAndCohortId`) |

- 401의 이름이 "Unauthorized"인데 뜻은 "미인증" - 역사적 오명. 면접에서 설명할 수 있어야 함
- Ondal의 선택: 비소속자가 없는 분반 id를 찔러도 **403**(권한 단계에서 먼저 막힘), ADMIN에게만 **404**. 하위 자원(다른 반의 과제 id 등)은 **404**로 존재 자체를 숨김(design.md 결정 11)

### 7.2 세션 방식의 동작

```
브라우저                                   서버(Tomcat 메모리)
  │ POST /api/auth/login {loginId}            │
  │ ─────────────────────────────────────────▶ │ 세션 저장소에 항목 생성 { id: 3F2A..., LOGIN_USER_ID: 1 }
  │ ◀───── 200 + Set-Cookie: JSESSIONID=3F2A ─ │
  │                                            │
  │ GET /api/auth/me  Cookie: JSESSIONID=3F2A  │
  │ ─────────────────────────────────────────▶ │ 쿠키의 id로 세션 조회 → LOGIN_USER_ID=1 → DB에서 User 1 조회
  │ ◀───────────── 200 {id:1,...} ──────────── │
```

- 서버 API
  - `request.getSession(false)`: 있으면 반환, 없으면 `null` - **새로 만들지 않음**. 검사용(인터셉터·리졸버·로그아웃)
  - `request.getSession(true)` 또는 `getSession()`: 없으면 생성 - 로그인 성공 시에만
  - `session.setAttribute(key, value)` / `getAttribute(key)` / `invalidate()`
- 세션에 **User 엔티티가 아니라 id만** 저장(`SessionConst.LOGIN_USER_ID`)하는 이유: 엔티티를 넣으면 이후 DB에서 이름·역할이 바뀌어도 세션 속 낡은 사본을 계속 믿게 됨. id만 두고 요청마다 DB에서 최신 상태를 읽음
- 만료: `timeout` 동안 접근이 없으면 서버가 세션을 버림 → 다음 요청은 401
- 세션 저장소가 **서버 메모리**: 재시작 시 전원 로그아웃, 서버 2대면 공유 불가 → 규모가 커지면 Spring Session(Redis). P1은 감수

### 7.3 세션 vs 토큰(JWT) - 비교와 Ondal의 선택

| | 세션 | 토큰(JWT) |
|---|---|---|
| 상태 | 서버가 기억(stateful) | 서버 무기억(stateless), 서명만 검증 |
| 전달 | 쿠키(브라우저 자동 첨부) | 보통 `Authorization: Bearer` 헤더(JS가 직접 첨부) |
| 즉시 무효화(강제 로그아웃) | 쉬움 - 서버에서 지움 | 어려움 - 만료까지 유효, 블랙리스트 필요 |
| 서버 확장 | 세션 공유 필요 | 자유로움 |
| 구현 복잡도 | 낮음 | 만료·갱신(refresh)·저장 위치(XSS) 고민 |
| Ondal(P1) | **채택** | 보류 |

- 채택 근거([decisions/5](../decisions/5-세션-인증-채택-spring-security-보류.md)): 단일 서버, 즉시 무효화, 토큰 갱신 로직 불필요, 개발 중 `localhost` 포트가 달라도 same-site라 쿠키 문제 없음
- 재검토 조건: 수평 확장, 모바일 네이티브 클라이언트, 홈페이지 SSO가 JWT로 확정

### 7.4 쿠키 속성

| 속성 | 뜻 | Ondal |
|---|---|---|
| `Path`, `Domain` | 어느 경로·호스트에 보낼지 | 기본(`/`, 현재 호스트) |
| `Expires` / `Max-Age` | 수명. 없으면 **세션 쿠키**(브라우저 종료 시 삭제) | 세션 쿠키. 서버 측 만료는 별도 `timeout` |
| `HttpOnly` | JS에서 읽기 불가 | `true` - XSS로 세션 탈취 완화 |
| `Secure` | HTTPS에서만 전송 | 운영에서 켜야 함 |
| `SameSite` | 다른 사이트에서 시작된 요청에 실을지 - `Strict`/`Lax`/`None` | `Lax` - 외부 사이트의 `<form>` POST·iframe·cross-site fetch에 안 실림 |

- `SameSite=Lax` 세부: 다른 사이트에서 링크를 눌러 **최상위 GET 이동**할 때만 실림. `Strict`는 그마저도 안 실려 외부 링크로 들어오면 로그아웃 상태, `None`은 `Secure` 필수
- "같은 사이트"의 기준은 **등록 도메인**(포트 무관). `localhost:5173`과 `localhost:8080`은 same-site → 개발 중 문제 없음. 그러나 **출처(origin)**는 포트까지 보므로 CORS는 필요(7.6절)

### 7.5 대표 공격과 Ondal의 방어

| 공격 | 내용 | Ondal 방어 |
|---|---|---|
| XSS | 악성 스크립트가 페이지에서 실행 | `HttpOnly` 쿠키(세션 id 탈취 방지). 출력 이스케이프는 FE(React 기본) |
| CSRF | 로그인된 브라우저를 이용해 다른 사이트가 몰래 요청 | `SameSite=Lax`(1차). CSRF 토큰은 P1 미도입(design.md 결정 15) |
| 세션 고정(session fixation) | 공격자가 미리 정한 세션 id로 피해자를 로그인시킴 | 로그인 성공 시 **기존 세션 파기 후 새 세션 발급**(`AuthController.login`) |
| IDOR(안전하지 않은 직접 객체 참조) | URL의 id를 바꿔 남의 자원 접근 | 분반 권한 인터셉터 + 하위 자원은 `findByIdAndCohortId` 스코프 조회 → 404 |
| 정보 노출·열거 | 에러 메시지·응답 필드로 내부 정보 누출 | 401/403 메시지 고정, 500 원인은 로그에만, 학생 응답에서 타인 `loginId`·`globalRole`·인원수 제거 |
| 권한 누락 | 새 API에 권한 검사를 잊음 | default-deny 인터셉터 + 기동 시 검증(fail-closed) |

### 7.6 CORS 상세

- 동일 출처 정책: 브라우저는 다른 출처의 응답을 스크립트가 읽지 못하게 차단(요청 자체는 나갈 수 있음)
- 서버가 응답 헤더로 허용을 알려 주면 브라우저가 해제 = CORS
  - `Access-Control-Allow-Origin: http://localhost:5173` (자격증명을 쓰면 `*` 금지, 정확한 출처)
  - `Access-Control-Allow-Credentials: true` (쿠키를 주고받으려면 필수)
  - 클라이언트도 `fetch(url, { credentials: "include" })`로 쿠키 전송을 명시해야 함
- **사전 요청(preflight)**: `Content-Type: application/json`인 POST, PUT/DELETE 등 "단순 요청"이 아닌 경우 브라우저가 먼저 `OPTIONS` + `Origin` + `Access-Control-Request-Method` 헤더로 허용 여부를 물음 → 서버가 허용 헤더로 응답 → 본 요청 전송
  - 사전 요청에는 쿠키가 없음 → 인증 검사를 하면 안 됨 → `AuthInterceptor`·`AuthorizationInterceptor` 첫 줄의 `CorsUtils.isPreFlightRequest(request)` 통과 처리
- 스프링 구현 위치: `WebMvcConfigurer.addCorsMappings` → `HandlerMapping`이 CORS 설정을 보고 사전 요청은 `PreFlightHandler`로, 일반 요청은 `CorsInterceptor`를 체인 앞에 붙여 응답 헤더 추가(2.4절)
- CORS는 **브라우저에만** 적용. curl·다른 서버 간 호출은 무관 → CORS는 CSRF 방어 수단이 아님(면접 함정)
- 운영에서 FE와 BE를 같은 도메인 아래(리버스 프록시) 두면 출처가 같아져 CORS 불필요

### 7.7 Spring Security 개요와 P1에서 보류한 이유

- Spring Security: 서블릿 **필터 체인**으로 인증·인가를 처리하는 표준 프레임워크. `SecurityContextHolder`에 인증 객체(`Authentication`) 보관, `UserDetails`·`GrantedAuthority`(ROLE_XXX)의 전역 역할 모델, 로그인 폼·OAuth2·CSRF 토큰 등 내장
- Ondal이 P1에서 쓰지 않는 이유([decisions/5](../decisions/5-세션-인증-채택-spring-security-보류.md))
  1. 학습 부하: 필터 체인·인증 객체 모델이 주니어에게 "마법 상자". 코드를 읽고 복제할 수 있어야 함
  2. 권한 모델 불일치: Ondal 권한의 핵심 = "**분반 범위** 역할"(`Enrollment.role`). 시큐리티의 전역 Role로 표현 불가 → 커스텀 판정 컴포넌트가 어차피 필요 → 밑 프레임워크는 단순할수록 유리
  3. 세션의 이점(7.3절)
- 대신 직접 구현한 것 = `HttpSession` + `HandlerInterceptor`(default-deny) + `@LoginUser` 리졸버 + 어노테이션 3종. 2부 13장·15장
- 면접: "시큐리티 없이 인증을 구현했다"고만 하면 약함 → 위 근거 세 가지 + 재검토 조건까지 말할 수 있어야 함

### 7.8 Ondal 보안 설계의 세 원칙

- **default-deny(기본 거부)**: `/api/**`는 기본 잠금. 공개 경로는 `AuthPaths.PUBLIC` 한 곳에만. 새 API는 아무 설정 없이 로그인 필수
- **fail-closed(실패 시 닫힘)**: 권한 어노테이션을 잊으면 "열림"이 아니라 **부팅 실패**(기동 검증) 또는 **500**(인터셉터). 판단 불가 = 거부
- **단일 출처(single source of truth)**: 공개 경로(`AuthPaths`), 권한 규칙(`CohortAuthorizer`), 유효 어노테이션 선택 규칙(`AuthorizationAnnotations`), 직책 명칭(`RoleTitle`) - 각각 한 곳. 두 군데에 복제되어 어긋나는 사고를 구조로 차단

---

## 8. 테스트 기초 - JUnit 5·MockMvc·Testcontainers

### 8.1 왜, 무엇을 테스트하나

- 목적: 회귀 방지(고치다 다른 곳이 깨지는 것), 규약 집행(역할 × 엔드포인트 → 상태코드가 문서가 아니라 코드로 고정)
- 테스트 피라미드
  - 단위 테스트: 클래스 하나, 빠름, 스프링 없음 - Ondal `AuthorizationMappingValidatorTest`
  - 통합 테스트: 여러 층을 실제로 연결 - Ondal API 테스트 3개(인터셉터·컨트롤러·서비스·JPA·실제 DB 전부)
  - E2E: 브라우저까지 - Ondal BE 범위 밖(FE가 헤드리스 브라우저로 수행)
- Ondal 선택: **API 통합 테스트 위주**(요청 → 상태코드·JSON 검증) + 검증기 규칙은 단위 테스트. 이유 = 권한·트랜잭션·LAZY 같은 문제는 층을 합쳐야 드러남

### 8.2 JUnit 5 기본과 보조 라이브러리

| 어노테이션·도구 | 뜻 | Ondal 예 |
|---|---|---|
| `@Test` | 테스트 메서드 | 전부 |
| `@Nested` | 내부 클래스로 묶음 - 엔드포인트별 그룹 | `class Create { ... }` |
| `@DisplayName` | 표시 이름 | `"POST /api/cohorts - 분반 생성"` |
| `@BeforeEach` / `@AfterEach` | 각 테스트 전·후 실행 | `ApiTestSupport.cleanDatabase()` |
| 테스트 인스턴스 생명주기 | **메서드마다 새 인스턴스**(기본) - 필드 상태가 테스트 간에 섞이지 않음 | |
| 메서드명 | 한국어·언더스코어 허용 - 시나리오가 곧 이름 | `미로그인이면_401` |
| AssertJ | `assertThat(x).isEqualTo(...)` 체이닝 단언 | `assertThat(violations).isEmpty()` |
| Hamcrest | 매처 - MockMvc `jsonPath`와 함께 | `jsonPath("$.operators", hasSize(2))` |
| Mockito | 가짜 객체 | `mock(RequestMappingHandlerMapping.class)`, `when(...).thenReturn(...)` |

### 8.3 스프링 테스트의 종류와 컨텍스트 캐싱

| 어노테이션 | 띄우는 것 | 용도 |
|---|---|---|
| `@SpringBootTest` | **전체** ApplicationContext(모든 빈). 기본 `webEnvironment = MOCK`(실제 포트 안 열림) | Ondal API 테스트 - 인터셉터·리졸버·예외 핸들러·트랜잭션·JPA 전부 실제로 |
| `@WebMvcTest` | 컨트롤러 층만(서비스는 `@MockitoBean`) | 컨트롤러 단독 검증 - Ondal 미사용 |
| `@DataJpaTest` | JPA 층만 | 리포지토리 검증 - Ondal 미사용 |

- Ondal이 `@SpringBootTest`만 쓰는 이유: 검증하려는 것이 "권한 + 규칙 + DB"의 합작이라 슬라이스 테스트로는 의미가 없음
- **컨텍스트 캐싱**: 같은 설정(어노테이션·프로필·Import)의 테스트 클래스들은 컨텍스트를 **한 번만** 기동해 공유 → `ApiTestSupport`를 모든 API 테스트가 상속하는 구조 = 컨텍스트 1개, PostgreSQL 컨테이너 1개
- `@ActiveProfiles("test")`: local 시더를 빈에서 제외. 픽스처는 각 테스트가 직접 생성

### 8.4 MockMvc

- 서블릿 컨테이너(Tomcat)를 띄우지 않고 `DispatcherServlet`을 **직접 호출**하는 테스트 클라이언트. 필터·인터셉터·리졸버·컨트롤러·예외 핸들러를 전부 태움(2.4절 그림이 그대로 실행됨)
- 사용 형태: `mockMvc.perform(post("/api/cohorts").session(...).contentType(JSON).content(body)).andExpect(status().isCreated()).andExpect(jsonPath("$.id").exists())`
- `MockHttpSession`: 가짜 `HttpSession`. 속성을 직접 넣을 수 있어 "로그인된 상태"를 로그인 API 없이 만들 수 있음(`LoginHelper`)
- `@AutoConfigureMockMvc`로 `MockMvc` 빈 주입(Boot 4 패키지 `org.springframework.boot.webmvc.test.autoconfigure`)

### 8.5 Testcontainers와 `@ServiceConnection`

- Testcontainers: 테스트 중 Docker로 실제 DB 컨테이너를 띄우고 끝나면 정리. Docker 실행 필수
- `@Bean @ServiceConnection PostgreSQLContainer postgres()` (`PostgresContainerConfig`): Boot가 컨테이너 타입을 보고 `spring.datasource.*`에 해당하는 접속 정보를 **자동 등록** → test용 yml 불필요
- H2(인메모리 DB) 대신 실제 PostgreSQL을 쓰는 이유: 방언 차이(타입·함수·제약 동작)로 "테스트는 통과, 운영은 실패"를 막기 위해. 이미지 버전은 docker-compose와 동일(`postgres:16`)

### 8.6 테스트 간 격리 - 롤백 방식 vs TRUNCATE 방식

- 흔한 방식: 테스트 클래스에 `@Transactional` → 각 테스트가 트랜잭션 안에서 돌고 끝나면 롤백
- Ondal이 **의도적으로 미채택**한 이유(`DatabaseCleaner` 주석): MockMvc 요청 전체가 테스트 트랜잭션 **안**에서 돌아 버림 → 영속성 컨텍스트가 요청 끝까지 열려 있어 `open-in-view=false`가 드러내야 할 `LazyInitializationException`·N+1이 테스트에서는 안 보이고 실제 서버에서만 터짐
- 대신: 요청은 실제와 같이 **자기 트랜잭션**으로 돌게 두고, 각 테스트 후 `TRUNCATE ... RESTART IDENTITY CASCADE`로 전부 비움(`@AfterEach`)

### 8.7 테스트 작성 스타일

- 검증 축: **역할 × 엔드포인트 → 상태코드**(+ 응답 필드). 한 엔드포인트에 미로그인/일반 부원/수강생/운영진/관리자/다른 반 사람을 차례로 대입
- 이름: 시나리오 그대로(`보관된_분반은_관리자도_409_COHORT_ARCHIVED`)
- 픽스처는 **API로** 만든다(`createCohort`, `enrollStudent`가 실제 API 호출) - 리포지토리로 직접 넣으면 규칙(보관 409 등)을 우회해 비현실적인 상태가 됨
- 한 테스트 안에서 "실패 → 해제 → 성공"처럼 흐름을 잇는 것 허용(`보관된_분반은_409_해제하면_다시_200`)

---

## 9. 1부 점검 - 면접 예상 질문

- 답을 말할 수 없으면 괄호의 절로 복귀. 2부를 읽은 뒤 다시 한 번

**HTTP·REST (1장)**
1. 401과 403의 차이, 404로 존재를 숨기는 경우 (1.4, 7.1)
2. 멱등성이란, PUT과 POST와 DELETE 중 멱등인 것 (1.3)
3. 경로 변수와 쿼리 파라미터의 용도 차이 (1.2, 1.5)
4. 출처(origin)의 세 요소, `localhost:5173`과 `:8080`이 다른 출처인 이유 (1.8)

**서블릿·MVC (2장, 5장)**
5. Filter와 Interceptor의 차이, Ondal이 인터셉터를 고른 이유 (2.3)
6. DispatcherServlet이 요청을 처리하는 순서 - HandlerMapping → 인터셉터 → 아규먼트 리졸버 → 컨트롤러 → 메시지 컨버터 (2.4)
7. 인터셉터에서 던진 예외가 `@RestControllerAdvice`에 도달하는가 (2.4, 17장)
8. `@RestController`와 `@Controller`의 차이 (2.5, 5.1)
9. `@RequestBody`·`@Valid`가 각각 언제 실행되고 실패하면 어떤 예외인가 (5.3)
10. `@EnableWebMvc`를 붙이면 무슨 일이 생기나 (5.6)
11. 커스텀 어노테이션에 `@Retention(RUNTIME)`이 필요한 이유 (5.8)

**스프링 핵심 (3장)**
12. IoC와 DI의 차이 (3.2)
13. 생성자 주입을 권장하는 이유 3가지 (3.3)
14. `@SpringBootApplication`의 세 구성과 컴포넌트 스캔 기준 패키지 (3.6)
15. 빈의 기본 스코프와 그로 인한 코딩 제약 (3.5)
16. 자동 설정이 물러나는 조건 (3.7)
17. `SmartInitializingSingleton`과 `CommandLineRunner`의 실행 시점 차이 (3.8)
18. 같은 타입 빈이 둘일 때 해결 방법 (3.10)
19. 순환 의존을 어떻게 풀었나 (3.12)
20. 프록시란, `@Transactional`이 자기 호출에서 안 먹는 이유 (3.13, 6.5)

**빌드·설정 (4장)**
21. 스타터에 버전이 없는 이유 (4.1)
22. `runtimeOnly`와 `implementation`의 차이, DB 드라이버가 `runtimeOnly`인 이유 (4.2)
23. `ddl-auto` 옵션과 운영 권장값 (4.5, 6.7)
24. OSIV란, 끄면 무엇이 달라지나 (4.5, 6.3)
25. 프로필별 설정 분리 방법과 우선순위 (3.9, 3.10, 4.5)

**JPA (6장)**
26. JPA·Hibernate·Spring Data JPA의 관계 (6.1)
27. 영속성 컨텍스트의 네 기능 (6.3)
28. 더티 체킹 - `save()` 없이 UPDATE가 나가는 이유 (6.3)
29. `IDENTITY` 전략이 쓰기 지연과 충돌하는 이유 (6.2)
30. `EnumType.ORDINAL`이 위험한 이유 (6.2)
31. 지연 로딩·프록시·`LazyInitializationException` (6.4)
32. N+1 문제와 해법 3가지, Ondal 적용 (6.4)
33. `@Transactional`의 전파 기본값과 롤백 규칙 (6.5)
34. `readOnly = true`의 효과 (6.5)
35. 쿼리 메서드 파생 규칙, `findByCohortIdAndUserId`가 조인 없이 동작하는 이유 (6.6)
36. fetch join과 일반 join의 차이 (6.4, 6.6)

**보안 (7장)**
37. 세션 방식과 토큰 방식의 비교, Ondal 선택 근거 (7.3)
38. 세션에 엔티티 대신 id만 넣는 이유 (7.2)
39. `SameSite=Lax`·`HttpOnly`·`Secure`가 각각 막는 것 (7.4, 7.5)
40. 세션 고정 공격과 방어 (7.5)
41. CORS 사전 요청이란, 왜 인터셉터가 통과시키나 (7.6)
42. CORS가 CSRF 방어가 아닌 이유 (7.6)
43. Spring Security를 보류한 근거 3가지 (7.7)
44. default-deny와 fail-closed의 뜻 (7.8)

**테스트 (8장)**
45. `@SpringBootTest`와 `@WebMvcTest`의 차이 (8.3)
46. MockMvc는 무엇을 띄우고 무엇을 안 띄우나 (8.4)
47. H2 대신 Testcontainers를 쓰는 이유 (8.5)
48. 테스트 `@Transactional` 롤백을 안 쓴 이유 (8.6)
49. 컨텍스트 캐싱이 테스트 구조에 끼친 영향 (8.3)

---

# 2부. Cohort 슬라이스 - 코드가 실제로 움직이는 방식

## 10. 무엇을 만들었나 - 수직 슬라이스

### 10.1 문제

- Ondal에 앞으로 붙는 도메인: 분반, 과제, 제출, 현황판
- 도메인마다 구조가 다르면 읽는 사람이 매번 새로 학습해야 함
- → 첫 도메인 **분반(Cohort)** 을 만들면서 "이후 도메인이 그대로 따를 틀"까지 함께 수립
- 분반 = "2026-2 C언어"처럼 학기·트랙 단위로 열리는 반. 모든 과제·제출은 어떤 분반에 속함 → 첫 도메인으로 선정
- 뒤이어 같은 틀로 **수강생 배정**(PR #5, [enrollment/design.md](../enrollment/design.md))까지 구현 완료 - 이 문서는 둘을 합쳐 설명

### 10.2 수직 슬라이스

- 수직 슬라이스 = 한 도메인을 위에서 아래까지 세로로 한 줄 다 뚫는 것
- Cohort 슬라이스의 여섯 층(1부의 장과 대응):

| 층 | 역할 | 파일 | 기초 |
|---|---|---|---|
| 엔티티 | DB 테이블과 1:1로 대응하는 자바 객체 | `Cohort`, `Enrollment`, `User` | 6.2 |
| 리포지토리 | DB에서 엔티티를 읽고 쓰는 창구 | `CohortRepository`, `EnrollmentRepository`, `UserRepository` | 6.6 |
| 서비스 | 업무 규칙. 트랜잭션의 경계 | `CohortService`, `EnrollmentService`, `UserService` | 6.5 |
| 컨트롤러 | HTTP 요청을 받아 서비스를 부르고 응답 반환 | `CohortController`, `EnrollmentController` | 5.1 |
| DTO | 요청·응답 전용 데이터 그릇 | `CohortCreateRequest`, `CohortResponse` 등 | 5.3, 5.4 |
| 테스트 | 역할 × API → 상태코드 검증 | `CohortApiTest`, `EnrollmentApiTest` | 8장 |

- 함께 깔린 "모든 도메인이 공유하는 바닥"
  - 권한 판정 장치: `auth/authorization/` (13장)
  - 예외 → HTTP 응답 변환 장치: `common/error/` (17장)
  - 테스트 지원: `test/support/` (19장)

### 10.3 패키지 구조

- 도메인 = 패키지 하나(`cohort/`, `enrollment/`, `user/`, `auth/`), 그 안을 계층별 하위 패키지로: `controller/`, `service/`, `repository/`, `entity/`, `dto/` (BE PR #3)
- `auth/`는 공통 기반이라 계층 폴더는 `controller/`·`service/`·`dto/`만, 인터셉터·리졸버·상수는 루트, 권한은 `authorization/`
- 패키지 간 의존은 서비스·리포지토리 층에서 **단방향**(cohort → enrollment: 응답 조립용 읽기 / enrollment → cohort: 존재·보관 확인). 컨트롤러는 **자기 패키지의 서비스만** 주입

### 10.4 용어 빠른 참조 (1부 어디를 볼지)

| 용어 | 절 |
|---|---|
| 빈·주입·생성자 | 3.2~3.4 |
| 프록시·`@Transactional`·readOnly | 3.13, 6.5 |
| 영속성 컨텍스트·더티 체킹·지연 로딩·N+1 | 6.3, 6.4 |
| 쿼리 메서드·`@Query`·fetch join | 6.6 |
| 인터셉터·아규먼트 리졸버·커스텀 어노테이션 | 2.3, 5.7, 5.8 |
| 세션·쿠키·CORS | 7.2, 7.4, 7.6 |
| record·Jackson·Bean Validation | 5.3, 5.4 |
| MockMvc·Testcontainers·TRUNCATE | 8.4~8.6 |

---

## 11. 데이터 - 사람·분반·소속 (엔티티 코드 읽기)

### 11.1 테이블 세 개

```
users ──────< enrollments >────── cohorts
                (cohort_id, user_id) UNIQUE
                role: OPERATOR | STUDENT
```

- **users** - 사람. `loginId`(고유), `name`, `globalRole`(ADMIN | MEMBER), `createdAt`
- **cohorts** - 분반. `name`, `description`, `status`(ACTIVE | ARCHIVED), `createdAt`
- **enrollments** - 소속. "누가, 어느 분반에서, 무슨 역할인가." `cohort_id`, `user_id`, `role`(OPERATOR | STUDENT), `createdAt`. 같은 사람이 같은 분반에 두 번 소속 불가(유니크 제약)
- 전체 P1 스키마(과제·제출 포함)의 단일 출처: [db/schema.md](../db/schema.md)

### 11.2 `User` - 사람

```java
@Entity
@Table(name = "users")                       // "user"는 PostgreSQL 예약어
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;
    @Column(nullable = false, unique = true, length = 50) private String loginId;
    @Column(nullable = false, length = 50) private String name;
    @Enumerated(EnumType.STRING) @Column(nullable = false, length = 20) private GlobalRole globalRole;
    @Column(nullable = false, updatable = false) private Instant createdAt;

    protected User() {}                                          // JPA용
    private User(String loginId, String name, GlobalRole role) { ... createdAt = Instant.now(); }
    public static User member(String loginId, String name)       // 일반 부원 - 스텁 로그인·배정이 사용
    public static User admin(String loginId, String name)        // 부트스트랩 전용(시더·수동 SQL)
    public boolean isAdmin()
}
```

- 홈페이지 연동 후에는 "홈페이지 계정의 로컬 사본" - 신원의 원본은 홈페이지, Ondal은 `loginId`로 매칭
- 정적 팩토리가 둘인 이유: `admin(...)`은 일반 코드 경로에서 호출 금지를 이름으로 표시
- `GlobalRole {ADMIN, MEMBER}`: **사람에게** 붙는 전역 역할

### 11.3 `Cohort` - 분반

```java
@Entity @Table(name = "cohorts")
public class Cohort {
    ... id, name(100), description(text), status(STRING), createdAt(updatable=false)
    public static Cohort create(String name, String description)   // status = ACTIVE
    public void update(String name, String description)            // PUT 전체 교체
    public void archive()  / restore()                             // 멱등 - 이미 그 상태여도 예외 없음
    public boolean isActive() / isArchived()
    public void ensureActive()   // ARCHIVED면 CohortArchivedException(409) - 분반 스코프 쓰기의 첫 줄
}
```

- `CohortStatus {ACTIVE, ARCHIVED}`: 하드 삭제 없음(원칙). 학기가 끝나면 **보관** - 소속자 열람은 유지, 변경은 누구도 불가(13.6절)
  - 2026-08-19 결정: 보관이 기본인 것은 유지하되 ADMIN 전용 **영구 삭제**를 최후 수단으로 별도 슬라이스에서 추가 예정(미구현, [db/schema.md](../db/schema.md) 결정 5)
- `Cohort` 안에 소속 목록(`List<Enrollment>`)이 **없음** - 11.5절

### 11.4 `Enrollment` - 소속

```java
@Entity
@Table(name = "enrollments",
       uniqueConstraints = @UniqueConstraint(name = "uk_enrollment_cohort_user", columnNames = {"cohort_id", "user_id"}))
public class Enrollment {
    @ManyToOne(fetch = FetchType.LAZY, optional = false) @JoinColumn(name = "cohort_id", nullable = false) private Cohort cohort;
    @ManyToOne(fetch = FetchType.LAZY, optional = false) @JoinColumn(name = "user_id",   nullable = false) private User user;
    @Enumerated(EnumType.STRING) private EnrollmentRole role;
    ...
    public static Enrollment create(Cohort cohort, User user, EnrollmentRole role)
    public void promoteToOperator()   // STUDENT → OPERATOR. 유일한 역할 변경 경로, 강등 없음
    public boolean isOperator()
}
```

- `EnrollmentRole {OPERATOR, STUDENT}`: **사람-분반 관계에** 붙는 역할. `satisfies(required)`: OPERATOR는 STUDENT가 할 수 있는 것을 전부 할 수 있음(OPERATOR가 STUDENT를 포함)
- 하드 삭제되는 관계 테이블이며 **다른 엔티티의 FK 대상이 아님** - 제출(Submission)은 `user_id`로 사용자를 직접 참조, 소속이 해제돼도 제출물은 남음(design.md 결정 9)
- 두 연관 모두 LAZY(6.4절). 읽을 때 필요한 쪽만 fetch join으로 채움(11.7절)

### 11.5 역할이 두 층인 이유와 `Cohort`가 소속 목록을 갖지 않는 이유

- 역할 두 층([permissions.md](../permissions.md) 1절, [decisions/3](../decisions/3-권한-모델-2층-구조.md))
  - `User.globalRole`: ADMIN = 동아리 임원(모든 분반에서 운영자 이상, 소속 없어도 됨) / MEMBER = 일반 부원
  - `Enrollment.role`: 한 사람이 A반 운영진이면서 B반 수강생일 수 있음 → "이 사람은 운영진인가"는 답이 없고 "**이 분반의** 운영진인가"만 답이 있음
  - 기각한 대안: 전역 슈퍼유저 통합(남의 반 조작·학기 후 권한 잔존), 전역 3단 등급(결국 "어느 반의"를 저장할 테이블이 필요)
- `Cohort`에 컬렉션을 두지 않는 이유: 양방향으로 엮으면 조회 한 번에 소속 전체가 딸려 오거나(N+1), 서로 참조하다 꼬이는 문제가 처음부터 발생. 소속이 필요하면 항상 `EnrollmentRepository`로 별도 조회(6.4절)

### 11.6 화면용 직책 명칭 - `RoleTitle`

```java
public enum RoleTitle {
    EXECUTIVE("해구르르"), OPERATOR("교육운영진"), MEMBER("일반 수강생");
    @JsonValue public String label()                               // JSON으로는 문자열 그대로
    public static RoleTitle of(User user, EnrollmentRole roleOrNull)  // ADMIN > OPERATOR > 나머지
}
```

- 역할 = 코드용 이름, 화면 = 동아리 용어. 매핑은 서버 한 곳(단일 출처). API는 문자열을 그대로 내려 주고 프런트는 매핑 없이 표시
- 임원이 어느 분반에 운영진으로 들어가 있어도 **해구르르**로 표시. "일반 수강생"은 미확정 - label 한 줄만 고치면 전체 반영

### 11.7 리포지토리 세 개

| 메서드 | 생성 방식 | 용도 |
|---|---|---|
| `UserRepository.findByLoginId(loginId)` | 파생 | 스텁 로그인·배정의 find-or-create, 해제 시 사용자 찾기 |
| `CohortRepository.findAllByStatusOrderByCreatedAtDesc(status)` | 파생 | 관리자 목록(상태별, 최신순) |
| `EnrollmentRepository.findByCohortIdAndUserId(cohortId, userId)` | 파생 | **권한 판정용** - 연관을 안 건드리고 role만. FK 컬럼 비교라 조인 없음 |
| `EnrollmentRepository.findAllByUserIdWithCohort(userId)` | `@Query` fetch join cohort | 내 분반 목록 |
| `EnrollmentRepository.findAllByCohortIdWithUser(cohortId)` | `@Query` fetch join user, `order by role asc, createdAt asc` | 명부 - 운영진 먼저(알파벳순 우연, 주석 명시), 등록순 |
| `EnrollmentRepository.findAllByCohortIdInWithUser(cohortIds)` | `@Query` fetch join user, `in :cohortIds` | 분반 N개의 응답을 **쿼리 1번**으로 조립 |

- 규약: fetch join 조회는 이름 끝에 `WithXxx` + **반드시** `@Query`(6.6절의 기동 실패 사유)

### 11.8 엔티티 작성 규약 (모든 도메인이 따름)

- `protected` 기본 생성자 + `private` 전체 생성자(`createdAt = Instant.now()`) + 정적 팩토리 `create(...)`
- **setter 없음** - 상태 변경은 `update`, `archive`, `restore`, `promoteToOperator`처럼 이름 있는 메서드로만
- enum은 `EnumType.STRING`, 시각은 `Instant`(UTC), Lombok 없음
- 하드 삭제 없음(원칙) - 분반은 `ARCHIVED`로 전환. 예외는 11.3절의 영구 삭제(예정)

---

## 12. 요청 하나의 여행 - 여기까지 읽으면 뼈대 완성

추적 대상(가장 흔한 요청): **학생 `student1`의 자기 분반 상세 조회 - `GET /api/cohorts/1`** (쿠키에 세션 id 포함)

### 12.1 도착 - Tomcat과 DispatcherServlet

- Tomcat이 요청을 파싱해 스레드 하나에 배정 → Filter 체인 → `DispatcherServlet`(2.4절)
- `HandlerMapping`이 `GET /api/cohorts/{cohortId}` 패턴을 찾아 `CohortController#get`을 핸들러로 결정. 경로 변수 `cohortId="1"`을 request attribute에 문자열로 저장
- 실행 체인 = [`AuthInterceptor`, `AuthorizationInterceptor`] + 핸들러

### 12.2 문지기 둘 - 인터셉터

- 등록: `WebConfig.addInterceptors`. 경로 `/api/**`, 제외 `AuthPaths.PUBLIC`(`/api/auth/login`, `/api/auth/logout`, `/api/health`)

**① `AuthInterceptor` - 로그인 여부**

```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
    if (CorsUtils.isPreFlightRequest(request)) return true;          // 사전 요청은 세션이 없음
    HttpSession session = request.getSession(false);
    if (session == null || session.getAttribute(SessionConst.LOGIN_USER_ID) == null) {
        throw new UnauthenticatedException();                        // → 401
    }
    return true;
}
```
- 역할은 이것 하나. 세션이 없으면 끝

**② `AuthorizationInterceptor` - 이 API에 필요한 권한과 요청자의 보유 여부**

```
사전 요청/OPTIONS        → 통과 (데이터가 나가지 않음)
컨트롤러 메서드가 아니면 → 통과 (정적 파일, 에러 페이지 등)
effective = 이 핸들러의 "유효 권한 어노테이션" 하나  (없거나 둘 이상이면 IllegalStateException → 500)
effective가 @CohortRole이면 → 경로의 {cohortId}를 먼저 읽음 ("abc"면 400)
user = 세션의 id로 DB 조회 (없으면 401)
@LoginOnly  → 통과
@AdminOnly  → user.isAdmin() 아니면 403
@CohortRole → CohortAuthorizer.isAllowed(user, cohortId, 요구 역할) 아니면 403
```

- 이 요청의 핸들러 `CohortController.get`에는 `@CohortRole(STUDENT)` → cohortId=1을 읽고, student1을 로드하고, `isAllowed` 판정
- `CohortAuthorizer.isAllowed`:

```
ADMIN이면 true
아니면 enrollments에서 (cohort 1, student1) 한 줄을 찾음 - findByCohortIdAndUserId
  있으면 → 그 role이 요구 역할을 만족하는가?
           OPERATOR.satisfies(STUDENT)=true, STUDENT.satisfies(STUDENT)=true, STUDENT.satisfies(OPERATOR)=false
  없으면 → false → 403
```

- = [permissions.md](../permissions.md) 2절의 판정 4단계 ①로그인 ②ADMIN ③소속 ④역할
- 인터셉터는 **분반 엔티티를 읽지 않고, HTTP 메서드(GET/POST)도 보지 않음** - 보관 여부는 권한이 아니라 서비스가 다루는 도메인 규칙(13.6절)

### 12.3 컨트롤러 - 4줄 패턴과 파라미터 채우기

```java
@CohortRole(EnrollmentRole.STUDENT)
@GetMapping("/{cohortId}")
public CohortResponse get(@PathVariable Long cohortId, @LoginUser User me) {
    return cohortService.findOne(cohortId, me);
}
```

- `HandlerAdapter`가 파라미터를 채움(5.2절, 5.7절)
  - `@PathVariable Long cohortId`: 문자열 "1" → `Long`
  - `@LoginUser User me`: `LoginUserArgumentResolver`가 세션 id로 `User` 조회해 주입 → 컨트롤러가 세션을 직접 만질 일 없음. 세션은 살아 있는데 DB에서 사용자가 삭제됐으면 401
- 컨트롤러 규약: **권한 어노테이션 → (`@Valid`) → 서비스 호출 → 서비스가 준 DTO 반환.** 그 외 로직 없음. `CohortController`의 여섯 메서드 전부 같은 형태
- ※ 인터셉터도 `User`를 한 번 조회 → 같은 요청에서 두 번 조회하는 구조. PK 조회라 비용은 무시할 수준이나 인지해 둘 사항(request attribute로 넘기는 최적화는 미채택 - design.md 2절)

### 12.4 서비스 - 트랜잭션이 열리는 곳

```java
@Service
@Transactional
public class CohortService {
    @Transactional(readOnly = true)
    public CohortResponse findOne(Long cohortId, User viewer) {
        return assembler.toResponse(requireCohort(cohortId), viewer);
    }
    private Cohort requireCohort(Long cohortId) {
        return cohortRepository.findById(cohortId)
                .orElseThrow(() -> new NotFoundException("분반을 찾을 수 없습니다."));
    }
}
```

- 프록시가 `findOne` 진입 시 읽기 전용 트랜잭션 + 영속성 컨텍스트를 열고, 반환 시 닫음(6.5절)
- `requireCohort` = 없으면 404. 모든 서비스에 같은 이름의 private 메서드 존재
- `open-in-view=false` → **트랜잭션은 서비스에서 종료.** 컨트롤러로 나간 엔티티는 준영속 → **서비스가 DTO까지 완성해서 반환.** 이 규칙 하나가 구조 전체를 단순하게 유지

### 12.5 조립기 - 보는 사람에 따라 달라지는 응답

- `CohortResponse`: 목록·단건·생성·수정 응답이 전부 같은 형태. 단, 몇 필드는 **누가 보느냐**에 따라 상이 → 조립은 `CohortResponseAssembler` 담당(서비스에서 분리한 이유: 3.12절 순환 의존)
- `toResponses(cohorts, viewer)`가 하는 일
  1. 분반 id 목록으로 소속을 **쿼리 한 번**에 조회(`findAllByCohortIdInWithUser`, user까지 fetch join) → `groupingBy(e -> e.getCohort().getId())`(프록시의 `getId()`는 DB를 치지 않음 - 6.4절)
  2. 분반마다 필드를 채움
     - `operators` - OPERATOR 소속들을 `UserSummary{id, name, title}`로. `loginId`·`globalRole` 미포함(학생에게도 내려가는 응답). 이름순 정렬
     - `myRole` - 소속 중 `user.id == viewer.id`인 것의 role. 없으면 `null`(ADMIN이 남의 분반을 볼 때)
     - `myTitle` - `RoleTitle.of(viewer, myRole)`
     - `canManage` - `CohortAuthorizer.canManage` = 분반이 ACTIVE이고 (ADMIN이거나 OPERATOR). 프런트는 이 값만 보고 운영 버튼 표시
     - `studentCount` - ADMIN 또는 OPERATOR에게만 숫자, 학생에게는 `null`. 인원수도 타인 정보로 간주
- 단건도 리스트 1개짜리로 같은 경로(`toResponse` → `toResponses(List.of(cohort))`)
- 자체 `@Transactional` 없음 - 호출한 서비스의 트랜잭션 안에서 실행(전제)

student1이 받는 응답 예:

```json
{
  "id": 1, "name": "2026-2 C언어", "description": "...", "status": "ACTIVE", "createdAt": "2026-08-18T...Z",
  "operators": [{ "id": 2, "name": "operator1", "title": "교육운영진" }],
  "studentCount": null,
  "myRole": "STUDENT",
  "myTitle": "일반 수강생",
  "canManage": false
}
```

- `RoleTitle`이 `"교육운영진"` 문자열로 나가는 이유: enum에 `@JsonValue`(5.4절)

### 12.6 돌아가는 길 - 직렬화와 예외

- 컨트롤러 반환 `CohortResponse`(record) → Jackson이 JSON으로 직렬화 → 200 본문
- 도중 어디서든 예외(401/403/404/409/400/500)가 나면 `GlobalExceptionHandler`가 `{code, message}`로 변환(17장). 컨트롤러·서비스는 예외를 **던지기만**

> **뼈대 요약**: 요청 → 문지기 둘 → 컨트롤러(파라미터 주입) → 서비스(트랜잭션) → 조립기(쿼리 1번) → JSON. 실패 시 예외 핸들러. 이 흐름이 모든 API에 동일하게 적용

---

## 13. 권한 장치 상세 - `auth/authorization/`

- 사고 시 "남의 반 과제가 새는" 최고 위험 영역 → 가장 신경 써서 제작. 수정 담당: PM

### 13.1 어노테이션 세 개

`/api/**`의 모든 컨트롤러 메서드는 셋 중 **정확히 하나** 부착

| 어노테이션 | 뜻 | Ondal 사용 예 |
|---|---|---|
| `@LoginOnly` | 로그인만 되어 있으면 됨. 분반과 무관한 API | `GET /api/auth/me`, `GET /api/me/cohorts` |
| `@AdminOnly` | 전역 ADMIN만. 경로에 `{cohortId}`가 있어도 소속은 보지 않음 | 분반 생성·수정·보관·목록, 운영진 지정/해제 |
| `@CohortRole(역할)` | 경로의 `{cohortId}` 분반에서 그 역할 **이상**. ADMIN은 자동 통과 | `STUDENT`: 분반 상세 / `OPERATOR`: 명부, 수강생 배정/제외 |

- `@CohortRole(STUDENT)` = "소속자 누구나" / `@CohortRole(OPERATOR)` = "운영진 이상"
- 세 어노테이션 모두 `@Target({METHOD, TYPE}) @Retention(RUNTIME)` - 메서드·클래스에 붙일 수 있고 런타임에 읽힘(5.8절)

### 13.2 유효 어노테이션을 하나만 고르는 규칙 - `AuthorizationAnnotations`

```java
static Annotation resolve(HandlerMethod handlerMethod) {
    List<Annotation> onMethod = findAll(handlerMethod.getMethod());      // 메서드에 붙은 3종
    if (!onMethod.isEmpty()) return single(onMethod, ...);               // 하나면 그것, 둘 이상이면 예외
    List<Annotation> onClass = findAll(handlerMethod.getBeanType());     // 없으면 클래스 것
    if (!onClass.isEmpty()) return single(onClass, ...);
    throw new IllegalStateException("권한 어노테이션이 없는 API: ...");
}
```

- 규칙 = **위치 우선**: 메서드에 하나라도 있으면 메서드 것, 없으면 클래스 것. 같은 위치에 둘 이상 → 설정 오류(`IllegalStateException`)
- `findAll`은 `AnnotatedElementUtils.findMergedAnnotation(element, type)` 사용 - 스프링 유틸. 메타 어노테이션·인터페이스·상위 클래스까지 찾음
- 이 규칙을 한 클래스에 모은 이유(사고 이력, design.md 결정 14)
  - 초기 구현: 종류별로 따로 찾아 "`@LoginOnly`가 있으면 통과" 식으로 판정
  - → 클래스에 `@LoginOnly`, 메서드에 `@AdminOnly`를 단 경우 클래스 것이 먼저 걸려 **일반 부원이 관리자 API를 통과**(fail-open)
  - 리뷰에서 발견 → "유효 어노테이션 하나를 먼저 고르고, 그것으로만 판정"으로 수정
- 인터셉터(런타임)와 기동 검증기(부팅 시)가 같은 클래스를 사용 → 두 판정이 어긋날 수 없음. `violation(handlerMethod)`은 검증기용으로 예외 대신 설명 문자열을 반환

### 13.3 판정 규칙의 단일 출처 - `CohortAuthorizer`

```java
public Optional<EnrollmentRole> roleOf(User user, Long cohortId)               // 소속 역할. 비소속(ADMIN 포함)이면 empty
public boolean isAllowed(User user, Long cohortId, EnrollmentRole required)    // ADMIN || roleOf.satisfies(required)
public boolean canManage(User user, Cohort cohort, EnrollmentRole myRoleOrNull) // cohort ACTIVE && (ADMIN || OPERATOR)
```

- `isAllowed` - 요청 통과 여부. 인터셉터가 사용 (②③④ 단계)
- `canManage` - 프런트의 "운영 버튼" 표시 여부. 조립기가 사용. 인터셉터는 Cohort를 로드하지 않으므로 이미 로드된 Cohort를 받는 별도 메서드. 보관 분반은 누구도 운영 불가 → false
- 규칙 변경 시 이 클래스만 수정 - 두 군데에 복제되어 서로 다르게 자라는 일 방지

### 13.4 `AuthorizationInterceptor`가 경로 변수를 직접 읽는 이유

```java
private static Long readCohortId(HttpServletRequest request, HandlerMethod handlerMethod) {
    Map<String, String> variables = (Map<String, String>) request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE);
    String raw = variables == null ? null : variables.get("cohortId");
    if (raw == null) throw new IllegalStateException("@CohortRole 핸들러의 경로에 {cohortId} 변수가 없다: ...");   // 설정 오류 → 500
    try { return Long.parseLong(raw); }
    catch (NumberFormatException e) { throw new InvalidInputException("cohortId: 숫자여야 합니다."); }          // → 400
}
```

- 인터셉터는 `@PathVariable` 바인딩보다 **먼저** 돌기 때문에(2.4절) 숫자가 아닌 값(`/api/cohorts/abc`)의 400 처리를 여기서 직접 함 - 핸들러 쪽 400 변환에는 도달하지 않음
- `{cohortId}`는 **ADMIN 판정보다 먼저** 읽음 - ADMIN 세션으로만 테스트해도 경로 규약 위반·비숫자 id가 드러나도록
- `@AdminOnly` 경로의 `/api/cohorts/abc/archive`는 인터셉터가 cohortId를 안 읽으므로 `@PathVariable` 바인딩에서 400(`MethodArgumentTypeMismatchException`) - 결과는 같은 `INVALID_INPUT`

### 13.5 인터셉터가 보장하는 것과 보장하지 않는 것

- 보장: "요청자가 `{cohortId}` 분반에서 그 역할 이상"
- 보장하지 않음: 경로 뒤쪽의 하위 id(예: `assignmentId`)가 **그 분반 것인지**
- → 이후 슬라이스의 서비스는 반드시 `findByIdAndCohortId(assignmentId, cohortId)`처럼 분반으로 범위를 좁혀 조회(손자는 체인 전부). `findById` 단독 호출 금지. 다른 반 과제 id를 지정하면 존재 여부를 알려 주지 않는 **404**가 정답(IDOR 방지, 7.5절)
- OJ(P3, `cohort_id NULL`) 엔드포인트는 `@CohortRole` 미사용 - 풀이 `@LoginOnly`, 출제 `@AdminOnly`

### 13.6 보관은 권한이 아니라 규칙 - `ensureActive()`와 409

- ARCHIVED 분반은 누구도 변경 불가(ADMIN 포함). 열람은 유지
- 인터셉터에서 막지 않는 이유: "권한이 없다(403)"가 아니라 "지금은 바꿀 수 없는 상태다(409)"이기 때문. 403은 프런트가 홈으로 보내는 코드라 충돌
- → **분반 스코프의 쓰기 서비스 메서드는 첫 줄에서 `cohort.ensureActive()` 호출** - `update`, `assign`, `promoteToOperator`, `remove`가 해당. `archive`/`restore` 자체는 호출하지 않음(보관 해제는 보관 상태에서 호출해야 하므로)
- 프런트: 409 `COHORT_ARCHIVED` 수신 시 안내만 표시, `status == ARCHIVED`면 쓰기 UI를 사전 비활성
- 기각 대안(design.md 결정 3): 인터셉터에서 GET 외 403(HTTP 동사에 인가를 결합, FE 403→홈 규칙과 충돌) / 소속자에게 완전 비노출(과거 과제·제출물 열람 필요)

### 13.7 부팅 때 잡는다 - `AuthorizationMappingValidator`

- 동작 시점: 모든 싱글톤 생성 완료 직후(`SmartInitializingSingleton`, 3.8절). `RequestMappingHandlerMapping.getHandlerMethods()`로 **모든 컨트롤러 매핑**을 순회
- 규칙 - 위반이 하나라도 있으면 목록을 모아 `IllegalStateException` → **서버 기동 실패**
  - (a) `AuthPaths.PUBLIC`이 아닌 `/api/**` 핸들러에 어노테이션 없음
  - (b) `@CohortRole`인데 경로에 `{cohortId}` 없음
  - (c) 경로에 `{cohortId}`가 있는데 유효 어노테이션이 `@CohortRole`도 `@AdminOnly`도 아님 - 분반 자원을 로그인만으로 열면 안 됨
  - (d) 한 위치(메서드 또는 클래스)에 어노테이션 둘 이상
  - (e) 우리 패키지(`kr.haedal.ondal`)의 컨트롤러가 `/api/` 밖에 매핑 - `/apo/...` 같은 오타로 인터셉터 두 개를 통째로 비껴가는 것 방지
- `/swagger-ui`, `/v3/api-docs`, `/error`처럼 `/api/` 밖의 라이브러리 핸들러는 대상 아님
- 효과: "어노테이션 붙이는 걸 잊는" 실수를 컴파일 다음 단계에서, 요청이 오기 전에 차단. 런타임의 500(13.2절)은 2중 안전망
- `validate(...)`는 정적·순수 함수로 분리 → 스프링 없이 가짜 매핑으로 단위 테스트(19장)

---

## 14. 소속 다루기 - `EnrollmentService` (수강생 배정 포함)

소속은 별도 패키지 `enrollment/`. 서비스 메서드 여섯 개:

| 메서드 | 하는 일 | 규칙 |
|---|---|---|
| `findMyCohorts(me)` | 내가 소속된 분반 전부 | 보관 분반 포함. ACTIVE가 먼저, 그 안에서 최신순(`Comparator.comparing(Cohort::isArchived).thenComparing(createdAt, reverseOrder())`). 응답은 조립기로 |
| `findMembers(cohortId)` | 분반 명부 | 운영진 먼저, 그다음 등록순. 항목 `MemberResponse{user(UserResponse), role, title, enrolledAt}` - 여기엔 `loginId`·`globalRole`이 있음(운영진 이상만 호출) |
| `assignStudents(cohortId, loginIds)` | 수강생 일괄 배정(UC-O2) | `assign(..., STUDENT)` 후 **갱신된 명부 전체** 반환 - FE가 재조회할 필요 없게 |
| `assign(cohortId, loginIds, role)` | loginId 목록을 role로 일괄 소속 | `ensureActive()`. `LinkedHashSet`으로 중복 제거(입력 순서 유지). 없는 사람은 만듦(MEMBER). 같은 role로 이미 있으면 그대로(멱등). **다른 role로 있으면 409** - 역할을 몰래 바꾸지 않음 |
| `promoteToOperator(cohortId, loginId)` | 운영진 지정 | `ensureActive()`. 미소속이면 OPERATOR로 소속, STUDENT면 승격, 이미 OPERATOR면 그대로. 멱등. 승격 경로는 이것 하나, 강등 API 없음(해제 후 다시 배정) |
| `remove(cohortId, loginId, expected)` | 소속 해제 | `ensureActive()`. `expected` 역할의 소속만 지움. 대상이 없거나 역할이 다르면 404. 성공 204 |

- `assign`이 "없는 사람은 만든다"인 이유
  - 아직 Ondal에 로그인한 적 없는 부원도 loginId만으로 사전 등록 가능해야 함(개강 전 세팅)
  - 그때 생성된 `User`의 이름은 임시로 loginId와 동일 → 홈페이지 연동 시 실제 이름으로 갱신
  - 이 find-or-create는 `UserService.findOrCreateMember`에 위치 - 스텁 로그인과 소속 배정이 공용. loginId가 비었거나 50자 초과면 400(경로 변수는 Bean Validation을 안 거치므로 여기서 한 번 더)
- `CohortService.create`: 분반 저장 후 `enrollmentService.assign(..., OPERATOR)` 호출 - 같은 트랜잭션(6.5절 전파) → 운영진 지정이 409로 실패하면 **분반 생성도 함께 롤백**
- 수강생 배정 슬라이스(PR #5)에서 추가된 것: 컨트롤러 메서드 2개(`POST/DELETE .../students`) + `assignStudents` 래퍼 + `StudentAssignRequest{loginIds}` DTO. `assign`/`remove`는 Cohort 슬라이스 때 이미 준비되어 있었음 - "틀 복제"가 실제로 이만큼 간단함을 보인 사례
- `/{role}s/{loginId}` 하위 자원은 해당 role의 Enrollment만 처리(design.md 결정 7): `/operators/{loginId}` 삭제는 OPERATOR만, `/students/{loginId}` 삭제는 STUDENT만. 시그니처 `remove(..., EnrollmentRole expected)`가 이를 강제

---

## 15. 로그인 - `auth/`

- P1의 인증 방식: 홈페이지 연동이 유력하나 미확정 → 현재는 **가짜 문**(스텁)만 달고, 나머지 흐름은 실물과 동일하게 구성(7.7절)

### 15.1 구성 요소

| 파일 | 역할 |
|---|---|
| `AuthService` | 인터페이스 `User login(String loginId)`. "이 loginId가 진짜 그 사람인가"를 확인하는 경계. **이 뒤편만 교체**하면 실제 인증으로 전환(3.11절) |
| `StubAuthService` | 현재 구현. 검증 없이 `userService.findOrCreateMember(loginId)`. 없으면 MEMBER로 생성 - 실제 연동에서도 "첫 방문자에게 로컬 User를 만든다"는 흐름은 동일, 검증 단계만 생략 |
| `AuthController` | `/api/auth/login`, `/me`, `/logout` |
| `LoginRequest` | `record { @NotBlank @Size(max=50) String loginId }` |
| `AuthPaths.PUBLIC` | 로그인 없이 되는 `/api` 경로의 **단 하나의 목록** - `WebConfig` 제외 목록과 기동 검증기가 둘 다 참조 |
| `SessionConst.LOGIN_USER_ID` | 세션 키 이름 |
| `AuthInterceptor` | 문지기 ① (12.2절) |
| `@LoginUser` + `LoginUserArgumentResolver` | 컨트롤러 매개변수에 세션 id로 조회한 **최신** `User` 주입 |

### 15.2 `AuthController.login` 한 줄씩

```java
@PostMapping("/login")
public UserResponse login(@RequestBody @Valid LoginRequest request, HttpServletRequest httpRequest) {
    User user = authService.login(request.loginId());           // 신원 확인(지금은 스텁)
    HttpSession oldSession = httpRequest.getSession(false);     // 세션 고정 공격 방지: 로그인 전 세션은 버리고
    if (oldSession != null) oldSession.invalidate();
    HttpSession session = httpRequest.getSession(true);         // 새로 발급
    session.setAttribute(SessionConst.LOGIN_USER_ID, user.getId());   // 엔티티가 아니라 id만 (7.2절)
    return UserResponse.from(user);                             // 연관 없는 단일 엔티티라 컨트롤러에서 DTO 변환 - auth만의 예외
}
```

- `GET /api/auth/me` - `@LoginOnly`. 프런트가 앱 시작 시 로그인 상태·전역 역할 확인. `@LoginUser User user` → `UserResponse.from(user)`
- `POST /api/auth/logout` - 공개 경로. 세션이 이미 없어도 조용히 성공 - 만료된 사용자가 로그아웃을 눌렀을 때 401을 보지 않게
- 세션: 서버 메모리, 12시간. 서버 재시작 시 전원 로그아웃 - P1 감수(4.5절, 7.2절)

---

## 16. 설정·CORS·시더·springdoc

- `application.yml` 전체 해설은 4.5절. Ondal 관점 요약: `local` 기본 프로필, `ddl-auto: update`(운영 전 Flyway), `open-in-view: false`, Jackson UTC, 세션 12h + `SameSite=Lax` + `HttpOnly`, Hibernate SQL 로그, local datasource = docker compose

### 16.1 `WebConfig` - 세 가지 등록 (5.6절)

- CORS: `ondal.cors.allowed-origins`(기본 `http://localhost:5173`) + `allowCredentials(true)`. 운영에서 Pages 도메인을 환경변수로 추가, 리버스 프록시로 같은 도메인에 두면 불필요
- 인터셉터 2개: 순서·경로·제외 목록(12.2절)
- `LoginUserArgumentResolver` 등록

### 16.2 `LocalDataSeeder` - local 프로필 전용 부트스트랩

- `@Component @Profile("local")` + `CommandLineRunner` → 기동 완료 후 1회 실행(3.8절)
- `admin`(ADMIN) 계정이 없으면 생성(`User.admin`) - [permissions.md](../permissions.md) 4절 "최초 관리자"의 개발 환경 버전. 운영은 수동 SQL
- 분반이 하나도 없으면 샘플 둘 생성: "2026-2 C언어"(ACTIVE: operator1 + student1~3), "2026-1 파이썬"(ARCHIVED: student1)
- 계정 전부 스텁 로그인으로 즉시 접속 가능 → FE가 역할별 화면을 바로 확인. 테스트(`test` 프로필)에서는 빈 자체가 없음(3.9절). FE mock 데이터는 이 시더와 동일하게 유지

### 16.3 springdoc - 프런트와의 계약

- 의존성 `springdoc-openapi-starter-webmvc-ui:3.1.0`. 기동 후 `/swagger-ui/index.html`에서 API 목록·DTO 설명 확인, `/v3/api-docs`는 JSON 명세
- 어노테이션: 컨트롤러 `@Tag`, 메서드 `@Operation(summary)`, 규칙이 있는 필드만 `@Schema(description)`. 최소한만 - 코드가 문서
- `/swagger-ui`·`/v3/api-docs`는 `/api/**` 밖이라 인터셉터·검증기 미적용. Swagger UI에서 로그인 API를 먼저 호출하면 이후 요청에 세션 쿠키 자동 첨부
- **프런트와의 계약 기준 = 이 화면.** API 계약 변경 PR은 본문에 `[API 변경]` 명시

---

## 17. 예외 처리 - `common/error/`

### 17.1 구성

- `ErrorResponse(String code, String message)`: 모든 에러 응답의 공통 모양. `code`는 프런트가 분기하는 기계용 문자열, `message`는 사람에게 보여줄 한국어
- 예외 6종 - 전부 `RuntimeException` 상속(롤백 규칙·선언 불필요, 6.5절)

| 예외 | HTTP | code | 메시지 출처 | 던지는 곳 |
|---|---|---|---|---|
| `UnauthenticatedException` | 401 | `UNAUTHENTICATED` | 핸들러 고정 "로그인이 필요합니다." | `AuthInterceptor`, 리졸버, `AuthorizationInterceptor` |
| `ForbiddenException` | 403 | `FORBIDDEN` | 핸들러 고정 "권한이 없습니다." | `AuthorizationInterceptor` |
| `NotFoundException(message)` | 404 | `NOT_FOUND` | 던진 쪽 문구 | 서비스 `requireCohort`, `remove` |
| `ConflictException(message)` | 409 | `CONFLICT` | 던진 쪽 문구 | `EnrollmentService.assign` |
| `CohortArchivedException` | 409 | `COHORT_ARCHIVED` | 생성자 고정 "보관된 분반은 변경할 수 없습니다..." | `Cohort.ensureActive()` |
| `InvalidInputException(message)` | 400 | `INVALID_INPUT` | 던진 쪽 문구 | 인터셉터 `readCohortId`, `UserService.findOrCreateMember` |

- 401·403 메시지가 고정인 이유: 내부 사정 비노출(7.5절). 404·409·400은 자원별로 다르므로 던진 쪽 문구

### 17.2 `GlobalExceptionHandler` - 예외 → HTTP 응답

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse("NOT_FOUND", e.getMessage()));
    }
    ...
    @ExceptionHandler(Exception.class)          // 최후 방어 - 그 외 전부 500
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception e) {
        log.error("처리되지 않은 예외", e);     // 원인은 서버 로그에만
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(new ErrorResponse("INTERNAL_ERROR", "서버 오류가 발생했습니다."));
    }
}
```

- `@RestControllerAdvice` = 모든 컨트롤러에 적용되는 `@ExceptionHandler` 모음 + 반환값을 JSON 본문으로. 예외 발생 위치와 무관하게(인터셉터·리졸버·컨트롤러·서비스) `DispatcherServlet`이 잡아 여기로(2.4절)
- 핸들러 선택: 발생한 예외 타입에 **가장 가까운(구체적인)** `@ExceptionHandler`가 선택됨 → `Exception.class` 핸들러가 있어도 `NotFoundException`은 자기 핸들러로
- 스프링·서블릿이 던지는 프레임워크 예외도 같은 모양으로 변환

| 프레임워크 예외 | 언제 | HTTP / code |
|---|---|---|
| `MethodArgumentNotValidException` | `@Valid` 실패 | 400 `INVALID_INPUT` - 첫 필드 오류 `"필드명: 메시지"` |
| `MethodArgumentTypeMismatchException` | 경로 변수·쿼리 타입 불일치(`?status=FOO`, `/api/cohorts/abc`의 `@AdminOnly` 경로) | 400 `INVALID_INPUT` |
| `HttpMessageNotReadableException` | 본문이 JSON이 아니거나 깨짐 | 400 `INVALID_INPUT` |
| `HttpRequestMethodNotSupportedException` | 그 경로에 없는 메서드 | 405 `METHOD_NOT_ALLOWED` |
| `NoResourceFoundException` | 매핑되지 않은 경로 | 404 `NOT_FOUND` "존재하지 않는 경로입니다." |
| `Exception` | 그 외 전부(`IllegalStateException` 포함 = 권한 어노테이션 누락의 500) | 500 `INTERNAL_ERROR` |

- 프런트 규칙: **홈 리다이렉트는 403만.** 401은 로그인으로(작성 내용 보존), 404·409는 안내

---

## 18. API 한 장 (#1~#12)

| # | 메서드·경로 | 권한 | 컨트롤러 | 하는 일 | 성공 | 주요 실패 |
|---|---|---|---|---|---|---|
| - | `POST /api/auth/login {loginId}` | 공개 | Auth | 스텁 로그인 | 200 `UserResponse` | 400 |
| - | `POST /api/auth/logout` | 공개 | Auth | 세션 파기 | 200 | - |
| - | `GET /api/auth/me` | `@LoginOnly` | Auth | 내 정보 | 200 `UserResponse` | 401 |
| - | `GET /api/health` | 공개 | Health | 생존 확인 | 200 `{status: UP}` | - |
| 1 | `GET /api/me/cohorts` | `@LoginOnly` | Enrollment | 내 분반 목록(보관 포함, ACTIVE 먼저). 빈 배열 = 미소속 | 200 `[CohortResponse]` | 401 |
| 2 | `GET /api/cohorts?status=ACTIVE` | `@AdminOnly` | Cohort | 분반 목록(상태별, 기본 ACTIVE, 최신순) | 200 `[CohortResponse]` | 400(status 값), 403 |
| 3 | `POST /api/cohorts` | `@AdminOnly` | Cohort | 생성 + 운영진 동시 지정(UC-A1) | 201 + `Location` + `CohortResponse` | 400, 409(운영진이 다른 역할로 소속) |
| 4 | `GET /api/cohorts/{cohortId}` | `@CohortRole(STUDENT)` | Cohort | 상세 | 200 `CohortResponse` | 400(비숫자 id), 403, 404(ADMIN) |
| 5 | `PUT /api/cohorts/{cohortId}` | `@AdminOnly` | Cohort | 이름·설명 전체 교체 | 200 | 400, 404, 409 보관 |
| 6 | `POST /api/cohorts/{cohortId}/archive` | `@AdminOnly` | Cohort | 보관(멱등) | 200 | 404 |
| 7 | `POST /api/cohorts/{cohortId}/restore` | `@AdminOnly` | Cohort | 보관 해제(멱등) | 200 | 404 |
| 8 | `GET /api/cohorts/{cohortId}/members` | `@CohortRole(OPERATOR)` | Enrollment | 명부(운영진 먼저) | 200 `[MemberResponse]` | 403(학생), 404 |
| 9 | `PUT /api/cohorts/{cohortId}/operators/{loginId}` | `@AdminOnly` | Enrollment | 운영진 지정(멱등, 승격 포함, 선등록 가능) | 200 `MemberResponse` | 400(loginId 길이), 404, 409 보관 |
| 10 | `DELETE /api/cohorts/{cohortId}/operators/{loginId}` | `@AdminOnly` | Enrollment | 운영진 해제(OPERATOR만) | 204 | 404(미소속·수강생·모르는 loginId), 409 보관 |
| 11 | `POST /api/cohorts/{cohortId}/students {loginIds}` | `@CohortRole(OPERATOR)` | Enrollment | 수강생 일괄 배정(UC-O2, 멱등, 중복 1회) | 200 `[MemberResponse]` 갱신된 명부 | 400(빈 목록·길이), 403, 404, 409(이미 운영진), 409 보관 |
| 12 | `DELETE /api/cohorts/{cohortId}/students/{loginId}` | `@CohortRole(OPERATOR)` | Enrollment | 수강생 제외(STUDENT만) | 204 | 403, 404(미소속·운영진·모르는 loginId), 409 보관 |

- 공통: 응답 시각 UTC ISO-8601, 에러 `{code, message}`(17장), 목록 페이징 없음(P1), 정렬 `createdAt desc`(`/me/cohorts`만 status 우선)
- 생성 응답의 계약 = **본문의 `id`**. `Location`은 REST 관례상 덧붙임(CORS `exposedHeaders` 없이는 브라우저에서 못 읽음 - 의존 금지)

DTO 요약:

- 요청
  - `LoginRequest{loginId(필수, 50자 이하)}`
  - `CohortCreateRequest{name(필수, 100자 이하), description(2000자 이하), operatorLoginIds[](각 필수·50자 이하, 선택)}` + `operatorLoginIdsOrEmpty()`
  - `CohortUpdateRequest{name, description}` - PUT 전체 교체
  - `StudentAssignRequest{loginIds[](비어 있으면 400, 각 필수·50자 이하)}`
  - 검증 어노테이션은 요청 DTO 필드에만(5.3절)
- 응답
  - `CohortResponse{id, name, description, status, createdAt, operators[UserSummary], studentCount|null, myRole|null, myTitle, canManage}` - 12.5절. 정적 팩토리 `of(...)`(여러 값 조합)
  - `UserSummary{id, name, title}` - 타인에게 보여도 되는 최소 정보(학생 화면의 운영진)
  - `UserResponse{id, loginId, name, globalRole}` - 본인(`/me`), 또는 운영진 이상이 보는 명부. `from(user)`
  - `MemberResponse{user: UserResponse, role, title, enrolledAt}` - `from(enrollment)`(user가 fetch join 되어 있다는 전제)
  - 규약: 엔티티 하나면 `from(entity)`, 여러 값 조합이면 `of(...)`

---

## 19. 테스트 - 69개가 도는 방식

### 19.1 기반 - `test/support/` 네 파일 (PM 담당, 슬라이스 작성자는 수정 안 함)

| 파일 | 역할 | 기초 |
|---|---|---|
| `PostgresContainerConfig` | `@TestConfiguration(proxyBeanMethods=false)` + `@Bean @ServiceConnection PostgreSQLContainer("postgres:16")`. 컨테이너는 테스트 전체에서 1번(컨텍스트 캐싱) | 8.5 |
| `ApiTestSupport` | 추상 베이스 `@SpringBootTest @AutoConfigureMockMvc @ActiveProfiles("test") @Import(PostgresContainerConfig.class)`. `MockMvc`, `ObjectMapper`, `LoginHelper`, 리포지토리 주입. `@AfterEach`에서 `DatabaseCleaner.clean()`. 공용 픽스처 `createCohort(name, operators...)`, `enrollStudent(cohortId, loginId)`(수강생 API 호출), `archiveCohort`, `restoreCohort`, JSON 유틸 `json(obj)`, `readJson(result)` | 8.3, 8.4 |
| `LoginHelper` | "이 사람으로 로그인된 세션" - `MockHttpSession`에 `LOGIN_USER_ID`를 직접 세팅. `admin()`, `member(loginId)`, `as(user)`, `memberUser`, `adminUser`. 로그인 API를 부르지 않으므로 인증 방식이 바뀌어도 그대로 | 8.4 |
| `DatabaseCleaner` | 매 테스트 후 메타모델의 모든 엔티티 테이블을 `TRUNCATE ... RESTART IDENTITY CASCADE`. 테스트 `@Transactional` 롤백 방식을 의도적으로 미채택 | 8.6 |

### 19.2 테스트 파일 네 개 (총 69개)

| 파일 | 개수 | 내용 |
|---|---|---|
| `cohort/CohortApiTest` | 22 | 분반 API #2~#7. `@Nested`로 엔드포인트별 묶음(`Create`, `GetOne`, `ListAll`, `Update`, `ArchiveRestore`), 메서드명은 한국어 시나리오. 검증 축 = **역할 × 엔드포인트 → 상태코드** |
| `enrollment/EnrollmentApiTest` | 34 | #1, #8~#12. `MyCohorts`, `Members`, `AssignOperator`, `RemoveOperator`, `AssignStudents`, `RemoveStudent` |
| `auth/AuthApiTest` | 5 | 실제 로그인/로그아웃/me/health 흐름. 세션은 있는데 사용자가 삭제되면 401 |
| `auth/authorization/AuthorizationMappingValidatorTest` | 8 | 규칙 (a)~(e) + 위치 우선 해석 + 위반 시 기동 중단을 **스프링 컨텍스트 없이** 가짜 `HandlerMethod`·`RequestMappingInfo`로 단위 테스트. 실제 컨텍스트의 위반은 모든 `@SpringBootTest`가 기동 실패로 함께 알림 |

- 대표 케이스(역할 × 엔드포인트)
  - 미로그인 → 401 / MEMBER `POST /cohorts` → 403 / ADMIN → 201 + Location + operators(중복 loginId는 1번)
  - 비소속 `GET /cohorts/{id}` → 403 / STUDENT → 200 myRole=STUDENT canManage=false **studentCount=null**, operators에 loginId 없음 / OPERATOR → 200 canManage=true studentCount 값 / 비소속 ADMIN → 200 myRole=null myTitle=해구르르 canManage=true
  - 임원이 운영진으로 소속되면 학생 화면에도 "해구르르"
  - `/api/cohorts/abc` → 400(MEMBER·ADMIN 세션 모두, `@AdminOnly` 경로도) / 없는 id: ADMIN → 404, MEMBER → 403 / **교차 분반**: 다른 반의 STUDENT·OPERATOR가 이 반 GET·members·students → 403, `/me/cohorts`에 남의 소속이 섞이지 않음
  - archive → ARCHIVED, restore → ACTIVE(멱등) / 보관 후 OPERATOR GET → 200 canManage=false / 보관 후 쓰기(PUT cohort, operators 지정·해제, students 배정·제외) → 409 COHORT_ARCHIVED, 해제 후 → 200 / PUT 수정은 **별도 요청으로 재조회**해 더티 체킹 flush 확인
  - 운영진 지정 멱등 2회 200 / STUDENT → OPERATOR 승격 / `DELETE operators`에 STUDENT loginId → 404, 소속 유지 / 마지막 운영진 해제도 허용
  - 수강생 배정: 자기 반 OPERATOR·ADMIN 200 + 갱신 명부(운영진 먼저) / 같은 반 STUDENT·다른 반 OPERATOR 403 / 이미 수강생 재배정 멱등 / 운영진 loginId 섞이면 409 CONFLICT / 빈 목록 400 / 51자 400 / 없는 분반 404
  - 수강생 제외: 204 후 명부에서 소멸 / 운영진 loginId → 404 소속 유지 / 모르는 loginId 404
  - 깨진 JSON 400 / `?status=FOO` 400 / `assign` 다른 역할 충돌은 서비스 레벨에서도 409

### 19.3 이후 슬라이스의 테스트

- `extends ApiTestSupport` 한 줄로 시작, `CohortApiTest`의 구조를 그대로 복제
- 슬라이스 고유 픽스처(`createAssignment` 등)는 그 테스트 클래스의 private 헬퍼로 - `support/`는 PM 파일
- 다음 슬라이스 필수 케이스: 분반 A의 OPERATOR가 `/api/cohorts/A/assignments/{B의 과제}` → 404

---

## 20. 다음 도메인을 만들 때 - 복제 절차

과제(Assignment) 기준 절차 - 권한 코드는 한 줄도 작성하지 않음. 2nd guide의 출발점

1. `docs/assignment/design.md` 먼저 작성 - 도메인마다 `docs/<domain>/` 폴더. 스키마는 [db/schema.md](../db/schema.md)가 원본(`due_at` 기반, 제출 타입 열 없음)
2. 엔티티 `Assignment` - `Cohort`의 규약대로(11.8절). `cohort`를 `@ManyToOne(LAZY)`로 보유. 파일은 `assignment/entity/`, 이후 계층도 `repository/`·`service/`·`controller/`·`dto/` 하위 패키지
3. 리포지토리 - 분반 스코프 조회 `findByIdAndCohortId` 필수. fetch join은 `WithXxx` + `@Query`
4. 서비스 - 클래스 `@Transactional`, 조회만 `readOnly`, `requireXxx`, 쓰기 첫 줄 `cohort.ensureActive()`, DTO 반환. 서비스 메서드는 `(Long cohortId, Long childId, ...)`로 cohortId를 첫 인자로
5. 컨트롤러 - 경로는 `/api/cohorts/{cohortId}/assignments/...`로 **반드시 `{cohortId}` 아래**. 메서드마다 어노테이션 하나 - 학생이 보는 것은 `@CohortRole(STUDENT)`, 운영진이 만드는 것은 `@CohortRole(OPERATOR)`. 자기 패키지의 서비스만 주입
6. DTO - record, 요청 검증은 필드에, 응답은 `from()`/`of()`. 학생에게 내려가는 응답에 타인 정보가 섞이지 않는지 확인
7. 테스트 - `extends ApiTestSupport`, 역할 × 엔드포인트 → 상태코드, 교차 분반 404 케이스 포함
8. 서버 기동 - 검증기가 규약 위반을 알려 주면 수정

규약 총정리:

- `/api/**` 핸들러에는 권한 어노테이션이 정확히 하나(위치 우선)
- 분반 스코프 자원은 `/api/cohorts/{cohortId}/...` 아래. 변수 이름은 `cohortId` 고정
- 하위 자원 조회는 `findByIdAndCohortId` - 다른 반 것이면 404
- 분반 스코프 쓰기는 `ensureActive()` 먼저 - 보관이면 409
- 서비스가 DTO를 반환 - 컨트롤러는 엔티티를 받지 않음
- 학생에게 타인 정보 비노출 - 명부는 운영진 이상, 인원수도 동일
- 직책 명칭은 `RoleTitle` 한 곳 - 프런트는 매핑 미보유
- Lombok 없음 - record와 정적 팩토리, 로거는 `LoggerFactory.getLogger`
- 검증 어노테이션은 요청 DTO 필드에만
- fetch join 조회는 `WithXxx` + `@Query`
- 시각은 UTC `Instant`, enum은 STRING, 하드 삭제 없음(영구 삭제는 별도 슬라이스)
- 컨트롤러 반환: 200 DTO / 201 `ResponseEntity.created` / 204 `void` + `@ResponseStatus`. 상태 전이 `POST /{id}/{동사}` 멱등, 수정 PUT, 삭제 DELETE

---

## 21. 파일 지도

```
src/main/java/kr/haedal/ondal/
├─ OndalApplication.java                    부팅 진입점 (3.6)
├─ user/
│  ├─ entity/User.java                    사람. globalRole ADMIN|MEMBER. member()/admin() 팩토리, isAdmin() (11.2)
│  ├─ entity/GlobalRole.java              ADMIN | MEMBER
│  ├─ entity/RoleTitle.java               화면 직책 명칭 해구르르/교육운영진/일반 수강생. @JsonValue (11.6)
│  ├─ repository/UserRepository.java      findByLoginId
│  ├─ service/UserService.java            findOrCreateMember(loginId) - 스텁 로그인·소속 배정 공용 (14장)
│  └─ dto/UserResponse.java, UserSummary.java   (18장)
├─ cohort/
│  ├─ entity/Cohort.java                  create/update/archive/restore/ensureActive (11.3)
│  ├─ entity/CohortStatus.java            ACTIVE | ARCHIVED
│  ├─ repository/CohortRepository.java    findAllByStatusOrderByCreatedAtDesc
│  ├─ service/CohortService.java          findAll/findOne/create/update/archive/restore (12.4)
│  ├─ service/CohortResponseAssembler.java  뷰어별 CohortResponse 조립(쿼리 1번) (12.5)
│  ├─ controller/CohortController.java    /api/cohorts (#2~#7)
│  └─ dto/CohortCreateRequest, CohortUpdateRequest, CohortResponse
├─ enrollment/
│  ├─ entity/Enrollment.java              소속. unique(cohort_id,user_id). promoteToOperator/isOperator (11.4)
│  ├─ entity/EnrollmentRole.java          OPERATOR | STUDENT. satisfies()
│  ├─ repository/EnrollmentRepository.java  findByCohortIdAndUserId, ...WithCohort, ...WithUser, ...InWithUser (11.7)
│  ├─ service/EnrollmentService.java      findMyCohorts/findMembers/assignStudents/assign/promoteToOperator/remove (14장)
│  ├─ controller/EnrollmentController.java  /api/me/cohorts, /api/cohorts/{cohortId}/members|operators|students (#1, #8~#12)
│  └─ dto/MemberResponse.java, StudentAssignRequest.java
├─ auth/                                  ※ 공통 기반이라 계층 폴더는 controller/service/dto만. 나머지는 루트
│  ├─ service/AuthService.java            인터페이스 - 실제 인증으로 교체할 지점 (15장)
│  ├─ service/StubAuthService.java        지금의 구현(검증 없음)
│  ├─ controller/AuthController.java      login/logout/me
│  ├─ dto/LoginRequest.java
│  ├─ AuthInterceptor.java                ① 로그인 여부 (12.2)
│  ├─ AuthPaths.java                      공개 경로 목록(단일 출처)
│  ├─ SessionConst.java                   세션 키
│  ├─ LoginUser.java, LoginUserArgumentResolver.java   @LoginUser 주입 (5.7, 12.3)
│  └─ authorization/                      (13장)
│     ├─ LoginOnly.java, AdminOnly.java, CohortRole.java   어노테이션 3종
│     ├─ AuthorizationAnnotations.java    유효 어노테이션 하나 고르기(위치 우선)
│     ├─ AuthorizationInterceptor.java    ②③④ 판정
│     ├─ CohortAuthorizer.java            isAllowed / canManage - 규칙의 단일 출처
│     └─ AuthorizationMappingValidator.java   기동 시 규칙 a~e 검증
└─ common/
   ├─ HealthController.java               /api/health (5.5)
   ├─ config/WebConfig.java               CORS, 인터셉터 2개, 리졸버 (5.6, 16.1)
   ├─ config/LocalDataSeeder.java         local 전용 admin + 샘플 분반 (16.2)
   └─ error/                              ErrorResponse{code,message}, 예외 6종, GlobalExceptionHandler (17장)

src/main/resources/application.yml        (4.5)
src/test/java/kr/haedal/ondal/
├─ support/ApiTestSupport, LoginHelper, DatabaseCleaner, PostgresContainerConfig   (19.1)
├─ cohort/CohortApiTest                   22개
├─ enrollment/EnrollmentApiTest           34개
└─ auth/AuthApiTest 5개, auth/authorization/AuthorizationMappingValidatorTest 8개
build.gradle, settings.gradle, docker-compose.yml   (4장)
```

---

## 22. 전체 요약 + 2부 면접 질문

### 22.1 전체 요약

- 사람(`users`)과 분반(`cohorts`), 둘을 잇는 소속(`enrollments`)에 분반 안 역할이 부착. 전역 역할은 사람에게, 분반 역할은 관계에
- `/api/**` 요청은 문지기 둘을 통과 - ① 로그인 여부, ② 핸들러에 붙은 어노테이션 하나(`@LoginOnly`/`@AdminOnly`/`@CohortRole`)를 보고 ADMIN·소속·역할 순으로 판정
- 컨트롤러는 서비스 호출만, 서비스는 트랜잭션 안에서 규칙(없으면 404, 보관이면 409, 충돌이면 409)을 적용해 DTO를 완성해 반환
- 응답은 보는 사람에 따라 상이, 학생에게는 타인 정보 비노출
- 예외는 한 곳에서 `{code, message}`로 변환
- 어노테이션 누락 시 서버 기동 실패(fail-closed)
- 다음 도메인은 이 슬라이스를 복제해 작성 - 권한 코드는 작성하지 않음

### 22.2 2부 면접 예상 질문

1. 권한이 두 층인 이유와 기각한 대안 두 가지 (11.5)
2. `Cohort`에 `List<Enrollment>`를 두지 않은 이유 (11.5, 6.4)
3. `Enrollment`가 다른 테이블의 FK 대상이 아닌 이유 (11.4)
4. 기본 생성자가 `protected`인 이유, setter가 없는 이유 (6.2, 11.8)
5. `GET /api/cohorts/1` 요청이 거치는 컴포넌트를 순서대로 (12장)
6. 인터셉터가 `{cohortId}`를 직접 파싱하는 이유 (13.4)
7. `AuthorizationAnnotations`가 "위치 우선 단일 유효"인 이유 - 사고 사례 (13.2)
8. `isAllowed`와 `canManage`가 같은 클래스에 있는 이유 (13.3)
9. 인터셉터가 보장하는 것과 하지 않는 것, `findByIdAndCohortId` 규약 (13.5)
10. 보관 분반 변경이 403이 아니라 409인 이유 (13.6)
11. 기동 검증기의 규칙 5개와 실행 시점 (13.7, 3.8)
12. `CohortResponseAssembler`를 서비스에서 분리한 이유, 쿼리가 1번인 이유 (12.5, 3.12, 6.4)
13. `studentCount`가 학생에게 `null`인 이유, `UserSummary`와 `UserResponse`의 차이 (12.5, 18장)
14. `CohortService.create`에서 운영진 지정이 실패하면 분반은 어떻게 되나 (14장, 6.5)
15. `assign`의 멱등·충돌 규칙, `LinkedHashSet`을 쓴 이유 (14장)
16. 운영진 해제 API에 수강생 loginId를 주면 404인 이유 (14장)
17. 세션에 id만 저장하는 이유, 로그인 시 세션을 새로 만드는 이유 (15.2, 7.2, 7.5)
18. `AuthService`를 인터페이스로 둔 이유 (15.1, 3.11)
19. 시더가 테스트에서 돌지 않는 원리 (16.2, 3.9)
20. Ondal 예외가 전부 `RuntimeException`인 이유 (17.1, 6.5)
21. 인터셉터에서 던진 401이 JSON으로 나가는 경로 (17.2, 2.4)
22. `DatabaseCleaner`가 롤백 대신 TRUNCATE를 쓰는 이유 (19.1, 8.6)
23. 검증기 테스트가 스프링 없이 도는 방법 (19.2)
24. 새 도메인을 추가할 때 권한 코드를 한 줄도 안 쓰는 이유 (20장)
