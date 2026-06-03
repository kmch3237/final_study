# Day 5 — 로그인 풀스택 + 세션 + role 파생 설계

> 2026-06-03 · 회원가입에 이어 **로그인 풀스택 4계층 완성** + 강사님 try/catch 패턴 도입 + 권한(role)을 컬럼이 아닌 **다른 테이블 데이터로 파생**하는 설계 결정. 면접에서 내 코드 설명할 수 있게 정리.

---

## 0. 오늘의 전체 흐름

```
[로그인 폼]                                  [로그인 후 세션]
loginForm.jsp                                session = {
   ↓ POST /user/login                          "loginUser" : User 객체,
UserController.login(User user, HttpSession)   "role"      : "USER" or "OWNER"
   ↓                                         }
UserService.login(user)
   ├─ userMapper.selectByLoginId(loginId)   ← DB에서 아이디로 회원 조회
   └─ password 비교 (Java에서)
   ↓ User dto 반환 (실패 시 null)
UserService.findRole(userId)
   └─ userMapper.countCafeByUserId(userId) ← CAFE 테이블에 본인 USER_ID 있나
   ↓ "OWNER" or "USER" 반환
session.setAttribute("loginUser", dto)
session.setAttribute("role", role)
   ↓
redirect:/
```

오늘 핵심 = **로그인 풀스택 + 권한 파생 + 세션 저장**까지 한 흐름으로 묶음.

---

## 1. 로그인 — Mapper 단

회원가입 때 만든 `countByLoginId` 옆에 한 줄 더.

### UserMapper.java (인터페이스)

```java
public int countByLoginId(String loginId);       // 회원가입 때 만든 거 (중복확인)
public User selectByLoginId(String loginId);     // ← 오늘 추가 (로그인)
public int countCafeByUserId(Long userId);       // ← 오늘 추가 (role 파생)
```

### userMapper.xml

```xml
<select id="selectByLoginId" parameterType="string"
        resultType="com.noexit.app.model.User">
    SELECT USER_ID, LOGIN_ID, PASSWORD, NICKNAME, CREATE_AT
    FROM USER_ACCOUNT
    WHERE LOGIN_ID = #{loginId}
</select>

<select id="countCafeByUserId" parameterType="long" resultType="int">
    SELECT COUNT(*) FROM CAFE WHERE USER_ID = #{userId}
</select>
```

### 왜 SELECT에 `AND PASSWORD = #{password}` 안 넣었나

| 방식 | SELECT 쿼리 | 문제점 |
|---|---|---|
| A. SQL에서 비교 | `WHERE LOGIN_ID=? AND PASSWORD=?` | 나중에 비밀번호 암호화 붙이면 SQL에서 비교 불가능 |
| **B. Java에서 비교** ✅ | `WHERE LOGIN_ID=?` | DB에서 꺼낸 뒤 Java에서 비교 → 학원 sp14~16 복호화 패턴 그대로 적용 가능 |

> 면접 톤: "USER_ACCOUNT.PASSWORD는 암호화 컬럼이라(스키마 주석에 명시) DB 비교가 아니라 애플리케이션단에서 비교하는 패턴을 적용했습니다."

### 함정 — CREATE_AT (단수형!)

```sql
SELECT ..., CREATE_AT     ← 팀 스키마 컬럼명. CREATED_AT 아님!
```

→ Day 4에 메모해뒀던 함정 그대로. 다른 도메인 쿼리 짤 때도 계속 헷갈리니까 한 번 더 짚어둠.

---

## 2. 로그인 — Service 단 (강사님 try/catch 패턴 도입)

### 강사님 패턴이 뭔지

학원 `sb02_FileUploadNotice/NoticeServiceImpl.java` 패턴 참고:

| 어노테이션 | 역할 |
|---|---|
| `@Service` | Spring 빈 등록 |
| `@RequiredArgsConstructor` | `private final` 필드 생성자 자동 주입 (Lombok) |
| `@Slf4j` | `log.info("...", e)` 변수 자동 생성 (Lombok) |

메서드 본문 패턴:

```java
public 반환타입 메서드명(...) {
    반환타입 result = null;        // 기본값 null
    try {
        result = mapper.xxx();
    } catch (Exception e) {
        log.info("메서드명 : ", e);
        // 조회(SELECT) 계열은 throw e 안 함
        // INSERT/UPDATE/DELETE 계열은 throw e 함 (트랜잭션 롤백 위해)
    }
    return result;
}
```

### UserService 인터페이스

```java
public interface UserService {
    public void enroll(User user);
    public int countByLoginId(String loginId);
    public User login(User user);             // ← 오늘 추가
    public String findRole(Long userId);      // ← 오늘 추가
}
```

### UserServiceImpl — login()

```java
@Service
@RequiredArgsConstructor
@Slf4j                                                  // ← Lombok 로그
public class UserServiceImpl implements UserService {

    private final UserMapper userMapper;

    @Override
    public User login(User user) {
        User dto = null;
        try {
            dto = userMapper.selectByLoginId(user.getLoginId());

            if (dto != null && ! dto.getPassword().equals(user.getPassword())) {
                dto = null;       // 아이디 있고 비번 다름 → 실패 처리
            }
        } catch (Exception e) {
            log.info("login : ", e);
        }
        return dto;
    }
}
```

### 왜 try/catch?

- DB는 언제든 죽을 수 있음 (네트워크 끊김, 락, SQL 오타)
- 예외가 컨트롤러까지 그대로 가면 사용자 화면이 **500 에러 (흰 화면)**
- `log.info`로 개발자가 추적할 단서를 남기고, 사용자에겐 "정상 실패 메시지"로 응답

→ 강사님이 모든 DB 메서드를 try/catch로 감싼 이유.

### SELECT는 왜 throw 안 해?

| 메서드 종류 | throw e | 이유 |
|---|---|---|
| INSERT/UPDATE/DELETE | ✅ | 트랜잭션 롤백 시키려면 위로 던져야 함 |
| **SELECT** (login, findRole, listXxx) | ❌ | 조회 실패는 "없으면 그만". 상위로 안 던져도 됨 |

> 면접 톤: "조회 메서드는 swallow, 변경 메서드는 rethrow하는 강사님 컨벤션을 따랐습니다. 트랜잭션 롤백 책임 분리 때문입니다."

---

## 3. 권한(role) — 컬럼 추가 ❌, 데이터 파생 ✅

### 문제 상황

원래 계획: `USER_ACCOUNT.ROLE` 컬럼 추가해서 'USER'/'OWNER' 저장.

### 팀 스키마 확인 결과

```sql
CREATE TABLE USER_ACCOUNT (
    USER_ID NUMBER,
    LOGIN_ID VARCHAR2(50),
    PASSWORD VARCHAR2(500),
    NICKNAME VARCHAR2(50),
    CREATE_AT DATE,
    ...
);
-- ROLE 컬럼 없음!
```

```sql
CREATE TABLE CAFE (
    CAFE_ID NUMBER,
    USER_ID NUMBER NOT NULL,    ← 카페 사장 USER_ID
    ...
);

CREATE TABLE ADMIN_ACCOUNT (    ← 관리자는 아예 별도 테이블
    ...
);
```

### 결론 — 팀이 의도적으로 정규화한 설계

| 권한 | 어디서 옴 |
|---|---|
| **USER** (일반) | `USER_ACCOUNT`에 그냥 있음 |
| **OWNER** (사장) | `CAFE.USER_ID`에 본인 USER_ID 있음 → 사장 |
| MANAGER (매니저) | `MANAGER_HISTORY`에 등록 이벤트 있음 |
| ADMIN (관리자) | 별도 `ADMIN_ACCOUNT` 테이블 |

→ **role은 컬럼이 아니라 "다른 테이블 존재 여부"로 파생됨.**

### 그래서 어떻게 풀었나 — findRole()

```java
@Override
public String findRole(Long userId) {
    String role = "USER";                  // 기본값
    try {
        int cafeCount = userMapper.countCafeByUserId(userId);
        if (cafeCount > 0) {
            role = "OWNER";                // CAFE에 본인 USER_ID 있으면 사장
        }
    } catch (Exception e) {
        log.info("findRole : ", e);
    }
    return role;
}
```

> 면접 톤: "권한을 USER_ACCOUNT 컬럼으로 두지 않고 CAFE 테이블 존재 여부로 파생했습니다. 팀 스키마가 의도적으로 정규화한 설계라 컬럼 추가는 팀 합의를 깨는 일이고, 도메인 의미상으로도 '카페를 가진 사람 = 사장'이 자연스럽기 때문입니다."

### 만약 컬럼 추가했다면 안 좋았을 점

1. 팀 DDL 멋대로 변경 → git push 시 충돌
2. 데이터 중복: ROLE='OWNER'인데 CAFE에 없음? → 어느 쪽이 진실?
3. 이력서/면접: "그냥 컬럼 박았어요"보다 "데이터 파생으로 처리했어요"가 훨씬 좋음

---

## 4. Controller — 세션 저장 패턴

### UserController.login() POST 본문

```java
@PostMapping("/login")
public String login(User user, HttpSession session) {
    User dto = null;
    try {
        dto = service.login(user);
    } catch (Exception e) {
        log.info("login : ", e);
    }

    if (dto == null) {
        return "redirect:/user/login";           // 실패 → 로그인 폼
    }

    String role = service.findRole(dto.getUserId());

    session.setAttribute("loginUser", dto);
    session.setAttribute("role", role);

    return "redirect:/";                          // 성공 → 메인
}
```

### HttpSession은 어디서 나왔나 — Spring 매직

| 강사님 sp15 옛날 방식 | Spring Boot 어노테이션 방식 |
|---|---|
| `request.getSession()` 호출 | 메서드 파라미터 `HttpSession session` 적으면 끝 |

→ `User user`, `HttpServletResponse response`, `HttpSession session`, `Model model` 다 같은 원리. **Spring이 알아서 꽂아주는 박스들.**

### 세션 키 컨벤션 정함

| 키 | 값 타입 | 용도 |
|---|---|---|
| `loginUser` | `User` 객체 | 닉네임/아이디 등 JSP에서 꺼내 쓰기 |
| `role` | `String` "USER"/"OWNER" | 인터셉터에서 권한 분기 |

> 면접 톤: "세션 키 이름을 컨벤션으로 고정해서 JSP·인터셉터·다른 컨트롤러에서 같은 이름으로 꺼낼 수 있게 했습니다."

---

## 5. logout — invalidate()

```java
@GetMapping("/logout")
public String logout(HttpSession session) {
    session.invalidate();              // ← 세션 전체 폭파
    return "redirect:/user/login";
}
```

`invalidate()` = 세션에 담긴 모든 값 + 세션 자체 폐기. 로그아웃의 정석.

---

## 6. 트러블슈팅 기록

### `selectByLoginId` 빨간 줄

- **증상**: `UserServiceImpl`에서 `userMapper.selectByLoginId(...)` 빨간 줄
- **원인**: 인터페이스 `UserMapper.java`에 메서드 선언이 없었음 (저장 누락)
- **해결**: 인터페이스에 `public User selectByLoginId(String loginId);` 추가

### NPE 함정 (외워두기)

```java
// ❌ 위험 — role이 null이면 NullPointerException
role.equals("OWNER")

// ✅ 안전 — 문자열 상수가 먼저
"OWNER".equals(role)
```

→ 비로그인 상태에서 `session.getAttribute("role")` 결과는 null. 인터셉터/JSP에서 비교할 때 항상 **문자열 상수.equals(변수)** 순서.

---

## 7. 오늘 만든/수정한 파일 목록

| 파일 | 변경 |
|---|---|
| `UserMapper.java` | `selectByLoginId`, `countCafeByUserId` 추가 |
| `userMapper.xml` | `selectByLoginId`, `countCafeByUserId` SELECT 추가 |
| `UserService.java` | `login`, `findRole` 인터페이스 |
| `UserServiceImpl.java` | `@Slf4j` 도입, `login()`, `findRole()` 구현 |
| `UserController.java` | `HttpSession` 주입, `login()` POST 본문, `logout()` invalidate, role 세션 저장 |

---

## 8. 내일 할 것 — `/owner/**` 인터셉터

```
브라우저: GET /owner/cafeEnroll
         ↓
   OwnerInterceptor.preHandle()  ← 세션 role 체크
         ↓
   "OWNER".equals(role) ? 통과 : redirect:/user/login
         ↓
   OwnerController
```

만들 파일:
1. `com.noexit.app.interceptor.OwnerInterceptor` — `HandlerInterceptor` 구현, `preHandle()`에서 role 체크
2. `com.noexit.app.config.WebMvcConfig` — `WebMvcConfigurer` 구현, `addInterceptors()`로 `/owner/**` 패턴에 등록

`@Component`로 인터셉터 빈 등록 → Config에서 주입받아 등록. preHandle return값 의미: `true`=통과, `false`=차단. 차단 시 `response.sendRedirect(...)` 호출 필수 (안 하면 흰 화면).

---

## 9. 오늘 배운 핵심 3줄

1. **try/catch + @Slf4j**는 DB 작업의 기본 옷차림. 화면이 500 에러로 죽지 않게 막는 안전장치.
2. **role은 컬럼이 아니라 다른 테이블로 파생**할 수 있다. 정규화된 스키마에선 이 선택이 자연스럽다.
3. **HttpSession을 메서드 파라미터로 받으면 Spring이 자동 주입**한다. `request.getSession()` 안 써도 됨.
