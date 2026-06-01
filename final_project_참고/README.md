# 📚 final_project 참고자료 (방탈출 카페 noExit)

> 학원에서 혼자 작업할 때 보고 쓸 수 있게 만든 **강사님 패턴 총정리**.
> 강사님 학원 폴더 sp14~sp16 + JqueryApp + AjaxJquery + WebApp30/36/37 의 코드를 전수 분석.
> **모든 예시는 그대로 복붙해도 동작하는 강사님 표준 코드.**

---

## 📁 파일 인덱스 (순서대로 보면 됨)

| 파일 | 다룰 내용 | 언제 봄? |
|------|----------|----------|
| **[01_JSP기본_taglib_EL.md](01_JSP기본_taglib_EL.md)** | `<%@ page %>`, `<%@ taglib %>`, JSTL(`c:forEach/c:if/c:choose/c:url/c:out/c:import`), EL `${}`, 주석 | JSP 파일 맨 위 헤더 / EL 표현식 / 반복문·조건문 |
| **[02_HTML엘리먼트.md](02_HTML엘리먼트.md)** | form/input(10가지)/button/select/textarea/table/div/span/a/h1/br/hr — 강사님 사용 패턴 + 모든 속성 | 폼 만들 때 / 표 만들 때 / 화면 구성 |
| **[03_CSS.md](03_CSS.md)** | CSS 로드 3가지, 셀렉터, 박스모델, 색, hover/active, **강사님 표준 클래스(.table/.btn/.form-control) 풀 코드** | 디자인 만질 때 / 색 바꿀 때 |
| **[04_jQuery.md](04_jQuery.md)** | 로드, `$(function(){})`, 셀렉터, 이벤트(.click/.change/.keyup/.hover/.mousedown), DOM 조작(.val/.html/.text/.css/.attr/.prop), 체이닝, each, eq/gt/lt | JavaScript 동작 추가할 때 |
| **[05_Ajax_폼검증.md](05_Ajax_폼검증.md)** | `$.ajax` 5종세트(type/url/data/success/error/beforeSend), `$.post`, `$.get`, JSON 처리, **강사님 #error 빨간 글씨 검증 패턴 풀 코드** | 중복확인 / 인증 / 비동기 요청 / 폼 검증 |
| **[06_기능별_완성코드.md](06_기능별_완성코드.md)** | 로그인폼 / 등록폼 / 목록(페이징+검색) / 상세(삭제확인) / 이메일전송 / 메뉴(include) — **noExit에 바로 차용 가능한 풀 코드** | 새 기능 만들 때 통째로 복붙 |

---

## 🎯 사용 흐름 (학원에서)

```
1. git pull origin main          → 이 폴더 받기
2. 작업할 기능 결정              → 예: 이메일 인증 추가
3. 06_기능별_완성코드.md 찾아봄  → 비슷한 패턴 있나?
4. 없으면 04 + 05 조합          → jQuery + Ajax 직접 조립
5. 디자인은 03_CSS.md에서        → .table, .btn 등 표준 클래스 차용
6. JSP 헤더는 01 참고             → taglib 빼먹지 말기
```

---

## ⚠️ 절대 주의 7가지 (강사님 강조 / 실수 자주 일어남)

1. **JSP 맨 위 `<%@ page contentType="text/html; charset=UTF-8"%>`** — 빠뜨리면 한글 깨짐
2. **JSTL 쓰면 `<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>` 필수** — 안 쓰면 `<c:if>` 가 빈 출력
3. **URL은 절대경로 + contextPath** — `${pageContext.request.contextPath}/...` 안 붙이면 배포 환경에서 깨짐
4. **CSS 위치는 `src/main/webapp/resources/css/`** — `WEB-INF` 안에 두면 안 됨 (보안상 차단)
5. **jQuery 셀렉터는 `id` 기준** — `name`은 서버 전송용, `id`는 JS용. **둘은 다름!**
6. **`<form>` 안에 `id="signUpForm"` 같이 id 지정** — jQuery `$("#signUpForm").submit()` 호출 때 필요
7. **return 값과 JSP 파일명 일치** — `return "user/enrollForm"` ↔ `enrollForm.jsp` (이름 한 글자라도 다르면 404)

---

## 📚 출처 (강사님 코드 어디서 뽑았나)

| 카테고리 | 강사님 소스 |
|---------|-----------|
| 로그인/등록 폼 검증 | `sp14/15/16 LoginForm.jsp`, `sp15 EmployeeInsertForm.jsp`, `EmployeeUpdateForm.jsp` |
| 게시판 목록/페이징/검색 | `sp16_board/list.jsp` |
| 게시판 등록/수정 | `sp16_board/write.jsp` |
| 게시판 상세/삭제 | `sp16_board/article.jsp` |
| Ajax 5종세트 | `AjaxJquery/ajaxTest01.jsp`, `postTest01.jsp`, `getTest01.jsp`, `jsonTest01.jsp` |
| Ajax + 검증 통합 | `sp15 EmployeeInsertForm.jsp` (positionId change → ajax) |
| 이메일 전송 | `WebApp36/sendMail.jsp`, `WebApp37/sendMailForm.jsp` (파일첨부 포함) |
| jQuery 셀렉터/이벤트/DOM | `JqueryApp/jQueryTest01~20.html` (21개 전수) |
| CSS 표준 | `sp16_board/resources/css/main.css`, `listStyle.css`, `writeStyle.css`, `articleStyle.css`, `menuStyle.css` |
| 메뉴/include 패턴 | `sp16_board/views/include/EmployeeMenu.jsp` |

> ⚠️ `doit5601`은 팀원이 만든 코드라 제외 (강사님 패턴 아님).

---

## 🚀 commit & push (학생 직접)

```bash
cd C:\Users\Owner\final_study
git add final_project_참고/
git commit -m "feat: final_project 참고자료 추가 (강사님 패턴 총정리)"
git push origin main
```

학원 PC에서:
```bash
git clone https://github.com/kmch3237/final_study.git
# 또는 이미 있으면
git pull origin main
```

→ `final_project_참고/` 폴더 안에 위 6개 .md 파일 다 보임. 작업하면서 옆에 띄워놓고 참조하면 됨.
