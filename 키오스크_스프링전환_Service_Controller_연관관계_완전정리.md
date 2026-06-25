# 콘솔 키오스크 → Spring Boot 전환 ②: Service · Controller · REST/JSON/JPA · 연관관계 완전 정리

> 2026-06-25 · 콘솔 키오스크를 Spring Boot REST API로 옮기는 2일차.
> 오늘 범위: **Service → Controller** 층 완성(Restaurant) → **REST vs SSR / JSON / JPA 개념 정리** → **Menu 도메인 + `@ManyToOne` 연관관계**.
> **(전 구간 기동 + curl + DB 저장까지 검증 성공! HTTP→Controller→Service→Repository→MySQL 한 바퀴 완주.)**

**스택:** Spring Boot 3.3.5 / Java 21 / Maven / Spring Data JPA / MySQL 9.6

---

## 0. 오늘 완성한 전체 흐름

```
                                  ┌──────────── 우리 Spring 서버 ────────────┐
  브라우저 / curl / 프론트         │                                          │      MySQL
      │   ① JSON 요청              │  Controller  →  Service  →  Repository   │  ▲   (kiosk DB)
      └──────────────────────────▶│  (URL/JSON)    (로직)      (DB 창구)      │  │
                                   │      ▲                          │ ◀──JPA─┘
      ◀───── ④ JSON 응답 ─────────┘      └── ② @PathVariable/@RequestBody    ③ save/findAll
                                                                          (객체↔SQL 자동변환)
   └─── JSON/REST: "서버 ↔ 바깥" ───┘       └────── JPA: "서버 ↔ DB" ──────┘
```

핵심 한 줄: **콘솔의 893줄짜리 `RestrauntAdminManagement` 한 덩어리를, 역할별로 Controller(입구)·Service(두뇌)·Repository(창고)로 쪼갰다.**

---

## 1. 콘솔 vs 스프링 — 계층 분리

원본 콘솔은 **한 클래스가 모든 걸 다 했다:**

```
RestrauntAdminManagement (893줄)
├── 화면 출력 (System.out.println)        ← 콘솔 UI
├── 입력 받기 (BufferedReader.readLine)    ← 콘솔 UI
├── 파일 읽기/쓰기 (ObjectOutputStream)    ← 저장소
└── 등록·수정·삭제 규칙, 최소금액 계산       ← 비즈니스 로직 ★
```

스프링은 이걸 역할별로 분리:

| 원본이 하던 일 | 스프링에서 누가 | 핵심 애너테이션 |
|---|---|---|
| 화면 출력 / 입력 받기 | **Controller** | `@RestController`, `@GetMapping`, `@PostMapping` |
| 파일 읽기/쓰기 | **Repository** | `JpaRepository` |
| 등록·수정 **규칙**, 계산 | **Service** | `@Service` |

> ⭐ **Service = "무엇을 할지 결정하는 두뇌".** 화면도 모르고(Controller 담당) SQL도 직접 안 짠다(Repository에 시킴). 그 사이에서 흐름·규칙만 담당. 원본 `writeRestraunt()`에서 `System.out`·`readLine`·`ObjectOutputStream`을 다 걷어내면 남는 알맹이가 Service다.

---

## 2. ⭐ DI (의존성 주입, 생성자 주입)

오늘 가장 중요한 개념.

```java
@Service
public class RestaurantService {
    private final RestaurantRepository restaurantRepository;

    // 내가 'new RestaurantRepository()' 하지 않는다!
    // 매개변수로 "필요해요"라고 선언만 하면 스프링이 만들어서 넣어준다.
    public RestaurantService(RestaurantRepository restaurantRepository) {
        this.restaurantRepository = restaurantRepository;
    }
}
```

- **원본:** 생성자가 직접 `getFile()`로 저장소를 **스스로 마련**(`new`로 직접 생성).
- **스프링:** `new` 안 한다. **생성자 매개변수로 선언만** 하면 스프링이 찾아서 **주입(넣어줌)**.
- **왜?** 직접 `new` 안 하니 → 테스트 때 가짜로 바꿔치기 쉽고, 결합이 느슨해짐. **"필요한 걸 직접 구하지 말고 받아라"**가 DI의 핵심.

> 💡 검증: 기동 시 `Error creating bean`·`UnsatisfiedDependency` 에러가 **하나도 없으면** DI 성공. 내가 `new`를 한 번도 안 썼는데 객체가 연결된 게 그 증거.

> 💡 Repository를 **여러 개** 주입받을 수도 있다(아래 MenuService). 생성자 매개변수에 나열하면 스프링이 다 넣어준다.

---

## 3. ⭐ REST API vs Thymeleaf/JSP(SSR) — "누가 HTML을 만드냐"

이전에 했던 JSP 프로젝트와 지금의 차이를 확실히 정리.

### ① Thymeleaf/JSP 방식 = 서버 사이드 렌더링(SSR)
```
브라우저 ──"식당목록 줘"──▶ 서버
                            │ 1. DB에서 식당 가져오고
                            │ 2. 서버가 직접 HTML을 조립 (<table>...식당...</table>)
브라우저 ◀── 완성된 HTML ──┘ 3. 다 그려진 화면을 통째로 보냄
(브라우저는 받은 HTML 표시만)
```
→ **서버가 "데이터 + 화면"을 둘 다** 만들어 완성품 HTML을 보냄.
→ 등록 폼: `GET /new`(빈 폼 화면) → 사용자 입력 → `POST`(저장). **GET·POST가 짝**으로 필요.

### ② REST API 방식 = 지금 우리
```
프론트/앱/curl ──"식당목록 줘"──▶ 서버
                                 │ DB에서 가져와
프론트 ◀──── JSON 데이터만 ──────┘ 데이터만 보냄 (화면 X)
[{"id":1,"name":"김밥천국",...}]
   └─ 프론트가 이 데이터로 화면을 알아서 그림
```
→ **서버는 JSON만**, 화면은 프론트가 따로. **등록 폼 화면이 아예 없다** → POST로 JSON 한 방.

### 장단점 / 왜 REST를 쓰나

| | ① SSR (JSP/Thymeleaf) | ② REST API (지금) |
|---|---|---|
| 화면 만드는 곳 | 서버 | 프론트 |
| 서버가 보내는 것 | 완성된 HTML | JSON 데이터만 |
| **재사용성** | 웹 전용 | 웹·앱·타서버가 **같은 API 공유** ⭐ |
| **역할 분리** | 화면+로직 한 서버 | 백엔드/프론트 **분업** ⭐ |
| SEO | 유리 | 신경 써야 |
| 만들기 | 한 서버로 끝(간단) | 서버+프론트 둘 필요 |

> ⭐ **REST가 좋은 이유 2가지:** (1) API 하나 만들면 웹·아이폰·안드로이드가 다 재사용. (2) 백엔드는 "JSON 잘 주기"만 책임 → 화면 바뀌어도 내 코드 안 바뀜.
> ⭐ **SSR이 나쁜 건 아니다:** 블로그·상세페이지처럼 SEO·콘텐츠 중심이면 SSR이 유리. 요즘은 Next.js처럼 섞기도. 단, **순수 백엔드 감각엔 REST가 정석**이라 우리가 이걸 함.

---

## 4. ⭐ JSON 이란

**JSON = 데이터를 글자로 표현하는 약속된 형식.** "자바 객체를 네트워크로 보낼 수 있게 글자로 바꾼 것".

```java
Restaurant r = new Restaurant(1L, "김밥천국", "분식");  // 자바 객체 (메모리에만 존재)
```
이걸 인터넷으로 못 보내서 → 글자(JSON)로 변환:
```json
{ "id": 1, "name": "김밥천국", "category": "분식" }
```
- `{ }` = 객체 하나 ↔ 자바 객체
- `"name": "김밥천국"` = 필드이름:값 ↔ 자바 필드
- `[ ]` = 목록 ↔ 자바 `List` (식당 여러 개면 `[{...},{...}]`)

> ⭐ **왜 JSON?** 어느 언어든(자바/JS/파이썬) 다 읽는 **공통어**라서. 서버는 자바, 프론트는 JS여도 JSON으로 통한다.

> ⭐ **변환 코드는 내가 안 짠다.** `@RestController`가 붙으면 `return 자바객체` 하는 순간 **Jackson** 라이브러리가 자동으로 JSON으로 바꿔준다. 반대로 `@RequestBody`는 들어온 JSON을 자바 객체로 자동 변환.
> → 이때 `getName()`으로 값을 빼고 `setName()`으로 값을 넣는다. **그래서 도메인에 getter/setter가 꼭 필요했던 것.**
> → boolean은 관례상 `isXxx()` (예: `isSoldOut()` → JSON 키 `"soldOut"`).

---

## 5. ⭐ JPA는 이 그림에서 어디냐

위(3·4)는 전부 **서버 ↔ 바깥(브라우저)** 얘기. **JPA는 반대 방향 — 서버 ↔ DB.**

```
  바깥세상              우리 서버                    DB
브라우저 ◀─JSON/REST─▶ [Controller→Service→Repository] ◀─JPA─▶ MySQL
         (Jackson 통역)                               (Hibernate 통역)
              └── 자바 객체(Restaurant)가 양쪽 통역의 공통 기준점 ──┘
```

**통역사 2명 비유:**
- **Jackson** = 손님(브라우저)과 대화 → "객체 ↔ JSON"
- **JPA(Hibernate)** = 창고(DB)와 대화 → "객체 ↔ SQL/테이블"
- 가운데 **자바 객체**가 공통 기준점.

**POST 요청 한 바퀴:**
```
POST /restaurants {"name":"김밥천국"}        ← 손님이 JSON으로 줌
 → @RequestBody가 JSON을 Restaurant 객체로    (Jackson)
 → Service가 받아 Repository에 저장 시킴
 → JPA가 객체를 INSERT SQL로 바꿔 MySQL 저장   (JPA)
 → 저장된 객체를 return
 → @RestController가 다시 JSON으로 응답         (Jackson)
{"id":1,"name":"김밥천국","category":null}    ← 손님이 JSON으로 받음 (id 자동 발급됨!)
```

---

## 6. Controller — URL을 Service로 연결하는 교환원

```java
@RestController
@RequestMapping("/restaurants")          // 공통 주소 앞부분
public class RestaurantController {
    private final RestaurantService restaurantService;
    public RestaurantController(RestaurantService s) { this.restaurantService = s; }  // DI

    @GetMapping                          // GET /restaurants  (전체)
    public List<Restaurant> getAll() { return restaurantService.findAll(); }

    @GetMapping("/{id}")                 // GET /restaurants/3 (한 건)
    public Restaurant getOne(@PathVariable Long id) { return restaurantService.findOne(id); }

    @PostMapping                         // POST /restaurants (등록)
    public Restaurant create(@RequestBody Restaurant r) { return restaurantService.register(r); }
}
```

### ⭐ 라우팅 충돌 (오늘 실수했던 부분)
스프링은 **(HTTP메서드 + URL)** 조합으로 어느 메서드를 부를지 정한다. 이 조합이 겹치면 **기동 자체가 실패**(`Ambiguous mapping`).
- `getAll()` = `@GetMapping` → `GET /restaurants`
- `getOne()` = `@GetMapping("/{id}")` → `GET /restaurants/3` ← **`("/{id}")`를 빠뜨리면** getAll과 겹쳐서 충돌!
- `getAll`과 `create`는 URL은 같지만 GET vs POST라 안 겹침.

### ⭐ @PathVariable vs @RequestBody
| | @PathVariable | @RequestBody |
|---|---|---|
| 받는 위치 | URL 경로 (`/restaurants/3`의 3) | body의 JSON 덩어리 |
| 쓸 때 | 작은 값 1개 (주로 id) | 필드 여러 개 객체 |

> 💡 `@PathVariable`은 **이름으로 연결**. URL의 `{id}`와 매개변수 `id` 이름이 같아야 자동 매칭. 다르면 `@PathVariable("id") Long restaurantId`로 명시.

---

## 7. ⭐ Menu 도메인 + 연관관계 (@ManyToOne)

오늘의 진짜 새 개념. **콘솔은 식당 객체 안에 `List<RestrauntMenu>`를 품었다.** 자바 객체를 `.ser`에 통째 저장하니 가능. 하지만 **DB는 표(table)라서 칸 안에 목록을 못 담는다.**

→ 해법: Menu를 **별도 테이블**로 빼고, **외래키(foreign key)**로 연결.

```
menu 테이블
┌────┬───────────┬───────┬───────────────┐
│ id │ menu_name │ price │ restaurant_id │ ← 연결고리(외래키)
├────┼───────────┼───────┼───────────────┤
│ 1  │ 불고기버거 │ 5000  │      2        │ → 2번 식당(맥도날드)
│ 3  │ 아메리카노 │ 4500  │      3        │ → 3번 식당(스타벅스)
└────┴───────────┴───────┴───────────────┘
```

```java
@Entity
public class Menu {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String menuName;
    private int price;
    private int menuQty;
    private boolean soldOut;

    @ManyToOne                              // 메뉴 여럿(Many) → 식당 하나(One)
    @JoinColumn(name = "restaurant_id")     // DB에 만들 외래키 칸 이름
    private Restaurant restaurant;          // 숫자 id가 아니라 '객체 통째로' 들고 있음!

    // 수량 setter에 도메인 규칙: 0 이하면 자동 품절
    public void setMenuQty(int menuQty) {
        this.menuQty = menuQty;
        this.soldOut = (this.menuQty <= 0);
    }
}
```

> ⭐ **@ManyToOne = 다대일.** 메뉴 입장에서 식당은 하나라서 필드 타입이 `Restaurant` 한 개. 자바에선 객체로 들고, DB에선 `restaurant_id` 숫자로 저장 — **변환은 JPA가 자동.**

### 연관관계 저장의 핵심 (Service)
```java
public Menu register(Long restaurantId, Menu menu) {
    Restaurant restaurant = restaurantRepository.findById(restaurantId).orElse(null); // (1) 식당 찾고
    menu.setRestaurant(restaurant);     // (2) 메뉴에 붙이고  ★핵심
    return menuRepository.save(menu);   // (3) 저장 (JPA가 restaurant_id 자동 기입)
}
```
> ⭐ 감각: **"숫자 id를 직접 넣는 게 아니라, 객체를 찾아서 붙인다."** (2)가 빠지면 `restaurant_id`가 null = 떠돌이 메뉴.
> → 이 Service는 Repository를 **2개** 주입받는다(Menu 저장 + Restaurant 찾기).

### 쿼리 메서드 (Repository)
```java
public interface MenuRepository extends JpaRepository<Menu, Long> {
    List<Menu> findByRestaurantId(Long restaurantId);  // 이름만 적으면 SQL 자동 생성!
    // → SELECT * FROM menu WHERE restaurant_id = ?
}
```

### 중첩 URL (Controller)
```java
@RequestMapping("/restaurants/{restaurantId}/menus")   // 메뉴는 식당에 '속한' 자원
...
@PostMapping
public Menu addMenu(@PathVariable Long restaurantId,    // URL에서 어느 식당
                    @RequestBody Menu menu) {            // body에서 메뉴 정보
    return menuService.register(restaurantId, menu);     // 둘을 합쳐 연결 완성
}
```
> ⭐ `@PathVariable`(소속)과 `@RequestBody`(데이터)를 **한 메서드에서 동시에** 사용.

---

## 검증 결과 (실제 실행)

### Restaurant — curl 한 바퀴
```bash
$ curl -X POST localhost:8080/restaurants -H "Content-Type: application/json" \
       -d '{"name":"김밥천국","category":"분식"}'
{"id":1,"name":"김밥천국","category":"분식"}        # ← id 자동 발급!

$ curl localhost:8080/restaurants
[{"id":1,...},{"id":2,"name":"맥도날드",...},{"id":3,"name":"스타벅스",...}]

$ curl localhost:8080/restaurants/2
{"id":2,"name":"맥도날드","category":"패스트푸드"}

$ curl localhost:8080/restaurants/999    # 없는 id → 빈 응답(null) ✅
```
DB 확인: `restaurant` 테이블에 3건 실제 저장됨 ✅

### Menu — 연관관계
```bash
$ curl -X POST localhost:8080/restaurants/2/menus -H "Content-Type: application/json" \
       -d '{"menuName":"감자튀김","price":2000,"menuQty":0}'
{"id":2,"menuName":"감자튀김",...,"soldOut":true,                  # ← 수량0 → 자동 품절! ✅
 "restaurant":{"id":2,"name":"맥도날드","category":"패스트푸드"}}   # ← 연관 식당이 JSON에 펼쳐짐

$ curl localhost:8080/restaurants/2/menus     # 2번 식당 메뉴만 (쿼리 메서드) ✅
```
DB 확인:
```
id  menu_name   price  menu_qty  sold_out  restaurant_id
1   불고기버거   5000   10        0         2     ← 객체로 붙였더니 숫자 2로 저장 ✅
2   감자튀김     2000   0         1         2
3   아메리카노   4500   50        0         3
```

---

## 헷갈렸던 점

1. **빈 생성자(`public Menu(){}`)가 왜 값을 안 받나?**
   JPA/Jackson은 "빈 깡통 객체 만들기 → setter로 채우기" 2단계로 일한다. 그래서 **매개변수 없는 생성자가 필수.** (값 받는 생성자는 코드에서 직접 만들 때용으로 따로 추가 가능 — 원본엔 둘 다 있었음.)
2. **boolean은 왜 `isSoldOut()`?** JavaBeans 관례. Jackson이 `isXxx()` → JSON 키 `"xxx"`로 변환. `getIsSoldOut`으로 쓰면 키가 `"isSoldOut"`으로 이상해짐.
3. **GET 등록폼 → POST?** 그건 JSP(SSR) 방식. REST는 등록 폼 화면이 없고 POST로 JSON 한 방. (3장 참고)
4. **`@GetMapping`만 두 번 쓰면?** URL이 겹쳐 `Ambiguous mapping`으로 기동 실패. `("/{id}")`로 구분. (6장)

---

## 다음 할 일

- [ ] 메뉴 JSON에 식당정보가 **통째로 중첩**(`"restaurant":{...}`)되는 것 정리 → **DTO** 또는 **`@JsonIgnore`** (응답 무거움 + 양방향이면 무한루프 위험)
- [ ] Kiosk / 주문(Order) / 결제(Payment) 도메인 확장
- [ ] (선택) `@OneToMany`로 식당→메뉴 양방향 연관, 더미데이터 코드 주입(값 받는 생성자)
