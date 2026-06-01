# 📚 페이징 처리 완전 정복 (PaginateUtil + NoticeController)

> 강사님이 실제 강의에서 사용한 **최종 코드 두 파일**을 기준으로,  
> 이 파일 하나만 보고 처음부터 끝까지 직접 작성할 수 있도록 만든 학습 문서.

---

## 📑 목차

1. [학습 개요](#1-학습-개요)
2. [한눈에 보는 두 파일의 역할](#2-한눈에-보는-두-파일의-역할)
3. [PaginateUtil.java 분석](#3-paginateutiljava-분석)
4. [NoticeController.java 분석](#4-noticecontrollerjava-분석)
5. [전체 흐름 시뮬레이션](#5-전체-흐름-시뮬레이션)
6. [JSP 출력 가이드](#6-jsp-출력-가이드)
7. [외우기용 핵심 패턴](#7-외우기용-핵심-패턴)
8. [자가 점검 체크리스트](#8-자가-점검-체크리스트)

---

## 1. 학습 개요

### 1-1. 왜 이걸 외워야 하나?

> "다음에 배우는 타임리프 이런 건 개념만 알고 있으면 되지만,  
>  이건(JSP) 파이널 프로젝트에 적용하는 거라서 혼자 안 보고 짤 줄 알아야 한다." — 강사님

페이징 처리는 **모든 게시판형 기능의 기본**입니다. Notice뿐만 아니라 Member, Order, Product, 어떤 도메인이든 거의 같은 패턴으로 들어갑니다. **한 번 완전히 외우면 평생 써먹습니다.**

### 1-2. 코드 구조 한눈에 보기

```
NoticeController          <-- 사용자 요청 받음
    │
    ├─ 1. 검색 조건 받기
    ├─ 2. service.dataCount(map) → 총 개수
    ├─ 3. paginateUtil.pageCount() → 전체 페이지 수
    ├─ 4. currentPage 보정
    ├─ 5. offset 계산
    ├─ 6. service.listNotice(map) → 목록 조회
    ├─ 7. paginateUtil.paging() → HTML 생성
    └─ 8. Model에 담아 JSP로 전달

PaginateUtil              <-- 페이징 계산 + HTML 생성
    ├─ pageCount(dataCount, size) → 전체 페이지 수
    ├─ paging(currentPage, totalPage, listUrl) → HTML
    └─ createLinkUrl(url, page, label, title) → <a> 태그 1개
```

---

## 2. 한눈에 보는 두 파일의 역할

### 2-1. PaginateUtil.java (110줄)
**역할**: 페이징 계산과 HTML 생성을 담당하는 유틸리티.

| 메서드 | 역할 |
|---|---|
| `pageCount()` | 총 데이터 수 + 페이지당 크기 → 전체 페이지 수 |
| `paging()` | 현재 페이지 + 전체 페이지 + URL → 페이지네이션 HTML |
| `createLinkUrl()` | 링크 한 개를 만드는 헬퍼 (`<a>` 태그) |

### 2-2. NoticeController.java (119줄)
**역할**: 사용자 요청을 받아서 서비스/유틸을 조립하고 JSP로 전달.

| 단계 | 내용 |
|---|---|
| 입력 받기 | `@RequestParam`으로 page, schType, kwd |
| 데이터 개수 조회 | `service.dataCount(map)` |
| 페이지 계산 | `paginateUtil.pageCount()` |
| **공지 분리** | 1페이지일 때만 상단 공지 따로 조회 |
| OFFSET 계산 | `(currentPage - 1) * size` |
| 게시물 조회 | `service.listNotice(map)` |
| URL 조립 | contextPath + query string |
| 페이징 HTML | `paginateUtil.paging()` |
| Model 전달 | JSP에서 쓸 모든 데이터 |

---

## 3. PaginateUtil.java 분석

### 3-1. 패키지 & 어노테이션

```java
package com.doit.app.common;

import org.springframework.stereotype.Service;

@Service
public class PaginateUtil {
```

- **`com.doit.app.common`**: 유틸은 도메인별 패키지가 아닌 공용 패키지에 둠
- **`@Service`**: Spring Bean으로 등록 → 다른 클래스에서 DI로 받아쓸 수 있게

### 3-2. `pageCount()` — 전체 페이지 수 계산

```java
public int pageCount(int dataCount, int size) {
    if (dataCount <= 0 || size <= 0)
        return 0;
    return dataCount / size + (dataCount % size > 0 ? 1 : 0);
}
```

#### 🔑 핵심 공식: 올림 나눗셈

```
페이지 수 = (몫) + (나머지가 있으면 1, 없으면 0)
```

| dataCount | size | 몫 | 나머지 | 결과 |
|---:|---:|---:|---:|---:|
| 100 | 10 | 10 | 0 | **10** |
| 95 | 10 | 9 | 5 | **10** |
| 5 | 10 | 0 | 5 | **1** |
| 101 | 10 | 10 | 1 | **11** |

#### 🔑 Guard Clause (방어 코드)

`size = 0`이면 `dataCount / 0` → **ArithmeticException** 터집니다. 그래서 0 또는 음수를 먼저 차단.

### 3-3. `paging()` — HTML 생성 (메서드 핵심)

#### 단계 1: 입력값 검증 및 보정 (⭐ 강사님 방식)

```java
if (currentPage > totalPage) {
    currentPage = totalPage;
}

if (currentPage < 1 || totalPage < 1) {
    return "";
}
```

#### 🔑 검증 순서가 중요한 이유

**clamp(보정)을 먼저, 거부를 나중에** — 이 순서가 핵심입니다.

```java
// 예: currentPage=999, totalPage=10
// 1. currentPage > totalPage? Yes → currentPage = 10
// 2. currentPage < 1? No && totalPage < 1? No → 통과
// → 마지막 페이지로 자동 이동! 사용자 친화적

// 예: currentPage=5, totalPage=0
// 1. 5 > 0? Yes → currentPage = 0
// 2. 0 < 1? Yes → return "" (페이징 안 보임)
// → 데이터 없을 때 빈 문자열
```

만약 거부를 먼저 하면 `?page=999` 같은 URL 조작 시 페이징이 사라져버립니다. **사용자 친화적인 처리.**

#### 단계 2: 기본 변수 + URL 처리

```java
int blockSize = 10;
StringBuilder sb = new StringBuilder();

String connector = listUrl.contains("?") ? "&" : "?";
String fullUrl = listUrl + connector;
```

#### 🔑 `connector` 변수 분리

```java
// 한 줄로도 쓸 수 있지만
String fullUrl = listUrl + (listUrl.contains("?") ? "&" : "?");

// 강사님은 두 단계로 분리
String connector = listUrl.contains("?") ? "&" : "?";
String fullUrl = listUrl + connector;
```

**`connector`라는 이름이 "이게 `?` 또는 `&` 연결자다"라는 의도를 명시**합니다. 의도를 변수명으로 표현하는 것이 좋은 코드의 핵심.

#### 🔑 `StringBuilder` vs `StringBuffer`

| 특징 | StringBuffer | StringBuilder |
|---|---|---|
| 등장 | Java 1.0 | Java 5 |
| 스레드 안전 | ✅ (느림) | ❌ (빠름) |
| 사용처 | 멀티스레드 공유 | 메서드 내 지역변수 |

메서드 안에서 만든 객체는 다른 스레드가 접근할 일이 없으므로 **항상 StringBuilder**가 정답.

#### 단계 3: ⭐⭐⭐ 블록 계산 (가장 중요)

```java
int currentBlock = (currentPage - 1) / blockSize;
int startPage = (currentBlock * blockSize) + 1;
int endPage = Math.min(startPage + blockSize - 1, totalPage);
```

#### 🔑 블록(Block)이란?

페이지 10개씩 묶은 단위.

```
블록 0: [1][2][3][4][5][6][7][8][9][10]
블록 1: [11][12][13][14][15][16][17][18][19][20]
블록 2: [21][22][23][24][25][26][27][28][29][30]
```

- 현재 페이지가 7이면 블록 0 표시
- 현재 페이지가 15면 블록 1 표시
- `<`, `>` 버튼은 **블록 단위로 이동**

#### 🔑 `currentBlock = (currentPage - 1) / blockSize` — `-1` 트릭

**왜 1을 먼저 빼는가?** 10, 20, 30 같은 경계값 처리 때문.

| currentPage | currentPage / 10 | (currentPage - 1) / 10 |
|---:|:---:|:---:|
| 1 | 0 ✅ | 0 ✅ |
| 9 | 0 ✅ | 0 ✅ |
| 10 | **1** ❌ | 0 ✅ |
| 11 | 1 ✅ | 1 ✅ |
| 20 | **2** ❌ | 1 ✅ |
| 21 | 2 ✅ | 2 ✅ |

**1을 먼저 빼면 경계값(10, 20, 30…)이 자연스럽게 이전 블록에 속하게 됩니다.**

페이지는 1-기반(1부터 시작)이지만 산술은 0-기반이라서, `-1`로 0-기반으로 바꿨다가 마지막에 `+1`로 복원.

#### 🔑 startPage 공식

```java
int startPage = (currentBlock * blockSize) + 1;
```

| currentBlock | × blockSize | + 1 | startPage |
|---:|---:|---:|---:|
| 0 | 0 | 1 | **1** |
| 1 | 10 | 1 | **11** |
| 9 | 90 | 1 | **91** |

#### 🔑 endPage with `Math.min`

```java
int endPage = Math.min(startPage + blockSize - 1, totalPage);
```

`Math.min(a, b)`는 **둘 중 작은 값**을 반환.

| startPage | + blockSize - 1 | totalPage | endPage |
|---:|---:|---:|---:|
| 1 | 10 | 50 | 10 |
| 11 | 20 | 50 | 20 |
| 91 | 100 | 95 | **95** (보정됨) |

**왜 `blockSize - 1`?** 한 블록에 페이지 10개. 시작이 11이면 끝은 20.

```
11, 12, ..., 20   ← 10개
└─ startPage    └─ endPage = startPage + 10 - 1
```

"포함 시작점 + 개수 - 1 = 포함 끝점" 공식.

#### 단계 4: div 시작

```java
sb.append("<div class='paginate'>");
```

#### 🔑 작은따옴표 vs 큰따옴표

```java
"<div class='paginate'>"     // 작은따옴표는 escape 없이 사용 가능
"<div class=\"paginate\">"   // 더블쿼트는 escape 필요
```

#### 단계 5: 처음과 이전 페이지

```java
if (currentBlock > 0) {
    int prevBlockPage = startPage - 1;
    sb.append(createLinkUrl(fullUrl, 1, "&#x226A;", "처음"));
    sb.append(createLinkUrl(fullUrl, prevBlockPage, "&#x003C;", "이전"));
}
```

#### 🔑 조건 `currentBlock > 0`

**"첫 블록이 아니면 표시"** — 매우 직관적.

#### 🔑 prevBlockPage = startPage - 1

이전 블록의 **마지막 페이지**로 이동.
- 현재 블록 21~30 → prevBlockPage = 20 (블록 11~20의 끝)

#### 🔑 HTML 엔티티 끝의 세미콜론 ⭐

```java
"&#x226A;"   // ← 끝에 세미콜론 (HTML 엔티티 정석)
"&#x226A"    // ← 세미콜론 없어도 대부분 브라우저는 인식하지만 비표준
```

**HTML 엔티티는 `&...;` 형태가 표준**입니다. 강사님 코드처럼 세미콜론을 붙이는 게 정석.

| 엔티티 | 문자 | 의미 |
|---|---|---|
| `&#x226A;` | « | 처음 |
| `&#x003C;` | < | 이전 |
| `&#x003E;` | > | 다음 |
| `&#x226B;` | » | 마지막 |

#### 단계 6: 페이지 번호 반복 (for문)

```java
for (int i = startPage; i <= endPage; i++) {
    if (i == currentPage) {
        sb.append("<span class='active' aria-current='page'>").append(i).append("</span>");
    } else {
        sb.append(createLinkUrl(fullUrl, i, String.valueOf(i), String.valueOf(i)));
    }
}
```

#### 🔑 for문의 장점

```java
for (int i = startPage; i <= endPage; i++)
//   ^초기화           ^조건       ^증가
```

while문은 초기화/조건/증가가 흩어져 있는데, for문은 한 줄에 모입니다. `i`도 for문 안에서만 살아있어서 스코프가 깨끗.

#### 🔑 현재 페이지 강조: `<span class='active' aria-current='page'>`

두 가지 속성이 둘 다 있는 이유:

| 속성 | 용도 |
|---|---|
| `class='active'` | **시각 사용자용** (CSS로 색상/굵기 강조) |
| `aria-current='page'` | **스크린 리더용** (시각장애인에게 "현재 페이지" 알림) |

**역할이 완전히 다릅니다.** 둘 다 있어야 모든 사용자에게 현재 페이지를 알릴 수 있습니다.

#### 🔑 `aria-current` 종류 (강사님 주석 정리)

```
aria-current="page"     - 현재 페이지 (페이지네이션용)
aria-current="step"     - 단계별 가이드의 현재 단계
aria-current="location" - 현재 위치 경로 (빵부스러기)
aria-current="date"     - 현재 날짜 (달력)
aria-current="time"     - 현재 시간
aria-current="true"     - 단순 현재 항목
aria-current="false"    - 단순 현재 아님 (기본값)
```

페이지네이션에서는 `"page"`를 쓰면 됨.

#### 🔑 `String.valueOf(i)` 두 번?

```java
createLinkUrl(fullUrl, i, String.valueOf(i), String.valueOf(i))
//            url   page  label              title
```

페이지 번호는 화면 표시(label)와 설명(title)이 같으므로 둘 다 `"5"` 같은 숫자.

기호 버튼은 다름:
```java
createLinkUrl(fullUrl, 1, "&#x226A;", "처음")
//                       label       title
```

#### 단계 7: 다음과 마지막 페이지

```java
if (endPage < totalPage) {
    int nextBlockPage = endPage + 1;
    sb.append(createLinkUrl(fullUrl, nextBlockPage, "&#x003E;", "다음"));
    sb.append(createLinkUrl(fullUrl, totalPage, "&#x226B;", "마지막"));
}
```

#### 🔑 조건 `endPage < totalPage`

**"현재 블록의 끝이 전체 끝보다 작으면 다음 블록이 있다"** — 직관적.

#### 🔑 nextBlockPage = endPage + 1

다음 블록의 **첫 페이지**로 이동.
- 현재 블록 11~20 → nextBlockPage = 21

#### 단계 8: 마무리

```java
sb.append("</div>");
return sb.toString();
```

### 3-4. `createLinkUrl()` — 헬퍼 메서드

```java
protected String createLinkUrl(String url, int page, String label, String title) {
    return "<a href='" + url + "page=" + page + "'"
         + " title='" + title + "' aria-label='" + title + " 페이지'>"
         + label + "</a>";
}
```

#### 🔑 4개 파라미터

| 파라미터 | 의미 | 예시 |
|---|---|---|
| `url` | 기본 URL (?/&까지 포함) | `"/sb02/notice/list?"` |
| `page` | 이동할 페이지 번호 | `5` |
| `label` | 화면에 **보이는** 텍스트 | `"5"` 또는 `"&#x003E;"` |
| `title` | **설명** 텍스트 | `"5"` 또는 `"다음"` |

#### 🔑 생성되는 HTML

```html
<!-- 페이지 번호 -->
<a href='/sb02/notice/list?page=5' title='5' aria-label='5 페이지'>5</a>

<!-- 다음 블록 -->
<a href='/sb02/notice/list?page=11' title='다음' aria-label='다음 페이지'>></a>
```

#### 🔑 세 가지 텍스트 속성

| 속성 | 화면 표시 | 마우스 툴팁 | 스크린 리더 |
|---|:---:|:---:|:---:|
| `label` (innerHTML) | ✅ | ❌ | ✅ |
| `title` 속성 | ❌ | ✅ | ⚠️ 일부 |
| `aria-label` 속성 | ❌ | ❌ | ✅ (우선) |

**`aria-label`이 있으면 스크린 리더는 그것만 읽고 innerHTML을 무시**합니다. 그래서 `>` 기호 대신 "다음 페이지"라고 정확히 들리게 됨.

#### 🔑 `protected` 접근 제어

자식 클래스에서 오버라이드 가능하도록 열어둔 것. Bootstrap 스타일로 바꾸고 싶으면 상속해서 이 메서드만 재정의하면 됨.

---

## 4. NoticeController.java 분석

이게 진짜 **파이널 프로젝트에 적용해야 하는 핵심**입니다. PaginateUtil은 외부 유틸이고, 컨트롤러는 **매번 직접 짜야 하는 부분**.

### 4-1. 클래스 선언과 어노테이션

```java
@RequiredArgsConstructor
@Slf4j
@Controller
@RequestMapping("/notice/*")
public class NoticeController {
    
    private final NoticeService service;
    private final PaginateUtil paginateUtil;
```

#### 🔑 어노테이션 4개

| 어노테이션 | 역할 |
|---|---|
| `@RequiredArgsConstructor` | `final` 필드로 생성자 자동 생성 (Lombok) |
| `@Slf4j` | `log` 객체 자동 주입 (Lombok) |
| `@Controller` | Spring MVC 컨트롤러 등록 |
| `@RequestMapping("/notice/*")` | 클래스 단위 URL 매핑 |

#### 🔑 `@RequestMapping("/notice/*")` — 와일드카드

`*`는 **`/notice/` 아래 한 단계**를 의미.
- `/notice/list` ✅
- `/notice/article` ✅
- `/notice/list/detail` ❌ (두 단계는 안 됨)

#### 🔑 `final` + DI

```java
private final NoticeService service;
private final PaginateUtil paginateUtil;
```

`@RequiredArgsConstructor`가 이 둘을 받는 생성자를 만들고, Spring이 그 생성자로 의존성 주입(DI).

### 4-2. 메서드 시그니처

```java
@GetMapping("list")
public String list(
        @RequestParam(name = "page", defaultValue = "1") int currentPage,
        @RequestParam(name = "schType", defaultValue = "all") String schType,
        @RequestParam(name = "kwd", defaultValue = "") String kwd,
        Model model,
        HttpServletRequest req) throws Exception {
```

#### 🔑 `@GetMapping("list")` 슬래시 없음

클래스에 `/notice/*`가 있고 메서드에 `"list"`만 있으면 최종 URL은 `/notice/list`.  
**중복 슬래시 방지**를 위해 메서드에서는 슬래시 생략하는 게 일반적.

#### 🔑 `@RequestParam`의 `name`과 `defaultValue`

```java
@RequestParam(name = "page", defaultValue = "1") int currentPage
```

| 옵션 | 의미 |
|---|---|
| `name = "page"` | URL의 `?page=...` 파라미터를 받음 |
| `defaultValue = "1"` | 파라미터 없으면 1로 기본값 |
| `int currentPage` | 자바 변수 이름은 자유롭게 (의미적으로 더 명확하게) |

**URL 파라미터 이름과 자바 변수명을 다르게 둘 수 있다는 점**이 중요합니다.

```
URL: /notice/list?page=3
                  ↓
Java: int currentPage = 3
```

#### 🔑 `Model`과 `HttpServletRequest`

| 객체 | 역할 |
|---|---|
| `Model` | JSP에 데이터를 전달하는 통로 |
| `HttpServletRequest` | 요청 정보 (특히 contextPath) 얻기 위함 |

#### 🔑 `throws Exception`

내부에서 예외가 발생할 수 있음을 선언. `@ControllerAdvice` 같은 글로벌 예외 처리기가 받아 처리.

### 4-3. try-catch + log 패턴

```java
try {
    // 모든 로직
} catch (Exception e) {
    log.info("list : ", e);
    throw e;
}
```

#### 🔑 왜 catch에서 다시 throw?

```java
catch (Exception e) {
    log.info("list : ", e);   // 로그 남기고
    throw e;                  // 다시 던짐 (서비스 단/글로벌 핸들러로)
}
```

**로그는 여기서, 처리는 위로** — 책임 분리 패턴. 이 컨트롤러가 모든 걸 다 처리하지 않고 상위 계층으로 위임.

### 4-4. ⭐ 검색 조건 처리 + URLDecoder

```java
int size = 10;
int totalPage = 0;
int dataCount = 0;

kwd = URLDecoder.decode(kwd, "utf-8");

Map<String, Object> map = new HashMap<String, Object>();
map.put("schType", schType);
map.put("kwd", kwd);
```

#### 🔑 변수 초기화

```java
int totalPage = 0;
int dataCount = 0;
```

`if (dataCount != 0)` 같은 분기를 안전하게 쓰기 위해 명시적으로 0으로 초기화.

#### 🔑 ⭐ `URLDecoder.decode(kwd, "utf-8")` — 한글 검색어 디코딩

이게 **꼭 알아야 할 부분**입니다.

URL에 한글이 들어가면 브라우저가 자동으로 인코딩:
```
검색어: "공지" → URL에서는 "%EA%B3%B5%EC%A7%80"
```

스프링이 URL 파라미터를 받을 때 기본적으로 디코딩하지만, **이중 인코딩(double encoding)이 발생할 수 있어서** 한 번 더 디코딩하는 게 안전.

```
원본:        공지
1차 인코딩:   %EA%B3%B5%EC%A7%80
2차 인코딩:   %25EA%25B3%25B5%25EC%25A7%2580   ← % 자체가 다시 %25로
```

만약 페이지 이동 시 그대로 URL을 다시 만들어 보내면 이중 인코딩이 발생할 수 있는데, `URLDecoder.decode`로 원본으로 되돌려 두는 거죠.

#### 🔑 검색 조건 Map

```java
Map<String, Object> map = new HashMap<String, Object>();
map.put("schType", schType);
map.put("kwd", kwd);
```

MyBatis로 넘길 때 여러 파라미터를 한 번에 보내려면 Map이 편함.

### 4-5. ⭐ 데이터 개수 조회 + 페이지 계산

```java
dataCount = service.dataCount(map);

if (dataCount != 0) {
    totalPage = paginateUtil.pageCount(dataCount, size);
    
    currentPage = Math.min(currentPage, totalPage);
    // 다른 사람이 데이터를 삭제하여 전체 페이지 수가 변경된 경우 대비
```

#### 🔑 `if (dataCount != 0)` — 데이터 없을 때 처리

데이터가 0개면 페이지 계산할 필요 없음. 그래서 큰 if 블록으로 감싸고, **데이터 있을 때만 모든 로직 실행.**

데이터 없으면 `noticeList`, `list` 같은 변수가 모델에 들어가지 않아서 JSP에서 "검색 결과 없음" 처리 가능.

#### 🔑 ⭐ `currentPage = Math.min(currentPage, totalPage)` — 동시성 대비

```java
currentPage = Math.min(currentPage, totalPage);
// 다른 사람이 데이터를 삭제하여 전체 페이지 추가 변경된 경우
```

이게 **세련된 부분**입니다. 시나리오를 봅시다:

```
[10:00:00] 사용자 A: 5페이지 보고 있음 (총 50페이지)
[10:00:30] 사용자 B: 게시물 대량 삭제 → 총 페이지 3페이지로 줄어듦
[10:01:00] 사용자 A: "4페이지" 링크 클릭
            → 4 > 3 (현재 totalPage) → 잘못된 페이지
            → Math.min(4, 3) = 3 → 마지막 페이지로 자동 보정 ✅
```

**동시성(concurrency) 문제**를 깔끔하게 대응하는 코드. PaginateUtil 안에서도 보정하지만, **컨트롤러에서도 OFFSET 계산을 위해 한 번 더 보정**해두는 것.

### 4-6. ⭐ 1페이지일 때만 상단 공지

```java
List<Notice> noticeList = null;
if (currentPage == 1) {
    noticeList = service.listNoticeTop();
}
```

#### 🔑 게시판에서 흔한 패턴

```
┌─────────────────────────┐
│ [공지] 서비스 점검 안내    │  ← 1페이지에만 표시
│ [공지] 이용약관 변경       │  ← 1페이지에만 표시
├─────────────────────────┤
│ 1. 일반 게시물 1          │
│ 2. 일반 게시물 2          │
│ ...                     │
└─────────────────────────┘
```

**중요 공지는 항상 위에 보여야 하지만, 모든 페이지에 똑같이 표시하면 화면 낭비.** 1페이지일 때만 상단 공지를 보여주는 패턴.

#### 🔑 `null`로 초기화

```java
List<Notice> noticeList = null;
if (currentPage == 1) {
    noticeList = service.listNoticeTop();
}
```

2페이지 이상이면 `noticeList`는 `null`. JSP에서 `<c:if test="${not empty noticeList}">`로 분기 처리.

### 4-7. ⭐ OFFSET 계산

```java
int offset = (currentPage - 1) * size;
if (offset < 0) {
    offset = 0;
}

map.put("offset", offset);
map.put("size", size);
```

#### 🔑 핵심 공식

```
offset = (현재 페이지 - 1) × 페이지당 개수
```

| currentPage | size | offset | 의미 |
|---:|---:|---:|---|
| 1 | 10 | 0 | 0개 건너뛰고 10개 → 1~10번째 |
| 2 | 10 | 10 | 10개 건너뛰고 10개 → 11~20번째 |
| 3 | 10 | 20 | 20개 건너뛰고 10개 → 21~30번째 |

**"OFFSET = 건너뛸 개수"** 라고 기억하면 됨.

#### 🔑 MyBatis XML과 연결

```xml
SELECT ...
FROM NOTICE
ORDER BY NUM DESC
OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
```

Oracle 12c 이상에서 사용 가능한 표준 SQL 페이징 구문.

#### 🔑 `if (offset < 0)` 방어

이론상 currentPage가 보정되어서 음수가 될 일은 없지만, **안전망**으로 추가.

### 4-8. ⭐ URL 조립 + 인코딩

```java
List<Notice> list = service.listNotice(map);
String cp = req.getContextPath();
String query = "";
String listUrl = cp + "/notice/list";

if (!kwd.isBlank()) {
    query = "schType=" + schType + "&kwd=" + URLEncoder.encode(kwd, "utf-8");
    listUrl += "?" + query;
}
```

#### 🔑 ⭐ `req.getContextPath()` — Context Path

**이게 진짜 중요합니다.** 웹 애플리케이션이 어디에 배포되어 있는지 알아내는 방법.

```
배포 1: http://localhost:8080/notice/list
        → contextPath = "" (루트에 배포)

배포 2: http://localhost:8080/sb02/notice/list
        → contextPath = "/sb02"

배포 3: http://example.com/myapp/notice/list
        → contextPath = "/myapp"
```

**개발할 때는 "/sb02"였는데 배포는 "/"로 바뀌면?** 하드코딩한 URL은 다 깨집니다. `contextPath`를 쓰면 어디에 배포되어도 정상 동작.

#### 🔑 `kwd.isBlank()` — Java 11+ 메서드

```java
"".isBlank()        → true
"   ".isBlank()     → true   ← 공백만 있어도 true
"hello".isBlank()   → false

"".isEmpty()        → true
"   ".isEmpty()     → false  ← 공백은 비어있다고 보지 않음
```

검색어 입력 시 공백만 입력하면 검색 안 하는 게 자연스러우므로 `isBlank()` 사용.

#### 🔑 ⭐ `URLEncoder.encode(kwd, "utf-8")` — 한글 인코딩

```java
String kwd = "공지사항";
URLEncoder.encode(kwd, "utf-8")
// → "%EA%B3%B5%EC%A7%80%EC%82%AC%ED%95%AD"
```

한글을 URL에 안전하게 넣기 위한 인코딩. **이거 빼먹으면 한글 검색어가 URL에서 깨집니다.**

흐름:
```
1. 사용자가 "공지" 검색
2. URLDecoder.decode → "공지" (위에서 처리)
3. Map에 "공지"로 들어가서 DB 조회 OK
4. 페이지 링크 만들 때 URLEncoder.encode → "%EA%B3%B5%EC%A7%80"
5. 사용자가 2페이지 클릭 → ?kwd=%EA%B3%B5%EC%A7%80 으로 요청
6. 다시 1번부터 (디코딩 → 검색)
```

**대칭 구조**가 됩니다: 받을 때 디코딩, 보낼 때 인코딩.

#### 🔑 query string 조립

```java
if (!kwd.isBlank()) {
    query = "schType=" + schType + "&kwd=" + URLEncoder.encode(kwd, "utf-8");
    listUrl += "?" + query;
}
```

검색어 있으면:
- `query = "schType=all&kwd=%EA%B3%B5%EC%A7%80"`
- `listUrl = "/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80"`

검색어 없으면:
- `listUrl = "/sb02/notice/list"` (?는 PaginateUtil에서 붙임)

### 4-9. 페이징 HTML 생성

```java
String paging = paginateUtil.paging(currentPage, totalPage, listUrl);
```

PaginateUtil 안에서 `listUrl`이 `?`를 포함하는지 자동 판단해서 `&` 또는 `?`를 붙임.

### 4-10. ⭐ Model에 데이터 담기

```java
model.addAttribute("noticeList", noticeList);   // 상단 공지 (1페이지에만)
model.addAttribute("list", list);                // 일반 게시물
model.addAttribute("page", currentPage);          // 현재 페이지
model.addAttribute("dataCount", dataCount);       // 총 개수
model.addAttribute("paging", paging);             // 페이징 HTML
model.addAttribute("size", size);                 // 페이지당 개수
model.addAttribute("totalPage", totalPage);       // 전체 페이지 수

model.addAttribute("schType", schType);           // 검색 타입 (다시 그리기 위해)
model.addAttribute("kwd", kwd);                   // 검색어 (input에 채우기 위해)
```

#### 🔑 무엇을 왜 담는가?

| 속성 | 용도 |
|---|---|
| `noticeList` | 1페이지 상단 공지 |
| `list` | 게시물 목록 (테이블) |
| `page` | 페이지네이션 강조 표시 (현재 페이지 표기) |
| `dataCount` | "총 X개" 같은 카운트 표시 |
| `paging` | 페이지네이션 HTML 그대로 출력 |
| `size` | 게시물 번호 계산 (1페이지 1번이 아니라 dataCount부터 역순 등) |
| `totalPage` | 페이지 정보 표시 |
| `schType` | `<select>`의 선택 유지 |
| `kwd` | `<input>`의 값 유지 |

#### 🔑 검색 상태 유지

검색해서 2페이지로 갔을 때, 검색 입력 칸이 비어있으면 사용자 혼란. **검색어와 검색 타입을 모델에 담아서 JSP에서 다시 표시**해야 합니다.

### 4-11. 컨트롤러 전체 흐름 요약

```
[입력] page=2, schType=subject, kwd=공지
   ↓
1. URLDecoder.decode(kwd) → "공지"
   ↓
2. map = {schType, kwd}
   ↓
3. dataCount = service.dataCount(map)  → 95개
   ↓
4. dataCount != 0인지 확인
   ↓
5. totalPage = paginateUtil.pageCount(95, 10) = 10
   ↓
6. currentPage = Math.min(2, 10) = 2
   ↓
7. (currentPage == 1) → false → 공지 목록 skip
   ↓
8. offset = (2-1) * 10 = 10
   map = {schType, kwd, offset:10, size:10}
   ↓
9. list = service.listNotice(map)  → 11~20번째 게시물
   ↓
10. listUrl = contextPath + "/notice/list?schType=subject&kwd=%EA%B3%B5%EC%A7%80"
    ↓
11. paging = paginateUtil.paging(2, 10, listUrl) → HTML
    ↓
12. model에 모든 데이터 담기
    ↓
13. return "notice/list" → /WEB-INF/views/notice/list.jsp로 forward
```

---

## 5. 전체 흐름 시뮬레이션

### 시나리오: totalPage=10, currentPage=2, kwd="공지"

#### 컨트롤러
```
URLDecoder.decode("%EA%B3%B5%EC%A7%80", "utf-8") = "공지"
dataCount = 95
totalPage = pageCount(95, 10) = 10
currentPage = min(2, 10) = 2
currentPage == 1? No → noticeList = null
offset = (2-1) * 10 = 10
list = 11~20번째 게시물
listUrl = "/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80"
paging = paginateUtil.paging(2, 10, listUrl)
```

#### PaginateUtil.paging(2, 10, listUrl)
```
2 > 10? No → currentPage 그대로
2 < 1 || 10 < 1? No → 통과
connector = "&"  ← listUrl에 이미 ?가 있음
fullUrl = "/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80&"

currentBlock = (2-1)/10 = 0
startPage = 0*10 + 1 = 1
endPage = min(1+10-1, 10) = 10

<div class='paginate'>
  // currentBlock > 0? No → 처음/이전 skip
  
  i=1: 1≠2 → <a ...page=1>1</a>
  i=2: 2==2 → <span class='active' aria-current='page'>2</span>
  i=3..10: <a ...page=N>N</a>
  
  // endPage < totalPage? → 10 < 10? No → 다음/마지막 skip
</div>
```

#### 최종 HTML
```html
<div class='paginate'>
  <a href='/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80&page=1' title='1' aria-label='1 페이지'>1</a>
  <span class='active' aria-current='page'>2</span>
  <a href='/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80&page=3' title='3' aria-label='3 페이지'>3</a>
  ...
  <a href='/sb02/notice/list?schType=all&kwd=%EA%B3%B5%EC%A7%80&page=10' title='10' aria-label='10 페이지'>10</a>
</div>
```

---

## 6. JSP 출력 가이드

### 6-1. 기본 구조

```jsp
<%@ page contentType="text/html; charset=UTF-8" %>
<%@ taglib prefix="c" uri="jakarta.tags.core" %>
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>공지사항</title>
<style>
.paginate { text-align: center; margin: 20px 0; }
.paginate a, .paginate span {
    display: inline-block;
    padding: 5px 10px;
    margin: 0 2px;
    border: 1px solid #ddd;
    text-decoration: none;
    color: #333;
}
.paginate a:hover { background: #f0f0f0; }
.paginate .active {
    background: #007bff;
    color: white;
    border-color: #007bff;
}
</style>
</head>
<body>

  <!-- 검색 폼 -->
  <form action="<%= request.getContextPath() %>/notice/list" method="get">
    <select name="schType">
      <option value="all" ${schType=='all' ? 'selected' : ''}>제목+내용</option>
      <option value="subject" ${schType=='subject' ? 'selected' : ''}>제목</option>
      <option value="content" ${schType=='content' ? 'selected' : ''}>내용</option>
      <option value="name" ${schType=='name' ? 'selected' : ''}>작성자</option>
    </select>
    <input type="text" name="kwd" value="${kwd}">
    <button type="submit">검색</button>
  </form>

  <!-- 상단 공지 (1페이지에만) -->
  <c:if test="${not empty noticeList}">
    <table>
      <c:forEach var="dto" items="${noticeList}">
        <tr style="background: #ffe;">
          <td>[공지]</td>
          <td>${dto.subject}</td>
          <td>${dto.name}</td>
          <td>${dto.regDate}</td>
        </tr>
      </c:forEach>
    </table>
  </c:if>

  <!-- 일반 게시물 -->
  <table>
    <thead>
      <tr>
        <th>번호</th>
        <th>제목</th>
        <th>작성자</th>
        <th>날짜</th>
      </tr>
    </thead>
    <tbody>
      <c:choose>
        <c:when test="${empty list}">
          <tr><td colspan="4">검색 결과가 없습니다.</td></tr>
        </c:when>
        <c:otherwise>
          <c:forEach var="dto" items="${list}" varStatus="status">
            <tr>
              <!-- 번호 = dataCount - (page-1)*size - status.index -->
              <td>${dataCount - (page-1)*size - status.index}</td>
              <td>${dto.subject}</td>
              <td>${dto.name}</td>
              <td>${dto.regDate}</td>
            </tr>
          </c:forEach>
        </c:otherwise>
      </c:choose>
    </tbody>
  </table>

  <!-- 페이징 HTML 출력 -->
  ${paging}

  <p>총 ${dataCount}개 / ${page}페이지 / 전체 ${totalPage}페이지</p>

</body>
</html>
```

#### 🔑 게시물 번호 계산

```jsp
${dataCount - (page-1)*size - status.index}
```

| 예 | 계산 | 결과 |
|---|---|---|
| dataCount=95, page=1, status.index=0 | 95 - 0 - 0 | **95** |
| dataCount=95, page=1, status.index=9 | 95 - 0 - 9 | **86** |
| dataCount=95, page=2, status.index=0 | 95 - 10 - 0 | **85** |

**최신 게시물부터 번호가 큰** 일반적 게시판 번호 매김.

#### 🔑 `${paging}` 출력

`${paging}`은 HTML이 그대로 출력됨. `<c:out>`을 쓰면 태그가 문자로 보여서 안 됨.

#### 🔑 검색 상태 유지

- `<select>`: `${schType=='subject' ? 'selected' : ''}` 로 선택 유지
- `<input>`: `value="${kwd}"`로 값 유지

---

## 7. 외우기용 핵심 패턴

### 7-1. PaginateUtil 핵심 공식 7개

```java
// 1. 전체 페이지 수
int totalPage = dataCount / size + (dataCount % size > 0 ? 1 : 0);

// 2. 입력 보정 순서
if (currentPage > totalPage) currentPage = totalPage;
if (currentPage < 1 || totalPage < 1) return "";

// 3. URL 연결자
String connector = listUrl.contains("?") ? "&" : "?";
String fullUrl = listUrl + connector;

// 4. 현재 블록 (-1 트릭!)
int currentBlock = (currentPage - 1) / blockSize;

// 5. 시작 페이지
int startPage = (currentBlock * blockSize) + 1;

// 6. 끝 페이지 (Math.min!)
int endPage = Math.min(startPage + blockSize - 1, totalPage);

// 7. OFFSET
int offset = (currentPage - 1) * size;
```

### 7-2. PaginateUtil 알고리즘 흐름

```
1. 입력 보정 (clamp → return)
2. blockSize=10, StringBuilder
3. connector, fullUrl
4. currentBlock, startPage, endPage
5. <div> 시작
6. if (currentBlock > 0) → 처음/이전
7. for (i = startPage..endPage) → 페이지 번호
8. if (endPage < totalPage) → 다음/마지막
9. </div> 닫고 return
```

### 7-3. NoticeController 흐름 9단계

```
1. 파라미터 받기 (@RequestParam)
2. try 시작, kwd 디코딩
3. map에 검색 조건 담기
4. dataCount 조회
5. if (dataCount != 0) {
     6. totalPage 계산
     7. currentPage = Math.min(currentPage, totalPage)
     8. (currentPage == 1)이면 noticeList 조회
     9. offset 계산, map에 추가
     10. list 조회
     11. listUrl 조립 (contextPath + query string)
     12. paging HTML 생성
     13. model에 모두 담기
   }
14. catch → log → throw
15. return "notice/list"
```

### 7-4. HTML 엔티티 4개

| 엔티티 | 문자 | 의미 |
|---|---|---|
| `&#x226A;` | « | 처음 |
| `&#x003C;` | < | 이전 |
| `&#x003E;` | > | 다음 |
| `&#x226B;` | » | 마지막 |

**세미콜론(;)을 꼭 붙이는 게 표준입니다.**

### 7-5. HTML 속성

```html
<!-- 현재 페이지 -->
<span class='active' aria-current='page'>5</span>

<!-- 일반 링크 -->
<a href='...?page=N'
   title='5'              ← 마우스 툴팁
   aria-label='5 페이지'   ← 스크린 리더용
>5</a>
```

### 7-6. 단계별 손코딩 연습

| 회차 | 시간 | 내용 |
|---|---|---|
| 1회차 | 60분 | 두 파일 모두 문서 보면서 따라 치기 |
| 2회차 | 10분 | `pageCount()` + `createLinkUrl()` 외워서 |
| 3회차 | 25분 | `paging()` 공식 외워서 |
| 4회차 | 40분 | NoticeController 흐름 외워서 |
| 5회차 | 90분 | **두 파일 모두 안 보고 처음부터 끝까지** |

---

## 8. 자가 점검 체크리스트

### 8-1. PaginateUtil — pageCount()
- [ ] `dataCount <= 0 || size <= 0` 검증?
- [ ] 정수 나눗셈 + 나머지 처리?

### 8-2. PaginateUtil — paging() 입력 처리
- [ ] `currentPage > totalPage` clamp **먼저**?
- [ ] `currentPage < 1 || totalPage < 1` 거부 **나중**?
- [ ] 순서가 정확? (clamp → 거부)

### 8-3. PaginateUtil — paging() 변수와 URL
- [ ] `blockSize = 10`?
- [ ] `StringBuilder` 사용?
- [ ] `connector` 변수 분리?
- [ ] `fullUrl = listUrl + connector`?

### 8-4. PaginateUtil — paging() 블록 계산
- [ ] `currentBlock = (currentPage - 1) / blockSize` — `-1` 트릭?
- [ ] `startPage = (currentBlock * blockSize) + 1`?
- [ ] `endPage = Math.min(startPage + blockSize - 1, totalPage)`?

### 8-5. PaginateUtil — paging() HTML
- [ ] `<div class='paginate'>` 시작?
- [ ] 처음/이전: `if (currentBlock > 0)`?
- [ ] `prevBlockPage = startPage - 1`?
- [ ] HTML 엔티티 끝에 `;` 붙임?
- [ ] for문: `i = startPage; i <= endPage; i++`?
- [ ] 현재 페이지: `<span class='active' aria-current='page'>`?
- [ ] 다음/마지막: `if (endPage < totalPage)`?
- [ ] `nextBlockPage = endPage + 1`?

### 8-6. PaginateUtil — createLinkUrl()
- [ ] 4개 파라미터 (url, page, label, title)?
- [ ] `title='...'` 속성?
- [ ] `aria-label='... 페이지'` 속성?
- [ ] `protected` 접근?

### 8-7. NoticeController — 어노테이션
- [ ] `@RequiredArgsConstructor`?
- [ ] `@Slf4j`?
- [ ] `@Controller`?
- [ ] `@RequestMapping("/notice/*")`?
- [ ] `final NoticeService service`, `final PaginateUtil paginateUtil`?

### 8-8. NoticeController — 메서드 시그니처
- [ ] `@GetMapping("list")` (슬래시 없음)?
- [ ] `@RequestParam` 3개 (page, schType, kwd) + defaultValue?
- [ ] `Model model`, `HttpServletRequest req`?
- [ ] `throws Exception`?

### 8-9. NoticeController — 로직
- [ ] `try-catch + log.info + throw e` 패턴?
- [ ] `URLDecoder.decode(kwd, "utf-8")`?
- [ ] Map에 schType, kwd 담기?
- [ ] `dataCount = service.dataCount(map)`?
- [ ] `if (dataCount != 0)` 분기?
- [ ] `currentPage = Math.min(currentPage, totalPage)` (동시성)?
- [ ] `if (currentPage == 1)` 일 때 listNoticeTop 조회?
- [ ] `offset = (currentPage - 1) * size` + 음수 방어?
- [ ] `req.getContextPath()` 사용?
- [ ] `kwd.isBlank()` 체크?
- [ ] `URLEncoder.encode(kwd, "utf-8")`?
- [ ] `paginateUtil.paging()` 호출?

### 8-10. NoticeController — Model 데이터
- [ ] noticeList, list, page, dataCount, paging, size, totalPage?
- [ ] schType, kwd (검색 상태 유지)?
- [ ] `return "notice/list"`?

---

## 🎓 마무리

페이징은 **외울 가치가 있는 패턴**입니다. 모든 게시판형 기능에 거의 그대로 적용 가능.

핵심을 다시 정리하면:

### PaginateUtil의 핵심
1. **`-1` 트릭** — `(currentPage - 1) / blockSize`로 10의 배수 자연 처리
2. **`Math.min`** — `if`문 한 줄로 압축
3. **변수 분리** — `connector`, `currentBlock` 등 의도를 이름으로

### NoticeController의 핵심
1. **`URLDecoder` + `URLEncoder` 대칭** — 받을 때 디코딩, 보낼 때 인코딩
2. **`req.getContextPath()`** — 배포 위치 변경에 안전
3. **`Math.min(currentPage, totalPage)`** — 동시성 대비
4. **`if (currentPage == 1)`** — 1페이지에만 상단 공지
5. **`if (dataCount != 0)`** — 데이터 없을 때 안전 처리

### 가장 외우기 어려운 부분 Top 3
1. 🥇 `currentBlock = (currentPage - 1) / blockSize` 의 `-1`
2. 🥈 검증 순서: clamp 먼저, 거부 나중
3. 🥉 contextPath + URLEncoder 조합

이 세 가지는 **반드시 5번 이상 손코딩**해서 외우세요.

> 💪 **외우는 데 시간이 걸려도 괜찮습니다. 한 번 외워두면 평생 써먹습니다.**

---

## 📎 부록: 두 파일 전체 코드 (참고용)

이 부분은 막막할 때 다시 볼 수 있도록 전체 코드를 그대로 박아둡니다.

### A. PaginateUtil.java
```java
package com.doit.app.common;

import org.springframework.stereotype.Service;

@Service
public class PaginateUtil {

    public int pageCount(int dataCount, int size) {
        if (dataCount <= 0 || size <= 0) return 0;
        return dataCount / size + (dataCount % size > 0 ? 1 : 0);
    }

    public String paging(int currentPage, int totalPage, String listUrl) {

        if (currentPage > totalPage) {
            currentPage = totalPage;
        }
        if (currentPage < 1 || totalPage < 1) {
            return "";
        }

        int blockSize = 10;
        StringBuilder sb = new StringBuilder();

        String connector = listUrl.contains("?") ? "&" : "?";
        String fullUrl = listUrl + connector;

        int currentBlock = (currentPage - 1) / blockSize;
        int startPage = (currentBlock * blockSize) + 1;
        int endPage = Math.min(startPage + blockSize - 1, totalPage);

        sb.append("<div class='paginate'>");

        if (currentBlock > 0) {
            int prevBlockPage = startPage - 1;
            sb.append(createLinkUrl(fullUrl, 1, "&#x226A;", "처음"));
            sb.append(createLinkUrl(fullUrl, prevBlockPage, "&#x003C;", "이전"));
        }

        for (int i = startPage; i <= endPage; i++) {
            if (i == currentPage) {
                sb.append("<span class='active' aria-current='page'>").append(i).append("</span>");
            } else {
                sb.append(createLinkUrl(fullUrl, i, String.valueOf(i), String.valueOf(i)));
            }
        }

        if (endPage < totalPage) {
            int nextBlockPage = endPage + 1;
            sb.append(createLinkUrl(fullUrl, nextBlockPage, "&#x003E;", "다음"));
            sb.append(createLinkUrl(fullUrl, totalPage, "&#x226B;", "마지막"));
        }

        sb.append("</div>");
        return sb.toString();
    }

    protected String createLinkUrl(String url, int page, String label, String title) {
        return "<a href='" + url + "page=" + page + "'"
             + " title='" + title + "' aria-label='" + title + " 페이지'>"
             + label + "</a>";
    }
}
```

### B. NoticeController.java (핵심 부분만)
```java
@RequiredArgsConstructor
@Slf4j
@Controller
@RequestMapping("/notice/*")
public class NoticeController {

    private final NoticeService service;
    private final PaginateUtil paginateUtil;

    @GetMapping("list")
    public String list(
            @RequestParam(name = "page", defaultValue = "1") int currentPage,
            @RequestParam(name = "schType", defaultValue = "all") String schType,
            @RequestParam(name = "kwd", defaultValue = "") String kwd,
            Model model,
            HttpServletRequest req) throws Exception {

        try {
            int size = 10;
            int totalPage = 0;
            int dataCount = 0;

            kwd = URLDecoder.decode(kwd, "utf-8");

            Map<String, Object> map = new HashMap<>();
            map.put("schType", schType);
            map.put("kwd", kwd);

            dataCount = service.dataCount(map);
            if (dataCount != 0) {
                totalPage = paginateUtil.pageCount(dataCount, size);
                currentPage = Math.min(currentPage, totalPage);

                List<Notice> noticeList = null;
                if (currentPage == 1) {
                    noticeList = service.listNoticeTop();
                }

                int offset = (currentPage - 1) * size;
                if (offset < 0) offset = 0;

                map.put("offset", offset);
                map.put("size", size);

                List<Notice> list = service.listNotice(map);

                String cp = req.getContextPath();
                String query = "";
                String listUrl = cp + "/notice/list";
                if (!kwd.isBlank()) {
                    query = "schType=" + schType + "&kwd=" + URLEncoder.encode(kwd, "utf-8");
                    listUrl += "?" + query;
                }

                String paging = paginateUtil.paging(currentPage, totalPage, listUrl);

                model.addAttribute("noticeList", noticeList);
                model.addAttribute("list", list);
                model.addAttribute("page", currentPage);
                model.addAttribute("dataCount", dataCount);
                model.addAttribute("paging", paging);
                model.addAttribute("size", size);
                model.addAttribute("totalPage", totalPage);
                model.addAttribute("schType", schType);
                model.addAttribute("kwd", kwd);
            }

        } catch (Exception e) {
            log.info("list : ", e);
            throw e;
        }

        return "notice/list";
    }
}
```

---

> 끝까지 따라오셨다면, 이제 손가락이 기억하도록 만들 차례입니다. 💪
