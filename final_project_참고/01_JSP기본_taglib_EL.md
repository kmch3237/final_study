# 01. JSP 기본 — page 디렉티브, taglib, EL, JSTL

> JSP 파일 맨 위에 들어가는 헤더 + 본문에서 데이터 표시/반복/조건 처리하는 문법.
> 모든 강사님 JSP에 공통으로 등장. 안 보고 쓸 수 있어야 하는 1순위.

---

## 1️⃣ `<%@ page %>` 디렉티브 — JSP 파일 맨 위 첫 줄

### 강사님 표준 (sp16 list.jsp 그대로)
```jsp
<%@ page contentType="text/html; charset=UTF-8"%>
```

### 변형 (`pageEncoding` 같이 쓸 때)
```jsp
<%@ page contentType="text/html; charset=UTF-8" pageEncoding="UTF-8"%>
```

| 속성 | 의미 | 빠뜨리면 |
|------|------|---------|
| `contentType="text/html; charset=UTF-8"` | 브라우저에 "이건 HTML이고 UTF-8 인코딩이다" 알림 | 한글 깨짐 |
| `pageEncoding="UTF-8"` | JSP 파일 자체의 인코딩 | 한글 깨짐 |

> 💡 **외울 거**: 무조건 첫 줄에 `<%@ page contentType="text/html; charset=UTF-8"%>` 박기.

---

## 2️⃣ `<%@ taglib %>` — JSTL 쓸 때 필수

### 강사님 표준 (sp16 list.jsp)
```jsp
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt"%>
```

| prefix | 어디 쓰나 |
|--------|----------|
| `c` (core) | `<c:forEach>`, `<c:if>`, `<c:choose>`, `<c:url>`, `<c:out>`, `<c:import>` 쓸 때 |
| `fmt` (format) | `<fmt:formatDate>`, `<fmt:formatNumber>` 등 날짜/숫자 포맷 쓸 때 |

> ⚠️ **이걸 빼먹으면** `<c:forEach>` 가 화면에 그대로 텍스트로 나오거나 빈 출력.

---

## 3️⃣ EL (Expression Language) — `${...}`

서버 데이터를 화면에 표시하는 가장 기본 문법.

### 강사님 사용 패턴 (sp16 list.jsp / article.jsp)

```jsp
<!-- 서버에서 보낸 변수 출력 -->
${dataCount}
${dto.name}
${dto.subject}
${dto.regDate}

<!-- 삼항 연산자 -->
${dataCount == 0 ? "등록된 게시물이 없습니다" : paging}

<!-- 조건 검사 -->
${schType=="all" ? "selected" : ""}

<!-- contextPath (URL의 기본 경로) -->
${pageContext.request.contextPath}

<!-- 세션 객체 -->
${sessionScope.admin}
${sessionScope.loginUser.name}

<!-- not empty (빈 값 체크) -->
${not empty prevDto}
```

### 자주 쓰는 EL 표현

| 문법 | 의미 | 예 |
|------|------|------|
| `${변수}` | 값 출력 | `${dto.name}` |
| `${A ? B : C}` | 삼항 | `${count == 0 ? "없음" : count}` |
| `${A==B}` | 같다 | `${schType=="all"}` |
| `${not empty X}` | X가 빈 값 아닌가? | `${not empty list}` |
| `${empty X}` | X가 빈 값인가? | `${empty errMsg}` |
| `${pageContext.request.contextPath}` | 컨텍스트 경로 | (필수!) |
| `${sessionScope.X}` | 세션 값 | `${sessionScope.loginUser}` |
| `${requestScope.X}` | request에 담은 값 | `${requestScope.errMsg}` |

> 💡 **contextPath 빠뜨리지 마**: 모든 링크/액션 앞에 붙여야 배포 환경에서도 동작.

---

## 4️⃣ JSTL — JSP에서 자바코드 없이 반복/조건/include

### A. `<c:forEach>` — 반복문 (★★★★★ 가장 많이 씀)

```jsp
<!-- 강사님 표준 (sp16 list.jsp) -->
<c:forEach var="dto" items="${list}" varStatus="status">
    <tr>
        <td>${dto.num}</td>
        <td><a href="${articleUrl}&num=${dto.num}">${dto.subject}</a></td>
        <td>${dto.name}</td>
        <td>${dto.regDate}</td>
        <td>${dto.hitCount}</td>
    </tr>
</c:forEach>
```

| 속성 | 의미 |
|------|------|
| `var="dto"` | 반복할 때 사용할 변수명 |
| `items="${list}"` | 반복할 리스트 (서버에서 받은 데이터) |
| `varStatus="status"` | 현재 반복 정보 (`status.index`, `status.count`, `status.first`, `status.last`) |

### B. `<c:if>` — 단순 조건 (else 없음)

```jsp
<!-- 강사님 표준 (sp16 article.jsp) -->
<c:if test="${not empty prevDto}">
    <a href="${pageContext.request.contextPath}/bbs/article?num=${prevDto.num}">
        <c:out value="${prevDto.subject}"/>
    </a>
</c:if>

<!-- mode가 update일 때만 hidden input 추가 (sp16 write.jsp) -->
<c:if test="${mode=='update'}">
    <input type="hidden" name="num" value="${dto.num}">
    <input type="hidden" name="page" value="${page}">
</c:if>
```

### C. `<c:choose>` / `<c:when>` / `<c:otherwise>` — if-else

```jsp
<!-- 강사님 표준 (sp16 EmployeeMenu.jsp) -->
<c:choose>
    <c:when test="${sessionScope.admin==null}">
        <li><a href="emplist.action" class="menu">직원 정보</a></li>
        <li><a href="logout.action" class="menu">로그 아웃</a></li>
    </c:when>
    <c:otherwise>
        <li><a href="employeelist.action" class="menu">직원 관리</a></li>
        <li><a href="logout.action" class="menu">로그 아웃</a></li>
    </c:otherwise>
</c:choose>
```

### D. `<c:url>` — URL 안전하게 만들기

```jsp
<!-- 강사님 표준 (sp16 list.jsp 코멘트) -->
<link rel="stylesheet" href="<c:url value='/resources/css/main.css'/>">
```

→ 자동으로 contextPath 앞에 붙음.

### E. `<c:out>` — HTML 이스케이프해서 안전하게 출력

```jsp
<!-- 강사님 표준 (sp16 article.jsp) -->
<c:out value="${dto.subject}"></c:out>

<!-- 한 줄 -->
<c:out value="${dto.subject}"/>
```

→ HTML 태그가 들어와도 그대로 텍스트로 표시 (XSS 방어).

### F. `<c:import>` — 다른 JSP 가져오기

```jsp
<!-- 강사님 표준 (sp14 EmployeeMenu 가져오기) -->
<c:import url="include/EmployeeMenu.jsp"></c:import>
```

→ 헤더/푸터/메뉴 공통으로 재사용할 때.

> 💡 **`<%@ include file="..." %>` vs `<c:import url="...">` 차이**: `include`는 컴파일 타임에 합침, `c:import`는 런타임에 가져옴. 보통 **`c:import` 추천** (유연).

---

## 5️⃣ 주석 종류

| 주석 | 어디서 보이나? | 언제 씀 |
|------|---------------|--------|
| `<!-- HTML 주석 -->` | 브라우저 소스 보기 OK | 임시 가림, 디버그 |
| `<%-- JSP 주석 --%>` | **브라우저 소스 보기에도 안 나옴** | 코드 설명, 민감 정보 |

```jsp
<!-- 이건 HTML 주석 — 브라우저에 노출됨 -->
<%-- 이건 JSP 주석 — 서버에서 처리, 브라우저에 안 보냄 --%>
```

> 💡 **민감한 주석은 JSP 주석으로** (sp16 list.jsp 패턴).

---

## 🎯 외워둘 미니 템플릿 (JSP 시작할 때 무조건 박기)

```jsp
<%@ page contentType="text/html; charset=UTF-8"%>
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>
<%@ taglib prefix="fmt" uri="http://java.sun.com/jsp/jstl/fmt"%>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>여기에_파일명.jsp</title>
<link rel="stylesheet" type="text/css" href="${pageContext.request.contextPath}/resources/css/main.css">
<script type="text/javascript" src="http://code.jquery.com/jquery.min.js"></script>
<script type="text/javascript">
    $(function(){
        // 여기에 jQuery 코드
    });
</script>
</head>
<body>

<!-- 여기에 본문 -->

</body>
</html>
```

→ **이 템플릿이 모든 작업의 출발점.** 외워둘 것.
