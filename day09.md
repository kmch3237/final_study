# Day 9 — 매니저 임명 풀스택 + 노쇼 프로시저 + 권한 시스템 완성

## 📌 오늘 한 작업 한 줄 요약
사용자 5개 담당 파트 마지막인 **매니저 임명 풀스택** + **출석체크 노쇼 프로시저(OUT 파라미터)** + **AuthUtil 권한 시스템(5단계 분기)** 마무리. 학원에서 안 배운 인터셉터/Map은 정적 메서드/객체 바인딩으로 우회.

---

## 1. AuthUtil 정적 메서드 패턴 (인터셉터 회피)

### 왜 만들었나
모든 `/owner/**` URL이 비로그인/일반회원 차단 + 사장/매니저 권한 분기 필요. `HandlerInterceptor` 학원에서 안 배웠으니 **정적 메서드로 같은 효과**.

### 코드
```java
public class AuthUtil {

    // 사장만 통과. 비로그인=로그인페이지, 일반회원=메인
    public static String checkOwner(HttpSession session) {
        String role = (String) session.getAttribute("role");
        if (role == null)            return "redirect:/user/login";
        if (!"OWNER".equals(role))   return "redirect:/theme/list";
        return null;  // 통과
    }

    // 사장 + 매니저 통과
    public static String checkStaff(HttpSession session) {
        String role = (String) session.getAttribute("role");
        if (role == null)            return "redirect:/user/login";
        if (!"OWNER".equals(role) && !"MANAGER".equals(role)) return "redirect:/theme/list";
        return null;
    }
}
```

### 컨트롤러 사용
```java
@GetMapping("/manager")
public String manager(HttpSession session, Model model) {
    String redirect = AuthUtil.checkOwner(session);
    if (redirect != null) return redirect;   // 차단 시 그 redirect 그대로 반환
    // 통과 시 본문 진행
    ...
}
```

### 핵심 포인트
- 반환 타입 `String` = redirect 경로 또는 null
- `null` = "차단 안 함" 신호
- 컨트롤러는 한 줄로 권한 체크 끝
- `"OWNER".equals(role)` 순서 = role이 null이어도 NullPointerException 안 남

---

## 2. 출석체크 노쇼 프로시저 (OUT 파라미터)

### 비즈니스 룰
노쇼 처리 시 해당 USER 매너온도 -1점 차감 + 갱신된 매너온도 화면에 표시.

### PL/SQL 프로시저
```sql
CREATE OR REPLACE PROCEDURE SP_INSERT_NOSHOW(
    P_USER_ID  IN  NUMBER,
    P_NEW_TEMP OUT NUMBER
)
IS
BEGIN
    -- (1) MANNER_HISTORY에 노쇼 사유로 INSERT (SCORE=-1)
    INSERT INTO MANAGER_HISTORY (MANAGER_HISTORY_ID, USER_ID, REASON_ID, SCORE)
    VALUES (MANAGER_HISTORY_SEQ.NEXTVAL, P_USER_ID, 1, -1);

    -- (2) 다른 분(JY)이 만든 FN_GET_MANNER 함수 호출해 갱신 매너온도 OUT
    P_NEW_TEMP := FN_GET_MANNER(P_USER_ID);
END;
/
```

### MyBatis CALLABLE
```xml
<update id="insertNoshow" statementType="CALLABLE" parameterType="com.noexit.app.model.Manner">
    { CALL SP_INSERT_NOSHOW(
        #{userId,  mode=IN,  jdbcType=NUMERIC},
        #{newTemp, mode=OUT, jdbcType=NUMERIC}
    ) }
</update>
```

### Service (학원에서 Map 안 배워서 객체 바인딩)
```java
public Manner noshow(Long userId) throws Exception {
    Manner manner = new Manner();
    manner.setUserId(userId);       // IN

    mapper.insertNoshow(manner);    // 호출 후 manner.newTemp 자동 채워짐

    return manner;
}
```

### 핵심 포인트
- `statementType="CALLABLE"` = 프로시저 호출 모드
- `mode=IN` / `mode=OUT` = OUT 파라미터는 같은 객체에 채워져 돌아옴
- VO 1개로 IN+OUT 다 처리 (Map 안 씀)
- 다른 팀원(JY)이 만든 `FN_GET_MANNER` 함수 호출 = **팀 협업 코드 활용**

---

## 3. V_ACTIVE_MANAGER 뷰 (복잡한 조인 캡슐화)

### 왜 만들었나
활성 매니저 = "각 (CAFE_ID, USER_ID) 그룹별 최근 행이 REG_EVENT_ID=1인 케이스" — 4테이블 조인 + 서브쿼리. 같은 SQL이 두 곳(매니저 목록 + role 판정)에서 쓰여서 **뷰로 한 번만 정의**.

### VIEW
```sql
CREATE OR REPLACE VIEW V_ACTIVE_MANAGER AS
SELECT mh.MANAGER_HISTORY_ID, mh.CAFE_ID, mh.USER_ID,
       mh.CREATED_AT,
       c.USER_ID  AS OWNER_USER_ID,   -- 사장 검색용
       c.CAFE_NAME,
       ua.LOGIN_ID, ua.NICKNAME,
       ui.PHONE
  FROM MANAGER_HISTORY mh
  JOIN CAFE         c  ON c.CAFE_ID  = mh.CAFE_ID
  JOIN USER_ACCOUNT ua ON ua.USER_ID = mh.USER_ID
  LEFT JOIN USER_INFO ui ON ui.USER_ID = mh.USER_ID
 WHERE mh.MANAGER_HISTORY_ID IN (
       SELECT MAX(MANAGER_HISTORY_ID) FROM MANAGER_HISTORY GROUP BY CAFE_ID, USER_ID
   )
   AND mh.REG_EVENT_ID = 1;
```

### 매퍼 단순화
```xml
<!-- 11줄 → 1줄 -->
<select id="selectActiveByOwnerUserId" parameterType="long" resultType="com.noexit.app.model.Manager">
    SELECT * FROM V_ACTIVE_MANAGER WHERE OWNER_USER_ID = #{ownerUserId} ORDER BY CREATED_AT DESC
</select>

<select id="countActiveByUserId" parameterType="long" resultType="int">
    SELECT COUNT(*) FROM V_ACTIVE_MANAGER WHERE USER_ID = #{userId}
</select>
```

---

## 4. findRole 데이터 파생 권한 (OWNER > MANAGER > USER)

### 왜 컬럼이 없나
팀 스키마에 USER_ACCOUNT.ROLE 같은 컬럼 없음. 권한은 **CAFE.USER_ID(소유)** 와 **MANAGER_HISTORY(임명 이력)** 으로부터 데이터 파생.

### 우선순위
```java
@Override
public String findRole(Long userId) {
    try {
        if (userMapper.countCafeByUserId(userId) > 0)        return "OWNER";    // 카페 1개라도 소유 = 사장
        if (managerService.countActiveByUserId(userId) > 0)  return "MANAGER";  // 활성 매니저
    } catch (Exception e) {
        log.info("findRole : ", e);
    }
    return "USER";   // 기본
}
```

### 핵심 포인트
- 사장이 동시에 매니저인 경우(본인 카페에 본인 임명) **사장 먼저** 반환
- 매니저 체크는 ManagerService 경유 — 학원 룰 `Controller→Service→Mapper` 일관

---

## 5. @RequestParam(name=) 명시 패턴

### 왜 명시해야 하나
Spring Boot 3+ 컴파일러 옵션 `-parameters` 없으면 reflection으로 파라미터 이름 못 찾음.

### 빈 파라미터 = 500 에러
```java
// ❌ 500: "Name for argument of type [String] not specified"
public String managerEnroll(String loginId, Long cafeId, ...) { ... }

// ✅ 학원 Theme.list 패턴
public String managerEnroll(
    @RequestParam(name = "loginId") String loginId,
    @RequestParam(name = "cafeId")  Long   cafeId,
    ...
) { ... }
```

### 두 가지 학원 패턴 정리
| 패턴 | 언제 |
|---|---|
| 객체 바인딩 (`User user`, `Cafe cafe`) | 폼 필드 많을 때 (회원가입 등) |
| `@RequestParam(name=)` 명시 | 검색/페이징/단일 파라미터 (Theme.list, 매니저 임명) |

---

## 6. 2단계 권한 차단 (JSP + Java)

같은 권한 룰이 두 곳에서 일치해야 함:

### ① JSP에서 메뉴 가리기
```jsp
<%-- 헤더 --%>
<c:if test="${role == 'OWNER'}">
    <li><a href=".../owner/res/open">테마관리</a></li>
</c:if>
<c:if test="${role == 'MANAGER'}">
    <li><a href=".../owner/attendance">출석체크</a></li>
</c:if>

<%-- 사이드바 ownerSide.jsp --%>
<c:if test="${role == 'OWNER'}">
    <li><a href=".../owner/theme/enroll">테마 등록</a></li>
    <li><a href=".../owner/manager">매니저 관리</a></li>
</c:if>
```

### ② Java에서 URL 직접 차단
```java
@GetMapping("/manager")
public String manager(HttpSession session, Model model) {
    String redirect = AuthUtil.checkOwner(session);
    if (redirect != null) return redirect;
    ...
}
```

### 핵심 포인트
- JSP만 가리면 URL 직접 입력으로 우회 가능
- Java만 차단하면 권한 없는 메뉴가 보임 (UX 안 좋음)
- **두 곳 모두 막아야 진짜 권한 시스템**

---

## 7. 회원가입 9단계 유효성 (학원 jQuery 패턴)

### 검증 흐름
가입하기 클릭 → `#error` span 빨간 글씨로 단계별 검증:

1. 아이디 중복 확인 했는지 (isIdChecked)
2. 비밀번호 8자 이상
3. 비밀번호 확인 일치
4. 닉네임 2~10자
5. 이름 필수
6. 이메일 형식 (`/^[\w.+-]+@[\w-]+(\.[\w-]+)+$/`)
7. 연락처 11자리 숫자 (`/^\d{11}$/`)
8. 성별 선택
9. 생년월일 선택

### 아이디 유효성 (중복확인 단계)
```js
if (!/^(?=.*[a-zA-Z])[a-zA-Z0-9]{6,15}$/.test(loginId)) {
    $("#error").html("아이디는 6~15자 영문+숫자(영문 1자 이상 필수)로 입력해주세요.")
               .css({color: "red", display: "inline"});
    return;
}
```

### 핵심 포인트
- `(?=.*[a-zA-Z])` = positive lookahead, 영문 1자 이상 포함 강제 (숫자만 차단)
- alert 안 씀, 학원 `#error` span 패턴 ([[feedback-no-alert-red-text]])
- 검증 순서 = 위에서 아래 (위 통과 시만 다음)

---

## 8. RedirectAttributes.addFlashAttribute

### 왜 쓰나
`redirect` 후 화면에 한 번만 표시되는 메시지가 필요. 보통 모델은 redirect 시 다 사라지는데 **Flash는 살아남음**.

### 노쇼 처리 예
```java
@PostMapping("/attendance/noshow")
public String noshow(@RequestParam(name = "userId") Long userId
                   , HttpSession session
                   , RedirectAttributes ra) {
    String redirect = AuthUtil.checkStaff(session);
    if (redirect != null) return redirect;

    try {
        Manner result = attendanceService.noshow(userId);
        ra.addFlashAttribute("resultMessage",
            "노쇼 처리 완료. 매너온도: " + result.getNewTemp());
    } catch (Exception e) {
        log.info("noshow : ", e);
        ra.addFlashAttribute("resultMessage", "노쇼 처리 실패");
    }
    return "redirect:/owner/attendance";
}
```

### JSP에서 받기
```jsp
<c:if test="${not empty resultMessage}">
    <div class="alert alert-info">${resultMessage}</div>
</c:if>
```

### 핵심 포인트
- `addAttribute` ≠ `addFlashAttribute`
- 전자는 URL 파라미터, 후자는 세션에 임시 저장 후 한 번 읽으면 사라짐
- 새로고침해도 또 안 뜸 (한 번만)

---

## 9. UserMapper.findByLoginId 분리 (의도가 다른 SQL은 메서드 분리)

### 왜 분리했나
기존 `selectByLoginId(User user)`는 **로그인용**이라 LOGIN_ID + 암호화된 PASSWORD 둘 다 매치해야 함:
```sql
SELECT ... FROM USER_ACCOUNT
 WHERE LOGIN_ID = #{loginId}
   AND PASSWORD = CRYPTPACK.ENCRYPT(#{password}, '12341234')
```

매니저 임명에서 PW 없이 호출하면 `CRYPTPACK.ENCRYPT(null, ...)` 비교 실패 → null 반환 → "해당 아이디 회원 없음" 오인.

### 해결
```java
// 로그인용 (PW 비교) — 기존
public User selectByLoginId(User user);

// LOGIN_ID만 비교 (매니저 임명 등) — 신규
public User findByLoginId(String loginId);
```

```xml
<select id="findByLoginId" parameterType="string" resultType="com.noexit.app.model.User">
    SELECT USER_ID, LOGIN_ID, NICKNAME, CREATED_AT
    FROM USER_ACCOUNT
    WHERE LOGIN_ID = #{loginId}
</select>
```

### 핵심 포인트
- **의도가 다른 SQL은 메서드도 분리** (DRY보다 명확성 우선)
- 함수명에 의도 드러내기 (`select` = 로그인 검증, `find` = 단순 식별)

---

## 🐛 트러블슈팅

### Bug 1: 500 "Name for argument of type [String]"
- **원인**: Spring Boot 3+ 컴파일러 `-parameters` 옵션 미설정
- **해결**: `@RequestParam(name = "...")` 명시 또는 객체 바인딩

### Bug 2: STS 모든 view 빨간불
- **원인**: 새 파일이 디스크에는 있지만 Eclipse Workspace 인덱스에 안 들어감 → 컴파일 실패 → JSP 연쇄 빨간불
- **해결**: F5 (Refresh) 또는 Project → Clean
- **포인트**: 실행은 잘 됨 (Tomcat은 디스크 직접 읽음). STS 빨간불 ≠ 실제 컴파일 에러

### Bug 3: 매니저 등록 시 "해당 아이디 회원 없음"
- **원인**: `selectByLoginId`가 PW 비교 쿼리라 PW null이면 매치 실패
- **해결**: `findByLoginId(String)` 분리 (위 9번)

### Bug 4: catch 안에 셀렉트박스 model 4개 채워야 하는 이유
- forward(`return "theme/themeEnrollForm"`) 시 GET에서 채운 model이 살아남지 않음
- 셀렉트박스 비면 사용자 다시 등록 불가
- 해결: `private void loadFormData()` 헬퍼 메서드로 추출 권장 (DRY)

---

## 📊 데이터 검증 (DB 시뮬레이션)

### 5단계 role 시뮬레이션
```sql
SELECT ua.USER_ID, ua.LOGIN_ID,
       (SELECT COUNT(*) FROM CAFE WHERE USER_ID = ua.USER_ID) AS CAFE_CNT,
       (SELECT COUNT(*) FROM V_ACTIVE_MANAGER WHERE USER_ID = ua.USER_ID) AS MGR_CNT,
       CASE
         WHEN (SELECT COUNT(*) FROM CAFE WHERE USER_ID = ua.USER_ID) > 0 THEN 'OWNER'
         WHEN (SELECT COUNT(*) FROM V_ACTIVE_MANAGER WHERE USER_ID = ua.USER_ID) > 0 THEN 'MANAGER'
         ELSE 'USER'
       END AS ROLE
  FROM USER_ACCOUNT ua
 ORDER BY ua.USER_ID;
```

### 테스트 계정 (비번 전부 `12341234`)
| Role | LOGIN_ID |
|---|---|
| 비로그인 | (로그아웃 상태) |
| USER (일반) | user0009 |
| MANAGER (매니저) | user0014 |
| OWNER (사장) | user0002 |
| ADMIN (관리자) | admin001 |

---

## 🎤 면접 톤 정리 (한 줄씩)

1. **AuthUtil** — "인터셉터 대신 정적 메서드로 같은 효과. 컨트롤러 첫 줄 한 줄로 권한 체크."
2. **노쇼 프로시저** — "INSERT 후 다른 팀원 함수(FN_GET_MANNER) 호출해 OUT으로 갱신값 반환. MyBatis CALLABLE."
3. **V_ACTIVE_MANAGER 뷰** — "4테이블 조인 + 그룹별 MAX(ID) 서브쿼리를 뷰로 캡슐화, 매퍼 1줄."
4. **findRole** — "USER에 ROLE 컬럼 없이 CAFE/MANAGER_HISTORY로 매번 파생. 사장 우선."
5. **@RequestParam(name=)** — "Spring Boot 3+ 컴파일러 옵션 없어도 동작하도록 이름 명시."
6. **2단계 차단** — "JSP c:if (UI) + Java AuthUtil (URL) 양쪽 일치."
7. **회원가입 9단계** — "학원 jQuery #error span + 빨간 글씨. 정규식 lookahead로 영문 1자 강제."
8. **RedirectAttributes** — "redirect 후 한 번만 표시 (Flash). 새로고침 안 떠."
9. **메서드 분리** — "로그인 검증과 단순 식별은 의도가 다르니 SQL/메서드도 분리."

---

## 🔜 D-Day 남은 작업

| 우선순위 | 작업 |
|---|---|
| 🥇 | 5권한 시연 리허설 (5계정 순차 클릭) |
| 🥈 | VIEW 도입 팀원 카톡 공유 ("V_ACTIVE_MANAGER 만들었음") |
| 보너스 | findId 단순 구현, 카페등록 파일처리, 카페관리 |

---

**오늘 핵심**: 사용자 5개 담당 파트 풀스택 + 권한 시스템 완성 + 다른 팀원 코드(`FN_GET_MANNER`) 활용. 학원에서 안 배운 인터셉터/Map은 우회 패턴(AuthUtil 정적 메서드 / 객체 바인딩)으로 해결.
