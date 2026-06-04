# Day 4 — 회원가입 연결 마무리 + Ajax 중복확인 풀스택 + 뷰 정리

> 2026-06-02 · 팀 git 합류 후 `noExit` 프로젝트에서 회원가입 풀스택 살리기 + 학원 패턴으로 아이디 중복확인 Ajax 풀스택 추가. 면접에서 내 코드 설명할 수 있게 정리.

---

## 0. 오늘의 전체 흐름

```
[회원가입 폼]                                       [중복확인 Ajax]
enrollForm.jsp                                     enrollForm.jsp ($.ajax)
   ↓ form submit (POST /user/enroll)                  ↓ POST /user/id-check
UserController.enroll(User user)                    UserController.idCheck(User user, HttpServletResponse response)
   ↓                                                   ↓
UserService.enroll(user)  @Transactional            UserService.countByLoginId(loginId)
   ↓                                                   ↓
UserMapper.insertAccount + insertInfo               UserMapper.countByLoginId
   ↓                                                   ↓
USER_ACCOUNT + USER_INFO INSERT                     SELECT COUNT(*) → "OK" or "NO" 응답
```

오늘 핵심 = **풀스택 4계층(JSP → Controller → Service → Mapper) 두 흐름 동시 완성**.

---

## 1. 팀 git 합류 후 바뀐 환경

| 항목 | Day 2 (개인) | Day 4 (팀 noExit) |
|---|---|---|
| 패키지 | `com.doit.app` | `com.noexit.app` |
| VO 이름 | `Member` | `User` |
| 컨트롤러 | `MemberController` | `UserController` |
| 작업 폴더 | `C:\SpringBoot\final_escape` | `C:\escapeRoom\noExit` |
| 시퀀스 이름 | `SEQ_USER_ID` | **`USER_ACCOUNT_SEQ`** ⚠️ |
| 컬럼 (USER_ACCOUNT) | `CREATED_AT` | **`CREATE_AT`** (단수형, 주의) |

> 면접 톤: "팀에 합류한 뒤 개인 작업본을 팀 패키지·VO 이름·DB 스키마에 맞춰 마이그레이션했습니다."

---

## 2. ⭐ @Transactional — 면접 1순위 개념

회원가입은 **테이블 두 개에 INSERT**(USER_ACCOUNT + USER_INFO). 둘은 같은 USER_ID로 묶임. 한쪽만 성공하면 데이터 깨짐.

```java
@Service
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final UserMapper userMapper;

    @Override
    @Transactional                              // ← 이게 없으면 위험
    public void enroll(User user) {
        userMapper.insertAccount(user);          // 1️⃣ USER_ACCOUNT INSERT
        userMapper.insertInfo(user);             // 2️⃣ USER_INFO INSERT
    }
}
```

**왜 필요?**
- `insertAccount` 성공 → `insertInfo`에서 예외 → DB 상태? → **둘 다 롤백** (`@Transactional` 덕분)
- 없으면: USER_ACCOUNT엔 행 있는데 USER_INFO엔 없음 = 부정합 데이터

**면접 답변**: "한 비즈니스 작업이 여러 SQL을 동반할 때, 중간에 실패하면 전부 롤백되도록 `@Transactional`로 묶었습니다. 회원가입에서 USER_ACCOUNT와 USER_INFO 두 테이블 INSERT가 한 묶음이어서 적용했습니다."

---

## 3. ⭐ selectKey — 시퀀스로 PK 받아 두 곳에 쓰기

```xml
<insert id="insertAccount" parameterType="com.noexit.app.model.User">

    <selectKey keyProperty="userId" resultType="Long" order="BEFORE">
        SELECT USER_ACCOUNT_SEQ.NEXTVAL FROM DUAL
    </selectKey>

    INSERT INTO USER_ACCOUNT (USER_ID, LOGIN_ID, PASSWORD, NICKNAME)
    VALUES (#{userId}, #{loginId}, #{password}, #{nickName})
</insert>


<insert id="insertInfo" parameterType="com.noexit.app.model.User">
    INSERT INTO USER_INFO (USER_ID, EMAIL, NAME, PHONE, GENDER, BIRTHDATE)
    VALUES (#{userId}, #{email}, #{name}, #{phone}, #{gender}, TO_DATE(#{birthDate}, 'YYYY-MM-DD'))
</insert>
```

**`<selectKey order="BEFORE">`가 하는 일**:
1. INSERT 실행 **전에** 시퀀스에서 NEXTVAL 가져옴
2. 그 값을 `user.userId`에 set
3. 이후 INSERT의 `#{userId}`에서 사용

**핵심**: insertAccount에서 받은 그 USER_ID가 user 객체에 박혀있어서, insertInfo에서 **같은 #{userId}**를 재사용 → FK 연결 OK.

**면접 답변**: "Oracle 시퀀스로 USER_ACCOUNT의 PK를 만들고, 같은 값을 USER_INFO의 FK로 재사용하기 위해 selectKey의 BEFORE 모드를 썼습니다."

---

## 4. ⭐ Ajax 중복확인 풀스택 — 학원 패턴 vs 실무 비교

### 학원 패턴 (HttpServletResponse) — 지금 내 코드

```java
import jakarta.servlet.http.HttpServletResponse;
import java.io.IOException;

@PostMapping("/id-check")
public void idCheck(User user, HttpServletResponse response) throws IOException {
    int count = service.countByLoginId(user.getLoginId());
    response.setContentType("text/html; charset=UTF-8");
    response.getWriter().print(count == 0 ? "OK" : "NO");
}
```

### 실무 패턴 (어노테이션)

```java
@PostMapping("/id-check")
@ResponseBody                                   // ← 리턴값을 뷰가 아니라 body에 직접
public String idCheck(@RequestParam String loginId) {
    int count = service.countByLoginId(loginId);
    return count == 0 ? "OK" : "NO";
}
```

| 비교 | 학원 (HttpServletResponse) | 실무 (@ResponseBody) |
|---|---|---|
| 리턴 타입 | `void` | `String` |
| 응답 방식 | `response.getWriter().print(...)` 직접 | 리턴값이 자동으로 응답 body |
| 파라미터 받기 | `User user` VO 통째 | `@RequestParam`으로 단일 값 |
| 학원 doit5601 | ✅ 같은 패턴 | ❌ |
| 실무 비중 | 5% (레거시) | 95%+ |

**왜 학원 패턴 선택?**
- 팀 다른 컨트롤러들과 일관 (Theme/Owner 등도 Servlet 톤)
- 학원에서 본 doit5601 패턴 그대로 (시연/발표 톤 일치)

**면접 답변**: "프로젝트에선 팀 일관성을 위해 HttpServletResponse 패턴을 썼지만, 모던 Spring에선 `@ResponseBody`나 `@RestController`로 같은 일을 어노테이션 하나로 처리한다는 걸 알고 있습니다."

---

## 5. ⭐ Ajax 흐름 — JSP fail/success 콜백 동작 원리

```js
$.ajax({
    type: "POST",
    url: "${pageContext.request.contextPath}/user/id-check",
    data: { loginId: loginId },
    success: function(result){              // ← 서버가 200 OK + 응답 body로 뭐 보냈을 때
        if (result === "OK") { ... }
        else { ... }
    },
    error: function(){                       // ← 404, 500 등 실패 시
        $("#error").html("오류가 발생했습니다.");
    }
});
```

- `success`의 `result` = 서버가 `response.getWriter().print("OK")`로 보낸 그 문자열
- 404가 뜨면 (= 서버에 매핑 없음) `error` 콜백 실행 → "오류가 발생했습니다"

**오늘 겪은 실수**: `/user/id-check` 매핑을 Controller에 안 만들고 Ajax 시도 → 404 → error 콜백 → 빨간 글씨.  
**해결**: Controller에 `@PostMapping("/id-check")` 추가.

---

## 6. ⭐ @RequiredArgsConstructor + final 필드 = 생성자 주입

```java
@Controller
@RequiredArgsConstructor                    // ← Lombok이 생성자 자동 생성
public class UserController {

    private final UserService service;       // ← final이라 생성자 매개변수에 들어감
    ...
}
```

**Lombok이 컴파일 시점에 만들어주는 것**:
```java
public UserController(UserService service) {
    this.service = service;
}
```

**Spring DI 흐름**:
1. Spring이 `UserController` 빈 만들려고 함
2. 생성자 매개변수에 `UserService`가 필요 → 컨테이너에서 찾음
3. `UserServiceImpl` 빈을 service 자리에 꽂아줌

**면접 답변**: "Spring 생성자 주입 패턴입니다. `@RequiredArgsConstructor`가 final 필드를 모아 생성자를 만들어주고, Spring이 그 생성자로 빈을 주입합니다. 필드 주입(@Autowired)보다 불변성·테스트 용이성 측면에서 권장되는 방식입니다."

---

## 7. 🔥 오늘 겪은 트러블 (다시 겪지 말자)

### 트러블 1: `ORA-02289: 시퀀스가 존재하지 않습니다`

**증상**: 회원가입 클릭 → 500 에러. 콘솔에 `SEQ_USER_ID.NEXTVAL` 관련 에러.

**원인**: Day 2에 만든 시퀀스(`SEQ_USER_ID`)와 팀 DB 스키마의 시퀀스(`USER_ACCOUNT_SEQ`)가 **다른 이름**. 팀 git 합류 시 mapper.xml은 옛 이름을 그대로 가져옴.

**해결**: mapper.xml의 selectKey에서 `SEQ_USER_ID` → `USER_ACCOUNT_SEQ`로 변경.

**교훈**: DB 스키마(TEST.sql) **시퀀스 명명 패턴은 `{테이블명}_SEQ`**. CAFE_SEQ, ROOM_SEQ, MANAGER_HISTORY_SEQ 등 같은 패턴.

### 트러블 2: `ORA-01861: literal does not match format string`

**증상**: BIRTHDATE에 폼 값 그대로 INSERT 시도 → 에러.

**원인**: HTML `<input type="date">`는 `"1995-03-15"` 문자열로 전송. Oracle의 DATE 컬럼은 문자열을 자동 변환 못 함.

**해결**: mapper.xml에서 `TO_DATE` 적용.
```xml
TO_DATE(#{birthDate}, 'YYYY-MM-DD')
```

### 트러블 3: 중복확인 풀스택 누락 — "오류가 발생했습니다" 빨간 글씨

**증상**: 아이디 입력 후 중복확인 클릭 → 빨간 "오류가 발생했습니다".

**원인**: JSP Ajax 코드는 있는데 **서버 측 4계층(Controller/Service/ServiceImpl/Mapper xml) 중 일부만 작성**.

**해결**: 4곳 다 채우기.
- `userMapper.xml`: `<select id="countByLoginId">` 추가
- `UserService.java`: `countByLoginId(String)` IF 추가
- `UserServiceImpl.java`: 구현 추가
- `UserController.java`: `@PostMapping("/id-check")` + import 추가

**교훈**: Ajax = 클라이언트(JS) + 서버 둘 다 있어야 동작.

### 트러블 4: 같은 id 두 번 = 폼 깨짐

**증상**: 옛 폼 div 안 지우고 새 폼 div 추가 → 한 페이지에 `#findPwForm`, `#name` 등 id 중복 → JS 셀렉터 깨짐.

**해결**: 옛 div 통째 삭제.

**교훈**: HTML의 id는 페이지에 **유일해야** 함. jQuery `$("#name")`이 어느 input을 잡을지 못 정함.

### 트러블 5: 중복확인 후 아이디 수정 불가

**원인**: 학원 패턴이 중복확인 성공 시 `$("#loginId").prop("readonly", true)` 처리.

**해결**: readonly 빼고 input 이벤트로 `isIdChecked` 리셋.
```js
$("#loginId").on("input", function(){
    if (isIdChecked) {
        isIdChecked = false;
        $("#error").html("아이디가 변경되어 중복확인이 필요합니다.")
                   .css({color: "red", display: "inline"});
    }
});
```

---

## 8. 뷰 정리 — 공통 CSS 활용

### 인라인 `<style>` 0줄 원칙

팀 공통 `common.css`에 디자인 토큰·컴포넌트 클래스가 다 있음. 페이지마다 인라인 `<style>` 블록을 박는 건 **AI 냄새**.

### 자주 쓴 공통 클래스

| 클래스 | 용도 |
|---|---|
| `.ne-sc` | 흰 카드 박스 |
| `.ne-sc-title` | 노랑 밑줄 제목 |
| `.ne-hint` | 회색 작은 안내 글씨 |
| `.content-box-yellow` | 노란 배경 결과 박스 |

### Bootstrap utility 위주

- 가운데 정렬: `<body>` flex 마법 X → `container my-5` + `style="max-width: 480px;"`
- 행 분할: 임의 `.num-row` X → `row` + `col`
- 버튼 위치: 임의 `.btn-row` X → `text-end mt-4`
- 폼 입력: `mb-3 form-label form-control`

### 단위 — `rem` 안 쓰기

- 학원 표준 = `px`
- 인라인 `style="font-size: 28px;"`처럼 px로
- (단 `fs-5`처럼 Bootstrap 클래스가 내부에서 rem 쓰는 건 OK — 우리 코드 아님)

---

## 9. 테마등록의 `cafeId` — 폼에서 받지 않는 이유 (보안)

ROOM 테이블의 `CAFE_ID`는 "이 테마가 어느 카페 소속이냐". **로그인한 사장님의 카페**여야 함.

```jsp
<!-- 폼에서 직접 입력 받으면 보안 사고 -->
<input type="number" id="cafeId" name="cafeId" required>   ❌

<!-- 세션의 userId로 자동 조회한 cafeId 주입 -->
<input type="hidden" name="cafeId" value="${loginCafeId}">  ✅
```

**왜 보안 사고?**: 폼 입력이면 매니저가 **남의 카페 ID를 적어 등록** 가능. 다른 사람 카페에 마음대로 테마 등록 = 무결성 깨짐.

**면접 답변**: "테마는 사장님 본인의 카페에 속해야 하므로, 폼 입력이 아닌 세션에서 사용자 ID를 꺼내 본인 소유 카페를 조회하고 hidden으로 주입했습니다. 폼 입력으로 받으면 남의 카페에 등록할 수 있는 권한 우회 취약점이 됩니다."

---

## 10. 코드 테이블 FK — `isAdult`가 `0/1`이 아닌 이유

ROOM.IS_ADULT는 `NUMBER NOT NULL FK → COMMON(COMMON_ID)`.  
COMMON 테이블에는 Y/N이 들어있고 COMMON_ID는 시퀀스로 부여.

```jsp
<!-- 잘못 -->
<option value="0">일반</option>      <!-- COMMON_ID 0은 존재 X → FK 위반 -->
<option value="1">성인 전용</option>

<!-- 정답 -->
<c:forEach var="c" items="${commonList}">
    <option value="${c.commonId}">       <!-- COMMON 테이블에서 조회한 실제 COMMON_ID -->
        <c:choose>
            <c:when test="${c.commonName == 'N'}">일반</c:when>
            <c:when test="${c.commonName == 'Y'}">성인 전용</c:when>
        </c:choose>
    </option>
</c:forEach>
```

**면접 답변**: "Y/N 같은 boolean을 별도 COMMON 테이블로 정규화한 팀 스키마라, 폼 옵션 값은 그 테이블의 PK를 동적으로 채워야 합니다. 하드코딩하면 FK 제약 위반(ORA-02291)이 발생합니다."

---

## 11. 면접에서 회원가입 한 번에 설명하기 — 짧은 버전

> "회원가입 폼은 USER_ACCOUNT와 USER_INFO 두 테이블로 분리되어 있어, Spring `@Transactional`로 한 묶음 처리했습니다. Oracle 시퀀스로 USER_ID를 받고 MyBatis `selectKey`의 BEFORE 모드로 PK를 먼저 받은 뒤, 같은 값을 두 INSERT의 FK로 재사용했습니다. 아이디 중복확인은 JSP에서 jQuery Ajax POST → Controller에서 HttpServletResponse로 결과 문자열을 직접 응답하는 학원 표준 방식으로 구현했습니다. 모던 스타일인 `@ResponseBody`나 `@RestController`로도 동일하게 짤 수 있다는 점을 알고 있습니다."

---

## 다음 할 일 (Day 5 예정)
- 로그인 본문 (POST /user/login) + 세션 저장
- (선택) Member.role 필드 + 인터셉터
- 카페 풀스택 (Model/Service/Mapper)
- 테마 풀스택
- 매니저 임명 풀스택 (폼 + 백엔드)
