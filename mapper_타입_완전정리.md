# Mapper 리턴타입 & 파라미터타입 완전 정리
작성일: 2026-05-31  
기준: Spring_Boot_Study2 (sb01, sb02) + finalProject 실제 코드

---

## 핵심 원칙

```
리턴타입  → SQL이 뭘 돌려주느냐
파라미터  → SQL에 뭘 넘겨주느냐
```

---

## 1. 리턴타입

### void — 결과 확인 안 해도 될 때

```java
// sb01 MemberMapper.java
public void insertMember(Member dto) throws Exception;

// sb02 NoticeMapper.java
public void insertNotice(Notice dto) throws SQLException;
public void updateNotice(Notice dto) throws SQLException;
public void deleteNotice(long num) throws SQLException;
public void updateHitCount(long num) throws SQLException;
```

- INSERT / UPDATE / DELETE 후 **성공 여부를 굳이 확인 안 해도 될 때**
- 실무에서는 `int` 를 더 많이 씀 (결과 확인 가능하니까)

---

### int — 영향받은 행 수 확인할 때

```java
// finalProject MemberMapper.java
public int insertAccount(Member member);
public int insertInfo(Member member);

// finalProject CafeMapper.java
public int insertCafe(Cafe cafe);
```

- INSERT / UPDATE / DELETE 실행 후 **성공이면 1, 실패면 0** 반환
- 서비스나 컨트롤러에서 결과 확인할 때 유용

```java
// 컨트롤러에서 결과 확인 예시
int result = cafeService.insertCafe(cafe);
if (result > 0) {
    // 성공
} else {
    // 실패
}
```

---

### int — COUNT 쿼리 (숫자 하나 조회)

```java
// sb01 MemberMapper.java
public int dataCount() throws Exception;

// sb02 NoticeMapper.java
public int dataCount(Map<String, Object> map);
```

```xml
<!-- memberMapper.xml -->
<select id="dataCount" resultType="Integer">
    SELECT COUNT(*) AS COUNT
    FROM TESTMEMBER
</select>

<!-- NoticeMapper.xml -->
<select id="dataCount" parameterType="map" resultType="integer">
    SELECT COUNT(*)
    FROM NOTICE
    <where>
        <if test="kwd != null and kwd != ''">
            <include refid="where-list" />
        </if>
    </where>
</select>
```

- `COUNT(*)` 처럼 **숫자 하나만 뽑을 때** → `int` 또는 `Integer`
- 페이징 처리할 때 전체 글 수 구할 때 많이 씀

---

### VO타입 — 한 행 조회

```java
// sb02 NoticeMapper.java
public Notice findById(long num);
public Notice findByPrev(Map<String, Object> map);
public Notice findByNext(Map<String, Object> map);
public Notice findByFileId(long fileNum);
```

```xml
<!-- NoticeMapper.xml -->
<select id="findById" parameterType="long" resultType="com.doit.app.model.Notice">
    SELECT NUM, NOTICE, NAME, SUBJECT, CONTENT, HITCOUNT,
           TO_CHAR(REGDATE,'YYYY-MM-DD') AS "regDate"
    FROM NOTICE
    WHERE NUM = #{num}
</select>
```

- SELECT 결과가 **딱 1건**일 때 → VO 타입 반환
- 상세보기, 이전글, 다음글 조회에 사용

---

### List\<VO\> — 여러 행 조회

```java
// sb01 MemberMapper.java
public List<Member> listMember() throws Exception;

// sb02 NoticeMapper.java
public List<Notice> listNoticeTop();
public List<Notice> listNotice(Map<String, Object> map);
public List<Notice> listNoticeFile(long num);
```

```xml
<!-- memberMapper.xml -->
<select id="listMember" resultType="com.doit.app.domain.Member">
    SELECT NUM, NAME, TEL
    FROM TESTMEMBER
    ORDER BY NUM
</select>
```

- SELECT 결과가 **여러 건**일 때 → `List<VO>` 반환
- 목록 조회, 첨부파일 목록 등에 사용
- JSP에서 `<c:forEach>` 로 반복 출력

---

### 리턴타입 한눈에 보기

| 리턴타입 | SQL | 예시 |
|---------|-----|------|
| `void` | INSERT/UPDATE/DELETE (결과 확인 불필요) | insertNotice, updateHitCount |
| `int` | INSERT/UPDATE/DELETE (결과 확인 필요) | insertCafe, insertAccount |
| `int` | COUNT 쿼리 | dataCount |
| `VO타입` | SELECT 1건 | findById, findByPrev |
| `List<VO>` | SELECT 여러 건 | listMember, listNotice |
| `String` | 문자열 하나 조회 | 아이디 중복체크 등 |

---

## 2. 파라미터타입

### VO (DTO) — 여러 필드를 한번에 넘길 때

```java
// 주로 INSERT, UPDATE
public void insertMember(Member dto);
public void insertNotice(Notice dto);
public int insertCafe(Cafe cafe);
```

```xml
<insert id="insertMember" parameterType="com.doit.app.domain.Member">
    INSERT INTO TESTMEMBER(NUM, NAME, TEL)
    VALUES(TESTMEMBER_SEQ.NEXTVAL, #{name}, #{tel})
</insert>
```

- 넘길 값이 **3개 이상**이고 VO가 있을 때
- `#{필드명}` 으로 VO의 getter 자동 호출

---

### long / Long — 값 하나만 넘길 때

```java
// 주로 DELETE, 조회
public void deleteNotice(long num);
public void updateHitCount(long num);
public Notice findById(long num);
public List<Notice> listNoticeFile(long num);
```

```xml
<delete id="deleteNotice" parameterType="long">
    DELETE FROM NOTICE
    WHERE NUM = #{num}
</delete>

<update id="updateHitCount" parameterType="long">
    UPDATE NOTICE
    SET HITCOUNT = HITCOUNT + 1
    WHERE NUM = #{num}
</update>
```

- 넘길 값이 **ID 하나**뿐일 때
- `parameterType="long"` → XML에서 `#{num}` (이름은 아무거나 써도 됨)

---

### @Param — 값 2~3개 넘길 때

```java
// finalProject MemberMapper.java
public Member selectByLoginIdAndPassword(
    @Param("loginId") String loginId,
    @Param("password") String password
);
```

```xml
<select id="selectByLoginIdAndPassword" resultType="com.doit.app.model.Member">
    SELECT USER_ID, LOGIN_ID, NICKNAME AS NICK_NAME
    FROM USER_ACCOUNT
    WHERE LOGIN_ID = #{loginId} AND PASSWORD = #{password}
</select>
```

- 넘길 값이 **2~3개**인데 딱 맞는 VO가 없을 때
- `@Param("이름")` → XML에서 `#{이름}` 으로 사용
- parameterType 생략 가능

---

### Map — 여러 값 + 타입이 다를 때 (주로 페이징, 검색)

```java
// sb02 NoticeMapper.java
public int dataCount(Map<String, Object> map);
public List<Notice> listNotice(Map<String, Object> map);
public Notice findByPrev(Map<String, Object> map);
public Notice findByNext(Map<String, Object> map);
public void deleteNoticeFile(Map<String, Object> map);
```

---

#### Map<String, Object> 란?

Map은 **이름표(Key) + 값(Value)** 쌍으로 데이터를 담는 바구니다.

```
Map<String, Object>
     ↑       ↑
   Key 타입  Value 타입
  (이름표)   (아무 타입이나 가능)
```

실생활로 비유하면 **사물함**과 같다.
```
사물함 번호(Key)  →  내용물(Value)
"kwd"           →  "맛집"         (String)
"offset"        →  0              (int)
"size"          →  10             (int)
"schType"       →  "subject"      (String)
```

---

#### 어떻게 만들고 넣고 꺼내나

```java
// 1. Map 만들기
Map<String, Object> map = new HashMap<>();

// 2. 값 넣기 — map.put("이름표", 값)
map.put("kwd", "맛집");        // 검색어
map.put("schType", "subject"); // 검색 타입
map.put("offset", 0);          // 페이지 시작점
map.put("size", 10);           // 한 페이지 개수

// 3. 값 꺼내기 — map.get("이름표")
String kwd = (String) map.get("kwd");   // "맛집"
int offset = (int) map.get("offset");   // 0
```

---

#### XML에서는 #{이름표} 로 꺼냄

```xml
<select id="listNotice" parameterType="map" resultType="com.doit.app.model.Notice">
    SELECT NUM, NOTICE, NAME, SUBJECT, HITCOUNT
    FROM NOTICE
    <where>
        <if test="kwd != null and kwd != ''">
            <include refid="where-list" />
        </if>
    </where>
    ORDER BY NUM DESC OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
    <!--                        ↑ map.put("offset", 0) 에서 꺼냄  -->
</select>
```

- `#{offset}` → `map.get("offset")` 과 같은 의미
- `map.put("key", 값)` 할 때 쓴 이름표를 XML에서 `#{이름표}` 로 그대로 사용

---

#### 서비스에서 넘기는 전체 흐름

```java
// NoticeServiceImpl.java
public List<Notice> listNotice(String schType, String kwd, int offset, int size) {

    // 값이 4개인데 @Param 쓰기 애매하고, 딱 맞는 VO도 없음
    // → Map에 담아서 한번에 넘김
    Map<String, Object> map = new HashMap<>();
    map.put("schType", schType);
    map.put("kwd", kwd);
    map.put("offset", offset);
    map.put("size", size);

    return noticeMapper.listNotice(map);  // Map 통째로 넘김
}
```

---

#### 언제 Map을 쓰나 vs 다른 방법 비교

| 상황 | 방법 |
|------|------|
| 값 1개 | `long num` 그냥 넘김 |
| 값 2~3개, VO 있음 | VO 넘김 |
| 값 2~3개, VO 없음 | `@Param` 사용 |
| 값 4개 이상, 타입 다양 | `Map<String, Object>` |
| 페이징 + 검색 조건 같이 | `Map<String, Object>` |

---

#### 왜 Object인가?

```java
Map<String, String> → 값이 전부 String일 때만 가능
Map<String, Object> → String, int, Long 뭐든 다 담을 수 있음
```

```java
map.put("kwd", "맛집");   // String
map.put("offset", 0);     // int
map.put("num", 123L);     // Long
// → Object 이기 때문에 다 담을 수 있음
```

---

### 파라미터타입 한눈에 보기

| 파라미터타입 | 넘기는 값 | 언제 씀 |
|------------|---------|---------|
| `VO (Cafe, Member...)` | 여러 필드 | INSERT, UPDATE (필드 많을 때) |
| `long` / `Long` / `String` | 값 1개 | DELETE, 단건 조회 |
| `@Param` | 값 2~3개 | 로그인 (아이디+비번), 2개짜리 조건 |
| `Map<String, Object>` | 여러 값 (타입 혼합) | 페이징+검색, VO 없을 때 |
| 없음 (생략) | 없음 | 전체 목록 조회 (조건 없을 때) |

---

## 3. 실제 패턴 정리 — 기능별

```java
// 등록 (필드 많음 → VO)
public int insertCafe(Cafe cafe);
public void insertNotice(Notice dto);

// 삭제 (ID 하나 → long)
public void deleteNotice(long num);
public int deleteCafe(long cafeId);

// 단건 조회 (ID → VO 반환)
public Notice findById(long num);
public Cafe selectCafe(long cafeId);

// 목록 조회 (파라미터 없거나 Map → List 반환)
public List<Member> listMember();
public List<Notice> listNotice(Map<String, Object> map);

// 개수 조회 (없거나 Map → int 반환)
public int dataCount();
public int dataCount(Map<String, Object> map);

// 조회수/상태 업데이트 (ID 하나 → void 또는 int)
public void updateHitCount(long num);
public int updateCafeStatus(long cafeId);

// 로그인 (값 2개 → @Param → VO 반환)
public Member selectByLoginIdAndPassword(
    @Param("loginId") String loginId,
    @Param("password") String password
);
```

---

> **한 줄 요약**  
> 넘길 값이 많으면 VO, 하나면 그 타입, 2~3개면 @Param, 조건이 복잡하면 Map  
> 받을 값이 없으면 void/int, 하나면 그 타입, 1건이면 VO, 여러건이면 List
