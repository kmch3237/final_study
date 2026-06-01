# 03. CSS — 강사님 표준 패턴 + 자주 쓰는 속성

> 강사님 sp16_board의 main.css, listStyle.css, writeStyle.css, articleStyle.css, menuStyle.css 분석.
> **표준 클래스를 그대로 복붙해서 쓰면 강사님 디자인 그대로 나옴.**

---

## 1️⃣ CSS 로드 3가지 방법

### A. 외부 CSS 파일 (가장 추천)
```jsp
<!-- 강사님 표준 (sp16 list.jsp) -->
<link rel="stylesheet" type="text/css" 
      href="${pageContext.request.contextPath}/resources/css/listStyle.css">
```

### B. `<style>` 블록 (head 안)
```jsp
<style type="text/css">
    .errMsg { display: none; }
    .outer { width: 200px; height: 200px; background: orange; }
</style>
```

### C. inline style (속성에 직접)
```jsp
<span id="error" style="color: red; font-size: small; display: none;"></span>
<input type="text" style="width: 100px;">
```

| 방법 | 언제 |
|------|------|
| **외부 파일** | 여러 페이지에 공통 적용. 강사님이 가장 많이 쓰는 방식 |
| **`<style>` 블록** | 한 페이지만 적용. 짧은 스타일 |
| **inline** | 한 엘리먼트만. 동적 변경 (jQuery로) |

---

## 2️⃣ CSS 위치 ★ 중요!

```
src/main/webapp/
└── resources/        ← 여기 안에 두기!
    └── css/
        ├── main.css
        ├── listStyle.css
        └── writeStyle.css
```

> ⚠️ **`WEB-INF` 안에 두면 안 됨**: 보안상 브라우저가 직접 접근 못함. CSS는 외부 접근 가능해야 하니까 `resources/` 안에 둠. (sp16 list.jsp 주석에 강사님이 강조함)

> ⚠️ **CSS 안 적용될 때**: 브라우저 캐시 문제 99%. → `Ctrl+F5`로 강제 새로고침.

---

## 3️⃣ 셀렉터 (Selector)

```css
/* 1. 전체 */
* {
    padding: 0;
    margin: 0;
}

/* 2. 태그 */
body { font-family: "맑은 고딕"; }
a { color: #222; text-decoration: none; }

/* 3. id (jQuery $("#id") 와 같은 #) */
#error { color: red; }
#menu { display: block; }

/* 4. class (jQuery $(".class") 와 같은 .) */
.btn { background-color: #fff; }
.form-control { border: 1px solid #999; }

/* 5. 태그 + 클래스 */
textarea.form-control { height: 170px; resize: none; }

/* 6. 자식 (직계) */
.table th, .table td { padding: 5px; }
.table-list thead > tr { background: #f8f9fa; }

/* 7. 후손 */
.body-container .body-title { font-size: 16px; }

/* 8. 가상 클래스 (hover, active, focus) */
a:hover, a:active { color: #f28011; }
.btn:hover { background-color: #f8f9fa; }
input:focus { outline: none; }

/* 9. 속성 */
input[type=checkbox] { vertical-align: middle; }
input[type=text], input[type=date] { width: 97%; }
.form-control[readonly] { background-color: #f8f9fa; }

/* 10. 첫째/마지막 */
tr:first-child { border-top: 2px solid #454545; }
tr:last-child { border-bottom: none; }

/* 11. n번째 */
tr > td:first-child { width: 110px; }
tr > td:nth-child(2) { padding-left: 10px; }

/* 12. 가상 요소 (after, before) */
.clear::after {
    content: '';
    display: block;
    clear: both;
}
```

---

## 4️⃣ 자주 쓰는 속성

### 🎨 색

```css
color: red;                    /* 글자색 */
color: #f28011;                /* 16진수 (오렌지) */
color: #222;                   /* 거의 검정 */
color: rgb(255, 0, 0);         /* RGB */

background-color: #fff;        /* 배경색 (흰색) */
background-color: #f8f9fa;     /* 연한 회색 (강사님 자주 씀) */
background: orange;            /* background-color 약식 */
```

### 강사님 자주 쓰는 색
| 색 | 코드 | 용도 |
|----|------|------|
| 거의 검정 | `#222` | 본문 글자 |
| 회색 (강조 X) | `#787878` | 헤더 셀 글자 |
| 옅은 회색 | `#999` | 테두리 |
| 매우 옅은 회색 | `#f8f9fa` | 배경 / 호버 |
| 매우 옅은 회색 2 | `#dee2e6` | 가로선 |
| 강조 오렌지 | `#f28011` | 링크 호버 |
| 에러 빨강 | `red` | 에러 메시지 |

### 📦 박스 모델 (margin / padding / border / width / height)

```css
.box {
    width: 200px;
    height: 100px;
    
    padding: 10px;                /* 안쪽 여백 - 모든 방향 */
    padding: 5px 10px;            /* 위아래 5px, 좌우 10px */
    padding: 10px 20px 30px 40px; /* 위, 오른, 아래, 왼 (시계방향) */
    padding-top: 13px;
    padding-bottom: 10px;
    
    margin: 30px auto;            /* 위아래 30, 좌우 자동(가운데정렬) */
    margin-left: auto;
    margin-right: auto;
    
    border: 1px solid #999;       /* 두께 종류 색 */
    border: 2px solid red;
    border-bottom: 1px solid #c6c7c8;  /* 한쪽만 */
    border-radius: 4px;           /* 모서리 둥글게 */
}
```

### 📐 display (가장 중요!)

```css
.menu { display: block; }         /* div처럼 (자동 줄바꿈) */
.error { display: inline; }       /* span처럼 (같은 줄) */
.error { display: none; }         /* 안 보임 (자리도 안 차지) */
.error { display: inline-block; } /* inline인데 width/height 됨 */
```

> ⭐ **강사님 빨간 글씨 패턴 핵심**:
> ```css
> #error { display: none; }    /* 처음엔 숨김 */
> ```
> ```js
> $("#error").css("display", "inline");   /* 보이게 */
> $("#error").css("display", "none");     /* 다시 숨김 */
> ```

### 📝 폰트

```css
body {
    font-family: "맑은 고딕", "Malgun Gothic", NanumGothic, 나눔고딕, 돋움, sans-serif;
    font-size: 14px;
    font-weight: bold;     /* normal / bold / 100~900 */
}

* {
    font-family: 맑은 고딕;   /* 강사님 main.css */
    color: #343;
}
```

### 🎯 정렬

```css
text-align: center;        /* 가로 정렬: left / center / right */
text-align: left;
vertical-align: middle;    /* 세로 정렬: top / middle / bottom / baseline */
```

### 🔤 텍스트 효과

```css
text-decoration: none;          /* 밑줄 제거 (a 태그) */
text-decoration: underline;     /* 밑줄 */
text-transform: uppercase;      /* 대문자 변환 */
cursor: pointer;                /* 손가락 커서 */
```

### 🧩 기타

```css
border-collapse: collapse;       /* table 셀 사이 간격 없애기 */
border-spacing: 0;
box-sizing: border-box;          /* padding/border 포함해서 width 계산 */
list-style-type: none;           /* ul/ol 점 제거 */
overflow: hidden;                /* 넘치는 부분 자르기 */
float: left;                     /* 옆으로 띄우기 (메뉴 가로 배치) */
clear: both;                     /* float 해제 */
opacity: .65;                    /* 투명도 (0~1) */
resize: none;                    /* textarea 크기 조절 막기 */
white-space: nowrap;             /* 줄바꿈 막기 */
```

---

## 5️⃣ 강사님 표준 클래스 (복붙해서 쓰면 됨)

### 📦 `.body-container` — 가운데 정렬 컨테이너
```css
.body-container {
    margin: 30px auto;      /* 위아래 30px, 좌우 자동 = 가운데 정렬 */
    width: 700px;
}
```
```html
<div class="body-container">
    <!-- 본문 -->
</div>
```

### 📦 `.body-title` — 제목 영역
```css
.body-title {
    width: 100%;
    font-size: 16px;
    font-weight: bold;
    padding: 13px 0;
}
```
```html
<div class="body-title">
    <h3>|게시판</h3>
</div>
```

### 🔘 `.btn` — 표준 버튼
```css
.btn {
    color: #333;
    border: 1px solid #999;
    background-color: #fff;
    padding: 5px 10px;
    border-radius: 4px;
    font-weight: 500;
    font-size: 14px;
    font-family: "맑은 고딕";
    vertical-align: baseline;
    cursor: pointer;
}
.btn:active, .btn:focus, .btn:hover {
    background-color: #f8f9fa;
    color: #333;
}
```
```html
<button type="button" class="btn">버튼</button>
```

### 📝 `.form-control` — 표준 입력 필드
```css
.form-control {
    border: 1px solid #999;
    border-radius: 4px;
    background-color: #fff;
    padding: 5px 5px;
    font-family: "맑은 고딕";
    vertical-align: baseline;
}
.form-control[readonly] {
    background-color: #f8f9fa;
}
textarea.form-control {
    height: 170px;
    resize: none;
}
```
```html
<input type="text" name="name" class="form-control">
<textarea name="content" class="form-control"></textarea>
```

### 📋 `.form-select` — 표준 select
```css
.form-select {
    border: 1px solid #999;
    border-radius: 4px;
    background-color: #fff;
    padding: 4px 5px;
    font-family: "맑은 고딕";
}
```
```html
<select name="schType" class="form-select">
    <option value="all">전체</option>
</select>
```

### 🟦 `.table` — 표준 표
```css
.table {
    width: 100%;
    border-spacing: 0;
    border-collapse: collapse;
}
.table th, .table td {
    padding-top: 10px;
    padding-bottom: 10px;
}
.table-border thead > tr { border-bottom: 1px solid #c6c7c8; }
.table-border tbody > tr { border-bottom: 1px solid #dee2e6; }
.table-hover > tbody > tr:hover { background: #e9ecef; }
```
```html
<table class="table table-border table-hover">
    <thead><tr><th>번호</th><th>제목</th></tr></thead>
    <tbody>
        <tr><td>1</td><td>...</td></tr>
    </tbody>
</table>
```

### 📋 `.table-list` — 게시판 목록 표
```css
.table-list thead > tr:first-child {
    background: #f8f9fa;
    border-top: 2px solid #454545;
}
.table-list .num { width: 60px; color: #787878; }
.table-list .name { width: 100px; color: #787878; }
.table-list .date { width: 100px; color: #787878; }
.table-list .hit { width: 70px; color: #787878; }
.table-list .left { text-align: left; padding-left: 5px; }
```

### ✏️ `.table-form` — 등록/수정 폼 표
```css
.table-form { margin-top: 1.5rem; }
.table-form td { padding: 7px 0; }
.table-form tr:first-child { border-top: 2px solid #212529; }
.table-form tr > td:first-child {
    width: 110px;
    text-align: center;
    background: #f8f9fa;
}
.table-form input[type=text], 
.table-form input[type=date],
.table-form textarea {
    width: 97%;
}
.table-form input[type=password] { width: 50%; }
```

### 📃 `.paginate` — 페이징
```css
.page-navigation { padding: 20px 0; text-align: center; }
.paginate a {
    border: 1px solid #ccc;
    color: #000;
    padding: 3px 7px;
    margin-left: 3px;
}
.paginate a:hover { color: #6771ff; }
.paginate span {  /* 현재 페이지 */
    border: 1px solid #e28d8d;
    color: #cb3536;
    padding: 3px 7px;
}
```

---

## 6️⃣ 강사님 main.css 전체 (sp16_board) — 복붙해서 쓰면 됨

```css
@charset "UTF-8";

* { padding: 0; margin: 0; }
*, *::after, *::before { box-sizing: border-box; }

body {
    font-family: "Malgun Gothic", "맑은 고딕", NanumGothic, 나눔고딕, 돋움, sans-serif;
    font-size: 14px;
    color: #222;
}

a {
    color: #222;
    text-decoration: none;
    cursor: pointer;
}
a:active, a:hover {
    color: #f28011;
    text-decoration: underline;
}

/* button */
.btn {
    color: #333;
    border: 1px solid #999;
    background-color: #fff;
    padding: 5px 10px;
    border-radius: 4px;
    font-weight: 500;
    font-size: 14px;
    font-family: "맑은 고딕";
    vertical-align: baseline;
    cursor: pointer;
}
.btn:active, .btn:focus, .btn:hover {
    background-color: #f8f9fa;
    color: #333;
}

/* form */
.form-control {
    border: 1px solid #999;
    border-radius: 4px;
    background-color: #fff;
    padding: 5px;
    font-family: "맑은 고딕";
}
.form-control[readonly] { background-color: #f8f9fa; }
textarea.form-control { height: 170px; resize: none; }

.form-select {
    border: 1px solid #999;
    border-radius: 4px;
    background-color: #fff;
    padding: 4px 5px;
}

textarea:focus, input:focus { outline: none; }
input[type=checkbox], input[type=radio] { vertical-align: middle; }

/* table */
.table {
    width: 100%;
    border-spacing: 0;
    border-collapse: collapse;
}
.table th, .table td {
    padding-top: 10px;
    padding-bottom: 10px;
}
.table-border thead > tr { border-bottom: 1px solid #c6c7c8; }
.table-border tbody > tr { border-bottom: 1px solid #dee2e6; }
.table-hover > tbody > tr:hover { background: #e9ecef; }

/* container */
.body-container {
    margin: 30px auto;
    width: 700px;
}
.body-title {
    width: 100%;
    font-size: 16px;
    font-weight: bold;
    padding: 13px 0;
}

.mx-auto { margin-left: auto; margin-right: auto; }
.clear::after { content: ''; display: block; clear: both; }
```

→ **이 파일 하나만 만들어 두면 강사님 디자인 다 됨.**
