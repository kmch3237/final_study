# JSP/HTML 폼 문법 정리

> 방탈출 카페 프로젝트에서 쓰는 모든 HTML 폼 태그 레퍼런스

---

## 📋 JSP 파일 기본 구조

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>제목</title>
</head>
<body>
  <!-- 여기에 내용 -->
</body>
</html>
```

| 줄 | 뜻 |
|---|---|
| `<%@ page ... %>` | JSP 지시문. 인코딩 UTF-8 설정 (한글 깨짐 방지) |
| `<!DOCTYPE html>` | HTML5 문서임을 선언 |
| `<html>` | 문서 시작 |
| `<head>` | 메타 정보 (제목, 스타일 — 화면에 안 보임) |
| `<body>` | 실제 사용자에게 보이는 내용 |

---

## 🎯 폼 관련 태그 (★ 가장 중요)

### `<form>` — 입력 폼 컨테이너

```html
<form action="/cafe/enroll" method="post">
  <!-- input들이 여기 들어감 -->
</form>
```

| 속성 | 뜻 |
|---|---|
| `action` | 데이터 보낼 URL (컨트롤러의 @PostMapping 경로) |
| `method` | `get` (조회) / `post` (생성·수정·삭제) |

**⚠️ 함정**: `action`은 항상 `/`로 시작! 안 그럼 상대경로로 잡힘.

---

### `<input>` — 입력칸 (가장 많이 씀)

```html
<input type="text" name="loginId" value="" placeholder="아이디 입력" required>
```

| 속성 | 뜻 |
|---|---|
| `type` | 입력 종류 (아래 표 참고) |
| **`name`** | **서버에 전달될 키. VO 필드명과 똑같아야 함!** |
| `value` | 초기값 |
| `placeholder` | 회색 안내문구 (실제값 X) |
| `required` | 필수입력 (브라우저가 검증) |
| `min` / `max` | 숫자 범위 |
| `readonly` | 수정 불가 |
| `disabled` | 비활성 (전송도 안 됨) |

#### `type` 종류 (자주 쓰는 것)

| type | 용도 | 예시 |
|---|---|---|
| `text` | 일반 텍스트 | 아이디, 이름 |
| `password` | 비밀번호 (●●●로 표시) | 비밀번호 |
| `number` | 숫자만 | 가격, 인원 |
| `date` | 달력 선택 | 생년월일 |
| `time` | 시각 선택 | 영업시간 |
| `radio` | 라디오 (그룹 중 1개) | 성별 |
| `checkbox` | 체크박스 (다중 선택) | 약관 동의 |
| `email` | 이메일 형식 자동검증 | 이메일 |
| `tel` | 전화번호 | 연락처 |
| `file` | 파일 업로드 | 프로필 사진 |
| `hidden` | 화면 안 보이는 값 | 사용자ID 등 내부값 |

---

### `<input type="radio">` — 라디오 (1개만 선택)

```html
성별
<input type="radio" name="gender" value="M" id="male">
<label for="male">남</label>
<input type="radio" name="gender" value="F" id="female">
<label for="female">여</label>
```

**핵심**:
- `name`이 같아야 같은 그룹 = 1개만 선택 가능
- `value`가 서버에 전달되는 값 (M 또는 F)
- 사용자에게 보이는 글자(남/여)는 그냥 텍스트

---

### `<input type="checkbox">` — 체크박스 (다중 선택)

```html
<input type="checkbox" name="hobby" value="movie">영화
<input type="checkbox" name="hobby" value="game">게임
<input type="checkbox" name="hobby" value="travel">여행
```

→ 서버에 `hobby=movie,game` 같은 식으로 여러 값 전달.

---

### `<select>` + `<option>` — 드롭다운

```html
<select name="difficulty">
  <option value="1">★ (쉬움)</option>
  <option value="2">★★</option>
  <option value="3" selected>★★★ (보통)</option>  <!-- 기본 선택 -->
  <option value="4">★★★★</option>
  <option value="5">★★★★★ (어려움)</option>
</select>
```

**핵심**:
- `option`의 `value`가 서버 전달값
- 글자(`★★★`)는 사용자에게 보이는 것
- `selected` 속성으로 기본값 지정

---

### `<textarea>` — 여러 줄 입력

```html
<textarea name="description" rows="3" cols="40" placeholder="설명 입력"></textarea>
```

| 속성 | 뜻 |
|---|---|
| `rows` | 세로 줄 수 |
| `cols` | 가로 글자 수 |

**⚠️ 함정**: `<input>`과 달리 **닫는 태그 `</textarea>` 필수!** 안 닫으면 뒤 코드 전체가 textarea 안으로 빨려들어감.

---

### `<button>` — 버튼

```html
<button type="submit">로그인</button>
<button type="button" onclick="alert('취소')">취소</button>
<button type="reset">초기화</button>
```

| `type` | 동작 |
|---|---|
| `submit` | **폼 전송** (form의 action으로 데이터 보냄) — 기본값 |
| `button` | 그냥 버튼 (전송 안 됨, JS 처리용) |
| `reset` | 폼 초기화 |

**⚠️ 함정**: form 안의 button은 **기본값이 submit**! 
→ 취소 버튼은 반드시 `type="button"` 명시 (안 그럼 누르면 폼이 전송돼버림)

---

### `<label>` — 라벨 (접근성)

```html
<label for="loginId">아이디</label>
<input id="loginId" type="text" name="loginId">
```

- `label`을 클릭하면 연결된 `input`에 포커스 들어감
- 시각장애인 스크린리더에도 좋음 (필수는 아니지만 권장)

---

## 🏗 레이아웃 관련

### `<div>` — 블록 박스 (가장 흔한 컨테이너)

```html
<div class="form-row">
  아이디 <input type="text" name="loginId">
</div>
```

- **의미 없는** 박스 (그냥 영역 묶기용)
- 줄바꿈 됨 (블록 요소)
- CSS 적용용으로 가장 많이 씀

### `<span>` — 인라인 박스

```html
<p>가입은 <span style="color:red">필수</span>입니다</p>
```

- `div`와 똑같이 의미 없지만 **줄바꿈 안 됨** (인라인 요소)
- 글자 일부에만 색 입힐 때 등

### `<nav>` — 네비게이션 영역

```html
<nav>
  <a href="/">홈</a> |
  <a href="/member/login">로그인</a> |
  <a href="/member/enroll">회원가입</a>
</nav>
```

헤더의 메뉴 모음. 의미상 "여기는 네비게이션"이라고 알려주는 시맨틱 태그.

### 시맨틱 태그 (의미 부여)

| 태그 | 용도 |
|---|---|
| `<header>` | 상단 영역 (로고, 메뉴) |
| `<footer>` | 하단 영역 (저작권) |
| `<main>` | 핵심 본문 |
| `<section>` | 주제별 묶음 |
| `<article>` | 독립적인 글 (블로그 포스트 등) |
| `<aside>` | 사이드바 |

→ 보이는 건 `<div>`와 똑같음. 단, 검색엔진·접근성 도구가 의미 인식.

---

## 📝 텍스트/제목

| 태그 | 용도 | 예시 |
|---|---|---|
| `<h1>` ~ `<h6>` | 제목 (h1=가장 큼) | `<h1>회원가입</h1>` |
| `<p>` | 문단 | `<p>안녕하세요</p>` |
| `<br>` | 줄바꿈 (닫는 태그 X) | `한 줄<br>두 줄` |
| `<hr>` | 가로 구분선 | `<hr>` |
| `<strong>` | 굵게 (의미: 중요) | `<strong>필수</strong>` |
| `<em>` | 기울임 (의미: 강조) | `<em>새로운</em>` |
| `<a href="/url">` | 링크 | `<a href="/login">로그인</a>` |

---

## 📊 표 (테이블)

```html
<table border="1">
  <thead>
    <tr>
      <th>이름</th>
      <th>나이</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>홍길동</td>
      <td>30</td>
    </tr>
    <tr>
      <td>김철수</td>
      <td>25</td>
    </tr>
  </tbody>
</table>
```

| 태그 | 뜻 |
|---|---|
| `<table>` | 표 전체 |
| `<thead>` / `<tbody>` | 머리 / 본문 (생략 가능) |
| `<tr>` | 행 (가로줄, table row) |
| `<th>` | 헤더 셀 (table header) — 굵게 표시됨 |
| `<td>` | 데이터 셀 (table data) |

→ 리스트 페이지 만들 때 자주 씀!

---

## 🎨 스타일 적용 3가지 방법

### 1. 인라인 (제일 빠름, 비추)
```html
<div style="color:red; margin:8px">빨강</div>
```

### 2. `<style>` 태그 (head 안에) — ★ 지금 우리 방식
```html
<head>
<style>
  .form-row { margin-bottom: 8px; }
  h1 { color: blue; }
  input { padding: 4px; }
</style>
</head>
```

### 3. 외부 CSS 파일 (best, 나중에)
```html
<link rel="stylesheet" href="/css/style.css">
```

---

## ⚠️ JSP에서 자주 틀리는 5가지 (꼭 기억)

1. **`name` 속성 = VO 필드명 일치** (대소문자도!)
   - 폼: `name="nickName"` ↔ VO: `private String nickName;`
   - 어긋나면 → null 들어감 → DB 에러 (5/28에 학생이 여러 번 놓침)

2. **form의 `action`은 슬래시 `/`로 시작** 
   - O: `action="/member/enroll"` (절대경로)
   - X: `action="member/enroll"` (상대경로 — 현재 페이지 기준)

3. **button의 기본 type은 submit** 
   - 취소/뒤로 버튼은 반드시 `type="button"` 명시

4. **`<textarea>`는 닫는 태그 필수** 
   - `<input>`은 닫지 않아도 OK
   - `<textarea>`는 안 닫으면 망함

5. **인코딩 UTF-8** 
   - JSP 상단 `pageEncoding="UTF-8"` 빠뜨리면 한글 깨짐
   - `<meta charset="UTF-8">`도 같이

---

## 💡 통합 예시 (방탈출 예약 폼)

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>예약</title>
<style>
  .form-row { margin-bottom: 10px; }
  label { display: inline-block; width: 80px; }
  button { padding: 6px 16px; margin-right: 8px; }
</style>
</head>
<body>

<nav>
  <a href="/">홈</a> | <a href="/member/login">로그인</a>
</nav>
<hr>

<h1>예약하기</h1>

<form action="/reservation/enroll" method="post">

  <div class="form-row">
    <label for="name">예약자명</label>
    <input type="text" id="name" name="reserverName" required>
  </div>

  <div class="form-row">
    <label>연락처</label>
    <input type="tel" name="phone" placeholder="010-1234-5678">
  </div>

  <div class="form-row">
    <label>인원</label>
    <select name="people">
      <option value="2">2명</option>
      <option value="3">3명</option>
      <option value="4" selected>4명</option>
    </select>
  </div>

  <div class="form-row">
    <label>날짜</label>
    <input type="date" name="reserveDate">
  </div>

  <div class="form-row">
    <label>시간</label>
    <input type="time" name="reserveTime">
  </div>

  <div class="form-row">
    <label>요청사항</label>
    <textarea name="memo" rows="3" cols="40"></textarea>
  </div>

  <div class="form-row">
    <input type="checkbox" name="agree" value="Y" required>
    <label>약관에 동의합니다</label>
  </div>

  <button type="submit">예약</button>
  <button type="button">취소</button>

</form>

</body>
</html>
```

---

## 📌 패턴 카드 (자주 쓰는 input 묶음)

### 회원가입 패턴
```html
<input type="text"     name="loginId"   placeholder="아이디">
<input type="password" name="password"  placeholder="비밀번호">
<input type="text"     name="nickName"  placeholder="닉네임">
<input type="email"    name="email">
<input type="tel"      name="phone">
<input type="date"     name="birthDate">
<input type="radio"    name="gender" value="M">남
<input type="radio"    name="gender" value="F">여
```

### 로그인 패턴
```html
<input type="text"     name="loginId"  placeholder="아이디">
<input type="password" name="password" placeholder="비밀번호">
<input type="checkbox" name="remember" value="Y">로그인 유지
```

### 등록 신청 패턴 (카페/테마)
```html
<input type="text"   name="이름">
<input type="text"   name="주소" size="40">
<input type="number" name="가격">
<textarea name="설명" rows="3" cols="40"></textarea>
```
