# 콘솔 키오스크 → Spring Boot 전환 ③: DTO · 주문 도메인 · 트랜잭션 · 예외처리 · 검증 완전 정리

> 2026-06-26 · 콘솔 키오스크를 Spring Boot REST API로 옮기는 3일차(분량 大).
> 오늘 범위: **DTO** → **주문(Order/OrderItem) 도메인 전 계층** → **무한루프 발견·해결** → **결제/재고차감/@Transactional** → **예외처리(@RestControllerAdvice)** → **조회** → **@Valid 입력검증**.
> **(전 구간 curl + DB 검증 성공! 실무형 REST API 핵심 골격 한 바퀴 완성.)**

**스택:** Spring Boot 3.3.5 / Java 21 / Maven / Spring Data JPA / MySQL 9.6

---

## 0. 오늘 완성한 전체 흐름

```
[요청 검증]   @Valid       →  잘못된 입력(음수/빈값) 400으로 입구컷
     │
[Controller] OrderController / MenuController
     │  @RequestBody(요청 DTO)  ·  return 응답 DTO
[Service]    OrderService(주문생성) · PaymentService(@Transactional 재고차감)
     │
[Repository] OrderRepository (OrderItem은 cascade로 함께 저장)
     │
[MySQL]      orders · order_item · menu · restaurant  (FK로 연결)

[예외]   어디서든 throw → @RestControllerAdvice(콜센터) → 404/409/400 + 깔끔 메시지
```

핵심 한 줄: **"엔티티를 그대로 주고받지 말고 DTO로 포장하고, 위험한 작업은 트랜잭션으로 묶고, 실패는 콜센터가 정중히 응대한다."**

---

## 1. ⭐ DTO vs Entity — 냉장고 재료 vs 손님상 접시

| | Entity (`Menu`, `Order`) | DTO (`MenuResponse`, `OrderRequest/Response`) |
|---|---|---|
| 정체 | **DB 테이블과 1:1, JPA가 관리하는 살아있는 객체** | 데이터를 **옮기는(Transfer) 그릇** |
| 표시 | `@Entity` | 평범한 클래스 |
| 비유 | 냉장고 속 진짜 재료 | 손님상에 내갈 접시 |

> 💡 **예전 MyBatis/JSP 프로젝트엔 왜 DTO만 있었나?** MyBatis는 "살아있는 엔티티" 개념이 없어 VO/DTO 하나가 DB결과+화면을 1인 2역 했다. JPA는 `@Entity`가 DB가 추적·관리하는 특별한 객체라, 바깥엔 DTO로 분리하는 게 정석이 됐다.

### 왜 분리하나 (3가지)
1. **무거움** — 연관 객체가 통째로 딸려옴
2. **내부구조 노출** — 엔티티(DB 구조)를 바깥에 그대로 보이면 변경 시 API가 깨짐
3. **무한루프** — 양방향 연관이면 서로 끝없이 참조 (아래 4장)

### ⭐ static 팩토리 메서드 `from()` — "객체 없이 부른다"가 무슨 말?
```java
public static MenuResponse from(Menu menu) { ... }   // 호출: MenuResponse.from(menu)
```
**일반 메서드**는 객체를 먼저 만들어야 부른다:
```java
Menu m = new Menu();   // 1. 객체 먼저
m.getMenuName();       // 2. 그 객체에게 물어봄
```
`getMenuName()`은 *특정 메뉴 한 개*에 속한 일이라 객체가 있어야 말이 된다.

그런데 `from()`의 임무는 **"MenuResponse를 새로 만드는 것"**. 여기서 모순:
```java
MenuResponse r = r.from(menu);   // r을 만들려는데 r이 이미 있어야 한다? 닭-달걀!
```
그래서 **`static`** 을 붙여 *객체가 아니라 클래스 자체에 속한* 메서드로 만든다:
```java
MenuResponse r = MenuResponse.from(menu);   // 빈 객체 미리 안 만들고 클래스명으로 바로 호출 ✅
```

> 🐟 **붕어빵 비유:** `static from()` = **붕어빵 틀**(주방에 걸림 = 클래스 소속). 결과물 `MenuResponse` = **붕어빵**. 붕어빵을 만들려고 붕어빵이 먼저 필요하진 않다. 틀만 있으면 된다. `MenuResponse.from(menu)` = "틀에 menu 반죽 넣고 하나 뽑기".
>
> 정리: **인스턴스 메서드**(getName 등)는 "이 객체"가 하는 일 → 객체 필요. **정적(static) 메서드**는 "클래스"가 하는 일 → 객체 불필요, `클래스명.메서드()`로 호출.

### DTO에 setter가 없는 이유
- **나갈 때(응답)는 읽기만** → Jackson은 JSON 만들 때 **getter만** 쓴다.
- `from()`이 **같은 클래스 내부**라 `dto.id = ...`로 private 필드에 직접 접근 가능(자바는 같은 클래스 내 private 허용) → setter 불필요.
- 반대로 **들어올 때(요청)** 쓰는 `OrderRequest`엔 setter가 있다(Jackson이 JSON 값을 *넣어야* 하므로).

`from(X)` / `Response` / `Request`는 자바 문법이 아니라 **작명 관례**. `from(X)`="X로부터 만든다", `Response`="응답용", `Request`="요청용".

---

## 2. ⭐ 주문 도메인 — 영수증(Order) & 영수증 한 줄(OrderItem)

콘솔의 주문은 그냥 `List<RestrauntMenu>` 장바구니였고 **저장되는 주문 자체가 없었다.** DB엔 제대로 남기자.

```
Restaurant ──1:N──▶ Order ──1:N──▶ OrderItem ──N:1──▶ Menu
  (식당)             (영수증)        (영수증 한 줄)       (메뉴)
```

| 엔티티 | 뜻 | 핵심 필드 |
|---|---|---|
| `Order` | 주문 1건(영수증 전체) | restaurant, orderItems, orderedAt, totalPrice, paid |
| `OrderItem` | 주문 한 줄 | menu, quantity, orderPrice, order |

### ⭐ OrderItem이 menu_id·order_id를 **둘 다** 갖는 이유 (서로 다른 질문)
```
order_item 한 줄: "불고기버거를, 2개, 주문 3번에서 샀다"
                    ↑menu_id   ↑quantity  ↑order_id
```
| 칸 | 답하는 질문 |
|---|---|
| `order_id` | 이 줄이 **어느 영수증**에 속하나? |
| `menu_id` | 이 줄이 **무슨 상품**인가? |

> `order_id`만 두면 → "주문 3번에 뭔가 2개" — **그 '뭔가'가 뭔데?** 상품을 모른다. 그래서 `menu_id`가 따로 필요. 영수증 한 줄엔 `[상품명][수량][금액]` + 영수증번호가 다 있어야 한다.

### ⭐ 왜 `@ManyToOne`을 두 개나? (학생-반 비유)
- `OrderItem`(학생) → `Order`(반): 한 반에 학생 여럿, 학생은 한 반 → **@ManyToOne**
- `OrderItem`(주문항목) → `Menu`: 같은 불고기버거가 여러 주문에 등장 → **@ManyToOne**
- 둘 다 "Many 쪽"이 OrderItem이라 **FK(order_id, menu_id)가 order_item 테이블에 둘 다** 생긴다.

```java
@ManyToOne @JoinColumn(name = "menu_id")  private Menu menu;
@ManyToOne @JoinColumn(name = "order_id") private Order order;   // 연관관계의 '주인'(FK 가짐)
private int quantity;     // 주문 수량 (Menu.menuQty '재고'와는 다른 숫자!)
private int orderPrice;   // 주문 시점 단가 (가격이 후에 바뀌어도 영수증은 '그때 값' 유지)
```

### ⭐ @OneToMany + mappedBy — 주인 vs 거울
> **외래키(FK)는 항상 "여러 개(Many)" 쪽 테이블에 생긴다.** (order_item이 "나 3번 주문 소속"이라 적음. order가 "내 항목 5,6,7"을 한 칸에 적을 순 없음.)

| 쪽 | 애너테이션 | 역할 |
|---|---|---|
| `OrderItem`(Many) | `@ManyToOne @JoinColumn(order_id)` | **주인** — 진짜 FK 칸 가짐 |
| `Order`(One) | `@OneToMany(mappedBy="order")` | **거울** — 조회 편의용, FK 칸 안 만듦 |

```java
@Entity
@Table(name = "orders")   // ★ 'order'는 SQL 예약어(ORDER BY)라 테이블명 회피
public class Order {
    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)   // mappedBy = OrderItem의 'order' 필드가 FK 주인
    private List<OrderItem> orderItems = new ArrayList<>();

    // 양방향은 양쪽 다 맞춰야 한다
    public void addOrderItem(OrderItem item) {
        orderItems.add(item);      // 주문 목록에 넣고
        item.setOrder(this);       // 항목에도 "너 이 주문 소속" 알림 (FK는 이쪽이 채움)
    }
}
```

> 💡 **검증(DB):** 기동하니 `order_item` 테이블에 `menu_id`·`order_id` 둘 다 생기고, `orders`엔 항목 관련 칸이 **안 생김** → "거울 쪽은 FK 칸을 안 만든다"가 눈으로 확인됨.
> 💡 **cascade=ALL** : `order`만 save해도 딸린 OrderItem들이 함께 저장(전파)된다.

---

## 3. 주문 생성 로직 (OrderService)

```java
public Order create(OrderRequest request) {
    Restaurant restaurant = restaurantRepository.findById(request.getRestaurantId())
            .orElseThrow(() -> new NotFoundException("식당을 찾을 수 없습니다. ..."));
    Order order = new Order();
    order.setRestaurant(restaurant);
    order.setOrderedAt(LocalDateTime.now());

    int totalPrice = 0;
    for (OrderRequest.Item itemReq : request.getItems()) {
        Menu menu = menuRepository.findById(itemReq.getMenuId())
                .orElseThrow(() -> new NotFoundException("메뉴를 찾을 수 없습니다. ..."));
        OrderItem orderItem = new OrderItem();
        orderItem.setMenu(menu);
        orderItem.setQuantity(itemReq.getQuantity());
        orderItem.setOrderPrice(menu.getPrice());   // ★ 가격은 서버(DB)가 결정 — 조작 방지
        order.addOrderItem(orderItem);
        totalPrice += menu.getPrice() * itemReq.getQuantity();
    }
    order.setTotalPrice(totalPrice);
    return orderRepository.save(order);   // cascade로 OrderItem들도 함께 저장
}
```

> ⭐ **요청 DTO는 menuId·quantity만 받는다(가격 X).** 가격을 손님이 보내게 하면 "1원에 주문" 조작이 가능. → 클라이언트는 '무엇을 몇 개', '얼마인지'는 **서버가 DB에서 결정**.
> 💡 `Menu menu = item.getMenu();` 에서 `Menu menu`가 **선언**. `item.getMenu()`는 `@ManyToOne` 연관 → JPA가 `menu_id` FK로 Menu를 자동 로드(별도 SQL 불필요).

---

## 4. ⭐⭐ 무한루프 — 마주보는 두 거울 (가장 중요)

주문을 만들고 `Order` **엔티티를 그대로 응답**했더니 **99KB**가 나왔다.

### 왜 무한히 커지나
```
Order → orderItems → OrderItem → order(주문으로 되돌아감!) → orderItems → OrderItem → order → ... ∞
```
- `Order`는 자기 `orderItems`(항목들)를 가짐
- 각 `OrderItem`은 자기 `order`(소속 주문)를 **도로 가리킴** ← 되돌아가는 길!
- 그 `order`는 또 `orderItems`를 보고...

Jackson이 JSON을 만들 때 이 참조를 따라가다 **거울 두 개를 마주본 것처럼** 무한 반복한다.

> 🪞🪞 **두 거울 비유:** 거울을 마주보게 두면 거울 속에 거울 속에 거울... 끝없다. `Order ↔ OrderItem`이 서로를 비추는 거울이다.

### 왜 양방향을 뒀나 (그럼 없애면 안 되나?)
- `OrderItem.order` : **FK(order_id) 주인**이라 저장에 꼭 필요
- `Order.orderItems` : cascade 저장 + "주문의 항목들 조회" 편의
- 둘 다 자바에선 유용하다. **JSON 직렬화에서만** 루프가 문제. → 그래서 엔티티는 그대로 두고, **응답만 DTO로** 푼다.

### ⭐ OrderResponse = "한 방향 통행로" (해결)
```java
public static OrderResponse from(Order order) {
    OrderResponse dto = new OrderResponse();
    dto.id = order.getId();
    dto.restaurantId = (order.getRestaurant()!=null) ? order.getRestaurant().getId() : null; // 식당 객체 통째 X → id만
    dto.orderedAt = order.getOrderedAt();
    dto.totalPrice = order.getTotalPrice();
    dto.paid = order.isPaid();
    dto.items = order.getOrderItems().stream()   // 각 줄을 Item DTO로 변환
            .map(OrderResponse::toItem)          // ★ 여기서 order로 '되돌아가지 않음' → 루프 차단
            .toList();
    return dto;
}
private static Item toItem(OrderItem oi) {       // OrderItem → 응답용 Item (menu에서 id/이름만)
    Item item = new Item();
    item.menuId = oi.getMenu().getId();
    item.menuName = oi.getMenu().getMenuName();
    item.quantity = oi.getQuantity();
    item.orderPrice = oi.getOrderPrice();
    return item;                                 // order 역참조 없음!
}
```
**from() 흐름 단계별:** (1) 빈 DTO 생성 → (2) order의 단순값 복사 → (3) `getOrderItems().stream().map(toItem).toList()`로 각 줄을 Item DTO로 변환(**여기서 order로 돌아가는 길을 안 담음**) → (4) 반환.

> 🎯 거울 하나를 치운 셈. **99KB → 250B.** (`.stream().map(...).toList()` = 목록의 각 요소를 변환기에 하나씩 통과시켜 새 목록 만들기.)

---

## 5. ⭐ 결제 & @Transactional — 계좌이체처럼

결제 = 주문의 각 항목만큼 **메뉴 재고를 깎고** 주문을 '결제됨'으로.

> 💡 **수량이 두 종류:** `Menu.menuQty`(창고에 남은 **재고**) vs `OrderItem.quantity`(이번에 산 **주문량**). 결제 시 재고에서 주문량을 뺀다.

```java
@Service
public class PaymentService {
    @Transactional   // ★ 메서드 안 모든 DB 변경을 '하나의 거래'로 묶음
    public Order pay(Long orderId) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new NotFoundException("주문을 찾을 수 없습니다. ..."));
        if (order.isPaid())
            throw new IllegalStateException("이미 결제된 주문입니다. ...");   // 이중 차감 방지

        // (1) 먼저 전부 살 수 있는지 검증 (깎기 전에 — 한 줄이라도 모자라면 시작 안 함)
        for (OrderItem item : order.getOrderItems()) {
            Menu menu = item.getMenu();
            if (menu.getMenuQty() < item.getQuantity())
                throw new IllegalStateException("재고 부족: " + menu.getMenuName() + " ...");
        }
        // (2) 통과 → 실제 차감 (setMenuQty의 '0이면 품절' 규칙 자동 발동)
        for (OrderItem item : order.getOrderItems()) {
            Menu menu = item.getMenu();
            menu.setMenuQty(menu.getMenuQty() - item.getQuantity());
            // menu는 영속 상태 → 트랜잭션 끝에 변경이 자동 반영(dirty checking, save 명시 불필요)
        }
        order.setPaid(true);
        return order;
    }
}
```

> ⭐ **@Transactional이 핵심:** 메뉴 3개 차감 중 마지막에서 실패하면? 트랜잭션이 없으면 앞 2개는 이미 깎인 채 사고. `@Transactional`은 **"전부 성공 or 전부 롤백"**을 보장한다.
> 💳 **계좌이체 비유:** "A에서 빼고 B에 넣기" 둘 중 하나만 되면 돈이 증발. 그래서 하나의 거래(transaction)로 묶는다.
> 💡 **검증 먼저 / 차감 나중**이 1차 방어, **@Transactional**이 최후 안전망. 둘이 같이 지킨다.
> 💡 **dirty checking:** DB에서 조회한 `menu`는 영속 상태라 `setMenuQty`만 해도 트랜잭션 끝에 자동 UPDATE.

### ⭐ 프로시저(Stored Procedure) vs @Transactional + 서비스
| | 프로시저 | @Transactional + 서비스(우리) |
|---|---|---|
| 로직 위치 | DB 안 | 자바 코드 |
| 속도 | 대량데이터 빠름(왕복 없음) | 건건이 왕복하면 느릴 수 있음 |
| 버전관리/테스트 | 어려움 | 쉬움(git/JUnit) |
| DB 교체 | 종속 | 독립적 |

> 요즘 웹앱은 **압도적으로 @Transactional+서비스**. 프로시저는 ① 한국 SI/금융/공공 레거시(MyBatis+Oracle), ② 수백만 건 배치/집계에서 쓴다. 둘 다 트랜잭션은 보장된다.

---

## 6. ⭐ 예외처리 — @RestControllerAdvice = 전역 콜센터

처음엔 결제 실패가 **500 + 못생긴 JSON**으로 샜다. 콜센터를 세우자.

```java
@RestControllerAdvice   // 모든 @RestController를 감싸는 전역 처리기
public class GlobalExceptionHandler {
    @ExceptionHandler(NotFoundException.class)       // 없는 자원
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(new ErrorResponse(404, e.getMessage()));
    }
    @ExceptionHandler(IllegalStateException.class)   // 재고부족/이미결제
    public ResponseEntity<ErrorResponse> handleConflict(IllegalStateException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(new ErrorResponse(409, e.getMessage()));
    }
    @ExceptionHandler(MethodArgumentNotValidException.class)  // @Valid 위반
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream().findFirst()
                .map(err -> err.getField() + ": " + err.getDefaultMessage()).orElse("잘못된 요청입니다.");
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(new ErrorResponse(400, msg));
    }
}
```

> ☎️ 어디서 무슨 예외가 터지든 콜센터가 한 곳에서 받아 상황에 맞는 코드로 응대. 컨트롤러/서비스는 `throw`만 하면 된다. **한 번 만들면 앱 전체에 자동 적용**(OrderService에 코드만 추가해도 자동으로 404 처리됨).

> 💡 **IllegalStateException** = "지금 상태로는 작업 불가"용 자바 표준 예외(이미결제/재고부족). cf `IllegalArgumentException`(인자 잘못). 실무는 보통 커스텀 예외(`OutOfStockException`).
> 💡 **NotFoundException**은 `RuntimeException` 상속(unchecked → try-catch 강제 안 됨). 표준 예외 대신 만든 이유: "없음(404)"과 "상태문제(409)"를 이름으로 구분해 다른 코드로 응대하려고.
> 💡 **ResponseEntity** = 응답 몸통 + 상태코드를 함께 지정하는 도구.

---

## 7. ⭐ @Valid 입력검증 — 입구 문지기

```xml
<!-- pom.xml: 검증 애너테이션 쓰려면 필요 -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```
```java
public class OrderRequest {
    @NotNull(message="restaurantId는 필수입니다.")
    private Long restaurantId;

    @NotEmpty(message="items는 최소 1개 이상이어야 합니다.")
    @Valid   // ★ 목록 '안의 각 Item'도 검사 (없으면 Item 규칙 무시됨)
    private List<Item> items;

    public static class Item {
        @NotNull(message="menuId는 필수입니다.")        private Long menuId;
        @Min(value=1, message="quantity는 1 이상이어야 합니다.") private int quantity;
    }
}
// 컨트롤러: public OrderResponse create(@Valid @RequestBody OrderRequest request)
```

> ⭐ **3단 방어선 완성:** `@Valid`(400, 형식 틀림) → `NotFoundException`(404, 없음) → `IllegalStateException`(409, 상태 문제). 각 잘못을 맞는 코드로.

---

## 검증 결과 (실제 실행)

### 주문 생성 → 결제 (정상)
```bash
$ curl -X POST /orders -d '{"restaurantId":3,"items":[{"menuId":3,"quantity":1},{"menuId":4,"quantity":3}]}'
{"id":2,"restaurantId":3,"totalPrice":19500,"paid":false,"items":[...]}   # 4500*1+5000*3=19500 ✅ (99KB→250B)

$ curl -X POST /orders/2/payment
{... "paid":true ...}        # 재고 차감: 아메리카노 50→49, 카페라떼 7→4 ✅
```

### 예외 처리 (전부 깔끔)
```bash
$ curl -X POST /orders/999/payment   → 404 {"status":404,"message":"주문을 찾을 수 없습니다. orderId=999"}
$ curl -X POST /orders/1/payment     → 409 {"status":409,"message":"재고 부족: 감자튀김 (재고 0 < 주문 1)"}
$ curl -X POST /orders/2/payment     → 409 {"status":409,"message":"이미 결제된 주문입니다. orderId=2"}
$ POST /orders (restaurantId:99)     → 404 식당 없음   (예전엔 500)
$ POST /orders (menuId:99)           → 404 메뉴 없음   (예전엔 NPE 500)
```

### @Valid 입력검증
```bash
$ quantity:-5      → 400 "items[0].quantity: quantity는 1 이상이어야 합니다."
$ items:[]         → 400 "items: items는 최소 1개 이상이어야 합니다."
$ restaurantId 누락 → 400 "restaurantId: restaurantId는 필수입니다."
$ 정상(수량2)       → 200 totalPrice 10000 ✅
```

---

## 헷갈렸던 점 (Q&A 정리)

1. **`static`이 "객체 없이 호출"이라는 게 뭔지** → 붕어빵 틀(클래스 소속) vs 붕어빵(객체). 틀은 붕어빵 없이도 쓴다. (1장)
2. **DTO에 setter 왜 없나** → 응답은 getter로 읽기만. from()이 같은 클래스라 private 직접 접근. (1장)
3. **OrderItem이 menu_id·order_id 둘 다 갖는 이유** → 서로 다른 질문(무슨 상품/어느 영수증). (2장)
4. **order가 왜 @ManyToOne** → 학생-반: 학생 여럿이 한 반. (2장)
5. **무한루프가 왜·어떻게** → 마주보는 두 거울, OrderResponse가 한 방향 통행로로 끊음. (4장)
6. **`Menu menu = item.getMenu()` 어디서 가져오나** → `Menu menu`가 선언, getMenu()는 @ManyToOne으로 JPA가 자동 로드. (3장)
7. **프로시저 vs 트랜잭션** → 로직을 DB냐 자바냐. 요즘은 @Transactional+서비스. (5장)
8. **IllegalStateException** → "상태가 잘못됨"용 표준 예외. (6장)

---

## 다음 할 일
- [ ] RestaurantResponse DTO (식당 응답도 정리), GET /orders 목록 조회
- [ ] 테스트(JUnit + MockMvc) — 손으로 curl 말고 자동 검증
- [ ] (선택) 매출/잔돈 Money 로직, DB 마이그레이션(Flyway)
- [ ] Menu 등록 쪽 .orElse(null)도 NotFoundException으로 통일
