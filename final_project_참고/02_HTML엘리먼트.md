# 02. HTML 엘리먼트 — 강사님 사용 패턴 총정리

> 강사님 JSP에 등장하는 모든 HTML 엘리먼트 + 속성 사용 패턴.
> 폼 만들 때 / 표 만들 때 / 메뉴 만들 때 그대로 차용 가능.

---

## 📌 폼 관련 ★★★★★ (가장 많이 씀)

### A. `<form>` — 폼 컨테이너

```jsp
<!-- 강사님 표준 (sp16 LoginForm.jsp) -->
<form action="login.action" method="post" id="loginForm">
    ...
</form>

<!-- contextPath 붙인 절대 경로 (sp16 article.jsp) -->
<form action="${pageContext.request.contextPath}/bbs/list" method="post" id="boardForm">
    ...
</form>

<!-- 파일 업로드 (WebApp37 sendMailForm.jsp) -->
<form action="send.do" name="myForm" method="post" enctype="multipart/form-data">
    ...
</form>
```

| 속성 | 의미 | 언제 |
|------|------|------|
| `action="..."` | 폼 제출 시 보낼 URL | 항상 |
| `method="post"` / `"get"` | HTTP 방식 | 보통 post |
| `id="..."` | jQuery 선택자용 (`$("#loginForm").submit()`) | jQuery로 제출할 때 |
| `name="..."` | 자바스크립트 `document.폼이름` 접근용 | `document.searchForm` 같이 쓸 때 |
| `enctype="multipart/form-data"` | 파일 업로드용 | 파일 필드 있을 때 **필수** |

> ⚠️ **회원가입 같이 jQuery로 검증 후 제출하는 경우**: `<form id="signUpForm">` + `<button type="button" onclick="..."> + $("#signUpForm").submit()`. **button을 type="submit"으로 두면 검증 전에 그냥 제출됨.**

### B. `<input>` — 입력 필드 (10가지 type)

```jsp
<!-- 1. 텍스트 -->
<input type="text" id="userId" name="userId" placeholder="아이디" required="required">

<!-- 2. 비밀번호 -->
<input type="password" id="pw" name="pw" placeholder="패스워드" required="required">

<!-- 3. 이메일 (자동 형식 검증) -->
<input type="email" name="userEmail" required="required">

<!-- 4. 숫자 -->
<input type="number" name="age" min="0" max="120">

<!-- 5. 날짜 -->
<input type="date" name="birthDate">

<!-- 6. 시간 -->
<input type="time" name="openTime">

<!-- 7. 라디오 (한 그룹) -->
<input type="radio" value="0" id="lunar0" name="lunar" checked="checked">
<label for="lunar0">양력</label>
<input type="radio" value="1" id="lunar1" name="lunar">
<label for="lunar1">음력</label>

<!-- 8. 체크박스 -->
<input type="checkbox" id="admin" name="admin" value="0">
<label for="admin">admin</label>

<!-- 9. 파일 -->
<input type="file" name="fileName">

<!-- 10. 숨김 (서버로 데이터만 전송, 화면 X) -->
<input type="hidden" name="num" value="${dto.num}">
<input type="hidden" name="page" value="${page}">

<!-- 11. 버튼 (button과 거의 동일 — 강사님은 button 선호) -->
<input type="button" value="로그인" id="submitBtn" class="btn">
<input type="reset" value="취소" id="resetBtn" class="btn">
```

### input 공통 속성

| 속성 | 의미 |
|------|------|
| `type="..."` | 입력 종류 (text/password/email/number/date/time/radio/checkbox/file/hidden/button) |
| `id="..."` | jQuery 선택자용 (`$("#userId")`) |
| `name="..."` | **서버 전송 키** (`request.getParameter("userId")`) |
| `value="..."` | 초기값 또는 (radio/checkbox는) 선택될 때 보낼 값 |
| `placeholder="..."` | 입력 안내문 (회색 글씨) |
| `required="required"` | 빈 값으로 제출 못하게 (HTML5 자동 검증) |
| `readonly="readonly"` | 읽기 전용 (수정 불가) |
| `maxlength="N"` | 최대 글자수 |
| `style="width: 100px;"` | inline 스타일 |
| `class="form-control"` | CSS 클래스 |

> ⚠️ **id와 name 둘 다 쓰는 이유**: id는 JS용(`$("#userId")`), name은 서버 전송용(`request.getParameter("userId")`). 둘은 다름! 헷갈리지 마.

### C. `<button>` — 버튼 (강사님 선호)

```jsp
<!-- 일반 버튼 (JS로 처리) -->
<button type="button" class="btn" id="submitBtn">로그인</button>

<!-- 폼 제출 (검증 안 거치고 바로 제출) -->
<button type="submit">전송하기</button>

<!-- 폼 리셋 -->
<button type="reset" class="btn">다시입력</button>

<!-- onclick으로 페이지 이동 -->
<button type="button" class="btn" 
        onclick="location.href='${pageContext.request.contextPath}/bbs/list';">
    리스트
</button>

<!-- onclick으로 함수 호출 (sp16 write.jsp) -->
<button type="button" class="btn" onclick="sendOk();">등록완료</button>

<!-- mode에 따라 텍스트 다르게 (sp16 write.jsp) -->
<button type="button" class="btn" onclick="sendOk();">
    ${mode=='update' ? '수정완료' : "등록완료"}
</button>
```

| type | 의미 |
|------|------|
| `type="button"` | **그냥 버튼** (자바스크립트로 처리. 기본 동작 없음) |
| `type="submit"` | 폼 제출 (form 안에서 누르면 자동 submit) |
| `type="reset"` | 폼 리셋 |

> 💡 **강사님 패턴**: jQuery 검증 거치는 경우 무조건 `type="button"` + `onclick="함수()"` 또는 `$("#btn").click(...)`로 처리.

### D. `<select>` + `<option>` — 드롭다운

```jsp
<!-- 강사님 표준 (sp16 list.jsp) - 검색 타입 선택 + 선택 상태 유지 -->
<select name="schType" class="form-select" title="검색항목">
    <option value="all" ${schType=="all" ? "selected" : ""}>제목 + 내용</option>
    <option value="name" ${schType=="name" ? "selected" : ""}>작성자</option>
    <option value="subject" ${schType=="subject" ? "selected" : ""}>제목</option>
    <option value="content" ${schType=="content" ? "selected" : ""}>내용</option>
</select>

<!-- 강사님 표준 (sp15 EmployeeInsertForm.jsp) - DB에서 list 받아서 동적 -->
<select id="positionId" name="positionId">
    <c:forEach var="position" items="${positionList}">
        <option value="${position.positionId}">${position.positionName}</option>
    </c:forEach>
</select>

<!-- 수정 폼에서 selected 처리 (sp15 EmployeeUpdateForm.jsp) -->
<c:forEach var="position" items="${positionList}">
    <option value="${position.positionId}"
            ${employee.positionId == position.positionId ? "selected='selected'" : ""}>
        ${position.positionName}
    </option>
</c:forEach>
```

> 💡 **수정 폼에서 기존 값 selected 유지하는 패턴 — 자주 씀.**

### E. `<textarea>` — 여러 줄 입력

```jsp
<!-- 강사님 표준 (sp16 write.jsp) -->
<textarea name="content" maxlength="4000" class="form-control">${dto.content}</textarea>

<!-- 사이즈 지정 -->
<textarea name="content" rows="10" cols="50"></textarea>

<!-- 강사님 textarea CSS (writeStyle.css) -->
/* textarea.form-control { height: 170px; resize: none; } */
```

| 속성 | 의미 |
|------|------|
| `rows="N"` | 보일 줄 수 |
| `cols="N"` | 보일 글자수 (한 줄 길이) |
| `maxlength="N"` | 최대 글자수 |
| 안의 값 | `${dto.content}` 같이 초기값 (input과 다름!) |

---

## 📌 표 관련 ★★★★ (강사님 매우 자주 씀)

### F. `<table>` 구조 (강사님 표준)

```jsp
<!-- 강사님 표준 (sp16 list.jsp) -->
<table class="table table-border table-list">
    <thead>
        <tr>
            <td class="num">번호</td>
            <td class="subject">제목</td>
            <td class="name">작성자</td>
            <td class="date">작성일</td>			
            <td class="hit">조회수</td>
        </tr>
    </thead>
    <tbody>
        <c:forEach var="dto" items="${list}">
            <tr>
                <td>${dto.num}</td>
                <td class="left">
                    <a href="${articleUrl}&num=${dto.num}">${dto.subject}</a>
                </td>
                <td>${dto.name}</td>
                <td>${dto.regDate}</td>
                <td>${dto.hitCount}</td>
            </tr>
        </c:forEach>
    </tbody>
</table>
```

### 폼용 table (sp15 EmployeeInsertForm.jsp)
```jsp
<table>
    <tr>
        <th>이름</th>
        <td><input type="text" id="name" name="name" placeholder="이름"></td>
    </tr>
    <tr>
        <th>주민번호</th>
        <td>
            <input type="text" id="ssn1" name="ssn1" maxlength="6"> -
            <input type="text" id="ssn2" name="ssn2" maxlength="7">
        </td>
    </tr>
</table>
```

### table 관련 태그

| 태그 | 의미 |
|------|------|
| `<table>` | 표 전체 |
| `<thead>` | 헤더 영역 |
| `<tbody>` | 본문 영역 |
| `<tr>` | 가로 한 줄 (row) |
| `<th>` | 헤더 셀 (굵음, 가운데) |
| `<td>` | 본문 셀 |

### `<td>` 속성

| 속성 | 의미 | 예 |
|------|------|------|
| `colspan="N"` | 가로로 N칸 합침 | `<td colspan="2">` |
| `rowspan="N"` | 세로로 N칸 합침 | `<td rowspan="3">` |
| `width="N"` | 너비 (px 또는 %) | `<td width="50%">` |
| `align="center/left/right"` | 가로 정렬 | `<td align="center">` |
| `valign="top/middle/bottom"` | 세로 정렬 | `<td valign="top">` |
| `class="..."` | CSS 클래스 | `<td class="num">` |

---

## 📌 구조 관련 ★★★ (레이아웃)

### G. `<div>` / `<span>` — 컨테이너

```jsp
<!-- div: block (세로 쌓임) -->
<div class="body-container">
    <div class="body-title">
        <h3><span>|</span>게시판</h3>
    </div>
</div>

<!-- span: inline (가로 쌓임) -->
<span id="error" style="color: red; font-size: small; display: none;"></span>
```

| 태그 | 종류 | 차이 |
|------|------|------|
| `<div>` | block | 자동 줄바꿈 |
| `<span>` | inline | 같은 줄에 나란히 |

### H. `<a>` — 링크

```jsp
<!-- 강사님 표준 (sp16 list.jsp) -->
<a href="${articleUrl}&num=${dto.num}">${dto.subject}</a>

<!-- 외부 -->
<a href="https://www.google.com">구글</a>

<!-- 새 탭 -->
<a href="..." target="_blank">새 탭에서</a>

<!-- 클릭 시 JS 실행 (제자리) -->
<a href="#">클릭</a>

<!-- contextPath 붙인 절대 경로 (강사님 패턴) -->
<a href="${pageContext.request.contextPath}/bbs/article?num=${prevDto.num}">
    이전글
</a>
```

### I. `<h1>` ~ `<h6>` — 제목

```jsp
<h1>로그인</h1>      <!-- 가장 큼 -->
<h2>이메일 작성</h2>
<h3><span>|</span>게시판</h3>  <!-- sp16 list.jsp -->
```

### J. `<br>` / `<hr>` — 줄바꿈/구분선

```jsp
<br>          <!-- 줄바꿈 -->
<br><br>      <!-- 두 줄 띄움 -->
<hr>          <!-- 가로 구분선 -->
```

### K. `<ul>` / `<li>` — 리스트

```jsp
<!-- 강사님 표준 (sp16 EmployeeMenu.jsp) -->
<ul>
    <li><a href="emplist.action" class="menu">직원 정보</a></li>
    <li><a href="reglist.action" class="menu">지역 정보</a></li>
    <li><a href="logout.action" class="menu">로그 아웃</a></li>
</ul>
```

### L. `<label>` — 입력 필드 라벨

```jsp
<!-- for="..." 와 input의 id="..." 연결 -->
<label for="id">ID</label>
<input type="text" id="id" name="id">

<!-- 라벨 클릭하면 input에 포커스 가는 효과 -->
```

---

## 📌 기타 자주 쓰는 것

### M. `<link>` — CSS 로드 (head 안)

```jsp
<!-- 강사님 표준 3가지 방법 -->

<!-- 1. EL 방식 (가장 많이 씀) -->
<link rel="stylesheet" type="text/css" 
      href="${pageContext.request.contextPath}/resources/css/main.css">

<!-- 2. JSTL c:url -->
<link rel="stylesheet" type="text/css" 
      href='<c:url value="/resources/css/main.css"/>'>

<!-- 3. 더미 파비콘 (sp16 list.jsp - 404 에러 방지) -->
<link rel="icon" href="data:;base64,iVBORw0KGgo=">
```

### N. `<script>` — JS 로드 (head 또는 body 끝)

```jsp
<!-- jQuery CDN -->
<script type="text/javascript" src="http://code.jquery.com/jquery.min.js"></script>

<!-- 외부 JS 파일 -->
<script type="text/javascript" 
        src="${pageContext.request.contextPath}/resources/js/main.js"></script>

<!-- inline JS -->
<script type="text/javascript">
    $(function(){
        // jQuery 코드
    });
</script>
```

### O. `<meta>` — 메타 정보 (head 안 필수)

```jsp
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

---

## 🎯 강사님 폼 풀 템플릿 (외워둘 것 — sp15 EmployeeInsertForm 패턴)

```jsp
<form action="..." method="post" id="employeeForm">
    <table>
        <tr>
            <th>이름</th>
            <td><input type="text" id="name" name="name" placeholder="이름"></td>
        </tr>
        <tr>
            <th>전화번호</th>
            <td><input type="text" id="telephone" name="telephone" placeholder="전화번호"></td>
        </tr>
        <tr>
            <td colspan="2" align="center">
                <br><br>
                <button type="button" class="btn" id="submitBtn"
                        style="width: 40%; height: 50px;">추가</button>
                <button type="button" class="btn" 
                        onclick="location.href='list.action'">취소</button>
                <br><br>
                <span id="error" 
                      style="color: red; font-weight: bold; display: none;"></span>
            </td>
        </tr>
    </table>
</form>
```

→ **이 구조가 강사님 폼 90%의 골격.** 변형해서 쓰면 됨.
