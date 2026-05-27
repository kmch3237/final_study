# Day 1 — Spring Boot + JSP 환경 세팅 & 첫 화면 띄우기

> 2026-05-27 · 회원 도메인 시작. "컨트롤러 → 뷰리졸버 → JSP" 흐름을 처음으로 끝까지 연결하고, 브라우저에 회원가입 폼을 띄우는 것이 오늘의 목표.

---

## 1. Spring MVC 요청 처리 흐름

```
브라우저 요청
  → DispatcherServlet        (입구 — 스프링이 만들어 준 FrontController)
  → HandlerMapping           ("이 URL 누가 처리?" → @RequestMapping 보고 찾음)
  → Controller 메서드 실행
  → return "member/enrollForm"
  → ViewResolver             (prefix + 뷰이름 + suffix 로 JSP 경로 완성)
  → /WEB-INF/views/member/enrollForm.jsp 렌더링 → 응답
```

순수 Servlet에서 직접 만들던 FrontController/분기를 **Spring이 대신 해준다**는 게 핵심.

---

## 2. 컨트롤러 작성

```java
package com.doit.app.controller;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller                       // 스테레오타입 어노테이션 (옛날 implements Controller 인터페이스 아님!)
@RequestMapping("/member")        // 이 클래스의 공통 URL 접두어
public class MemberController {

    @GetMapping("/enroll")        // GET /member/enroll
    public String enrollForm() {
        return "member/enrollForm";        // 뷰 이름 → JSP
    }

    @PostMapping("/enroll")       // POST /member/enroll
    public String enroll() {
        // TODO: 가입 처리
        return "redirect:/member/login";   // 가입 후 로그인 페이지로
    }

    @GetMapping("/login")
    public String loginForm() {
        return "member/loginForm";
    }

    @PostMapping("/login")
    public String login() {
        // TODO: 로그인 처리 + 세션
        return "redirect:/";               // 로그인 성공 → 메인(홈)
    }

    @GetMapping("/logout")
    public String logout() {
        // TODO: session.invalidate()
        return "redirect:/member/login";
    }
}
```

### 헷갈렸던 것 ①: `@Controller` 어노테이션 vs 옛날 인터페이스
| | 정체 | 사용법 |
|---|---|---|
| `org.springframework.web.servlet.mvc.Controller` | 옛날 **인터페이스** (안 씀) | `implements` |
| `org.springframework.stereotype.Controller` | 지금 쓰는 **어노테이션** | 클래스 위에 `@Controller` |

---

## 3. ⭐ URL ≠ 뷰 이름 (가장 헷갈린 포인트)

| | 값 | 정하는 곳 | 역할 |
|---|---|---|---|
| **URL** (브라우저에 치는 주소) | `/member/enroll` | `@RequestMapping` + `@GetMapping` | 어떤 메서드를 부를까 |
| **뷰 이름** (컨트롤러가 `return`) | `member/enrollForm` | `return "member/enrollForm";` | 어떤 JSP를 띄울까 |

→ 둘 다 "member/enroll"이 들어가 비슷해 보이지만 완전히 다른 것.
브라우저엔 **`/member/enroll`** 을 쳐야 함. (`/member/enrollForm` 치면 404)

---

## 4. ⭐ forward(뷰) vs redirect(경로) — 슬래시 규칙이 반대

| 종류 | 예시 | 앞 슬래시 | 이유 |
|---|---|---|---|
| **forward (뷰 이름)** | `return "member/loginForm";` | ❌ 안 붙임 | prefix(`/WEB-INF/views/`) 뒤에 이어 붙는 상대경로 |
| **redirect (경로)** | `return "redirect:/member/login";` | ✅ 붙임 | 사이트 루트(`/`)부터 시작하는 **절대경로** (클래스 @RequestMapping 무시) |

암기: **뷰는 슬래시 없이, redirect는 슬래시 있게.**

> GET/POST 기준: "새로고침했을 때 또 일어나면 안 되는 일"(가입·로그인 등)은 POST → 처리 후 `redirect`로 보냄 (PRG 패턴).

---

## 5. JSP 세팅 (Spring Boot + Gradle)

### (1) build.gradle — 의존성
```gradle
plugins {
    id 'war'                       // war 패키징
}
dependencies {
    implementation 'org.apache.tomcat.embed:tomcat-embed-jasper'   // JSP 엔진 (필수)
    implementation 'jakarta.servlet.jsp.jstl:jakarta.servlet.jsp.jstl-api'
    implementation 'org.glassfish.web:jakarta.servlet.jsp.jstl'    // JSTL
}
```
> **pom.xml이 아니라 build.gradle** — 이 프로젝트는 Maven이 아니라 Gradle로 생성됨.

### (2) application.properties — 뷰 리졸버
```properties
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp
```

### (3) JSP 파일 위치 (★ 폴더 순서 주의)
```
src/main/
  webapp/            ← 웹 애플리케이션 루트 (이름 고정)
    WEB-INF/         ← 보호구역: 브라우저가 URL로 직접 접근 불가 → 컨트롤러 거쳐야만
      views/         ← prefix 로 지정한 폴더
        member/      ← 도메인별 폴더 (뷰 이름 앞부분)
          enrollForm.jsp
```
`webapp` 안에 `WEB-INF`! (순서 헷갈려서 한참 고생함)

### enrollForm.jsp (순수 HTML 폼)
```jsp
<%@ page contentType="text/html; charset=UTF-8" %>
<!DOCTYPE html>
<html>
<head><title>회원가입</title></head>
<body>
    <h1>회원가입 폼</h1>
    <form action="/member/enroll" method="post">
        아이디 <input type="text" name="accountId"><br>
        비번  <input type="password" name="password"><br>
        닉네임 <input type="text" name="nickname"><br>
        <button type="submit">가입</button>
    </form>
</body>
</html>
```

---

## 6. 🐛 오늘 막혔던 것 & 해결 (트러블슈팅 로그)

| 증상 | 원인 | 해결 |
|---|---|---|
| `Failed to configure a DataSource: 'url' is not specified` | DB 드라이버는 있는데 접속정보 없음 | application.properties에 datasource 설정 (또는 임시로 자동설정 제외) |
| `No static resource WEB-INF/views/member/enrollForm.jsp` | JSP 파일 위치 틀림 (`WEB-INF/webapp` 순서 뒤바뀜, member 폴더 없음) | `webapp/WEB-INF/views/member/`로 정정 |
| `No static resource member/enrollForm` | 브라우저에 URL 대신 **뷰 이름**을 침 | `/member/enroll` 로 접속 |
| `NoResourceFoundException` (파일 위치 맞는데도 안 뜸) | JSP 엔진(Jasper)이 실행 앱에 안 붙음 = STS가 build.gradle 변경 미반영 | **STS → Gradle Refresh + Project Clean + 완전 재시작** |

---

## 7. 오늘의 핵심 5가지
1. `@Controller`는 **어노테이션** (옛 인터페이스 아님), `@RequestMapping`으로 공통 URL 접두어.
2. **URL ≠ 뷰 이름.** 브라우저엔 URL(`/member/enroll`)을 친다.
3. **forward 뷰 이름엔 슬래시 X, redirect 경로엔 슬래시 O**(루트 절대경로).
4. JSP는 `src/main/webapp/WEB-INF/views/<도메인>/` 에. (Gradle이면 build.gradle + tomcat-embed-jasper)
5. JSP가 안 뜨면 **Gradle Refresh + Clean + 재시작**.

---

## ▶️ 내일(Day 2) 할 것 — 회원가입을 실제 DB까지 연결
수직 슬라이스 나머지 단계:
```
VO(Member) → @Mapper 인터페이스 → mapper.xml(INSERT) → Service(@Transactional) → enroll() 바디 → 폼 연결
```
→ 가입 버튼 누르면 실제 Oracle DB에 INSERT 되는 것 확인.
