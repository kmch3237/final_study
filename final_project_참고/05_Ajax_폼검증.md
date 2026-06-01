# 05. Ajax + 폼 검증 — 강사님 표준 빨간 글씨 패턴

> Ajax 5종세트 ($.ajax, $.post, $.get, JSON 응답, 에러 처리) + **강사님 표준 #error 빨간 글씨 검증 패턴**.
> noExit 회원가입 / 로그인 / 중복확인 / 이메일 인증 등에 바로 차용 가능.

---

## 1️⃣ Ajax 핵심 — `$.ajax()` (5종세트)

### 강사님 표준 풀 코드 (AjaxJquery/ajaxTest01.jsp)

```js
$("#sendBtn").click(function(){
    
    // data 구성 (1) URL 쿼리스트링 방식
    let params = "name=" + $.trim($("#name").val())
              + "&content=" + $.trim($("#content").val());
    
    $.ajax({
        "type": "POST",                     // GET 또는 POST
        "url": "ajaxTest01ok.jsp",          // 요청 URL
        "data": params,                     // 보낼 데이터
        "success": function(args){          // 성공 시
            $("#resultDiv").html(args);
            $("#name").val("");
            $("#content").val("");
            $("#name").focus();
        },
        "beforeSend": showRequest,          // 보내기 전 검증 (true/false 리턴)
        "error": function(e){               // 에러 시
            alert(e.responseText);
            console.log(e.responseText);
        }
    });
});

// beforeSend 검증 함수
function showRequest(){
    if (!$.trim($("#name").val())) {
        alert("이름을 입력해야 합니다");
        $("#name").focus();
        return false;  // ⭐ false 반환하면 Ajax 안 보냄
    }
    if (!$.trim($("#content").val())) {
        alert("내용을 입력해야 합니다");
        $("#content").focus();
        return false;
    }
    return true;
}
```

### `$.ajax()` 옵션 5종세트

| 옵션 | 의미 | 필수? |
|------|------|------|
| `type` | "POST" 또는 "GET" | ✅ |
| `url` | 요청 보낼 서버 주소 | ✅ |
| `data` | 보낼 데이터 (객체 또는 문자열) | ⭕ |
| `success` | 성공 시 콜백 함수 (응답 받음) | ✅ |
| `error` | 에러 시 콜백 함수 | ⭕ |
| `beforeSend` | 보내기 전 검증 함수 (false 리턴 시 중단) | ⭕ |
| `dataType` | 응답 데이터 타입 ("json" 등) | ⭕ |

### data 구성 2가지 방식

```js
// 방식 1: 문자열 (URL 쿼리스트링)
let params = "name=" + $("#name").val() + "&content=" + $("#content").val();

// 방식 2: 객체 (jQuery가 알아서 변환) - 더 깔끔
let params = {
    name: $("#name").val(),
    content: $("#content").val()
};

// 둘 다 동일하게 동작
```

---

## 2️⃣ `$.post()` — `$.ajax()` 의 POST 간편 함수

### 강사님 표준 (AjaxJquery/postTest01.jsp)
```js
$("#sendBtn").click(function(){
    $.post("postTest01ok.jsp", {
        "title": $("#title").val(),
        "content": $("#content").val()
    }, function(data){
        $("#resultDiv").html(data);
    });
});
```

### 형식
```js
$.post(URL, 데이터객체, 성공콜백);
```

→ `$.ajax()` 보다 짧음. 검증/에러 처리 안 필요할 때 추천.

---

## 3️⃣ `$.get()` — POST 대신 GET

### 강사님 표준 (AjaxJquery/getTest01.jsp)
```js
$("#loadBtn").click(function(){
    let nickName = $("#nickName").val();
    
    $.get("getTest01ok.jsp", {nickName: nickName}, function(data){
        $("#resultDiv").html(data);
    });
});
```

→ 데이터 조회용 (URL에 노출되어도 OK한 경우).

---

## 4️⃣ JSON 응답 처리 — `dataType: "json"`

### 강사님 표준 (AjaxJquery/jsonTest01.jsp)
```js
$("#sendBtn").click(function(){
    let params = "name=" + $.trim($("#name").val())
              + "&content=" + $.trim($("#content").val());
    
    $.ajax({
        "type": "POST",
        "url": "jsonTest01ok.jsp",
        "data": params,
        "dataType": "json",                   // ⭐ JSON으로 받음
        "success": function(jsonObj){
            // jsonObj는 자동으로 객체 변환됨
            let num = jsonObj.num;
            let name = jsonObj.name;
            let content = jsonObj.content;
            
            let out = "";
            out += "<br>이름: " + name;
            out += "<br>내용: " + content;
            $("#result").html(out);
        },
        "beforeSend": checkInput,
        "error": function(e){
            alert(e.responseText);
        }
    });
});
```

### 서버는?
- Spring: `@PostMapping + @ResponseBody public Map<String,Object> ...` 또는 DTO 반환
- 응답 JSON: `{"num": 1, "name": "강명철", "content": "안녕"}` 형태

---

## 5️⃣ ⭐ 강사님 표준 #error 빨간 글씨 패턴 (가장 중요)

### 4개 JSP에서 같은 패턴 발견:
- `sp16 LoginForm.jsp`
- `sp15 LoginForm.jsp`
- `sp15 EmployeeInsertForm.jsp`
- `sp15 EmployeeUpdateForm.jsp`

### HTML 부분 (폼 맨 아래)
```jsp
<span id="error" style="color: red; font-size: small; display: none;"></span>
```

또는 (강조 효과):
```jsp
<span id="error" style="color: red; font-weight: bold; display: none;"></span>
```

### JS 부분 — 강사님 풀 코드 (sp16 LoginForm.jsp)
```js
$(function(){
    $("#submitBtn").click(function(){
        // 검증 실패 시
        if($("#id").val() == "" || $("#pw").val() == ""){
            $("#error").html("항목을 모두 입력해야 합니다")
                       .css("display", "inline");
            return;
        }
        $("#loginForm").submit();
    });
});
```

### 강사님 패턴 핵심 3단계

```
1. <span id="error" style="display: none;">    ← 초기 숨김
2. $("#error").html("메시지").css("display", "inline");   ← 보이게
3. $("#error").css("display", "none");          ← 매번 초기화 (선택)
```

### 매번 초기화 추가 패턴 (sp13 memberList.jsp)
```js
$("#submitBtn").click(function(){
    $("#error").css("display", "none");           // ⭐ 함수 진입 시 숨김부터
    
    if ($("#name").val() == "" || $("#telephone").val() == "") {
        $("#error").css("display", "inline");
        return;
    }
    
    $("#memberForm").submit();
});
```

---

## 6️⃣ noExit 회원가입에 적용한 풀 코드 (아이디 중복확인)

### 클라이언트 (enrollForm.jsp)

```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>회원가입</title>
<script src="https://code.jquery.com/jquery.min.js"></script>
<script type="text/javascript">

$(function(){
    let isIdChecked = false;    // 중복확인 통과 플래그
    
    // 중복확인 버튼
    $("#idChkBtn").click(function(){
        $("#error").css("display", "none");    // ⭐ 매번 초기화
        
        let loginId = $("#loginId").val().trim();
        
        if (loginId === "") {
            $("#error").html("아이디를 입력해주세요.")
                       .css({color: "red", display: "inline"});
            $("#loginId").focus();
            return;
        }
        
        $.ajax({
            type: "POST",
            url: "${pageContext.request.contextPath}/user/id-check",
            data: { loginId: loginId },
            success: function(result){
                if (result === "AVAILABLE") {
                    $("#error").html("사용 가능한 아이디입니다.")
                               .css({color: "blue", display: "inline"});
                    $("#loginId").prop("readonly", true);
                    isIdChecked = true;
                } else {
                    $("#error").html("이미 사용 중인 아이디입니다.")
                               .css({color: "red", display: "inline"});
                    $("#loginId").val("").focus();
                }
            },
            error: function(){
                $("#error").html("서버 통신 오류가 발생했습니다.")
                           .css({color: "red", display: "inline"});
            }
        });
    });
    
    // 회원가입 버튼
    $("#signUpBtn").click(function(){
        $("#error").css("display", "none");
        
        if (!isIdChecked) {
            $("#error").html("아이디 중복 확인은 필수입니다.")
                       .css({color: "red", display: "inline"});
            $("#idChkBtn").focus();
            return;
        }
        $("#signUpForm").submit();
    });
});
</script>
</head>
<body>

<h1>회원가입</h1>

<form action="${pageContext.request.contextPath}/user/enroll" method="post" id="signUpForm">
    <div>아이디 <input type="text" name="loginId" id="loginId">
              <button type="button" id="idChkBtn">중복확인</button>
        <br>
        <span id="error" style="font-size: small; display: none;"></span>
    </div>
    <div>비밀번호 <input type="password" name="password"></div>
    <div>닉네임 <input type="text" name="nickName"></div>
    <div>이름 <input type="text" name="name"></div>
    <div>이메일 <input type="text" name="email"></div>
    <div>연락처 <input type="text" name="phone"></div>
    <div>
        성별
        <input type="radio" name="gender" value="M">남
        <input type="radio" name="gender" value="F">여
    </div>
    <div>생년월일 <input type="date" name="birthDate"></div>
    
    <button type="button" id="signUpBtn">확인</button>
    <button type="button">취소</button>
</form>

</body>
</html>
```

### 서버 (UserController.java)

```java
@PostMapping("/id-check")
@ResponseBody                                       // ⭐ JSP 안 찾고 응답 본문에 그대로
public String idCheck(String loginId) {
    int count = service.countByLoginId(loginId);
    return count == 0 ? "AVAILABLE" : "DUPLICATED";
}
```

### Mapper (UserMapper.java)

```java
public int countByLoginId(String loginId);
```

### Mapper XML (userMapper.xml)

```xml
<select id="countByLoginId" parameterType="String" resultType="int">
    SELECT COUNT(*)
    FROM USER_ACCOUNT
    WHERE LOGIN_ID = #{loginId}
</select>
```

---

## 7️⃣ 응용 — 이메일 인증 패턴 (학생이 만든 findId.jsp 같은 구조)

```js
// 1단계: 인증번호 발송
$("#emailBtn").click(function(){
    $("#error").css("display", "none");
    
    let email = $("#email").val().trim();
    if (email === "") {
        $("#error").html("이메일을 입력해주세요.").css({color: "red", display: "inline"});
        return;
    }
    
    $.post("${pageContext.request.contextPath}/user/send-auth", {email: email}, 
        function(result){
            if (result === "SUCCESS") {
                $("#error").html("인증번호가 발송되었습니다.").css({color: "blue", display: "inline"});
                $("#authCodeArea").show();
            } else {
                $("#error").html("발송 실패. 다시 시도해주세요.").css({color: "red", display: "inline"});
            }
        }
    );
});

// 2단계: 인증번호 확인
$("#verifyBtn").click(function(){
    $("#error").css("display", "none");
    
    let authCode = $("#authCode").val().trim();
    if (authCode === "") {
        $("#error").html("인증번호를 입력해주세요.").css({color: "red", display: "inline"});
        return;
    }
    
    $.post("${pageContext.request.contextPath}/user/verify-auth", {authCode: authCode},
        function(result){
            if (result === "VALID") {
                $("#error").html("인증되었습니다.").css({color: "blue", display: "inline"});
                $("#isVerified").val("true");
            } else {
                $("#error").html("인증번호가 일치하지 않습니다.").css({color: "red", display: "inline"});
            }
        }
    );
});
```

---

## 🎯 외워둘 Ajax 최소 골격

```js
$.ajax({
    type: "POST",
    url: "${pageContext.request.contextPath}/내경로",
    data: { 키1: 값1, 키2: 값2 },
    success: function(result){
        if (result === "성공") {
            $("#error").html("성공!").css({color: "blue", display: "inline"});
        } else {
            $("#error").html("실패").css({color: "red", display: "inline"});
        }
    },
    error: function(){
        $("#error").html("서버 오류").css({color: "red", display: "inline"});
    }
});
```

→ **회원가입/이메일/중복확인/검증 다 이 골격에서 변형.**

---

## 🎯 검증 패턴 미니 템플릿

```js
$("#버튼").click(function(){
    $("#error").css("display", "none");        // ⭐ 1. 초기화
    
    if (조건 위반) {                              // ⭐ 2. 검증
        $("#error").html("메시지")
                   .css({color: "red", display: "inline"});
        $("#대상").focus();
        return;
    }
    
    // ⭐ 3. 통과 시 진행
    $.ajax({ ... });
    // 또는
    $("#폼").submit();
});
```
