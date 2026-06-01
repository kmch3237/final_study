# 웹 + JSP + Spring Boot 핵심 개념 완전 정리
작성일: 2026-05-31  
기준: 수업 + 파이널 프로젝트 작업 내용

---

## 1. 페이지 이동 방법 총정리

> "버튼 눌렀을 때 어떻게 페이지 이동시키지?" 라는 고민이 생기면 이 표 보기

### 방법 비교표

| 방법 | 코드 예시 | GET/POST | 언제 씀 |
|------|-----------|----------|---------|
| `<a href>` | `<a href="/member/login">로그인</a>` | GET | 단순 링크 이동 |
| `location.href` | `location.href='/member/login'` | GET | JS에서 이동 |
| `location.replace` | `location.replace('/member/login')` | GET | 뒤로가기 막을 때 |
| `onclick` + `location.href` | `<button onclick="location.href='/url'">` | GET | 버튼으로 GET 이동 |
| `<form method="get">` | `<form action="/search" method="get">` | GET | 검색, 조회 |
| `<form method="post">` | `<form action="/member/enroll" method="post">` | POST | 회원가입, 등록 |
| jQuery `.submit()` | `$("#form").submit()` | form 설정 따름 | 검증 후 폼 제출 |
| `window.open` | `window.open('/popup', '_blank')` | GET | 새 창 열기 |

---

### 각 방법 상세 설명

#### ① `<a href>` — 텍스트/이미지 링크
```html
<a href="/member/login">로그인</a>
<a href="/member/list?page=1">목록</a>   <!-- GET 파라미터 같이 보낼 때 -->
```
- 클릭하면 GET 방식으로 이동
- 버튼처럼 보이게 하려면 CSS로 꾸밈
- 데이터 전송은 URL 뒤에 `?key=value` 형식으로만 가능

#### ② `location.href` — 자바스크립트로 이동
```javascript
// 그냥 이동
location.href = '/member/login';

// 조건 걸어서 이동
if (result == 'ok') {
    location.href = '/member/main';
}
```
- 뒤로가기 기록이 남음
- `onclick="location.href='/url'"` 형태로 버튼에 자주 사용

#### ③ `location.replace` — 뒤로가기 막기
```javascript
location.replace('/member/login');
```
- 로그아웃 후 → 로그인 페이지 이동할 때 뒤로가기 막으려면 이걸 씀
- 이전 페이지 기록을 덮어씌움

#### ④ `<form method="post">` + submit — 데이터 서버로 보낼 때
```html
<form action="/member/enroll" method="post" id="enrollForm">
    <input type="text" name="loginId">
    <button type="submit">확인</button>
</form>
```
- 비밀번호, 개인정보 등 **숨겨서** 보낼 때 반드시 POST
- 파일 업로드: `enctype="multipart/form-data"` 추가
- URL에 데이터 안 보임

#### ⑤ jQuery `.submit()` — 검증 후 폼 제출 패턴
```javascript
// 파이널 프로젝트에서 쓴 패턴
$("#submitBtn").click(function() {

    // 1. 빈 값 검증
    if ($("#loginId").val() == "") {
        $("#error").html("아이디를 입력하세요.");
        $("#error").css("display", "inline");
        return;  // 여기서 멈춤, submit 안 됨
    }

    // 2. 검증 통과하면 폼 제출
    $("#enrollForm").submit();
});
```
- `type="submit"` 버튼이면 클릭 시 바로 submit → 검증 불가
- `type="button"` + jQuery `.submit()` 조합으로 검증 먼저 하고 폼 제출

---

## 2. GET vs POST

| | GET | POST |
|--|-----|------|
| 데이터 위치 | URL 뒤에 노출 (`?key=value`) | HTTP body에 숨겨짐 |
| 보안 | 낮음 (누구나 볼 수 있음) | 높음 |
| 용도 | 조회, 검색, 단순 이동 | 등록, 수정, 삭제, 로그인 |
| 링크 가능 여부 | `<a href>` 로 가능 | 반드시 `<form>` 필요 |
| 북마크 | 가능 | 불가능 |

```
규칙: 
  데이터 읽는 것 → GET
  데이터 바꾸는 것 → POST
```

---

## 3. JavaScript vs jQuery — 언제 뭘 쓰나

> jQuery = JavaScript를 더 짧고 쉽게 쓰는 라이브러리

### 같은 기능 비교

| 기능 | 순수 JavaScript | jQuery |
|------|----------------|--------|
| 요소 찾기 | `document.getElementById("id")` | `$("#id")` |
| 텍스트 바꾸기 | `el.innerHTML = "내용"` | `$("#id").html("내용")` |
| 값 가져오기 | `el.value` | `$("#id").val()` |
| CSS 바꾸기 | `el.style.display = "none"` | `$("#id").css("display", "none")` |
| 클릭 이벤트 | `el.addEventListener("click", fn)` | `$("#id").click(fn)` |
| 숨기기/보이기 | `el.style.display = "none/block"` | `$("#id").hide()` / `show()` |

### 언제 jQuery 쓰나
- HTML 요소 값 꺼내기/바꾸기: `val()`, `html()`, `text()`
- CSS 동적으로 바꾸기: `.css("속성", "값")`
- 클릭, 변경 이벤트 처리: `.click()`, `.change()`
- 폼 제출: `.submit()`

### 언제 순수 JS 쓰나
- `location.href`, `location.replace` (페이지 이동)
- `history.back()` (뒤로가기)
- `alert()`, `confirm()` (알림창)
- jQuery 불러오기 전 (CDN 없을 때)

### jQuery 시작 패턴 — 반드시 이 안에 코드 작성
```javascript
// 페이지 로딩 완료 후 실행
$(function() {
    // 여기에 jQuery 코드 작성
    $("#btn").click(function() {
        // ...
    });
});
```
- `$(function(){})` = 문서 준비되면 실행
- 이 밖에 코드 쓰면 HTML 요소가 없어서 오류 날 수 있음

---

## 4. CSS 적용 방법 3가지

| 방법 | 형태 | 우선순위 |
|------|------|---------|
| 인라인 | `<div style="color:red">` | 가장 높음 |
| 임베디드 | `<style>` 태그 안에 작성 | 중간 |
| 외부 파일 | `.css` 파일 + `<link>` | 낮음 |

- 파이널 프로젝트: JSP마다 `<style>` 태그 안에 작성 (임베디드 방식)
- 실무: 공통 CSS 파일 1개 만들어서 모든 페이지에 `<link>`로 연결

---

## 5. CSS 선택자 핵심 정리

```css
*           { }   /* 전체 선택 */
div         { }   /* 태그 선택 */
.form-row   { }   /* 클래스 선택 */
#submitBtn  { }   /* ID 선택 */

.form-row input      { }   /* form-row 안의 모든 input */
.form-row > label    { }   /* form-row 바로 아래 label (직계 자식만) */

input[type="text"]   { }   /* type이 text인 input만 */
input[type="radio"]  { }   /* type이 radio인 input만 */

button:hover  { }   /* 버튼에 마우스 올렸을 때 */
button:active { }   /* 버튼 클릭 중일 때 */
```

### 자주 쓰는 패턴 — 폼에서 라디오만 따로 설정
```css
/* 모든 input에 width 적용 */
.form-row input {
    width: 100%;
    padding: 12px;
}

/* 라디오 버튼은 width: auto로 덮어씌움 */
input[type="radio"] {
    width: auto;
    padding: 0;
    border: none;
}
```
- 더 구체적인 선택자가 이긴다 (`input[type="radio"]` > `input`)

---

## 6. CSS display 속성

| 값 | 설명 | 대표 태그 |
|----|------|---------|
| `block` | 줄 전체 차지, 다음 요소는 아래로 | `div`, `p`, `h1`, `label` |
| `inline` | 자기 크기만큼만 차지, 옆에 붙음 | `span`, `a`, `strong` |
| `inline-block` | 옆에 붙으면서 width/height 지정 가능 | - |
| `flex` | 자식 요소를 가로/세로 정렬 | - |
| `none` | 화면에서 완전히 숨김 | - |

### Flexbox 핵심 (파이널 프로젝트에서 자주 씀)
```css
/* 부모에 flex 설정 */
.btn-area {
    display: flex;
    gap: 8px;              /* 자식들 간격 */
    justify-content: flex-end;  /* 오른쪽 정렬 */
    /* justify-content: center; → 가운데 */
    /* justify-content: space-between; → 양 끝 */
}

/* body 가운데 정렬 */
body {
    display: flex;
    justify-content: center;   /* 가로 가운데 */
    align-items: center;       /* 세로 가운데 */
    height: 100vh;             /* 화면 전체 높이 */
}
```

---

## 7. 포워드 vs 리다이렉트

| | 포워드 | 리다이렉트 |
|--|--------|-----------|
| URL 변화 | 안 바뀜 | 바뀜 |
| request 유지 | 유지됨 | 소멸됨 |
| 브라우저 인식 | 모름 | 알고 있음 |
| Spring Boot | `return "뷰이름"` | `return "redirect:/url"` |
| 언제 씀 | 조회 결과 보여줄 때 | 등록/수정/삭제 후 이동 |

```java
// 포워드: 데이터 보여줄 때
@GetMapping("/list")
public String list(Model model) {
    model.addAttribute("cafeList", list);
    return "cafe/cafeList";   // 포워드
}

// 리다이렉트: 처리 후 이동
@PostMapping("/enroll")
public String enroll(Cafe cafe) {
    service.insertCafe(cafe);
    return "redirect:/cafe/list";  // 리다이렉트
}
```
- **등록/수정/삭제 후에는 항상 redirect** → 새로고침 중복 등록 방지

---

## 8. Spring Boot MVC 흐름 총정리

```
브라우저 요청
    ↓
Controller (@GetMapping / @PostMapping)
    ↓ 메서드 호출
Service (비즈니스 로직)
    ↓ 메서드 호출
Mapper 인터페이스
    ↓ XML의 SQL 실행
DB (Oracle)
    ↓ 결과 반환
Controller → Model에 담아서 → JSP
    ↓
브라우저 화면 출력
```

### 파이널 프로젝트 URL 구조
```
/member/enroll   GET  → 회원가입 폼 보여주기
/member/enroll   POST → 회원가입 처리 → redirect:/member/login

/cafe/enroll     GET  → 카페등록 폼 (로그인된 모든 사용자)
/cafe/enroll     POST → 카페등록 처리

/owner/theme/regist  GET  → 테마등록 폼 (사장님만)
/owner/theme/regist  POST → 테마등록 처리
```
- URL은 @GetMapping/PostMapping 경로가 결정함 (JSP 파일명 아님)

---

## 9. 파이널 프로젝트 모델 클래스 (Lombok 패턴)

```java
// 수업에서 배운 기본 패턴
@Getter
@Setter
@NoArgsConstructor
public class Member {
    private Long userId;
    private String loginId, nickName, name, email, phone, gender, birthDate, password;
}

// DB 컬럼 타입 → Java 타입 변환 규칙
// NUMBER (큰 ID 값)  → Long
// NUMBER (작은 숫자) → int
// VARCHAR2          → String
// CHAR              → String
// DATE              → Date (java.util.Date)
```

---

## 10. 오늘 작업한 파일 목록

| 파일 | 내용 |
|------|------|
| `loginForm.jsp` | CSS 기준 틀 (회색 배경, 흰 박스, box-shadow) |
| `enrollForm.jsp` | 회원가입 폼, jQuery 검증, 생년월일 type="date" |
| `cafeEnrollForm.jsp` | 카페등록 폼, loginForm CSS 통일 |
| `themeEnrollForm.jsp` | 테마등록 폼, 파일 업로드 enctype, 라디오 |
| `ThemeController.java` | 리턴값 수정 (themeEnrollForm.jsp 경로 맞춤) |
| `Cafe.java` | DB CAFE 테이블 매핑 모델, Lombok 3개 |
| `Theme.java` | DB ROOM 테이블 매핑 모델, int 사용 |

---

> 다음 단계: CafeMapper 인터페이스 → CafeMapper.xml (INSERT SQL) → CafeService → CafeController 연결
