# 📚 Notice 게시판 버그 분석 학습 정리

> 작성일: 2026-06-01  
> 대상 코드: `NoticeServiceImpl.java`, `NoticeMapper.xml`  
> 발견 버그: 총 9곳 (치명적 4 + 중요 2 + 권장 3)

이 문서는 코드 리뷰 과정에서 발견된 버그들을 통해, 다음에 비슷한 실수를 반복하지 않기 위한 **개념 정리 노트**입니다.

---

## 🎯 학습 목표

이 문서를 끝까지 읽고 나면 아래 질문에 답할 수 있어야 합니다.

1. `Objects.requireNonNull(mapper, findByFileId(fileNum))`가 왜 잘못된 코드인가?
2. `Path.get()`, `Paths.get()`, `Path.of()`의 차이는?
3. MyBatis `${}`와 `#{}`의 차이와 각각 언제 쓰는가?
4. 자바에서 같은 이름의 다른 패키지 클래스를 어떻게 구분하는가?
5. try-catch에서 catch 블록에 return이 없으면 왜 컴파일 에러가 나는가?
6. Java의 "precise rethrow"란 무엇인가?
7. INSERT 문에서 컬럼 순서와 값 순서가 어긋나면 어떻게 되는가?

---

## 📑 목차

1. [Java 핵심 개념](#1-java-핵심-개념)
2. [Spring & 라이브러리 개념](#2-spring--라이브러리-개념)
3. [MyBatis 개념](#3-mybatis-개념)
4. [Exception Handling 패턴](#4-exception-handling-패턴)
5. [HTTP 응답 구성](#5-http-응답-구성)
6. [흔히 하는 실수 체크리스트](#6-흔히-하는-실수-체크리스트)

---

## 1. Java 핵심 개념

### 1-1. 메서드 호출 vs 인자 전달 (쉼표의 의미)

#### ❌ 내가 썼던 잘못된 코드

```java
Notice dto = Objects.requireNonNull(mapper, findByFileId(fileNum));
```

이 코드는 컴파일러에게 이렇게 보입니다.

| 위치 | 값 | 타입 |
|---|---|---|
| 1번 인자 | `mapper` | NoticeMapper |
| 2번 인자 | `findByFileId(fileNum)` | (메서드 호출 결과) |

#### ✅ 의도한 올바른 코드

```java
Notice dto = Objects.requireNonNull(mapper.findByFileId(fileNum));
```

#### 🔑 핵심 개념

> **`,`(쉼표)는 "다음 인자 구분"이고, `.`(점)은 "객체의 멤버 접근"이다.**

자바에서 `mapper.findByFileId(fileNum)`는 "mapper 객체에 대고 findByFileId 메서드를 호출"하는 것이고,  
`mapper, findByFileId(fileNum)`는 "mapper와 findByFileId의 결과, 두 개를 따로 전달"하는 것입니다.

#### 📖 참고: `Objects.requireNonNull`의 시그니처

```java
public static <T> T requireNonNull(T obj);                        // 1-인자 버전
public static <T> T requireNonNull(T obj, String message);        // 2-인자 버전
```

2번째 인자는 **null일 때 던질 NullPointerException의 메시지**입니다. Notice 타입이 아니라 String이어야 합니다.

```java
// 메시지를 주고 싶다면 이렇게
Notice dto = Objects.requireNonNull(
    mapper.findByFileId(fileNum),
    "파일을 찾을 수 없습니다: fileNum=" + fileNum
);
```

---

### 1-2. 정적 팩토리 메서드: `Paths.get()` vs `Path.of()`

#### ❌ 존재하지 않는 메서드

```java
Path path = Path.get(pathname);   // ❌ Path 인터페이스에 get() 없음
```

#### ✅ 올바른 두 가지 방법

```java
// 방법 1: 전통적 방식 (Java 7+)
import java.nio.file.Paths;
Path path = Paths.get(pathname);

// 방법 2: 권장 방식 (Java 11+)
Path path = Path.of(pathname);    // import java.nio.file.Path만 있으면 됨
```

#### 🔑 핵심 개념: 유틸리티 클래스 패턴

자바는 **인터페이스(Path)와 그것을 다루는 유틸 클래스(Paths)를 분리**해서 설계하는 경우가 많습니다.

| 비슷한 사례 | 인터페이스 / 클래스 | 유틸 클래스 |
|---|---|---|
| 파일 경로 | `Path` | `Paths` |
| 컬렉션 | `List`, `Set`, `Map` | `Collections` |
| 배열 | (배열) | `Arrays` |
| 객체 일반 | `Object` | `Objects` |

Java 9부터는 인터페이스에 `static` 메서드를 둘 수 있게 되어, `List.of()`, `Map.of()`, `Path.of()` 같은 형태가 가능해졌습니다. **이제는 `Path.of`가 더 선호됩니다.**

---

### 1-3. 같은 이름, 다른 패키지의 클래스 구분

#### ❌ 잘못된 import

```java
import java.net.http.HttpHeaders;   // Java 11+의 HTTP 클라이언트용
```

#### ✅ 올바른 import

```java
import org.springframework.http.HttpHeaders;   // Spring의 HTTP 헤더
```

#### 🔑 핵심 개념

두 클래스는 **이름만 같고 완전히 다른 클래스**입니다.

| 클래스 | 패키지 | 용도 | 가변성 |
|---|---|---|---|
| `HttpHeaders` | `java.net.http` | Java 표준 HTTP 클라이언트 응답 헤더 | 불변(Immutable) |
| `HttpHeaders` | `org.springframework.http` | Spring 응답/요청 헤더 빌드 | 가변(Mutable) |

`add()` 메서드와 `CONTENT_DISPOSITION` 같은 상수는 **Spring의 것에만 있습니다.**

#### 💡 이클립스 팁

- `Ctrl + Shift + O`: import 자동 정리. 같은 이름의 클래스가 여러 개일 때 선택 창이 뜸.
- import 추가 시 **패키지 경로를 꼭 확인**할 것.

---

### 1-4. 대소문자 구분 (Case Sensitivity)

자바는 **완전히 대소문자를 구분**합니다.

```java
Httpheaders.CONTENT_DISPOSITION   // ❌ 컴파일 에러 (그런 클래스 없음)
HttpHeaders.CONTENT_DISPOSITION   // ✅
```

오타가 아니라 **다른 식별자로 인식되는 것**입니다. 컴파일러는 "심볼을 찾을 수 없음"이라는 에러를 띄웁니다.

---

### 1-5. 모든 실행 경로에서의 return (Definite Assignment)

#### ❌ 컴파일 에러 패턴

```java
public ResponseEntity<?> downloadUploadFile(long fileNum) throws Exception {
    try {
        // ...
        return ResponseEntity.ok()...;   // try에서만 return
    } catch (Exception e) {
        // 빈 catch - return 없음!
    }
    // ⚠️ 여기까지 도달 가능한데 return이 없음 → 컴파일 에러
}
```

#### ✅ 해결 방법 3가지

**(1) catch에서도 return**
```java
} catch (Exception e) {
    log.error("error", e);
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
}
```

**(2) catch에서 throw (메서드 정상 종료가 아니므로 OK)**
```java
} catch (Exception e) {
    log.error("error", e);
    throw e;   // 또는 throw new RuntimeException(e);
}
```

**(3) try-catch 자체를 빼고 throws로 위임**
```java
public ResponseEntity<?> downloadUploadFile(long fileNum) throws Exception {
    // try-catch 없이 그냥 처리
    return ResponseEntity.ok()...;
}
```

#### 🔑 핵심 개념: Definite Assignment

자바 컴파일러는 **모든 가능한 실행 경로에서 반환값이 보장되어야 한다**는 규칙을 검사합니다.  
이를 "definite assignment" 또는 "exhaustive return" 분석이라고 합니다.

---

### 1-6. `return null` 실수 — 가장 자주 발생하는 버그

```java
public List<Notice> listNotice(Map<String, Object> map) {
    List<Notice> list = null;
    try {
        list = mapper.listNotice(map);   // 조회는 정상
    } catch (Exception e) {
        log.info("listNotice : ", e);
    }
    return null;   // ⚠️ list를 반환해야 하는데 null 반환
}
```

**증상**: 컴파일은 통과하고 코드는 돌아가지만, 호출자에게는 항상 null이 전달되어 **목록이 화면에 안 보임.**

#### 🔑 예방 팁

- 변수 선언 시 `null`로 초기화하지 말고 `new ArrayList<>()`처럼 빈 객체로 초기화하는 것도 방법
- IDE의 "unused variable" 경고가 뜨면 그 메서드를 다시 살펴볼 것
- 단위 테스트로 반환값을 검증하면 즉시 발견됨

---

## 2. Spring & 라이브러리 개념

### 2-1. `ResponseEntity<?>` — HTTP 응답 직접 만들기

```java
return ResponseEntity.ok()
    .headers(headers)
    .contentLength(resource.contentLength())
    .body(resource);
```

#### 🔑 핵심 개념

일반적인 컨트롤러는 View 이름(`return "notice/list"`) 또는 JSON 데이터(`return dto`)를 반환합니다.  
하지만 **파일 다운로드처럼 응답의 모든 요소(상태코드, 헤더, 본문)를 세밀하게 제어**해야 할 때는 `ResponseEntity`를 사용합니다.

| 구성 요소 | 의미 |
|---|---|
| 상태 코드 (`.ok()`, `.status(...)`) | HTTP 200, 404, 500 등 |
| 헤더 (`.headers(...)`) | Content-Type, Content-Disposition 등 |
| 콘텐츠 길이 (`.contentLength(...)`) | 응답 본문 크기 |
| 본문 (`.body(...)`) | 실제 데이터 (객체, Resource 등) |

#### `<?>` — 와일드카드 제네릭

`ResponseEntity<Notice>`처럼 타입을 특정하면 그 타입만 body로 넣을 수 있습니다.  
파일 다운로드는 본문이 `Resource`이지만, 에러 시 String이나 ErrorDto일 수도 있어서 **`<?>`로 타입을 열어둡니다.**

---

### 2-2. URL 인코딩 (한글 파일명 처리)

```java
String encodeFileName = URLEncoder.encode(dto.getOriginalFilename(), StandardCharsets.UTF_8);
```

#### 🔑 왜 필요한가?

HTTP 헤더 값에 한글이나 공백이 들어가면 브라우저가 다운로드 파일명을 깨지게 표시합니다.

- 원본: `회의록.pdf`
- 인코딩 후: `%ED%9A%8C%EC%9D%98%EB%A1%9D.pdf`

이렇게 URL-safe한 형태로 변환해야 Content-Disposition 헤더에 안전하게 들어갑니다.

---

## 3. MyBatis 개념

### 3-1. `${}` vs `#{}` — 가장 중요한 구분

#### `#{}` — 파라미터 바인딩 (PreparedStatement)

```xml
WHERE NUM = #{num}
```

내부적으로는 이렇게 처리됩니다.

```sql
WHERE NUM = ?
-- 그리고 별도로 ?에 num 값을 바인딩
```

✅ **SQL 인젝션 안전**  
✅ 값이 자동으로 따옴표 처리됨 (문자열의 경우)  
✅ **거의 모든 경우에 이걸 써야 함**

#### `${}` — 문자열 치환 (String Concatenation)

```xml
WHERE ${columnName} = #{value}
```

내부적으로는 이렇게 처리됩니다.

```sql
-- 만약 columnName="NUM"이면
WHERE NUM = ?
-- 만약 columnName="1=1 OR 1"이면 (SQL 인젝션!)
WHERE 1=1 OR 1 = ?
```

⚠️ **SQL 인젝션 위험**  
⚠️ 사용자 입력값에 절대 사용하면 안 됨  
✅ **컬럼명, 테이블명처럼 SQL 문법 구조 자체를 동적으로 바꿔야 할 때만 사용**

#### 🔑 사용 기준

| 상황 | 사용 |
|---|---|
| 값 비교 (WHERE NUM = ?) | `#{}` |
| 정렬 컬럼 동적 선택 (ORDER BY ?) | `${}` 만 가능 (컬럼명은 바인딩 불가) |
| 테이블명 동적 선택 | `${}` 만 가능 |
| 사용자 입력 검색어 | 무조건 `#{}` |

#### 우리 코드의 사례

```xml
<!-- BEFORE: 값에도 ${}를 써서 인젝션 위험 -->
WHERE ${fileId} = ${num}

<!-- AFTER: 값은 #{}로 안전하게 -->
WHERE ${fileId} = #{num}
```

`${fileId}`는 컬럼명을 의미하므로 `${}` 사용이 맞습니다. 단, **서비스에서 fileId 값을 "NUM"이나 "FILENUM"같이 허용 목록(화이트리스트)으로만 제한**해야 합니다.

---

### 3-2. `resultType`에는 반드시 Fully Qualified Name

```xml
<!-- ❌ 잘못된 경로 -->
<select id="listNoticeTop" resultType="com.doit.app.Notice">

<!-- ✅ 올바른 경로 -->
<select id="listNoticeTop" resultType="com.doit.app.model.Notice">
```

#### 🔑 핵심 개념

MyBatis는 `resultType`에 적힌 클래스명을 **문자열로 그대로 받아서 리플렉션으로 클래스를 찾습니다.**  
실제 패키지 경로와 한 글자라도 다르면 `ClassNotFoundException`이 런타임에 발생합니다.

#### 💡 typeAlias 활용

매번 풀 경로 쓰기 귀찮으면 `mybatis-config.xml`에 별칭 등록:

```xml
<typeAliases>
    <typeAlias alias="Notice" type="com.doit.app.model.Notice"/>
</typeAliases>
```

그러면 매퍼에서 짧게 쓸 수 있습니다.

```xml
<select id="..." resultType="Notice">
```

---

### 3-3. INSERT문의 컬럼 순서 ↔ 값 순서 일치

#### ❌ 우리가 했던 실수

```xml
INSERT INTO NOTICE (NUM, NAME, SUBJECT, CONTENT, NOTICE, HITCOUNT, REGDATE)
VALUES (#{num}, #{notice}, #{name}, #{subject}, #{content}, 0, SYSDATE)
```

매핑을 일렬로 비교해보면:

| 컬럼 | 값 | 의도 | 결과 |
|---|---|---|---|
| NUM | #{num} | num | ✅ |
| NAME | #{notice} | name | ❌ NAME 컬럼에 0/1 들어감 |
| SUBJECT | #{name} | subject | ❌ |
| CONTENT | #{subject} | content | ❌ |
| NOTICE | #{content} | notice | ❌ NOTICE(NUMBER)에 텍스트 → SQL 에러 |

#### ✅ 올바른 코드

```xml
INSERT INTO NOTICE (NUM, NAME, SUBJECT, CONTENT, NOTICE, HITCOUNT, REGDATE)
VALUES (#{num}, #{name}, #{subject}, #{content}, #{notice}, 0, SYSDATE)
```

#### 🔑 예방 팁

- 컬럼과 값을 **세로로 정렬해서 작성**하면 눈으로 확인하기 쉽습니다.
- 가능하면 IDE의 SQL 검증 기능 활용 (Eclipse DBeaver, IntelliJ Database 등).
- **단위 테스트 (실제 INSERT 후 SELECT로 확인)** 가 가장 확실합니다.

---

### 3-4. `<selectKey>` — 시퀀스로 키 미리 생성

```xml
<insert id="insertNotice" parameterType="com.doit.app.model.Notice">
    <selectKey keyProperty="num" resultType="Long" order="BEFORE">
        SELECT NOTICE_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO NOTICE (NUM, ...) VALUES (#{num}, ...)
</insert>
```

#### 🔑 동작 흐름

1. `<selectKey>` 실행 → 시퀀스에서 다음 값 가져옴
2. 결과를 Notice 객체의 `num` 필드에 저장 (`keyProperty="num"`)
3. INSERT 실행 시 `#{num}`으로 그 값 사용
4. INSERT 끝난 후에도 Notice 객체의 `num`에 값이 남아있어서, **다음 작업(예: 첨부파일 INSERT 시 FK로 사용)에 활용 가능**

`order="BEFORE"`는 INSERT 전에 실행한다는 의미. Oracle은 BEFORE, MySQL은 보통 AFTER (AUTO_INCREMENT라).

---

## 4. Exception Handling 패턴

### 4-1. 빈 catch 블록의 위험성

#### ❌ 안티패턴

```java
catch (Exception e) {
    // 비어있음
}
```

**예외가 발생했는데 흔적도 없이 사라집니다.** 디버깅이 거의 불가능해집니다.

#### ✅ 최소한 이렇게

```java
catch (Exception e) {
    log.error("작업 실패: ", e);   // 무조건 로그
    throw e;                        // 필요시 다시 던지기
}
```

#### 🔑 빈 catch가 정당화되는 매우 드문 경우

- 정말로 무시해도 되는 상황 (예: 임시 파일 삭제 실패 - 어차피 OS가 정리)
- 그래도 **주석으로 "왜 무시하는지" 명시**해야 합니다.

---

### 4-2. Java 7의 Precise Rethrow

#### 미스터리한 코드

```java
public int dataCount(Map<String, Object> map) {  // throws 없음
    try {
        return mapper.dataCount(map);
    } catch (Exception e) {
        log.info("dataCount : ", e);
        throw e;   // ??? Exception을 던지는데 throws 없어도 컴파일됨?
    }
}
```

#### 🔑 Java 7+의 Precise Rethrow 기능

컴파일러가 **try 블록 안에서 실제로 어떤 예외가 발생할 수 있는지 분석**합니다.

- `mapper.dataCount(map)`이 `throws`를 선언하지 않음 → RuntimeException만 발생 가능
- catch에서 잡은 `e`는 선언상 `Exception`이지만 **실제로는 RuntimeException** 임을 컴파일러가 추론
- `throw e`는 RuntimeException을 던지는 것으로 처리되어 throws 선언 없이 OK

#### 조건

- `e`가 final이거나 effectively final이어야 함 (재할당하면 이 기능 안 됨)
- try 블록에서 발생 가능한 예외 종류가 좁혀져야 함

#### 💡 권장 스타일

이 기능을 이해하더라도, **메서드 시그니처에 `throws Exception`을 명시**하는 게 코드 가독성에 좋습니다.

---

### 4-3. throw e의 의미

```java
try {
    something();
} catch (Exception e) {
    log.error("...", e);
    throw e;   // ⚡ 메서드 정상 종료가 아님
}
```

`throw`는 **메서드를 비정상 종료**시킵니다. 따라서:

1. 다음 줄 코드 실행 안 함
2. 메서드의 return 분석에서 "여기서 끝남"으로 처리됨
3. 호출자에게 예외가 전파됨

이런 특성 때문에 catch에서 `throw e`만 하면 메서드 끝에 return이 없어도 컴파일러는 만족합니다.

---

## 5. HTTP 응답 구성

### 5-1. Content-Disposition 헤더 문법

#### ❌ 흔한 실수

```java
"attatchment: filename=\"" + encodeFileName + "\""
//     ^^                ^
//     오타            세미콜론이어야 함
```

#### ✅ 정확한 문법

```java
"attachment; filename=\"" + encodeFileName + "\""
```

#### 🔑 HTTP 헤더 값 문법

```
Content-Disposition: <type>; <param1>=<value1>; <param2>=<value2>
```

- `<type>`: `inline`(브라우저에서 표시) 또는 `attachment`(다운로드)
- 파라미터는 `;`로 구분
- 값에 공백이나 특수문자가 있으면 `"`로 감쌈

---

### 5-2. MIME 타입 — `application/octet-stream`

```java
headers.add(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_OCTET_STREAM_VALUE);
```

#### 🔑 의미

- `application/octet-stream` = "이진 데이터, 타입 알 수 없음"
- 브라우저가 "어떻게 처리해야 할지 모르겠으니 다운로드 시킨다"고 판단
- **파일 다운로드용 기본 MIME 타입**

만약 PDF만 다룬다면 `application/pdf`처럼 구체적인 타입을 줘도 됩니다.

---

## 6. 흔히 하는 실수 체크리스트

코드 작성 후 self-review할 때 확인할 항목들입니다.

### Java

- [ ] `import` 경로가 의도한 패키지인가? (특히 동명의 다른 클래스 주의)
- [ ] 메서드 호출(`.`)을 인자 구분(`,`)으로 잘못 쓰지 않았나?
- [ ] try-catch에서 모든 경로에 return이 있는가? (또는 throw)
- [ ] catch 블록이 비어있지 않은가? 최소한 로그는 찍는가?
- [ ] `return null` 했는데 실제로는 값을 반환해야 하는 곳은 아닌가?
- [ ] null 체크 빠진 곳은 없는가? (특히 컬렉션, 외부 입력)
- [ ] 문자열 처리에서 `lastIndexOf`가 -1 반환할 경우 대비했는가?

### MyBatis

- [ ] `${}` 위치가 정말 컬럼명/테이블명인가? 값에 쓰지 않았나?
- [ ] `resultType`의 패키지 경로가 정확한가?
- [ ] INSERT의 컬럼 순서와 VALUES 순서가 일치하는가?
- [ ] `parameterType`이 실제 전달 타입과 맞는가?
- [ ] CDATA 섹션이 필요한 SQL (`<`, `>`, `&`) 처리는 했는가?

### Spring

- [ ] `HttpHeaders`를 `org.springframework.http`에서 import 했는가?
- [ ] ResponseEntity의 상태 코드가 적절한가?
- [ ] `Content-Disposition`의 문법 (`;`, `attachment`)이 정확한가?
- [ ] 한글 파일명은 URLEncoder로 인코딩했는가?

### 통합

- [ ] Service ↔ Mapper ↔ XML 사이의 메서드명/파라미터명이 일치하는가?
- [ ] DB 컬럼명과 Java 필드명 매핑이 맞는가? (대소문자, 언더스코어)

---

## 🎓 마무리: 학습 우선순위

지금 시점에서 가장 먼저 익혀야 할 개념 3가지:

### 1순위: MyBatis `${}` vs `#{}`
→ 보안과 정확성에 직결. 이걸 모르면 SQL 인젝션 취약점을 만듦.

### 2순위: try-catch 흐름과 return 경로
→ 컴파일 에러의 가장 흔한 원인. 디버깅 시간을 줄이려면 필수.

### 3순위: import 패키지 구분
→ 같은 이름의 다른 클래스를 구분 못 하면 시간 낭비 큼.  
→ Eclipse `Ctrl + Shift + O` 단축키와 함께 익힐 것.

---

## 📎 부록: 우리가 발견한 9개 버그 요약

| # | 파일 | 줄 | 분류 | 변경 |
|---|------|-----|------|------|
| 1 | Java | 193 | 로직 | `insertNotice` → `insertNoticeFile` |
| 2 | Java | 204 | 안티패턴 | 빈 catch → log 추가 |
| 3 | Java | 253~254 | 로직 | map key/value 수정 |
| 4 | Java | 331 | 반환값 | `return null` → `return list` |
| 5 | Java | 380 | 반환값 | `return null` → `return dto` |
| 6 | Java | 395 | 반환값 | `return null` → `return dto` |
| 7 | XML | 69 | 오타 | `model.` 누락 추가 |
| 8 | XML | 109 | 순서 | INSERT VALUES 순서 정정 |
| 9 | XML | 261 | 보안 | `${num}` → `#{num}` |

---

> 💪 **다음에 또 마주칠 거예요. 그땐 5초 안에 찾읍시다.**
