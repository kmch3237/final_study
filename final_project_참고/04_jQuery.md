# 04. jQuery — 강사님 사용 패턴 총정리

> 강사님 학원 폴더 `JqueryApp/jQueryTest01~20.html` (21개) + sp14~16 JSP 분석.
> 셀렉터/이벤트/DOM 조작/체이닝 — 노트 보면 그대로 따라 쓸 수 있게.

---

## 1️⃣ jQuery 로드 + 시작

### 강사님 표준 (모든 JSP/HTML 공통)
```html
<!-- head 안에 -->
<script type="text/javascript" src="http://code.jquery.com/jquery.min.js"></script>

<script type="text/javascript">
    $(function(){
        // 여기에 jQuery 코드
        // = "페이지 로드 끝나면 실행"
    });
</script>
```

### `$(function(){ ... })` 의 정체

```js
// 다 같은 의미 (점점 줄여 쓴 결과)
jquery(document).ready(function(){ ... });
$(document).ready(function(){ ... });
$(function(){ ... });    // ← 가장 많이 씀
```

> 💡 **외울 거**: 무조건 `$(function(){ ... })` 안에 코드 넣기. 페이지 로드 끝나야 셀렉터가 작동.

---

## 2️⃣ 셀렉터 (jQuery의 핵심)

| 셀렉터 | 의미 | 예 |
|--------|------|------|
| `$("#id")` | id로 1개 선택 | `$("#userId")` |
| `$(".class")` | class로 여러 개 선택 | `$(".btn")` |
| `$("tag")` | 태그 모두 선택 | `$("h1")`, `$("input")` |
| `$("tag.class")` | 태그 + 클래스 | `$("input.required")` |
| `$("부모 자식")` | 후손 | `$("form input")` |
| `$("부모 > 자식")` | 직계 자식 | `$("#navigation > li")` |
| `$("선택자, 선택자")` | 여러 개 OR | `$("#btn1, #btn2")` |
| `$("tag[속성=값]")` | 속성으로 | `$("input[name=kwd]")` |
| `$("tag:eq(N)")` | N번째 (0부터) | `$("td:eq(2)")` |
| `$("tag:gt(N)")` | N번 초과 | `$("td:gt(4)")` |
| `$("tag:lt(N)")` | N번 미만 | `$("td:lt(5)")` |
| `$("tag:first")` | 첫 번째 | `$("tr:first")` |
| `$("tag:last")` | 마지막 | `$("tr:last")` |
| `$("label + input")` | 형제 (바로 다음) | `$("label + input")` |
| `$(this)` | 이벤트가 발생한 자기 자신 | `$(this).addClass("active")` |

### 실제 강사님 예시 (jQueryTest20.html)
```js
$("td:eq(2)").css("color", "red");                                  // 3번째 td만
$("td:gt(4)").css("color", "yellow");                               // 5번째 이후
$("td:lt(5)").css("background-color", "orange");                    // 처음 5개
```

### 실제 강사님 예시 (jQueryTest15.html)
```js
$("label + input").css("color", "blue").val("Labeled!!!");
// label 바로 다음에 오는 input만
```

---

## 3️⃣ 이벤트 등록 (★★★★★ 가장 중요)

### A. `.click()` — 클릭

```js
// 기본 패턴
$("#btn").click(function(){
    // 실행할 코드
    alert("클릭됨!");
});

// 강사님 표준 (sp16 LoginForm.jsp)
$("#submitBtn").click(function(){
    if($("#id").val() == "" || $("#pw").val() == ""){
        $("#error").html("항목을 모두 입력해야 합니다").css("display", "inline");
        return;
    }
    $("#loginForm").submit();
});
```

### B. `.change()` — 값 변경

```js
// select 값 바뀌면 (sp15 EmployeeInsertForm.jsp)
$("#positionId").change(function(){
    ajaxRequest();    // Ajax로 새 정보 가져오기
});
```

### C. `.keyup()` / `.keydown()` — 키 입력

```js
// 강사님 패턴 (jQueryTest05.html) - 글자수 카운트
$("textarea").keyup(function(){
    let len = $(this).val().length;
    let remain = 30 - len;
    $("h1").html(remain);
    
    if (remain <= 10 && remain >= 1) {
        $("h1").css("color", "orange");
    } else if (remain <= 0) {
        $("h1").css("color", "red").html("더 이상 입력 불가!!");
        $(this).attr("disabled", "disabled");
    } else {
        $("h1").css("color", "black");
    }
});
```

### D. `.hover()` — 마우스 올림/뗌

```js
// 강사님 패턴 (jQueryTest02.html)
$("h1").hover(function(){
    // 마우스 올라올 때
    $(this).addClass("item");
}, function(){
    // 마우스 떠날 때
    $(this).removeClass("item");
});
```

### E. `.mousedown()` / `.mouseup()` — 마우스 버튼

```js
// 강사님 패턴 (jQueryTest05.html)
$(".inner").mousedown(function(){
    $(this).css("background-color", "yellow").text("누르고 있음!");
}).mouseup(function(){
    $(this).css("background-color", "red").text("버튼 뗌!");
});
```

### F. `.submit()` — 폼 제출 (이벤트 등록 또는 호출 둘 다)

```js
// 폼 제출 호출 (검증 후 보낼 때)
$("#loginForm").submit();

// 폼 제출 이벤트 가로채기
$("#loginForm").submit(function(e){
    e.preventDefault();   // 기본 제출 막기
    // 검증 후 직접 submit
});
```

### G. `.focus()` / `.blur()` — 포커스

```js
$("#userId").focus();   // 포커스 가게
$("#userId").blur();    // 포커스 잃게
```

### H. `.each()` — 반복

```js
// 강사님 패턴 (jQueryTest01.html)
$("#navigation a").each(function(){
    $(this).stop().animate({"marginTop": "-80px"}, 150);
});
```

---

## 4️⃣ DOM 조작 (값 가져오기/넣기/변경)

### A. `.val()` — input 값

```js
// 가져오기
let id = $("#userId").val();
let len = $(this).val().length;

// 넣기
$("#userId").val("");        // 비우기
$("#userId").val("kmch3237"); // 값 설정
$("label + input").val("Labeled!!!");
```

> ⚠️ **input은 `.val()`, textarea도 `.val()`, label/h1/span/div는 `.html()` 또는 `.text()`**

### B. `.html()` — HTML 내용

```js
// 가져오기
let content = $(this).html();

// 넣기 (HTML 태그도 해석)
$("#error").html("필수 입력 항목이 누락되었습니다.");
$("#result").html("<b>성공!</b>");   // <b> 태그가 굵게 적용됨

// 체이닝 (강사님 자주)
$("h1").html(remain).css("color", "red");
```

### C. `.text()` — 텍스트만 (HTML 무시)

```js
$("h1").text("<b>제목</b>");   // 화면에 "<b>제목</b>" 그대로 표시 (태그 무시)

let txt = $(".inner").text();
```

> 💡 **`.html()` vs `.text()`**: 사용자 입력값을 표시할 땐 **`.text()`** (XSS 방어). 직접 만든 HTML 표시할 땐 **`.html()`**.

### D. `.css()` — CSS 스타일

```js
// 단일 속성
$("h1").css("color", "red");
$("#error").css("display", "inline");

// 다중 속성 (객체)
$("td:eq(8)").css({"background-color": "green", "color": "white"});

// 체이닝
$("#error").html("...").css({color: "red", display: "inline"});
```

### E. `.attr()` — HTML 속성

```js
// 가져오기
let placeholder = $("#userId").attr("placeholder");

// 넣기
$(this).attr("disabled", "disabled");
$("img").attr("src", "new.jpg");

// 제거
$("textarea").removeAttr("disabled");
```

### F. `.prop()` — 속성 (boolean 계열)

```js
// 강사님 패턴 (doit5601 같은 거 - readonly 잠그기)
$("#loginId").prop("readonly", true);    // 읽기 전용 잠금
$("#loginId").prop("readonly", false);   // 풀기

// 체크박스 체크 상태
let checked = $("#admin").prop("checked");
$("#admin").prop("checked", true);
```

> 💡 **`.attr()` vs `.prop()`**: `disabled`, `readonly`, `checked` 같은 boolean 속성은 **`.prop()`** 추천. 일반 속성은 `.attr()`.

### G. `.addClass()` / `.removeClass()` — 클래스 추가/제거

```js
// 강사님 패턴 (jQueryTest02.html)
$(this).addClass("item");        // class 추가
$(this).removeClass("item");     // class 제거
$(this).toggleClass("active");   // 있으면 빼고 없으면 추가
```

### H. `.show()` / `.hide()` — 보이기/숨기기

```js
$("#error").show();    // display: inline 또는 block
$("#error").hide();    // display: none
$("#error").toggle();  // 토글
```

> 💡 **강사님은 `.css("display", "inline")` 직접 쓰는 패턴 선호**.

### I. `.append()` — 자식 끝에 추가

```js
// 강사님 패턴 (jQueryTest05.html)
$("body").append($("<div>mouseover</div>").css("color", "blue"));
$("ul").append("<li>새 항목</li>");
```

### J. `.length` — 개수/길이

```js
let count = $("li").length;          // li 개수
let len = $(this).val().length;      // 입력값 길이
```

---

## 5️⃣ 체이닝 (강사님 매우 자주!)

여러 메서드를 점(.)으로 연결해서 한 줄에 처리.

```js
// 강사님 패턴 (jQueryTest15.html, sp16 LoginForm 등)
$("label + input").css("color", "blue").val("Labeled!!!");

$("h1").css("color", "red").html("더 이상 입력 불가!!");

$("#error").html("필수 입력 항목이 누락되었습니다.")
           .css({color: "red", display: "inline"});

$(".inner").mousedown(function(){...}).mouseup(function(){...});
```

→ **연속 처리하면 한 줄에 다 묶기.**

---

## 6️⃣ `$(this)` — 핵심 키워드

이벤트가 발생한 그 엘리먼트 자신.

```js
$("h1").click(function(){
    alert($(this).html());    // 클릭한 h1의 텍스트
});

$("h1").hover(function(){
    $(this).addClass("item");
});

// keyup 이벤트가 발생한 textarea (강사님 패턴 jQueryTest05)
$("textarea").keyup(function(){
    let len = $(this).val().length;
});
```

> 💡 **외울 거**: 이벤트 안에서 자기 자신 가리킬 땐 `$(this)`.

---

## 7️⃣ 강사님 기본 패턴 BEST 5 (외워둘 것)

### 패턴 1: 클릭 → 검증 → 에러 메시지 또는 submit
```js
$("#submitBtn").click(function(){
    if ($("#id").val() == "" || $("#pw").val() == ""){
        $("#error").html("항목을 모두 입력해야 합니다")
                   .css("display", "inline");
        return;
    }
    $("#loginForm").submit();
});
```

### 패턴 2: 매번 에러 초기화 → 검증 → 통과 시 진행
```js
$("#submitBtn").click(function(){
    $("#error").css("display", "none");    // ⭐ 매번 초기화
    
    if ($("#name").val() == "") {
        $("#error").html("필수 입력 항목이 누락되었습니다.")
                   .css("display", "inline");
        return;
    }
    $("#form").submit();
});
```

### 패턴 3: $(this) 활용 (자기 자신 변경)
```js
$("h1").hover(function(){
    $(this).addClass("item");
}, function(){
    $(this).removeClass("item");
});
```

### 패턴 4: keyup 카운트
```js
$("textarea").keyup(function(){
    let remain = 30 - $(this).val().length;
    $("h1").html(remain);
    if (remain <= 0) {
        $(this).attr("disabled", "disabled");
    }
});
```

### 패턴 5: 페이지 이동
```js
$("button").click(function(){
    location.href = "${pageContext.request.contextPath}/bbs/list";
});

// 또는 onclick에 직접
// <button onclick="location.href='url';">
```

---

## 🎯 외워둘 최소 핵심 (백지 작성 가능 목표)

```js
$(function(){
    // 1. 셀렉터 + 이벤트
    $("#버튼ID").click(function(){
        
        // 2. 값 가져오기
        let 변수 = $("#입력ID").val().trim();
        
        // 3. 검증
        if (변수 === "") {
            $("#error").html("메시지").css("display", "inline");
            $("#입력ID").focus();
            return;
        }
        
        // 4. 폼 제출
        $("#폼ID").submit();
    });
});
```

→ **이 골격만 외우면 회원가입/로그인/등록/검색 다 만들 수 있음.**
