# Day 3 학습 노트 (2026-05-30, 토)

> 방탈출카페 매칭 플랫폼 — 담당 5개 화면 중 로그인/카페/테마 폼 작업

---

## 🎯 오늘 목표
내일까지 담당 5개 화면(폼) 완성 → 오늘은 로그인/카페/테마 3개

## ✅ 완료한 것
- `loginForm.jsp` — id/pw 2개 input
- `cafe/cafeEnrollForm.jsp` — 카페 등록 신청 폼 (화면 동작 ✅)
- `theme/themeEnrollForm.jsp` — 테마 등록 폼
- `CafeController.java` — `enrollForm()` GET 메서드 추가
- `ThemeController.java` — 동일 패턴
- GitHub repo 생성 + push: https://github.com/kmch3237/finalproject.git

## ⏭ 남은 것
- `manager/appointForm.jsp` (5번째 폼, 내일)
- 맥북 환경 셋업
- 로그인 DB 연결 (Day 2 회원가입처럼)

---

## 🔥 오늘 겪은 트러블 (다시 겪지 말자)

### 트러블 1: 404 — `@Controller` 어노테이션 빠뜨림

**증상**: `http://localhost:8080/cafe/enroll` → Whitelabel Error Page

**잘못된 코드**:
```java
@RequestMapping("/cafe")           // ← @Controller 없음!
public class CafeController { ... }
```

**고친 코드**:
```java
@Controller                        // ← 이게 필수
@RequestMapping("/cafe")
public class CafeController { ... }
```

**왜 필요한가**:
- `@Controller` = "이 클래스를 Spring이 관리하는 컨트롤러 빈으로 등록"
- 없으면 Spring이 클래스 자체를 인식 안 함 → URL 매핑도 안 잡힘 → 404

**비유**: `@RequestMapping`은 가게 간판이고, `@Controller`는 사업자등록증. 등록증 없이 간판만 달면 손님이 못 들어옴.

---

### 트러블 2: 404 — 파일명 ↔ return문 불일치

**증상**: `@Controller` 추가했는데도 여전히 404

**원인**: 컨트롤러의 return 문자열과 실제 JSP 파일명이 다름

| 항목 | 값 |
|---|---|
| 컨트롤러 return | `"cafe/enrollForm"` |
| Spring이 찾는 파일 | `cafe/enrollForm.jsp` |
| 실제 파일명 | `cafe/cafeEnrollForm.jsp` ❌ |

**해결**: 둘 중 하나로 통일
```java
return "cafe/cafeEnrollForm";   // ← 파일명에 맞춤 (옵션 1)
```
또는
```
파일명을 enrollForm.jsp로 rename   // ← 컨벤션 통일 (옵션 2, member 패턴)
```

**교훈**:
- Spring은 글자 하나만 달라도 못 찾음
- **컨벤션 정해서 일관되게**: 도메인별로 폴더 만들었으니 `enrollForm.jsp` (도메인 접두 X) 권장
- 회원가입은 `member/enrollForm.jsp`이고 `member/memberEnrollForm.jsp`가 아님

---

### 트러블 3: `/` 루트 404 (이건 정상!)

**증상**: `http://localhost:8080/` → 404

**원인**: 루트 경로에 매핑된 컨트롤러가 아예 없음

**해결**: 메인 페이지 만들기 전엔 정상. 동작 확인은 `/member/enroll`로!

---

## 📚 모르고 있던 개념 정리

### 1. `@Controller` (어노테이션)
- Spring에게 "이 클래스를 컨트롤러로 인식해주세요" 라고 알리는 표식
- **없으면 다른 어노테이션(@RequestMapping, @GetMapping)도 무용지물**
- Spring Boot는 시작할 때 `@Controller` 붙은 클래스만 찾아서 URL 매핑 등록

### 2. `@RequestMapping("/cafe")` + `@GetMapping("/enroll")` 결합
```java
@Controller
@RequestMapping("/cafe")           // 클래스 레벨 = URL 접두어
public class CafeController {

    @GetMapping("/enroll")          // 메서드 레벨 = 접두어 뒤에 붙음
    public String enrollForm() { ... }
}
```
→ 최종 URL: `/cafe` + `/enroll` = **`/cafe/enroll`**

### 3. 뷰 리졸버 (Spring 자동 변환)
```
return "cafe/enrollForm"
        ↓
prefix          뷰이름           suffix
/WEB-INF/views/ + cafe/enrollForm + .jsp
        ↓
실제 파일: /WEB-INF/views/cafe/enrollForm.jsp
```
`application.properties`의 `spring.mvc.view.prefix`, `suffix` 설정 덕분.

### 4. URL ≠ 뷰이름 (계속 헷갈리는 부분)

| 구분 | 슬래시? | 예시 |
|---|---|---|
| URL (브라우저 주소) | O (`/`로 시작) | `/cafe/enroll` |
| 뷰이름 (return값) | 안 함 (도메인/파일) | `cafe/enrollForm` |
| redirect | O (`/`로 시작) | `redirect:/cafe/list` |

### 5. JSP 위치 규칙 (외우자!)
```
src/
 └ main/
    └ webapp/         ← ★ 1) webapp 먼저
       └ WEB-INF/     ← ★ 2) 그 안에 WEB-INF
          └ views/    ← ★ 3) views
             └ cafe/  ← 도메인별 폴더
                └ enrollForm.jsp
```
**헷갈리지 말 것**: webapp이 먼저, WEB-INF가 그 안에! (반대 X)

### 6. forward vs redirect

| 구분 | return 형태 | 동작 | URL 바뀜? |
|---|---|---|---|
| forward (기본) | `"cafe/enrollForm"` | 서버 안에서 JSP 띄움 | X |
| redirect | `"redirect:/cafe/list"` | 브라우저에 "다른 URL로 다시 요청" | O |

**언제 redirect?**: POST 처리 후 (가입/로그인). 새로고침 시 중복요청 방지 = **PRG 패턴 (POST → Redirect → GET)**

---

## 🎓 이번 주 키워드
- `@Controller` 빈 등록
- 뷰 리졸버 prefix/suffix
- 파일명 ↔ return 문자열 일치
- webapp/WEB-INF/views/도메인/파일.jsp
- PRG 패턴
