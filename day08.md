# Day 8 (2026-06-04) — 풀스택을 백지에서 짜는 사고 절차

> 오늘 목표: "코드를 보면 이해는 가는데, 백지에서 어떤 순서로 머릿속 그림을 그리고 어디부터 손대야 할지 모르겠다"는 막힘을 해소.
>
> 이 노트는 **무엇을 짰는지(WHAT)** 보다 **왜 그렇게 짰고, 백지에서 어떤 순서로 떠올렸는지(WHY/HOW)** 에 집중한다.

---

## 0. 오늘 한 일 요약

| # | 영역 | 결과 |
|---|---|---|
| 1 | **관리자 로그인 풀스택** | AdminController(POST/세션/role) + Service + Mapper + 학원 PL/SQL 복호화 패키지 연결 |
| 2 | **테마등록 폼 — 카페 셀렉트박스 동적 조회** | hidden cafeId → DB에서 본인 카페 목록 SELECT → 셀렉트박스 |
| 3 | **로그인 시 role 세션 저장** | findRole(userId) → "OWNER"/"USER" 파생 → session.role |
| 4 | **header.jsp 권한 분기** | CAFE 메뉴를 사장(OWNER)일 때만 노출 |
| 5 | **MyBatis `map-underscore-to-camel-case` 설정** | USER_ID → userId 자동 매핑 (이게 빠져서 ORA-17004 났음) |

---

## 1부. 백지에서 풀스택 짤 때의 사고 절차 (일반화)

### 1-1. 풀스택을 "건물 짓기"로 비유하기

방탈출 사이트 같은 웹앱은 **7층 건물**이라고 생각한다. 사용자가 보는 층(JSP)이 꼭대기, DB가 지하실. 한 기능(예: 회원가입, 로그인, 카페 등록)은 한 채의 **수직 슬라이스**다 — 1층부터 7층까지 한 줄로 관통하는 엘리베이터를 새로 놓는 일.

```
┌──────────────────────────────────────┐
│  7층  JSP        (사용자가 보는 화면) │  ← 폼/리스트/상세
├──────────────────────────────────────┤
│  6층  Controller (URL ↔ 메서드)       │  ← @GetMapping, @PostMapping
├──────────────────────────────────────┤
│  5층  Service    (비즈니스 로직)      │  ← @Service, @Transactional
├──────────────────────────────────────┤
│  4층  Mapper IF  (자바 인터페이스)    │  ← @Mapper
├──────────────────────────────────────┤
│  3층  mapper.xml (SQL)                │  ← <select>, <insert>
├──────────────────────────────────────┤
│  2층  VO (Model) (데이터 컨테이너)    │  ← Lombok @Getter/@Setter
├──────────────────────────────────────┤
│  1층  Oracle DB  (실제 데이터)        │  ← USER_ACCOUNT, CAFE 등
└──────────────────────────────────────┘
```

기능마다 7개 층이 다 있어야 사용자가 화면에서 클릭한 동작이 DB까지 갔다가 다시 화면으로 돌아온다.

### 1-2. 백지에서 짤 때의 사고 순서 — Top-Down (위에서 아래로)

학원에서 본 모범 패턴, 그리고 오늘 우리가 실제로 따라간 순서:

```
① 화면(JSP)부터 그린다 — 사용자가 무엇을 입력하고 무엇을 보는가?
       ↓
② URL을 결정한다 — 폼이 어디로 POST 되는가? 결과 화면 URL은?
       ↓
③ Controller에 빈 메서드 자리를 만든다 — 어떤 객체를 받고, 어디로 보낼지
       ↓
④ Service 인터페이스 + Impl 자리를 만든다
       ↓
⑤ Mapper 인터페이스 + xml에 SQL을 짠다 (← DB 스키마 보고)
       ↓
⑥ VO를 만든다 (SQL이 어떤 필드를 채울지 확정된 다음에)
       ↓
⑦ 위 ①~⑥을 하나로 연결 — Controller에서 Service 호출, Service에서 Mapper 호출, JSP에 모델 담기
```

**왜 화면부터?** 사용자가 보는 결과(input/output)를 먼저 고정해야 그 안쪽을 무엇으로 채울지 결정된다. 안쪽부터 짜면 화면에서 필요한 정보랑 안 맞아서 다시 짜야 한다.

### 1-3. 디버깅할 때의 사고 순서 — Bottom-Up (아래에서 위로)

에러가 나면 보통 **마지막 층(SQL)** 에서 터진다. 거기서 시작해 한 층씩 거슬러 올라가며 "이 층까지는 값이 잘 들어왔나?" 점검한다.

예: 오늘 ORA-17004 에러 → "SQL 파라미터가 null" → "왜 null?" → "loginUser.getUserId()가 null" → "왜 null?" → "MyBatis가 USER_ID → userId 매핑을 안 함" → "왜 안 함?" → "yml 설정 누락"

**5번 "왜?"를 던지면 진짜 원인이 나온다** (Toyota 5 Whys 기법).

---

## 2부. 사례 1 — 관리자 로그인 풀스택 (사고 과정)

### 2-1. 머릿속 그림 그리기 — 3단계

백지 상태에서 "관리자 로그인을 만들자" 했을 때 머릿속에서 떠올린 그림:

**① 데이터 흐름 (어디서 무엇이 어디로?)**

```
[브라우저: adminLoginForm.jsp]
  ↓ POST /admin/login (loginId + password)
[AdminController.login(Admin admin, HttpSession, Model)]
  ↓ service.login(admin)
[AdminServiceImpl.login(Admin admin)]
  ↓ mapper.findByLoginId(admin.getLoginId())
[AdminMapper.findByLoginId(String)]
  ↓ SQL 실행
[ADMIN_ACCOUNT 테이블]
  ↑ 결과 row (ADMIN_ID, LOGIN_ID, PASSWORD)
[AdminMapper → Admin dto]
  ↑ .equals()로 비밀번호 비교
[AdminServiceImpl → null 또는 Admin]
  ↑ null이면 폼으로, 아니면 세션에 박고 redirect
[AdminController → 세션 + redirect]
```

**② User 로그인이랑 뭐가 같고 뭐가 다른가? (이미 만든 패턴 활용)**

| 항목 | User 로그인 (Day 7) | Admin 로그인 (오늘) | 차이 |
|---|---|---|---|
| Mapper 메서드 | `selectByLoginId(User)` | `findByLoginId(String)` | 이름만 다름, 같은 패턴 |
| 비교 위치 | WHERE 절에 `CRYPTPACK.ENCRYPT(#{password}) = PASSWORD` | SELECT 컬럼에 `CRYPTPACK.DECRYPT(PASSWORD)` | 학원 패턴 두 갈래 |
| 자바 비교 | 없음 (SQL이 다 함) | `dto.getPassword().equals(input.getPassword())` | 위 차이의 연쇄 |
| 세션 키 | `loginUser` | `loginAdmin` | 객체 타입이 다름 |
| role 값 | "USER" 또는 "OWNER" (파생) | "ADMIN" 고정 | 관리자는 단일 |
| 회원가입 | 있음 | 없음 (SQL로 직접 INSERT) | 도메인 단순화 |

→ **결론: User 로그인 패턴을 복사하되, 이름과 세션 키만 바꾼다.**

**③ 작업 단위로 쪼개기 (몇 파일?)**

- AdminController.java (POST 메서드 추가)
- AdminService.java (인터페이스, 메서드 1개)
- AdminServiceImpl.java (구현)
- adminMapper.xml (SELECT 1개)
- adminLoginForm.jsp (이미 존재, errorMessage 표시만 추가)

→ **5개 파일, 1시간 안에 끝남.**

### 2-2. 단계별로 짚어보기

#### 단계 A — Mapper xml (가장 안쪽부터 / 또는 정찰부터)

```xml
<select id="findByLoginId" parameterType="string"
        resultType="com.noexit.app.model.Admin">
    SELECT ADMIN_ID,
           LOGIN_ID,
           CRYPTPACK.DECRYPT(PASSWORD, '12341234') AS PASSWORD
    FROM ADMIN_ACCOUNT
    WHERE LOGIN_ID = #{loginId}
</select>
```

**왜 `CRYPTPACK.DECRYPT(PASSWORD, '12341234') AS PASSWORD`?**

학원 PL/SQL 표준 패키지 `CRYPTPACK`이 DB에 깔려있고, 키 `'12341234'`로 AES-256 복호화. DB에 저장된 PASSWORD는 32자 hex(예: `AB90DDB...`), 평문은 `12341234`.

- DECRYPT 결과를 `PASSWORD`라는 alias로 다시 내보내면 → 자바 `Admin.password` 필드에 **평문**이 들어옴.
- 그러면 자바에서 `.equals()`로 평문끼리 비교 가능.

**왜 SELECT에 박았나? WHERE에 박을 수도 있는데?**

학원에는 두 패턴이 있다:

```sql
-- 패턴 A (오늘 Admin에 적용)
SELECT ..., CRYPTPACK.DECRYPT(PASSWORD, '키') AS PASSWORD
FROM ...
WHERE LOGIN_ID = #{loginId}
-- → 자바에서 .equals() 비교

-- 패턴 B (User에 이미 적용됨)
SELECT ...
FROM ...
WHERE LOGIN_ID = #{loginId}
  AND PASSWORD = CRYPTPACK.ENCRYPT(#{password}, '키')
-- → 자바 비교 없음. 결과 0건이면 실패
```

| | 패턴 A | 패턴 B |
|---|---|---|
| SQL 복잡도 | 컬럼 한 자리만 함수 | WHERE 한 줄 추가 |
| 자바 코드 | `.equals()` 비교 필요 | 결과 null 체크만 |
| 보안 | 평문이 자바에 흘러옴 | 평문이 자바에 안 옴 |

오늘은 일관성보다 진척을 우선해서 A로 갔지만, **나중에 B로 통일하는 게 정석**이다. 면접에서 "왜 두 가지를 섞었어요?" 물으면 "MVP 단계에서 진척을 우선한 결정, 추후 일관성 위해 B로 통일 예정"이라고 답할 수 있다.

#### 단계 B — Service + Impl

```java
// AdminService.java (인터페이스)
public interface AdminService {
    Admin login(Admin admin);
}

// AdminServiceImpl.java
@Service
@RequiredArgsConstructor
@Slf4j
public class AdminServiceImpl implements AdminService {

    private final AdminMapper mapper;

    @Override
    public Admin login(Admin admin) {
        Admin result = null;
        try {
            Admin dto = mapper.findByLoginId(admin.getLoginId());
            if (dto != null && dto.getPassword().equals(admin.getPassword())) {
                result = dto;
            }
        } catch (Exception e) {
            log.info("login : ", e);
        }
        return result;
    }
}
```

**왜 `@RequiredArgsConstructor + private final` 패턴?** 강사님 표준 옷차림. 컴파일 타임에 의존성 누락이 잡히고, 생성자 주입이 불변성을 보장.

**왜 try-catch?** 학원 패턴 (NoticeServiceImpl 모범) 그대로. SELECT 계열이라 `throw e` 없음(INSERT/UPDATE/DELETE는 트랜잭션 롤백을 위해 `throw e` 필요).

**왜 `result = null` 초기화 후 if 안에서 채우는 패턴?**
- 실패해도 항상 result는 정의된 상태 → NullPointerException 안 남
- 컨트롤러는 `result == null`로 실패 판정 가능
- 한 메서드 = 한 return문 (가독성)

#### 단계 C — Controller

```java
@Controller
@Slf4j
@RequiredArgsConstructor
@RequestMapping("/admin")
public class AdminController {

    private final AdminService service;

    @GetMapping("/login")
    public String loginForm() {
        return "admin/adminLoginForm";
    }

    @PostMapping("/login")
    public String login(Admin admin, HttpSession session, Model model) {
        Admin dto = service.login(admin);

        if (dto == null) {
            model.addAttribute("errorMessage", "아이디 또는 비밀번호가 올바르지 않습니다.");
            return "admin/adminLoginForm";
        }

        session.setAttribute("loginAdmin", dto);
        session.setAttribute("role", "ADMIN");
        return "redirect:/";
    }

    @GetMapping("/logout")
    public String logout(HttpSession session) {
        session.invalidate();
        return "redirect:/";
    }
}
```

**왜 실패는 `return "admin/adminLoginForm"` (forward), 성공은 `return "redirect:/"` (redirect)?**

- 실패 = **같은 페이지에서 에러 메시지 보여주기** → Model에 담은 errorMessage를 폼에서 그대로 출력해야 함 → **forward** (URL 그대로, Model 살아있음)
- 성공 = **다른 페이지로 이동 + 새로고침 시 POST 재요청 방지** → **redirect** (URL 변경, 새 GET 요청)

**Post-Redirect-Get(PRG) 패턴**. 외워둘 것. 면접 단골.

**왜 세션에 `loginAdmin`이랑 `role` 둘 다 넣지?**

- `loginAdmin`: 객체 자체. JSP에서 `${loginAdmin.loginId}` 같이 꺼내쓸 때 필요.
- `role`: 문자열. JSP에서 `${role == 'ADMIN'}` 분기 조건에 쓸 때 가볍게.

`role`은 `loginAdmin != null`로 추론할 수도 있지만, 일반 사장님(loginUser+role=OWNER)이랑 통일된 분기 문법을 쓰려고 별도 키로 분리.

#### 단계 D — JSP (errorMessage 표시)

```jsp
<form action="${pageContext.request.contextPath}/admin/login" method="post">
    <div class="mb-3">
        <label for="loginId" class="form-label">관리자 아이디</label>
        <input type="text" id="loginId" name="loginId" class="form-control">
    </div>
    <div class="mb-3">
        <label for="password" class="form-label">비밀번호</label>
        <input type="password" id="password" name="password" class="form-control">
        <c:if test="${not empty errorMessage}">
            <div class="text-danger">${errorMessage}</div>
        </c:if>
    </div>
    <button type="submit" class="btn btn-dark w-100">관리자 로그인</button>
</form>
```

**왜 `alert()` 안 쓰고 `text-danger` 빨간 글씨?**

- alert는 흐름을 끊고 사용자 짜증을 유발
- 폼 옆 빨간 글씨 = 어디가 잘못됐는지 즉각 알려줌
- 강사님 표준 패턴

### 2-3. 함정 노트 (오늘 부딪힌 것)

#### 함정 1 — adminMapper.xml이 User 매퍼 복붙된 상태

처음 열어보니:
```xml
<mapper namespace="com.noexit.app.mapper.AdminMapper">
    <insert id="insertAccount" parameterType="com.noexit.app.model.User"> ...
    <insert id="insertInfo" parameterType="com.noexit.app.model.User"> ...
    <select id="countByLoginId" ...
    <select id="selectByLoginId" resultType="...User"> ...
    <select id="countCafeByUserId" ...
</mapper>
```

namespace는 Admin인데 안에 든 SQL은 전부 User. 그리고 `AdminMapper.java` 인터페이스는 `findByLoginId(String)` 1개만. → **xml에 그 id가 없어서 호출 시 `BindingException` 터짐.**

→ 학원에서 시간 부족할 때 옆 파일 복붙해놓고 까먹은 상태. **새 매퍼 만들 때 namespace는 바꿨는지, IF 메서드와 xml id가 1:1로 맞는지 더블체크**.

#### 함정 2 — 컨트롤러 이름 `Admin`이 모델 `Admin`과 충돌

```java
@Controller
public class Admin {  // 컨트롤러 이름
    // 안에서 com.noexit.app.model.Admin을 import할 수 없음
    // (같은 클래스 안에서 자기 이름과 같은 import는 불가능)
}
```

→ `AdminController`로 rename (STS의 `Refactor → Rename` 사용). 클래스명 변경 시 **파일명도 같이 바뀜** (Java 규칙: public class 이름 = 파일 이름).

**왜 STS Refactor를 쓰나? 손으로 바꿔도 되잖아.**
- 클래스명을 참조하는 모든 곳(다른 파일의 import, 생성자 호출 등)을 자동 동시 변경
- 손으로 하면 한 군데 빠뜨려서 NoSuchBeanDefinitionException 날 수 있음

#### 함정 3 — PASSWORD가 평문이 아니었다

DB에 `AB90DDBDEB450C515CD2A7B87750420B` — 32자 hex.
- 16바이트 × 2 = 32 hex chars
- AES 1블록(16바이트) 출력 → **암호화돼있음**

평문 `12341234` 입력 → 자바에서 `.equals("AB90...")` → 항상 false → 로그인 무조건 실패.

→ 학원 패키지 `CRYPTPACK.DECRYPT(PASSWORD, '키')`로 풀어야 함. 그리고 키 `'12341234'`도 학원 표준값. 운영 코드면 keystore/환경변수로 빼야 하지만 학원 프로젝트라 인라인 OK.

### 2-4. 면접 답변 시나리오

**Q: "관리자 로그인을 어떻게 구현하셨나요?"**

A: 회원과 관리자가 별도 테이블(USER_ACCOUNT vs ADMIN_ACCOUNT)이고 도메인이 다르기 때문에, 컨트롤러도 `UserController`와 `AdminController`로 분리했습니다. 세션 키도 `loginUser`/`loginAdmin`으로 분리해서 JSP에서 분기를 명확하게 했습니다.

비밀번호는 학원 표준 PL/SQL 패키지 `CRYPTPACK`로 AES-256 암호화되어 DB에 저장돼있어서, SELECT 컬럼에 `CRYPTPACK.DECRYPT()`를 걸어 평문으로 받아온 뒤 자바에서 `.equals()` 비교했습니다. 일반 회원 로그인은 학원의 또 다른 패턴인 WHERE 절 `ENCRYPT` 비교 방식을 써서, 두 가지 학원 패턴을 모두 경험해봤습니다.

성공 시 `loginAdmin` 객체와 `role="ADMIN"` 문자열을 세션에 동시에 박는 이유는, 객체는 JSP에서 사용자 정보 표시용, 문자열은 권한 분기용으로 역할을 나눠서 가독성을 높이기 위함입니다.

---

## 3부. 사례 2 — 테마등록 폼 카페 셀렉트박스 (사고 과정)

### 3-1. "화면 → 데이터" 거꾸로 추론

테마 등록 폼에 카페 셀렉트박스를 추가한다고 했을 때, 머릿속 추론 순서:

```
화면: 셀렉트박스가 보임
  → 셀렉트박스 = <option> 여러 개
    → <option> 여러 개 = List 데이터 필요
      → 각 option = 카페 한 개 (id, name)
        → List<Cafe>를 model에 담아야 함
          → Controller가 SELECT를 호출해야 함
            → CafeMapper.selectByUserId(userId)가 필요함
              → Cafe VO도 필요함
                → 어느 사용자의 카페? 세션 loginUser.userId
```

**핵심: 화면이 보여주는 데이터의 모양으로부터 거꾸로 내려가면 무엇을 만들어야 하는지 자동으로 도출된다.**

### 3-2. hidden vs 셀렉트박스 — 보안 사고

원래는 hidden cafeId를 박아두려고 했다 (`<input type="hidden" name="cafeId" value="${loginCafeId}">`). 셀렉트박스로 바꾼 이유:

**hidden의 문제:**
- F12 개발자도구로 value를 다른 값으로 바꿀 수 있음
- 단일 카페만 가정한 설계 (한 사장이 여러 카페 가질 수 있다면 한계)
- "자동 주입"이 모호 — 어디서 누가 박는지 코드 따라가야 함

**셀렉트박스의 장점:**
- 사장님 본인 소유 카페만 옵션에 노출 → 다른 카페 선택 시도 자체가 화면에 보이지 않음
- 여러 카페 소유 가능 (test01은 카페 4개 보유)
- 컨트롤러에서 명시적으로 `cafeMapper.selectByUserId(userId)` 호출 → 누가 어디서 박는지 명확

**그래도 셀렉트박스가 완전 안전한가? 아님.**
- F12로 `<option value>`를 다른 cafeId로 바꿀 수 있음
- → **서버에서 다시 검증해야 함**: POST 받은 cafeId가 정말 그 사장님 소유 카페인지 한 번 더 SQL로 체크
- 이건 카페 등록 풀스택 작업 때 추가 예정

### 3-3. 단계별로 짚어보기

#### 단계 A — Cafe VO 신규 (2층 데이터 컨테이너)

```java
@Getter @Setter @NoArgsConstructor
public class Cafe {
    private Long cafeId;
    private Long userId;
    private String brNo;
    private String cafeName;
    private String phone;
    private String postalCode;
    private String address;
    private String addressDetail;
    private Date createdAt;
}
```

**왜 모든 컬럼을 다 필드로?** 카페 풀스택을 나중에 짤 때 등록 폼/상세 화면/수정 등에서 다 필요. 한 번 만들고 재사용.

**왜 camelCase 필드명?** Java 표준. DB는 snake_case(CAFE_ID, USER_ID). MyBatis의 `map-underscore-to-camel-case: true` 설정이 자동 변환.

#### 단계 B — CafeMapper IF + xml

```java
@Mapper
public interface CafeMapper {
    List<Cafe> selectByUserId(Long userId);
}
```

```xml
<mapper namespace="com.noexit.app.mapper.CafeMapper">
    <select id="selectByUserId" parameterType="long"
            resultType="com.noexit.app.model.Cafe">
        SELECT CAFE_ID, USER_ID, BR_NO, CAFE_NAME, PHONE,
               POSTAL_CODE, ADDRESS, ADDRESS_DETAIL, CREATED_AT
        FROM CAFE
        WHERE USER_ID = #{userId}
        ORDER BY CAFE_ID
    </select>
</mapper>
```

**왜 `List<Cafe>` 반환?** 사장님이 여러 카페 소유 가능. 0개일 수도 있음 (빈 List 반환, null 아님).

**왜 `ORDER BY CAFE_ID`?** 셀렉트박스에서 옵션 순서가 매번 바뀌면 UX 나쁨. 안정적인 순서 보장.

#### 단계 C — Theme.java 컨트롤러 보완 (⚠️ 다른 분 코드 안 건드리기)

이 부분이 오늘의 **가장 중요한 학습 포인트** 중 하나.

`Theme.java`는 **다른 팀원이 작성한 컨트롤러**. 내가 추가한 부분은 `enrollForm()` 메서드 4줄뿐. 다른 메서드(`themeListPage`, `themeListItem`, `themeDetail`)와 클래스 헤더(`@Controller`, `@RequestMapping`, 옛 생성자, `CafeController cafeController` 필드)는 **건드리면 안 됨**.

그런데 `enrollForm()`에 `cafeMapper`를 호출하려면 의존성 주입이 필요한데, 보통 패턴은:

```java
// ❌ 못 씀 — @RequiredArgsConstructor는 클래스 어노테이션 변경 = 다른 분 영역 침범
@Controller
@RequiredArgsConstructor  // 추가하면 클래스 차원의 변경
public class Theme {
    private final CafeController cafeController;
    private final CafeMapper cafeMapper;  // 추가
    // 옛 수동 생성자 삭제해야 함 — 다른 분이 만든 코드
}
```

→ 다른 분 영역(옛 생성자, 옛 final 필드, 클래스 어노테이션)을 건드리지 않으려면 **차선책: `@Autowired` 필드 주입**:

```java
@Controller
@RequestMapping("/theme/*")
public class Theme {

    private final CafeController cafeController;  // 다른 분 코드 — 손 안 댐

    @Autowired                                     // ← 추가 (학생 영역)
    private CafeMapper cafeMapper;                 // ← 추가 (학생 영역, final 못 붙임)

    Theme(CafeController cafeController) {         // 다른 분 코드 — 손 안 댐
        this.cafeController = cafeController;
    }

    // ... 다른 분 메서드들 ...

    // 테마 등록 (학생 영역)
    @GetMapping("enroll")
    public String enrollForm(HttpSession session, Model model) {       // ← 시그니처 변경
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/user/login";
        }
        List<Cafe> cafeList = cafeMapper.selectByUserId(loginUser.getUserId());
        model.addAttribute("cafeList", cafeList);
        return "theme/themeEnrollForm";
    }
}
```

**왜 `@Autowired` 필드주입은 `final` 못 붙이나?**
- `final` 필드는 생성자에서만 값 할당 가능
- `@Autowired` 필드주입은 생성자 아닌 외부에서 reflection으로 주입
- → final 붙이면 컴파일 에러

**왜 강사님은 `@RequiredArgsConstructor` 권장하나?**
- 생성자 주입 = 컴파일 타임 의존성 검증
- 필드 주입 = 런타임에 NullPointerException 위험
- 필드 주입은 옛 패턴, 생성자 주입이 현대 표준

**그럼에도 오늘 필드 주입을 쓴 이유:**
- 다른 분 코드 영역을 침범하지 않는 게 더 중요한 가치 (협업 규칙)
- 학원 옛 코드에서도 필드 주입을 자주 쓰니 학원 패턴에서 벗어난 건 아님
- 면접 답변: "팀원이 작성한 클래스의 클래스 레벨 어노테이션을 변경하지 않기 위해, 차선책으로 필드주입을 선택했습니다."

#### 단계 D — JSP 셀렉트박스

```jsp
<div class="row mb-3">
    <div class="col-8">
        <label for="roomName" class="form-label">테마명<span class="form-required">*</span></label>
        <input type="text" id="roomName" name="roomName" class="form-control" required>
    </div>
    <div class="col-4">
        <label for="cafeId" class="form-label">카페<span class="form-required">*</span></label>
        <select id="cafeId" name="cafeId" class="form-select" required>
            <option value="">-- 카페 선택 --</option>
            <c:forEach var="cafe" items="${cafeList}">
                <option value="${cafe.cafeId}">${cafe.cafeName}</option>
            </c:forEach>
        </select>
    </div>
</div>
```

**왜 첫 `<option value="">-- 카페 선택 --</option>`?**
- 사용자가 의식적으로 선택하게 유도 (자동 선택된 첫 옵션이 무엇인지 모르고 제출하는 사고 방지)
- `required` 속성이 빈 value("") 시 제출 차단 → 카페 선택 강제

**왜 `col-8 + col-4`?** 테마명이 카페명보다 더 긴 경우가 많으니 비율을 더 줌. 시각적 균형.

### 3-4. 면접 답변 시나리오

**Q: "테마 등록 폼에서 카페를 어떻게 받았어요? 보안은요?"**

A: 처음엔 hidden input으로 cafeId를 박을까 했지만, hidden은 F12로 조작 가능하고, 사장이 여러 카페를 소유할 수도 있는 도메인이라 셀렉트박스로 변경했습니다. 컨트롤러에서 세션의 `loginUser.userId`로 `CafeMapper.selectByUserId()`를 호출해 본인 소유 카페만 모델에 담고, JSP에서 `c:forEach`로 옵션을 그렸습니다.

다만 셀렉트박스도 F12로 옵션 value를 바꿀 수 있어서 완전한 보안은 아니고, POST 받은 cafeId가 정말 그 사장 소유인지 서버에서 한 번 더 검증하는 단계가 추가로 필요합니다. 이건 카페 등록 풀스택 작업과 함께 추가할 계획입니다.

---

## 4부. 사례 3 — role 세션 + 헤더 분기 (사고 과정)

### 4-1. 권한 모델 설계 — "데이터에서 파생"

처음엔 "USER_ACCOUNT에 ROLE 컬럼 하나 추가하자"는 생각이 들었다 (Day 7 작업). 그런데 팀 DB 스키마를 보니:

- ROLE 컬럼 없음
- 사장 = CAFE 테이블에 USER_ID FK로 카페 소유
- 관리자 = 별도 ADMIN_ACCOUNT 테이블
- 매니저 = MANAGER_HISTORY에 활성 상태로 등록

→ 권한이 **컬럼이 아니라 데이터 관계에서 파생**되도록 설계되어 있음. 그래서 ROLE 컬럼 추가 계획을 취소하고:

```java
public String findRole(Long userId) {
    String role = "USER";
    try {
        int cafeCount = userMapper.countCafeByUserId(userId);
        if (cafeCount > 0) {
            role = "OWNER";
        }
    } catch (Exception e) {
        log.info("findRole : ", e);
    }
    return role;
}
```

**한 줄 짜리 SQL**:
```sql
SELECT COUNT(*) FROM CAFE WHERE USER_ID = #{userId}
```

→ 카페 한 개라도 있으면 사장, 없으면 일반회원.

**왜 컬럼 추가 안 하고 파생?**
- 카페를 폐업하면 사장에서 일반회원으로 자동 강등 (컬럼이면 일일이 UPDATE 필요)
- 데이터 일관성: ROLE 컬럼이 실제 카페 보유 상태와 어긋날 위험 없음
- DB 정규화 원칙: **계산 가능한 값은 저장하지 말 것**

면접 단골 개념. 외워둘 것.

### 4-2. 세션 키 컨벤션

| 세션 키 | 값 | 누가 박는가 | 누가 쓰는가 |
|---|---|---|---|
| `loginUser` | User 객체 | UserController.login() | JSP `${loginUser.nickname}`, 모든 컨트롤러에서 (User)session.getAttribute |
| `loginAdmin` | Admin 객체 | AdminController.login() | 관리자 페이지 |
| `role` | "USER" / "OWNER" / "ADMIN" | login() 둘 다 | JSP 분기, 인터셉터, 컨트롤러 권한 체크 |

**왜 `role` 키를 객체에서 분리?**
- JSP에서 분기할 때 `${role == 'OWNER'}` 한 줄로 가볍게
- `${loginUser != null && loginUser.cafeCount > 0}` 같은 무거운 조건문 회피
- 인터셉터에서도 동일 키로 일관 처리 가능

### 4-3. UserController.login() 보완 (2줄 추가)

```java
@PostMapping("/login")
public String login(User user, HttpSession session, Model model) {

    User dto = null;
    try {
        dto = service.login(user);
    } catch (Exception e) {
        log.info("login : ", e);
    }

    if (dto == null) {
        model.addAttribute("errorMessage", "아이디 또는 비밀번호가 올바르지 않습니다.");
        return "user/loginForm";
    }

    String role = service.findRole(dto.getUserId());   // ← 추가
    session.setAttribute("loginUser", dto);
    session.setAttribute("role", role);                 // ← 추가
    return "redirect:/theme/list";
}
```

**왜 findRole을 로그인 직후에 한 번만?**
- 매 요청마다 호출하면 DB 부하 증가
- 로그인 세션 동안 role은 변하지 않는다고 가정 (카페 등록/폐업 시 재로그인하면 됨)
- 세션 만료 또는 로그아웃 시 자동 소실 → 보안 OK

### 4-4. header.jsp 권한 분기

```jsp
<div class="nav-right">
    <ul class="d-flex m-0 gap-3">
        <c:if test="${role == 'OWNER'}">
            <li><a href="${pageContext.request.contextPath}/owner/res/open">CAFE</a></li>
        </c:if>

        <c:choose>
            <c:when test="${not empty sessionScope.loginAdmin}">
                <li><span>${sessionScope.loginAdmin.loginId}(관리자)</span></li>
                <li><a href="${pageContext.request.contextPath}/admin/dashboard">관리</a></li>
                <li><a href="${pageContext.request.contextPath}/admin/logout">LOGOUT</a></li>
            </c:when>
            <c:when test="${not empty sessionScope.loginUser}">
                <li><span>${sessionScope.loginUser.nickname}님</span></li>
                <li><a href="${pageContext.request.contextPath}/user/logout">LOGOUT</a></li>
            </c:when>
            <c:otherwise>
                <li><a href="${pageContext.request.contextPath}/user/login">LOGIN</a></li>
                <li><a href="${pageContext.request.contextPath}/user/enroll">회원가입</a></li>
            </c:otherwise>
        </c:choose>
    </ul>
</div>
```

| 상태 | 보이는 것 |
|---|---|
| 비로그인 | LOGIN + 회원가입 |
| 일반회원 (role=USER) | 닉네임님 + LOGOUT |
| 사장 (role=OWNER) | CAFE 메뉴 + 닉네임님 + LOGOUT |
| 관리자 (loginAdmin 있음) | 관리자 라벨 + 관리 + LOGOUT |

**왜 CAFE는 `c:if`, 로그인 상태는 `c:choose`?**
- `c:if` = 단일 조건 (참이면 보임, 거짓이면 안 보임)
- `c:choose / c:when / c:otherwise` = 다중 분기 (switch-case)
- CAFE는 사장 여부 하나만 보면 되니 `c:if`. 로그인 상태는 비로그인/일반/관리자 셋 분기라 `c:choose`.

### 4-5. 함정 노트

#### 함정 4 — 중복 LOGIN 링크

학원에서 헤더 작성 중 옛 줄(`<li><a href="...">LOGIN</a></li>` — c:choose 밖에 박힌 것)을 안 지움. 결과:

```
비로그인 시: LOGIN(밖에 박힌 것) + LOGIN(c:otherwise) + 회원가입  ← LOGIN 두 번
로그인 시: LOGIN(밖에 박힌 것) + LOGOUT(c:when)                 ← 의미 안 맞음
```

→ 밖에 박힌 LOGIN 한 줄 삭제. **헤더 작업 시 c:choose 범위 밖에 뭐가 있는지 한 번 더 확인**.

#### 함정 5 — 세션에 옛 dto가 살아있음

설정/코드 바꿔도 **세션은 살아있음** (서버 재시작 또는 로그아웃 전까지). 첫 로그인 때 role 없이 박힌 dto가 그대로 남아서 헤더 분기 안 됨.

→ **자바 변경 후 STS 재시작 + 로그아웃 + 재로그인** 3단계가 디버깅 절차의 표준.

### 4-6. 면접 답변 시나리오

**Q: "권한 관리를 어떻게 했나요?"**

A: USER_ACCOUNT 테이블에 ROLE 컬럼을 추가하는 대신, 데이터에서 파생하는 방식을 택했습니다. 사장 여부는 `CAFE.USER_ID`로 카페를 소유했는지 SQL `COUNT`로 확인하고, 관리자는 별도 ADMIN_ACCOUNT 테이블에서 로그인. 결과를 세션에 "OWNER" / "USER" / "ADMIN" 문자열로 박아 JSP에서 `c:if`나 `c:choose`로 분기했습니다.

이렇게 한 이유는, 카페를 폐업한 사장이 자동으로 일반회원으로 강등되어야 하는데, ROLE 컬럼을 따로 두면 폐업 시마다 UPDATE를 추가로 해야 하고 데이터 불일치 위험이 있기 때문입니다. **DB 정규화의 "계산 가능한 값은 저장하지 말라"는 원칙**을 따른 선택입니다.

---

## 5부. 사례 4 — MyBatis 카멜케이스 설정 (디버깅 사고)

### 5-1. 에러 메시지에서 원인까지 — 5 Whys

테마 등록 폼을 처음 띄웠을 때 터진 에러:

```
ORA-17004: 열 유형이 부적합합니다.: 1111
Caused by: Error setting null for parameter #1 with JdbcType OTHER
Theme.java:111 ← cafeMapper.selectByUserId(loginUser.getUserId())
```

#### Why 1 — 왜 ORA-17004?
- Oracle JDBC가 null을 받았는데 그 null의 SQL 타입을 결정 못 함
- → MyBatis가 null을 그대로 Oracle에 넘김

#### Why 2 — 왜 null이 넘어갔나?
- `cafeMapper.selectByUserId(loginUser.getUserId())` 호출 시 `getUserId()`가 null 반환
- → `loginUser.userId` 필드가 null

#### Why 3 — 왜 loginUser.userId가 null?
- 로그인 시 SELECT는 USER_ID 컬럼을 가져왔다 (콘솔 로그로 확인: `Row: 1, test01, 테스트01, ...`)
- → MyBatis가 USER_ID 컬럼을 User 객체의 userId 필드에 매핑 못 함

#### Why 4 — 왜 매핑 못 했나?
- DB 컬럼명 USER_ID (snake_case) ≠ Java 필드명 userId (camelCase)
- MyBatis는 기본적으로 정확히 일치하는 이름만 매핑 (USER_ID → user_id 필드를 찾음, 없으면 null)

#### Why 5 — 왜 카멜케이스 변환이 안 됨?
- MyBatis에 `mapUnderscoreToCamelCase` 옵션이 있음
- `application.yml`에 설정이 빠져있었음
- Day 5 메모에는 추가했다고 적혔지만 `.properties → .yml` 마이그레이션 때 누락

→ **해결**: yml에 한 줄 추가
```yaml
mybatis:
  configuration:
    map-underscore-to-camel-case: true
```

### 5-2. 다른 컬럼은 왜 매핑됐나? (NICKNAME 같은 한 단어)

- `NICKNAME` (언더스코어 없는 한 단어) → MyBatis가 대소문자 무시하고 nickname 필드 매칭 → OK
- `USER_ID` (언더스코어 있는 두 단어) → user_id 필드를 찾음. 없으면 null
- 한 단어 컬럼들은 운 좋게 매핑되니까 문제가 드러나지 않았는데, `USER_ID` 매핑 실패가 처음 드러난 게 오늘이었음

**교훈: 한 단어 컬럼만 쓰는 매퍼는 카멜케이스 설정 없이도 잘 도는 것처럼 보일 수 있다. snake_case 컬럼이 등장하는 순간 터진다. → 처음부터 설정해두기.**

### 5-3. 디버깅 도구 — 콘솔 SQL 로그

`application.yml`의 이 설정 덕분에 콘솔에서 SQL과 파라미터를 다 볼 수 있었다:
```yaml
mybatis:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

콘솔에서 본 것:
```
==> Preparing: SELECT USER_ID, LOGIN_ID, NICKNAME, CREATED_AT FROM USER_ACCOUNT
              WHERE LOGIN_ID = ? AND PASSWORD = CRYPTPACK.ENCRYPT(?, '12341234')
==> Parameters: test01(String), 12341234(String)
<== Columns: USER_ID, LOGIN_ID, NICKNAME, CREATED_AT
<== Row: 1, test01, 테스트01, 2026-06-02 00:00:00
<== Total: 1
```

→ SELECT는 USER_ID=1을 받아옴. 그런데 `loginUser.userId`가 null. 즉 **DB → 자바 변환에서 누락**임을 즉시 확인 가능.

만약 다음 SQL인 `SELECT COUNT(*) FROM CAFE WHERE USER_ID = ?`가 콘솔에 안 떴다면 → `findRole` 자체가 호출 안 된 것 → 자바 코드 반영 안 됨 → STS 재시작 필요.

**SQL 로그 = 디버깅의 1차 무기. 매 세션 활용할 것.**

### 5-4. 면접 답변 시나리오

**Q: "MyBatis 사용 중 부딪힌 문제와 해결 경험을 말해보세요."**

A: 테마 등록 폼에서 카페 셀렉트박스를 추가했을 때 ORA-17004 에러가 났습니다. 에러 메시지의 "Error setting null for parameter"부터 시작해서 한 단계씩 거슬러 올라가 봤습니다. SQL 파라미터가 null → 컨트롤러에서 `loginUser.getUserId()`가 null → 로그인 SELECT는 USER_ID를 가져왔는데 객체에 매핑 안 됨 → MyBatis `mapUnderscoreToCamelCase` 옵션이 application.yml에 빠져있었습니다.

한 줄 추가해서 해결했고, 이 경험으로 두 가지를 배웠습니다.

첫째, **한 단어 컬럼만 쓰는 매퍼는 카멜케이스 설정 없이도 도는 것처럼 보여서 문제가 늦게 드러난다**는 점. 그래서 프로젝트 초기부터 이 설정을 켜두는 게 안전합니다.

둘째, **MyBatis SQL 로그(`log-impl: org.apache.ibatis.logging.stdout.StdOutImpl`)가 디버깅의 1차 무기**라는 것. 어떤 SQL이 어떤 파라미터로 실행됐고 어떤 결과를 받았는지 콘솔에서 즉시 보면 문제 위치를 빠르게 좁힐 수 있습니다.

---

## 6부. 백지에서 짤 때 — 머릿속 순서표 (총정리)

### 6-1. "새 기능 하나 추가해주세요" 받았을 때

```
[1단계] 요구사항을 7층 건물에 그려본다
  - 사용자가 어떤 화면을 보고 어떤 동작을 하는가? (7층 JSP)
  - URL은 무엇인가? (6층 Controller 매핑)
  - 어떤 데이터가 DB에 들어가거나 나오는가? (1~3층)

[2단계] 비슷한 기능이 이미 있는가? (재사용 가능한 패턴?)
  - "회원가입 했으니 카페등록도 거의 똑같다"
  - "User 로그인 했으니 Admin 로그인도 거의 똑같다"
  - 차이점만 표로 정리

[3단계] 작업 단위로 쪼개기
  - 7파일? 5파일? 신규/수정 표기
  - 의존관계: 무엇부터 만들어야 다음 단계가 가능한가?

[4단계] 위에서 아래로 짜기 (Top-Down)
  ① JSP 폼 그리기 (input 이름, action URL 확정)
  ② Controller 빈 메서드 (GetMapping, PostMapping)
  ③ Service IF + Impl (try-catch + log 옷차림)
  ④ Mapper IF + xml (SQL은 DB 스키마 보면서)
  ⑤ VO (Mapper SQL의 SELECT 컬럼 보고 필드 만들기)
  ⑥ Controller에서 Service 호출, JSP에 모델 담기

[5단계] 동작 확인 + 디버깅
  - 콘솔 SQL 로그가 1차 도구
  - 에러 → 5 Whys → 원인까지 거슬러 올라가기
```

### 6-2. "에러가 났어요" — 디버깅 순서

```
[1단계] 에러 메시지 끝부분(Caused by) 읽기
  - 진짜 원인은 마지막 Caused by에 있음
  - 예: ORA-17004, NullPointerException, BindingException

[2단계] 스택트레이스에서 본인 코드 줄 찾기
  - "Theme.java:111" 같은 줄
  - 본인이 만진 코드 위치가 원인 근처

[3단계] 5 Whys
  - 왜 그 에러가? 왜 null? 왜 그 SQL? 왜 그 설정? 왜?
  - 5번이면 진짜 원인 도달

[4단계] 콘솔 SQL 로그 확인
  - Preparing: 어떤 SQL이?
  - Parameters: 어떤 값으로?
  - Columns / Row: 어떤 결과가?

[5단계] 수정 후 STS 재시작 + 세션 무효화
  - 자바 변경 → 재시작 필수
  - 세션 데이터에 영향 있는 변경 → 로그아웃 + 재로그인
```

### 6-3. "다른 사람 코드를 만져야 해요" — 협업 순서

```
[1단계] 누구 코드인지 확인
  - git blame으로 작성자 확인
  - 작성자의 스타일(주석 톤, 들여쓰기, 패턴) 파악

[2단계] 손대는 영역 최소화
  - 가능하면 본인 메서드만 추가
  - 클래스 어노테이션 추가 → 다른 분 코드 영역 침범
  - 차선책: @Autowired 필드주입 같은 비침습 방식

[3단계] 변경 코드를 채팅(또는 PR 설명)에 먼저 노출
  - 동의 받기 전에 디스크에 적용 X
  - 한 줄 Edit도 동일

[4단계] 적용 후 알림
  - "이 부분만 손댔고, 다른 분 영역(메서드 X, Y, Z)은 그대로입니다"
  - 작성자가 검토할 수 있도록
```

---

## 7부. 회고 — 오늘 새로 배운 패턴

| 패턴 | 어떻게 응용? |
|---|---|
| **학원 PL/SQL `CRYPTPACK` 패키지** | 비밀번호 등 민감 데이터 암복호화. AES-256 + 키 `12341234`. SELECT 컬럼 또는 WHERE 절에 적용 가능. |
| **권한 = 데이터 파생** | ROLE 컬럼 안 박고 `COUNT(*) > 0`로 사장 여부 판정. 정규화 원칙. |
| **다른 분 코드 안 건드리는 기술** | @RequiredArgsConstructor 대신 @Autowired 필드주입. 메서드 단위로 영역 분리. |
| **MyBatis 카멜케이스 매핑** | `map-underscore-to-camel-case: true`는 프로젝트 초기에 무조건 켜두기. snake_case 컬럼 매핑이 운 좋게 도는 함정 회피. |
| **PRG 패턴 (Post-Redirect-Get)** | 실패는 forward(Model 유지), 성공은 redirect(중복 POST 방지). 매 폼 처리의 표준. |
| **5 Whys 디버깅** | 에러 메시지에서 원인까지 5번 "왜?" 던지기. SQL 로그가 1차 도구. |
| **세션 키 컨벤션** | 객체(`loginUser`/`loginAdmin`) + 분기용 문자열(`role`) 분리. JSP/인터셉터에서 가벼운 분기 가능. |

---

## 8부. 다음 세션 (Day 9 예정)

오늘 완성 못 한 것:

| # | 작업 | 의존 |
|---|---|---|
| 1 | **매니저 임명 풀스택** | 사전정찰 완료. ManagerHistory VO/Mapper/Service/Controller/JSP 7파일. 활성 매니저 SQL은 `(USER_ID, MAX(MANAGER_HISTORY_ID))` IN 패턴. |
| 2 | **`/owner/**` 인터셉터** | role 세션 저장은 오늘 끝남 → 바로 작성 가능. HandlerInterceptor.preHandle + WebMvcConfigurer.addInterceptors. 모범 시연 후 백지 권장. |
| 3 | **header 매니저 분기** | 인터셉터 + 활성 매니저 SQL 완성 후. |
| 4 | **카페 등록 POST** | CafeController 풀스택. |
| 5 | **redirect 위치 통일** | `/theme/list` vs `/`. 어느 쪽으로 통일할지 결정. |

---

## 끝맺으며

오늘의 핵심 한 줄:

> **"코드를 외우지 말고, 그림을 그리고 순서를 외워라. 그림이 같으면 코드는 다 비슷하다."**

User 로그인을 짤 줄 알면 Admin 로그인도 90% 짤 수 있다. 회원가입을 짤 줄 알면 카페등록도 거의 똑같다. 다른 건 이름, 테이블, 키 몇 개. **머릿속에 7층 건물 + Top-Down 순서표가 있으면 백지에서도 손이 움직인다.**

면접에서 "이 코드를 왜 이렇게 짰나요?" 물으면 "User 로그인 패턴을 재사용했고, 차이는 X뿐입니다. 다른 패턴(B)도 가능했지만 A를 택한 이유는 Y입니다." 식으로 답할 수 있으면 합격이다.

내일 매니저 임명도 같은 절차로 가자.
