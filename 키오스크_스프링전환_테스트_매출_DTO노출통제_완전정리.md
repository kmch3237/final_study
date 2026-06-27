# 콘솔 키오스크 → Spring Boot 전환 ④: JUnit 테스트 · 매출 집계 · DTO 노출통제 완전 정리

> 2026-06-27 · 콘솔 키오스크를 Spring Boot REST API로 옮기는 4일차.
> 오늘 범위: **JUnit 테스트 3종(단위 / Mockito / MockMvc)** → **매출(sales) 집계** → **RestaurantResponse DTO로 매출 노출 차단**.
> **(테스트 7개 전부 통과 + 매출/노출차단 라이브 검증 성공!)**

**스택:** Spring Boot 3.3.5 / Java 21 / Maven / Spring Data JPA / MySQL 9.6 / JUnit5 · Mockito · MockMvc

---

## 0. 오늘 완성한 전체 흐름

```
[자동 테스트]  ./mvnw test  →  코드가 스스로를 검증 (서버/curl 불필요)
   ├─ MenuTest (도메인)        : new Menu() 만으로 품절 규칙 검증        ← 순수, 빠름
   ├─ PaymentServiceTest (서비스): 가짜(Mock) Repository로 결제 로직만   ← DB 없이
   └─ OrderControllerTest (웹)  : MockMvc로 'HTTP 요청 흉내' (200/400)   ← 진짜에 가까움

[새 기능] 결제 성공 시 → 식당.sales += 주문 총액  (dirty checking으로 자동 DB 반영)
[보안]    매출(sales)이 GET /restaurants에 새던 것 → RestaurantResponse DTO로 차단
```

핵심 한 줄: **"손으로 curl 치던 검증을 코드가 대신하게 만들고, 새 기능(매출)이 만든 정보 노출 구멍을 DTO로 막았다."**

---

## 1. ⭐ 테스트 코드 — 왜, 그리고 given/when/then

지금까진 코드 바꿀 때마다 **사람이** 서버 켜고 curl 치고 눈으로 확인했다. 테스트는 그 검증을 **코드로 박아 자동 실행**한다.

| | 손 검증 (어제까지) | 테스트 코드 (오늘) |
|---|---|---|
| 실행 | 매번 서버+curl | `./mvnw test` 한 방 |
| 반복 | 사람이 매번 | 자동 |
| 회귀 방지 | 까먹으면 끝 | 깨지면 **빨간불**로 즉시 알림 |

> ⭐ **핵심 가치 = 회귀 방지.** 나중에 코드를 고쳤을 때 "예전 기능이 깨졌네?"를 **사람이 아니라 테스트가** 잡는다. (실제로 오늘 매출 기능을 추가하니 기존 테스트가 깨지며 알려줬다 — 5장.)

모든 테스트는 **given(준비) → when(실행) → then(검증)** 3단:
```java
@Test                              // "이건 테스트다" 표시 — 없으면 '조용히 무시'된다(중요 함정!)
@DisplayName("수량이 0이면 품절 처리된다")  // 결과에 보일 한글 이름
void 수량0이면_품절() {
    Menu menu = new Menu();              // given: 준비
    menu.setMenuQty(0);                  // when:  검사할 동작
    assertThat(menu.isSoldOut()).isTrue(); // then: 기대와 같은지 단언(틀리면 실패)
}
```

> ⚠️ **오늘 직접 겪은 함정:** `@Test`를 빠뜨리면 그 메서드는 테스트로 **인식되지 않고 조용히 무시**된다(에러도 없음). "분명 테스트인데 왜 안 돌지?"의 단골 원인.

---

## 2. ⭐ 단위 테스트 (MenuTest) — 스프링도 DB도 없이

`Menu`의 "수량 0이면 품절" 규칙은 **객체 하나면** 검증된다. 스프링 컨텍스트도, DB도 필요 없다 → 가장 빠르고 단순한 테스트.

```java
class MenuTest {
    @Test @DisplayName("수량이 1 이상이면 품절이 아니다")
    void 수량양수면_품절아님() {
        Menu menu = new Menu();
        menu.setMenuQty(5);
        assertThat(menu.isSoldOut()).isFalse();
    }

    @Test @DisplayName("재입고하면 품절이 해제된다")
    void 재입고하면_품절해제() {
        Menu menu = new Menu();
        menu.setMenuQty(0);   // 품절시켰다가
        menu.setMenuQty(5);   // 재입고
        assertThat(menu.isSoldOut()).isFalse();
    }
}
```

---

## 3. ⭐ 서비스 테스트 (PaymentServiceTest) — Mock = 대역 배우

`PaymentService.pay()`는 내부에서 `orderRepository.findById()`를 부른다. 테스트하려고 매번 진짜 MySQL을 켜는 건 느리고 번거롭다. 그래서 **가짜(Mock) Repository**를 끼운다.

> 🎬 **대역 배우 비유:** 영화의 위험한 장면은 진짜 배우 대신 **스턴트 대역**을 쓴다. Mock이 그 대역. 진짜 DB(느리고 무거움) 대신 **"findById(1) 부르면 이 주문 줄게"라고 미리 짜둔 가짜**를 넣어, `PaymentService`의 **로직만** 떼어 빠르게 검증한다.

```java
@ExtendWith(MockitoExtension.class)   // Mockito 사용 선언
class PaymentServiceTest {
    @Mock        private OrderRepository orderRepository;   // 가짜 저장소(대역 배우)
    @InjectMocks private PaymentService paymentService;     // 가짜를 끼운 진짜 서비스(테스트 대상)

    @Test @DisplayName("재고가 충분하면 결제되고 재고가 차감된다")
    void 결제성공_재고차감() {
        Order order = 주문하나(10, 3);   // 재고10, 주문3
        given(orderRepository.findById(1L)).willReturn(Optional.of(order)); // 대본: 이렇게 물으면 이걸 답해라

        Order result = paymentService.pay(1L);

        assertThat(result.isPaid()).isTrue();
        assertThat(result.getOrderItems().get(0).getMenu().getMenuQty()).isEqualTo(7);   // 10-3
        assertThat(result.getRestaurant().getSales()).isEqualTo(15000);                  // 매출 누적
    }

    @Test @DisplayName("재고보다 많이 주문하면 결제 시 예외가 터진다")
    void 재고부족_예외() {
        Order order = 주문하나(2, 5);   // 재고2 < 주문5
        given(orderRepository.findById(1L)).willReturn(Optional.of(order));

        assertThatThrownBy(() -> paymentService.pay(1L))   // "실행하면 예외 나야 통과"
                .isInstanceOf(IllegalStateException.class);
    }
}
```

> 💡 `@Mock`(가짜 생성) → `@InjectMocks`(그 가짜를 진짜 서비스에 주입) → `given(...).willReturn(...)`(대본).
> 💡 `assertThatThrownBy(() -> 코드)` : 람다로 코드를 **넘겨서** 검증기가 실행 → 예외가 나는지 본다. (바로 실행하면 테스트가 거기서 죽으므로 람다로 감싼다.) 정상 케이스 `assertThat`의 '실패 검증' 짝꿍.

---

## 4. ⭐ 웹 계층 테스트 (OrderControllerTest) — MockMvc = 가짜 손님

MockMvc는 **HTTP 요청을 자바 코드로 흉내내서** 컨트롤러를 때리고 응답을 검증한다. 진짜 서버를 켜지 않는다. = "curl을 코드로".

> 🤖 **가짜 손님 비유:** 진짜 손님(브라우저/curl)을 부르는 대신, **코드로 만든 가짜 손님**이 요청을 보내고 응답이 맞는지 확인한다.

```java
@WebMvcTest(OrderController.class)   // 컨트롤러 + 검증/예외처리 웹 계층만 띄움(Service/DB 제외)
class OrderControllerTest {
    @Autowired private MockMvc mockMvc;          // 가짜 손님
    @MockBean  private OrderService orderService;     // 가짜 Service
    @MockBean  private PaymentService paymentService;

    @Test @DisplayName("정상 주문이면 200과 주문 결과를 응답한다")
    void 정상주문_200() throws Exception {
        Order order = new Order(); order.setTotalPrice(10000);
        given(orderService.create(any())).willReturn(order);

        mockMvc.perform(post("/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"restaurantId\":2,\"items\":[{\"menuId\":1,\"quantity\":2}]}"))
                .andExpect(status().isOk())                          // 200?
                .andExpect(jsonPath("$.totalPrice").value(10000));   // 응답 JSON의 totalPrice?
    }

    @Test @DisplayName("수량이 음수면 400 Bad Request")
    void 음수수량_400() throws Exception {
        // given 대본 없음! @Valid가 Service 가기 전에 막으므로 가짜 호출조차 안 됨.
        mockMvc.perform(post("/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content("{\"restaurantId\":2,\"items\":[{\"menuId\":1,\"quantity\":-5}]}"))
                .andExpect(status().isBadRequest());                 // 400?
    }
}
```

> 💡 `@MockBean` = 어제 `@Mock`과 같은 개념(컨트롤러가 의존하는 Service를 가짜로).
> 💡 음수수량 테스트엔 **대본(given)이 없다.** `@Valid`가 입구에서 끊어 Service까지 안 가므로. → 검증/콜센터(`@RestControllerAdvice`)가 **진짜로** 동작함을 확인.

### 테스트 피라미드 (오늘 3층 완성)
```
   OrderControllerTest (웹)    ← 느리지만 진짜에 가까움 (HTTP 전 과정)
   PaymentServiceTest (서비스)  ← Mock으로 로직만
   MenuTest (도메인)            ← 가장 빠르고 단순 (순수 객체)
```
> 아래로 갈수록 빠르고 많이, 위로 갈수록 적게. 실무 테스트 전략의 기본 모양.

---

## 5. ⭐ 매출(sales) 집계 — dirty checking & 한 루프 리팩터링

원본 콘솔 `Kiosk.java`에 있던 `private int sales;`(매출)를 되살려, **결제 성공 시 식당 매출에 주문 총액을 누적**한다.

```java
// Restaurant 엔티티
private int sales;   // 매출액 (결제 시 누적)

// PaymentService.pay() 끝부분
order.setPaid(true);
Restaurant restaurant = order.getRestaurant();                       // @ManyToOne으로 들고 있는 식당(영속)
restaurant.setSales(restaurant.getSales() + order.getTotalPrice());  // 매출 += 총액
return order;
```

> ⭐ **dirty checking:** `restaurantRepository.save()`를 **부르지 않았는데도** DB에 매출이 저장된다. 트랜잭션 안에서 영속 상태 객체를 바꾸면 JPA가 트랜잭션 끝에 **알아서 UPDATE**한다. (재고 차감도 같은 원리.)
> ⭐ **@Transactional 재사용:** 재고 차감과 매출 가산이 **같은 거래**라, 결제가 롤백되면 매출도 함께 취소된다(돈이라 중요).

### 두 루프 → 한 루프 리팩터링
재고 '검사' 루프와 '차감' 루프를 둘로 나눴던 것을, if/else로 **한 루프**에 합쳤다.
```java
for (OrderItem item : order.getOrderItems()) {
    Menu menu = item.getMenu();
    if (menu.getMenuQty() < item.getQuantity())
        throw new IllegalStateException("재고 부족: ...");   // 모자라면 throw → @Transactional 롤백
    menu.setMenuQty(menu.getMenuQty() - item.getQuantity()); // 통과한 줄만 차감
}
```
> 💡 `@Transactional`이 받쳐주니 중간에 실패해도 앞서 깎인 게 롤백된다 → 한 루프도 안전. (두 루프는 "전부 검사 후 변경"이라 의도가 더 명확하지만, 트랜잭션이 있으면 한 루프로 충분.)

### Order ↔ OrderItem 다시 (영수증 비유)
```
영수증(Order) 1장 ── 여러 줄(OrderItem) ── 각 줄은 메뉴 1개를 가리킴
for (OrderItem item : order.getOrderItems())  // "영수증의 줄들을 한 줄씩 짚어라" (item = 손가락이 짚은 한 줄)
item.getMenu()                                 // 그 줄이 가리키는 메뉴 (@ManyToOne, JPA가 자동 로드)
```

---

## 6. ⭐ RestaurantResponse DTO — DTO의 또 다른 역할: 노출 통제

매출(`sales`)을 추가하자 **보안 구멍**이 생겼다. `GET /restaurants`가 `Restaurant` 엔티티를 그대로 응답했기에, getter가 생긴 `sales`까지 JSON에 새어나갔다 → **손님이 가게 매출을 다 봄.**

```java
// 해결: 보여줄 것만 담는 DTO
public class RestaurantResponse {
    private Long id;
    private String name;
    private String category;
    // sales 없음 — 일부러 안 담는다(내부 정보)

    public static RestaurantResponse from(Restaurant r) {
        RestaurantResponse dto = new RestaurantResponse();
        dto.id = r.getId(); dto.name = r.getName(); dto.category = r.getCategory();
        return dto;
    }
    // getter만 (응답은 읽기 전용)
}
// 컨트롤러: return restaurantService.findAll().stream().map(RestaurantResponse::from).toList();
```

> ⭐ **DTO의 두 얼굴이 모였다:**
> - `MenuResponse`/`OrderResponse` → **무한루프 차단** (양방향 참조 끊기)
> - `RestaurantResponse` → **민감정보 노출 차단** (sales 가리기)
>
> 공통 원칙: **"엔티티(DB 구조)를 바깥에 그대로 주지 마라. DTO로 무엇을 내보낼지 통제하라."**

---

## 검증 결과 (실제 실행)

### 테스트 (7개 전부 통과)
```
MenuTest            : 3개 (품절 규칙)
PaymentServiceTest  : 2개 (결제 성공 + 재고부족 예외)
OrderControllerTest : 2개 (정상 200 + 음수수량 400)
→ Tests run: 7, Failures: 0, Errors: 0   BUILD SUCCESS
```

### 매출 집계 (라이브)
```bash
결제 전:  스타벅스 sales = 0
$ POST /orders {restaurantId:3, items:[{menuId:3, quantity:2}]}   → totalPrice 9000
$ POST /orders/5/payment                                         → paid:true
결제 후:  스타벅스 sales = 9000   ✅ (save() 안 했는데 dirty checking으로 DB 반영)
```

### 매출 노출 차단 (라이브)
```bash
$ GET /restaurants     → [{"id":1,"name":"김밥천국","category":"분식"}, ...]   # sales 없음 ✅
$ GET /restaurants/3   → {"id":3,"name":"스타벅스","category":"카페"}          # 매출 9000인데 안 보임 ✅
```

---

## 헷갈렸던 점 (Q&A)
1. **두 for문 합칠 수 있나?** → 가능. if/else 한 루프 + `@Transactional`로 안전. (5장)
2. **Order vs OrderItem** → 영수증(Order) vs 영수증 한 줄(OrderItem). `for`는 줄을 한 줄씩 짚는 것. (5장)
3. **이 헷갈림이 DTO 개념 부족 때문?** → 아니다. 여기 `order`/`item`은 전부 **엔티티**(영수증 실물)다. DTO(`OrderResponse`)는 응답할 때만 잠깐 등장 — 둘은 별개 관심사. (DTO 자체는 잘 이해함)
4. **`@Test` 빠지면?** → 테스트로 인식 안 되고 조용히 무시됨. (1장)

---

## 다음 할 일
- [ ] GET /orders 목록 조회
- [ ] 잔돈 계산(Money) — 원본 콘솔의 동전 단위 거스름돈
- [ ] 테스트 더 채우기 (OrderService, RestaurantController)
- [ ] (선택) DB 마이그레이션(Flyway), 인증/로그인
