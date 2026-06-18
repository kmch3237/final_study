# noExit 출석체크 3-Layer 구조 완전정리

`OwnerAttendanceController` → `AttendanceService` → `AttendanceServiceImpl` 구조를 왜 이렇게 짰는지, 그리고 코드 한 줄씩 무슨 의미인지 정리한 문서.

---

# 📌 Part 1. 전체 구조 한눈에 보기

## 1-1. 음식점 비유

```
                  [브라우저 = 손님]
                        │
                        │  GET /owner/attendance
                        ▼
   ┌────────────────────────────────────────────┐
   │  OwnerAttendanceController  (= 홀 직원)     │
   │  - 주문(요청) 받기, 손님 응대                │
   │  - 세션/권한 체크 (AuthUtil.checkStaff)     │
   │  - 페이징 보정 (paginateUtil.preparePage)   │
   │  - Model에 담아 JSP로 던지기                │
   │  ※ 직접 요리는 절대 안 함                    │
   └────────────────────────────────────────────┘
                        │
                        │  attendanceService.xxx(...)
                        ▼
   ┌────────────────────────────────────────────┐
   │  AttendanceService  (= 메뉴판 / 계약서)      │
   │  ▶ interface — "할 수 있는 일 목록"만 선언   │
   │    selectAttendListByRole()                │
   │    saveDraft()                              │
   │    finalizeAttendance() ...                 │
   │  ※ 구현 코드는 한 줄도 없음                  │
   └────────────────────────────────────────────┘
                        │
                        │  implements
                        ▼
   ┌────────────────────────────────────────────┐
   │  AttendanceServiceImpl  (= 주방장)          │
   │  - 실제 요리(비즈니스 로직) 담당              │
   │  - draft 누적/중복제거 (saveDraft)          │
   │  - 트랜잭션 묶기 @Transactional             │
   │  - OWNER/MANAGER 분기                       │
   │  - 노쇼면 매너온도 차감 등                    │
   └────────────────────────────────────────────┘
                        │
                        │  mapper.xxx(...)
                        ▼
              [AttendanceMapper → DB]
```

## 1-2. 각 층의 책임

| 층 | 파일 | 책임 | 절대 안 하는 일 |
|---|---|---|---|
| **Controller** | `OwnerAttendanceController` | URL 매핑, 세션/권한, 파라미터 받기, Model 담기, 뷰 이름 리턴 | 비즈니스 계산, Mapper 직접 호출, DB접근 |
| **Service (interface)** | `AttendanceService` | "이런 기능이 있다"는 **명세서** | 구현 |
| **ServiceImpl** | `AttendanceServiceImpl` | 실제 로직, 트랜잭션, 분기, 세션 가공, Mapper 호출 | 웹전용 객체 직접 다루기 최소화, JSP 경로 결정 |
| **Mapper** | `AttendanceMapper` | SQL 한 줄씩 실행 | 로직 분기, 계산 |

> 강사님 룰: **Controller → Service → Mapper**. Controller가 Mapper 직접 부르면 단순 SELECT라도 안 됨.

## 1-3. interface + Impl 나누는 이유

```
                ┌──────────────────────┐
   Controller ──▶ AttendanceService    │ ← 콘센트 규격
                │  (interface)         │
                └─────────▲────────────┘
                          │ implements
                ┌─────────┴────────────┐
                │ AttendanceServiceImpl│ ← 지금 꽂힌 플러그
                └──────────────────────┘
```

- Controller는 **"콘센트 규격(interface)"만 의존**
- Impl을 `AttendanceServiceImplV2`로 바꿔도 Controller 코드는 **한 글자도 안 바뀜**
- Spring AOP/`@Transactional`이 동작하려면 프록시가 필요한데, **interface 기반이면 깔끔하게 프록시 생성**
- 단위테스트할 때 Mock(가짜) Service 쉽게 끼워넣기 가능

## 1-4. 실제 한 흐름 — `/owner/attendance`

```
[GET /owner/attendance?page=1]
      │
      ▼
Controller.attendance()
  ├─ AuthUtil.checkStaff(session)              ← 권한 가드
  ├─ attendanceService.dataCountByRole(...)    ▶ Impl ▶ Mapper.dataCount...
  ├─ paginateUtil.preparePage(...)              ← 페이지 보정
  ├─ attendanceService.selectAttendListByRole ▶ Impl ▶ Mapper.selectList...
  ├─ attendanceService.checkStatus(...)        ▶ Impl (세션 draft 비교 로직)
  └─ model.addAttribute(...)  → "owner/attendance" (JSP)
```

| 증상 | 봐야 할 곳 |
|---|---|
| 로그인 안 했는데 들어와짐 | Controller의 `AuthUtil.checkStaff` |
| 페이징 숫자가 이상함 | `PaginateUtil` |
| SQL 결과가 이상함 | `Mapper` + xml |
| OWNER/MANAGER 분기가 이상함 | `ServiceImpl.selectAttendListByRole` |
| draft가 done으로 안 잡힘 | `ServiceImpl.checkStatus` |

---

# 📌 Part 2. 한 줄 한 줄 코드 리뷰

## 2-1. OwnerAttendanceController.java

### 상단 어노테이션

```java
@Controller
@RequiredArgsConstructor
@Slf4j
@RequestMapping("/owner/attendance")
public class OwnerAttendanceController {
```

| 줄 | 개념 | 왜 |
|---|---|---|
| `@Controller` | Spring에 "나 컨트롤러야" 등록. 디스패처서블릿이 URL 매핑 대상으로 인식 | `@RestController`는 모든 메서드에 `@ResponseBody`가 자동. 여기는 JSP 뷰를 리턴해야 해서 `@Controller` |
| `@RequiredArgsConstructor` | Lombok. **`final` 필드만 골라서 생성자 자동 생성** | DI 생성자 코드를 안 써도 됨 |
| `@Slf4j` | Lombok. **`log`라는 이름의 Logger 변수** 자동 생성 | `private static final Logger log = ...` 한 줄을 줄여줌 |
| `@RequestMapping("/owner/attendance")` | **클래스 공통 prefix** | 모든 메서드 URL 앞에 자동으로 붙음 |

### 필드 = DI

```java
private final AttendanceService attendanceService;
private final PaginateUtil paginateUtil;
```

- `private` → 외부 차단 (캡슐화)
- `final` → 한 번 주입되면 못 바꿈
- 타입이 **interface** → Impl 바뀌어도 컨트롤러 무수정

> 💡 **생성자 주입을 쓰는 이유**: `@Autowired` 필드 주입은 final 못 붙임 → 테스트할 때 mock 못 끼움 + 순환참조 감지 못 함. 강사님 표준이 **생성자 주입**.

### attendance() — 출석체크 목록

```java
@GetMapping("")
public String attendance(@RequestParam(name = "page", defaultValue = "1") int currentPage,
                         HttpSession session, HttpServletRequest req, Model model) {
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `@GetMapping("")` | HTTP GET + 추가 경로 없음 → `/owner/attendance` 정확히 매칭 | POST와 분리해서 의도 명확히 |
| `@RequestParam(name="page", defaultValue="1")` | 쿼리스트링 `?page=3`을 `int`에 바인딩, 없으면 1 | **`name=` 명시는 강사님 룰**. defaultValue로 NPE 방지 |
| `int currentPage` | 원시타입 | 페이징은 절대 null 아닌 값이라 Integer 안 씀 |
| `HttpSession session` | 로그인 사용자 보관함 | Spring이 메서드 파라미터로 자동 주입 |
| `HttpServletRequest req` | 요청 객체. contextPath용 | 톰캣 컨텍스트 앞에 붙이려고 |
| `Model model` | JSP로 데이터 넘기는 그릇 | `request.setAttribute()`의 Spring 추상화 |

```java
String redirect = AuthUtil.checkStaff(session);
if (redirect != null) return redirect;
```

- AuthUtil = static 유틸 클래스. 차단 시 `"redirect:/login"` 리턴, 통과 시 null
- **`return "redirect:/login"` 패턴** : Spring이 문자열 앞에 `redirect:`가 붙으면 **HTTP 302**로 처리 (브라우저 URL 실제로 바뀜). 그냥 `"login"`이면 forward(URL 안 바뀜)

```java
User loginUser = (User) session.getAttribute("loginUser");
String role = (String) session.getAttribute("role");
```

- `session.getAttribute()`는 `Object` 반환 → 다운캐스팅 필요
- 권한체크가 끝났으니 여긴 안전

```java
try {
    int size = 10;
    Map<String, Object> map = new HashMap<>();
    Long userId = loginUser.getUserId();
    map.put("ownerUserId", userId);
    map.put("managerUserId", userId);
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `try-catch` | 예외 처리 | DB 죽거나 NPE 나도 화면 안 깨지게 |
| `int size = 10` | 한 페이지 개수 | 학원 표준 |
| `Map<String, Object> map` | MyBatis 파라미터 컨테이너 | 여러 값을 한 번에 SQL로 |
| owner/manager 둘 다 같은 값 | SQL 쪽에서 role 따라 다른 컬럼 사용 | |

```java
    int dataCount = attendanceService.dataCountByRole(map, role);
    String listUrl = req.getContextPath() + "/owner/attendance";
    currentPage = paginateUtil.preparePage(currentPage, size, dataCount, map, listUrl, model);
    List<AttendanceListDTO> attendList = attendanceService.selectAttendListByRole(map, role);
    Map<String, List<Long>> status = attendanceService.checkStatus(session, attendList);
```

- **`req.getContextPath()`** : 톰캣 컨텍스트 경로. 로컬은 `""`, 서버 배포 시 자동 처리. 하드코딩 금지
- **`paginateUtil.preparePage(...)`** : 페이지 번호 보정 + `map`에 offset/limit 박기 + `model`에 페이지바 정보 박기를 한 번에
- **`Map<String, List<Long>>`** : done/partial 두 묶음을 한 번에 (키 두 개, 값은 Long 리스트)

```java
    model.addAttribute("doneList", status.get("done"));
    model.addAttribute("partialList", status.get("partial"));
    model.addAttribute("attendList", attendList);
} catch (Exception e) {
    log.info("attendance : ", e);
}
return "owner/attendance";
```

- **`log.info("attendance : ", e)`** : SLF4J 문법. **두 번째 인자가 Throwable이면 스택트레이스 같이 찍힘**. `+ e`로 쓰면 스택 안 찍힘
- **return `"owner/attendance"`** : ViewResolver가 `/WEB-INF/views/owner/attendance.jsp`로 변환

### check() — 출석 입력 화면

```java
@GetMapping("/check")
public String check(@RequestParam(name = "reservationId") Long reservationId, ...) {
```

- `Long reservationId` (래퍼 타입) : DB의 NUMBER → Long 매핑. `long`은 null 못 받음
- `@RequestParam`에 `required` 생략 시 기본 true → 안 넘어오면 400

### saveDraft() — 임시저장

```java
@PostMapping("/saveDraft")
public String saveDraft(AttendForm form, HttpSession session) {
```

- `@PostMapping` : 데이터 **변경**은 POST 원칙
- `AttendForm form` : **객체 바인딩**. 폼의 name 속성이 자동 매핑

```java
return "redirect:/owner/attendance";
```

- **PRG 패턴** (Post-Redirect-Get) : 새로고침 시 같은 POST 재전송 방지

### finalizeAttend() — 최종확인

```java
attendanceService.finalizeAttendance(session, loginUser.getUserId());
```

- 파라미터 거의 없음 → 모든 데이터를 **세션 누적 draft**에서 꺼냄
- `staffUserId`는 ATTENDANCE의 "누가 출석체크 했는가" 컬럼용

### historyDetail() — AJAX 응답

```java
@ResponseBody
@GetMapping("/history/detail")
public List<AttendCrew> historyDetail(...) {
```

- `@ResponseBody` : JSP 안 거치고 **HTTP 응답 본문으로 직렬화** (Jackson이 List → JSON)
- 모달에서 fetch/AJAX로 호출, 페이지 리프레시 X

```java
if (redirect != null) return null;
```
- AJAX라 redirect 문자열 못 돌려줌. 실패 시 null

---

## 2-2. AttendanceService.java (interface)

```java
public interface AttendanceService {
    public List<AttendanceListDTO> selectListByOwnerUserId(Map<String, Object> map);
    void saveDraft(AttendForm form, HttpSession session) throws Exception;
    ...
}
```

- **interface = 메뉴판**. 시그니처만, 몸체 없음
- interface 메서드는 자동 `public abstract`
- `throws Exception` 선언 → 호출자도 처리 강제

> 💡 **왜 인터페이스 따로?**
> 1. Spring AOP(@Transactional) JDK 동적 프록시 쉽게 생성
> 2. 다형성 — Mock으로 갈아끼우기 쉬움
> 3. Controller는 interface만 의존 → Impl 갈아도 무수정
> 4. 계약(Contract) — "이 기능들 반드시 있어야 함" 문서

---

## 2-3. AttendanceServiceImpl.java

### 클래스 헤더

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class AttendanceServiceImpl implements AttendanceService {
    private final AttendanceMapper mapper;
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `@Service` | "서비스 빈"으로 등록 | 컴포넌트 스캔 대상 |
| `implements AttendanceService` | interface 계약 이행 | 시그니처 다르면 컴파일 에러 |
| `private final AttendanceMapper mapper` | Mapper(DAO) 의존성 | **Service만 Mapper 호출 권한** |

### 단순 조회 메서드 패턴

```java
@Override
public List<AttendanceListDTO> selectListByOwnerUserId(Map<String, Object> map) {
    List<AttendanceListDTO> list = null;
    try {
        list = mapper.selectListByOwnerUserId(map);
    } catch (Exception e) {
        log.info("selectListByOwnerUserId : ", e);
    }
    return list;
}
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `@Override` | 인터페이스 메서드 재정의 표시 | 오타 시 컴파일 에러로 잡힘 |
| `List<...> list = null` | 초기값 null | catch에서 빠져도 컴파일 OK |
| `mapper.xxx(map)` | MyBatis SQL 실행 | XML의 `<select id="...">` 호출 |
| 조회는 throw 안 함 | 화면이 빈 리스트로 그려지면 됨 | |

### dataCountByRole — 분기

```java
if ("OWNER".equals(role)) {
    result = mapper.dataCountByOwnerUserId(map);
} else {
    result = mapper.dataCountByManagerUserId(map);
}
```

- **`"OWNER".equals(role)`** : **상수.equals(변수)** 순서 → role이 null이어도 NPE 안 남
- 분기 in Service : Controller에서 분기하면 Mapper 직접 호출 의혹. **Service가 분기하는 게 표준**

### saveDraft — 가장 까다로움

```java
@SuppressWarnings("unchecked")
List<AttendItemDTO> drafts = (List<AttendItemDTO>) session.getAttribute("attendDraft");
if (drafts == null) drafts = new ArrayList<>();
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `@SuppressWarnings("unchecked")` | "캐스팅이 unchecked인 거 안다, 경고 꺼라" | session은 Object라 제네릭 정보 소실 → 컴파일러 경고 |
| `session.getAttribute("attendDraft")` | 세션 보관소에서 꺼냄 | DB 안 쓰고 세션 누적, 페이지 오가도 유지 |
| null 가드 | 처음엔 키 자체가 없음 | |

```java
List<AttendItemDTO> newDrafts = new ArrayList<>();
for (AttendItemDTO d : drafts) {
    if (!form.getReservationId().equals(d.getReservationId())) {
        newDrafts.add(d);
    }
}
```

- **같은 예약의 이전 draft 제거** (덮어쓰기)
- **새 리스트로 옮기는 이유** : for-each 도중 `drafts.remove()` 호출하면 `ConcurrentModificationException`

```java
for (int i = 0; i < form.getUserIds().size(); i++) {
    Long statusId = form.getAttendStatusIds().get(i);
    if (statusId == null) continue;
    ...
}
```

| 요소 | 개념 | 왜 |
|---|---|---|
| 인덱스 for문 | 두 리스트(userIds/attendStatusIds)를 같은 i로 묶기 | for-each로 두 리스트 동시 순회 불가 |
| `if (statusId == null) continue` | "미정" 건너뜀 | 사용자가 안 골랐으면 저장 X |

### finalizeAttendance — 트랜잭션 핵심

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void finalizeAttendance(HttpSession session, Long staffUserId) throws Exception {
```

| 요소 | 개념 | 왜 |
|---|---|---|
| `@Transactional` | 메서드 내 모든 DB 작업을 **하나의 트랜잭션**으로. 예외 시 전부 롤백 | ATTENDANCE 됐는데 DETAIL 실패 → 누락 데이터 생김. 둘 다 되거나 둘 다 안 되거나 |
| `rollbackFor = Exception.class` | **체크예외도 롤백 대상**으로 명시 | 기본은 RuntimeException만 롤백 |

```java
Long existingId = mapper.selectAttendanceIdByReservationId(reservationId);
if (existingId != null) {
    head.setAttendanceId(existingId);
} else {
    head.setReservationId(reservationId);
    head.setUserId(staffUserId);
    mapper.insertAttendance(head);
}
```

- 스케줄러가 미리 ATTENDANCE 박았을 수 있음 → ID 재사용
- `mapper.insertAttendance(head)` 후 `head.getAttendanceId()`에 키 채워짐 (MyBatis `useGeneratedKeys` 또는 selectKey 가정)

```java
if (dto.getAttendStatusId() != null && dto.getAttendStatusId() == 2L) {
    Manner m = new Manner();
    m.setUserId(dto.getUserId());
    mapper.callInsertNoshow(m);
}
```

- 노쇼 상태코드 비교 (`2L` = long 리터럴)
- 노쇼시 매너온도 차감, 같은 트랜잭션 안에 묶임 → 출석 실패하면 매너도 복구

### checkStatus — done/partial 분류

```java
public Map<String, List<Long>> checkStatus(HttpSession session, List<AttendanceListDTO> attendList) {
    List<Long> doneList = new ArrayList<>();
    List<Long> partialList = new ArrayList<>();
    ...
    for (Long rid : resIds) {
        int draftCount = 0;
        for (AttendItemDTO d : draftList) {
            if (rid.equals(d.getReservationId())) draftCount++;
        }
        int total = 0;
        for (AttendanceListDTO a : attendList) {
            if (rid.equals(a.getReservationId())) {
                total = a.getTotalMember();
                break;
            }
        }
        if (draftCount >= total) doneList.add(rid);
        else partialList.add(rid);
    }
```

- 이중 for로 카운트, draft 개수 ≥ 총원 → done, 미만 → partial
- `break` : 일치하는 예약 찾으면 더 안 돔 (성능)

---

# 📌 Part 3. 헷갈리는 문법 깊게 파기

## 3-1. `Map<String, Object> map = new HashMap<>();`

### Map이란

```
       Map = 사물함 (key로 찾는 보관함)
   ┌─────────────────────────────────────┐
   │  "ownerUserId"   →   123L           │
   │  "managerUserId" →   123L           │
   │  "offset"        →   0              │
   │  "size"          →   10             │
   └─────────────────────────────────────┘
        ↑ key             ↑ value
       (String)          (Object = 아무거나)
```

- **List = 순서대로** (`get(0)`, `get(1)`)
- **Map = 이름표로** (`get("ownerUserId")`)

### 왜 `Map<String, Object>`인가 — 후보 비교

| 방식 | 코드 | 장점 | 단점 |
|---|---|---|---|
| ① 파라미터 줄줄이 | `mapper.select(userId, role, offset, size)` | 명시적 | 6개 넘으면 헬, 호출부 다 수정 |
| ② DTO 만들기 | `mapper.select(SearchCondDTO cond)` | 타입안전 | 매번 DTO 만들기 귀찮 |
| ③ **`Map<String, Object>`** | `map.put("ownerUserId", 123L)` | **유연, 빠른 추가** | 타입체크 X, 키 오타 시 런타임 |
| ④ `Map<String, String>` | 값 다 String | 단순 | Long/int 못 담음 |

→ **학원 + MyBatis 전통적 표준은 ③**. 페이징처럼 키가 자주 바뀌고 값 타입 섞일 때 가장 빠름.

### 제네릭 `<String, Object>` 와 `<>`

```java
Map<String, Object> map = new HashMap<>();
        ^^^^^^^^^^^^^^^                ^^
        선언부 (꼼꼼히)              생성부 (생략 OK)
```

- 제네릭 = "이 컨테이너에 뭐 들어갈지 컴파일러한테 알려주기"
- `<>` = **다이아몬드 연산자**. Java 7부터 타입 추론 가능

```java
Map<String, Object> map = new HashMap<String, Object>();  // Java 6
Map map = new HashMap();  // 더 옛날 (Raw Type, 쓰지 마)
```

### `Map` (interface) vs `HashMap` (class)

```
                ┌─────────────┐
                │  Map        │ ← interface (콘센트 규격)
                └──────┬──────┘
                       │ implements
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   HashMap        LinkedHashMap     TreeMap
   (순서 X,        (입력순서 유지)   (key 정렬)
    가장 빠름)
```

```java
Map<String, Object> map = new HashMap<>();
^^^^                      ^^^^^^^^^^
"콘센트 규격으로 선언"      "지금은 HashMap 꽂아둠"
```

- 나중에 `TreeMap`으로 바꾸려면 우측만 수정. Service interface + Impl 패턴이랑 똑같은 원리

### 핵심 메서드

```java
map.put("ownerUserId", userId);   // 넣기
map.get("ownerUserId");           // 꺼내기 → Object 반환 → 캐스팅 필요
map.containsKey("ownerUserId");   // 있는지 확인
map.remove("ownerUserId");        // 지우기
map.size();                       // 개수
```

⚠️ `map.get("없는키")` → **NPE 안 남, null 반환**. 그 뒤 `.equals()` 호출하면 그때 NPE.

### MyBatis와 만나는 지점

```java
// Service에서
map.put("ownerUserId", userId);
map.put("offset", 0);
map.put("size", 10);
mapper.selectListByOwnerUserId(map);
```

```xml
<!-- MyBatis XML -->
<select id="selectListByOwnerUserId" parameterType="map">
    SELECT * FROM ATTENDANCE
    WHERE OWNER_USER_ID = #{ownerUserId}
    OFFSET #{offset} ROWS FETCH NEXT #{size} ROWS ONLY
</select>
```

→ **`#{키이름}`이 자동으로 `map.get("키이름")`** 로 치환됨.

---

## 3-2. `AuthUtil.checkStaff(session)`

### 정적 유틸리티 클래스 패턴

```
   [매번 권한체크 if문 복붙하기 싫음]
                │
                ▼
   ┌────────────────────────────────────┐
   │  class AuthUtil {                  │
   │    public static String            │
   │      checkStaff(HttpSession s) {   │
   │        // 로그인 안 됨?             │
   │        // OWNER/MANAGER 아님?       │
   │        // → return "redirect:/..."  │
   │        // 통과 → return null         │
   │    }                                │
   │  }                                  │
   └────────────────────────────────────┘
```

대략 이런 코드:

```java
public class AuthUtil {
    public static String checkStaff(HttpSession session) {
        User loginUser = (User) session.getAttribute("loginUser");
        String role = (String) session.getAttribute("role");
        
        if (loginUser == null) return "redirect:/member/login";
        if (!"OWNER".equals(role) && !"MANAGER".equals(role)) {
            return "redirect:/";
        }
        return null;  // 통과
    }
}
```

### `static`의 의미

```java
AuthUtil.checkStaff(session)
^^^^^^^^
객체 만들지 않고 클래스 이름으로 직접 호출
```

| 일반 메서드 | static 메서드 |
|---|---|
| `AuthUtil util = new AuthUtil();`<br>`util.checkStaff(session);` | `AuthUtil.checkStaff(session);` |
| 객체 상태(필드)에 의존 | 객체 상태 없음, 순수 함수 |

- **언제 static?** : 상태 없이 **입력 → 출력만 있는 도구함**. 권한체크, 문자열 가공, 날짜 포맷팅
- `Math.max(a, b)`, `Integer.parseInt("123")`도 같은 패턴

### 왜 외부 유틸로?

만약 컨트롤러에 직접 박는다면:

```java
@GetMapping("")
public String attendance(...) {
    User loginUser = (User) session.getAttribute("loginUser");
    if (loginUser == null) return "redirect:/member/login";
    String role = (String) session.getAttribute("role");
    if (!"OWNER".equals(role) && !"MANAGER".equals(role)) return "redirect:/";
    // ...
}

@GetMapping("/check")
public String check(...) {
    User loginUser = (User) session.getAttribute("loginUser");  // ← 복붙
    if (loginUser == null) return "redirect:/member/login";     // ← 복붙
    ...
}
```

→ 모든 메서드 4줄씩 복붙. 권한 규칙 바뀌면 다 찾아 고쳐야 함.

→ AuthUtil 하나로 빼면 **한 곳만 고치면 끝** (DRY 원칙).

### 왜 Interceptor 안 쓰고 AuthUtil로?

Spring 정석은 **Interceptor / Spring Security**지만 학원에선 안 배웠음. **static 유틸 함수 방식**이 차선책으로 깔끔.

---

## 3-3. `if (redirect != null) return redirect;` — 가드 클로즈

### 핵심

> "권한 없으면 본 로직 들어가기 전에 **빨리 빠져나가기**" 패턴.

```
   [요청 들어옴]
       │
       ▼
   AuthUtil.checkStaff(session)
       │
       ├─ 통과 → null 반환 → 아래로 진행
       │
       └─ 차단 → "redirect:/login" 반환
                       │
                       ▼
               redirect 변수에 담김
                       │
                       ▼
              if (redirect != null)
                       │
                       ▼
              return redirect; ← 끝. 본 로직 진입 X
```

### 같은 동작 비교

#### ❌ if-else 중첩 (Arrow code)

```java
public String attendance(...) {
    String redirect = AuthUtil.checkStaff(session);
    if (redirect == null) {
        // 본 로직 100줄
        // 들여쓰기 한 단계 깊어짐
        return "owner/attendance";
    } else {
        return redirect;
    }
}
```

#### ✅ Guard Clause (지금 코드)

```java
public String attendance(...) {
    String redirect = AuthUtil.checkStaff(session);
    if (redirect != null) return redirect;  // ← 빨리 탈출
    
    // 본 로직 100줄, 들여쓰기 한 단계 적음
    return "owner/attendance";
}
```

**장점** : 본 로직 평평하고 읽기 좋음.

### 왜 redirect를 **String으로** 받나?

Spring 컨트롤러 메서드 리턴값이 **String**이면 곧 **뷰 이름 or 리다이렉트 명령**:

| 리턴 문자열 | Spring의 해석 |
|---|---|
| `"owner/attendance"` | `/WEB-INF/views/owner/attendance.jsp` forward |
| `"redirect:/login"` | 302 응답 → 브라우저가 `/login`으로 다시 요청 |
| `"redirect:/owner/attendance"` | 302 응답 → 본인 페이지 새로고침 |

→ AuthUtil이 차단할 때 `"redirect:/login"` 같은 **이미 완성된 명령문자열**을 돌려주니까, 컨트롤러는 그대로 `return`하면 끝.

---

## 3-4. `User loginUser = (User) session.getAttribute("loginUser");`

### 세션(Session) 그림

```
   [브라우저]                      [톰캣 서버]
      │                                │
      │  로그인 POST                    │
      │ ───────────────────────────▶  │
      │                                │  ┌───── 세션 보관소 ─────┐
      │  Set-Cookie: JSESSIONID=ABC  │  │ ABC: {                  │
      │ ◀───────────────────────────  │  │   loginUser: User(...) │
      │                                │  │   role: "OWNER"        │
      │                                │  │ }                       │
      │  GET /owner/attendance        │  │ XYZ: { ... }            │
      │  Cookie: JSESSIONID=ABC ───▶  │  └─────────────────────────┘
```

- 서버가 **로그인 사용자 정보를 메모리에 보관**, 브라우저는 **세션ID(쿠키)만** 들고 다님
- 매 요청마다 DB 안 가도 됨 → 빠름
- 단점: 서버 재시작하면 세션 날아감 (운영은 Redis 등에 저장)

### `getAttribute()` — 사물함에서 꺼내기

```java
session.setAttribute("loginUser", user);  // 로그인 성공 시점에 박음
                       ↓
session.getAttribute("loginUser");        // 다른 컨트롤러에서 꺼내 씀
```

→ `Map.put()` / `Map.get()`이랑 같음. 세션 = **서버 보관 Map**.

### 왜 `(User)` 캐스팅이 필요한가

`session.getAttribute()`의 시그니처:

```java
public Object getAttribute(String name);
```

**리턴타입이 `Object`**. 모든 클래스의 최상위 부모. 세션은 **"뭐든 들어올 수 있는 만능 가방"**이라 꺼낼 땐 원래 타입을 모름.

```
        세션 사물함
   ┌─────────────────────┐
   │ "loginUser" → Object│ ← 컴파일러 입장에선 그냥 Object
   └─────────────────────┘
              │ getAttribute()
              ▼
         Object obj
              │ (User) ← "이건 사실 User 객체야" 캐스팅
              ▼
         User loginUser
              │
              ▼
         loginUser.getUserId() 가능
```

캐스팅 안 하면:
```java
User loginUser = session.getAttribute("loginUser");
// ❌ 컴파일 에러: Object를 User로 자동변환 못 함
```

**다운캐스팅(downcasting)** : 부모(Object) → 자식(User). 개발자가 "내가 확신한다"고 알려주는 행위.

### 캐스팅 위험 — `ClassCastException`

```java
session.setAttribute("loginUser", "문자열");  // 실수로 String 박음
User loginUser = (User) session.getAttribute("loginUser");
// 💥 ClassCastException 런타임 에러
```

→ **세션 키 이름 표준화**가 중요. `"loginUser"` 키엔 무조건 `User` 객체만 박기로 약속.

### 왜 `User`/`role` 따로 박나?

이론적으론 `loginUser.getRole()`로 통합 가능. 따로 박는 이유:

| 이유 | 설명 |
|---|---|
| 빠른 접근 | role만 필요한 곳에서 User 안 꺼내도 됨 |
| 권한 변경 용이 | 권한위임 등 할 때 role만 갈아끼우기 쉬움 |
| 팀 컨벤션 | 팀원 설계 |

---

# 🎯 한 줄 핵심 정리

| 코드 | 핵심 |
|---|---|
| `Map<String,Object> map = new HashMap<>();` | 여러 값을 한 덩어리로 묶어 SQL에 넘기는 만능 보관함. 좌측은 규격(interface), 우측은 도구(class) |
| `AuthUtil.checkStaff(session)` | 권한 4줄 복붙 방지용 static 유틸. 통과 = null, 차단 = "redirect:..." 문자열 |
| `if(redirect!=null) return redirect;` | 가드 클로즈 — 본 로직 들어가기 전 탈출. 들여쓰기 방지 |
| `(User) session.getAttribute("loginUser")` | 세션은 Object 만능가방 → 꺼낼 땐 다운캐스팅 필요 |
| `private final` + `@RequiredArgsConstructor` | 생성자 주입 (권장 DI) |
| `"OWNER".equals(role)` | NPE 안전 비교 (상수.equals(변수) 순서) |
| `@Transactional(rollbackFor=Exception.class)` | 체크예외도 롤백 포함 |
| `return "redirect:/path"` | 302 리다이렉트 (URL 바뀜) |
| `return "owner/attendance"` | forward + JSP 뷰 (URL 안 바뀜) |
| `@ResponseBody` | JSON 응답 (AJAX용) |
| 인덱스 for문 | 두 리스트 i로 묶어 처리 |
| 새 리스트로 옮기기 | for-each 중 컬렉션 수정 회피 (ConcurrentModificationException) |
