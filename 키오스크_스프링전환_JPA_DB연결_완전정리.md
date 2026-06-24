# 콘솔 키오스크 → Spring Boot 전환: JPA + MySQL + yml 완전 정리

> 2026-06-24 · 기존 **Java 콘솔 키오스크**(파일 직렬화 저장)를 **Spring Boot REST API + MySQL**로 옮기는 학습.
> 오늘 범위: 프로젝트 부트스트랩 → 도메인(@Entity) → **DB 연결 + application.yml** → Repository 층. **(전 구간 기동/테이블 자동생성 검증 성공!)**

**스택:** Spring Boot 3.3.5 / Java 21 / Maven / **Spring Data JPA** / **MySQL 9.6**

---

## 0. 오늘 완성한 전체 흐름

```
[빌드/실행]   mvnw (Maven Wrapper)  →  ./mvnw spring-boot:run  →  내장 톰캣 8080 기동
[설정]        application.yml        →  포트 + DB 접속정보 + JPA 옵션
[도메인]      Restaurant.java        →  @Entity (DB의 restaurant 테이블과 자동 매핑)
[저장소]      RestaurantRepository   →  extends JpaRepository  (CRUD 자동 생성)
                       │
                       ▼
              MySQL `kiosk` DB  ──(Hibernate가 ddl-auto로 자동 생성)──▶  restaurant 테이블 ✅
```

핵심 한 줄: **"손으로 짜던 ArrayList + .ser 파일 저장"을, 스프링이 @Entity 한 클래스 + Repository 한 인터페이스로 대신해준다.**

---

## 1. 콘솔 vs 스프링 — 무엇이 바뀌나

| 구분 | 콘솔 (원본) | 스프링 (전환본) |
|------|------------|----------------|
| 실행 | `Main.main()` 직접 실행 | `SpringApplication.run()` 이 객체 생성 + 웹서버 기동 대행 |
| 입출력 | `BufferedReader` 콘솔 입력 | HTTP 요청/응답(JSON), URL로 자원 지정 |
| 데이터 저장 | `ArrayList` + `Serializable` + `.ser` 파일 | DB(MySQL) + JPA |
| 식별 | 리스트의 index(몇 번째) | 자원마다 고유 **id** (`GET /restaurants/1`) |
| 설정 | 코드에 하드코딩 | `application.yml` 외부 설정 |

> 💡 콘솔엔 "외부 설정"이라는 개념 자체가 없었음. 포트·DB주소 같은 걸 코드 밖 yml로 빼는 게 스프링 방식.

---

## 2. 프로젝트 부트스트랩 (지난 단계 복습)

- **`SpringApplication.run()`** : 객체 자동 생성 + 내장 톰캣 웹서버 기동을 대신함.
- **`mvnw` (Maven Wrapper)** : Maven 미설치 환경에서도 `./mvnw spring-boot:run` 으로 실행 가능.
- 8080 기동 후 `localhost:8080` → **404**. 이건 정상 — "서버는 살아있는데 그 URL에 매핑된 컨트롤러가 아직 없다"는 뜻.

### 표준 4계층 패키지
```
com/kiosk/
├── domain/      ← 데이터 (Restaurant)        [오늘 @Entity로]
├── repository/  ← 데이터 보관/조회           [오늘 생성]
├── service/     ← 비즈니스 로직              [다음]
└── controller/  ← URL 받기 (REST)            [다음]
```
> 타이핑은 **아래층(domain)부터** — 위층이 아래층을 쓰기 때문. (설계는 밖→안, 구현은 안→밖)

---

## 3. ⭐ 도메인을 @Entity 로 — 콘솔 클래스에서 딱 3가지만 변함

원본 `Restraunt.java` → 스프링 `Restaurant.java`:

```java
@Entity                                  // ① 이 클래스를 DB 테이블과 연결 (Restaurant → restaurant)
public class Restaurant {

    @Id                                  // ② 이 필드가 기본키(PK)
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 값 자동 발급(MySQL auto_increment)
    private Long id;                     //   ← REST에서 /restaurants/{id} 로 콕 집기 위해 신규 추가

    private String name;                 // 원본의 resName
    private String category;

    // 기본 생성자(필수) + getter/setter는 그대로 유지
}
```

| 변화 | 이유 |
|------|------|
| ① `implements Serializable` **제거** | 파일 직렬화 안 함. JSON으로 주고받고 DB에 저장하므로 |
| ② `id` 필드 **추가** + `@Id @GeneratedValue` | REST는 URL로 자원을 지정 → 자원마다 고유번호 필요 (콘솔 `serialNumber` 개념의 일반화) |
| ③ `private 필드 + getter/setter` **유지** | 오히려 더 중요해짐 → 스프링이 객체를 **JSON으로 바꿀 때 getter를 호출**해 값을 꺼냄. getName() 없으면 JSON에 name이 안 나옴 |

> 🔑 지난번 "getter = 읽기 문지기"가 여기서 실제로 쓰인다. 그리고 `setMenuQty()`처럼 **규칙이 있는 setter**(수량 0 → 품절 true)는 단순 대입이 아니라 변경 규칙을 한 곳에 가두는 장치.

---

## 4. ⭐ DB 연결 + application.yml

### (1) properties → yml 전환
Spring Boot는 둘 다 인식하지만, 설정이 많아지면 **들여쓰기로 계층이 보이는 yml**이 읽기 좋다.

```yaml
spring:
  application:
    name: kiosk-spring

  # ===== DB 연결 (MySQL) =====
  datasource:
    url: jdbc:mysql://localhost:3306/kiosk?serverTimezone=Asia/Seoul&characterEncoding=UTF-8
    username: root
    password: ""            # Homebrew MySQL 초기 root는 비밀번호 없음
    driver-class-name: com.mysql.cj.jdbc.Driver

  # ===== JPA / Hibernate =====
  jpa:
    hibernate:
      ddl-auto: update      # @Entity 보고 테이블 자동 생성/수정 (학습용. 운영은 validate/none)
    show-sql: true          # 실행되는 SQL을 콘솔에 출력 (공부에 유용)
    properties:
      hibernate:
        format_sql: true    # SQL 보기 좋게 들여쓰기

server:
  port: 8080
```

### (2) pom.xml 의존성 2개 추가
```xml
<!-- 객체(@Entity) ↔ 테이블 자동 매핑, JpaRepository 제공 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- 스프링 ↔ MySQL 통신 연결선 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

### (3) MySQL 준비 (macOS / Homebrew)
```bash
brew services start mysql                       # 서버 기동 (3306)
mysql -u root -e "CREATE DATABASE IF NOT EXISTS kiosk
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

> ⚠️ DB(`kiosk`)는 **사람이 직접 만들어야** 한다. 그 안의 **테이블**은 `ddl-auto: update`가 @Entity를 보고 자동 생성.

### `ddl-auto` 옵션 요약
| 값 | 동작 |
|----|------|
| `none` | 아무것도 안 함 |
| `validate` | 엔티티 ↔ 테이블 일치만 검사 (운영 권장) |
| `update` | 없으면 만들고, 바뀌면 컬럼 추가 (**학습용 — 오늘 사용**) |
| `create` / `create-drop` | 매번 다시 만듦(데이터 날아감) |

---

## 5. ⭐ Repository — 인터페이스 한 줄로 CRUD 끝

```java
public interface RestaurantRepository extends JpaRepository<Restaurant, Long> { }
//                                                          └ 엔티티 ┘  └ id타입 ┘
```

이 **한 줄**로 아래가 공짜로 생긴다 (구현 클래스 직접 안 만듦):

| 메서드 | SQL | 콘솔 대응 |
|--------|-----|----------|
| `save(r)` | INSERT / UPDATE | 등록·수정 |
| `findById(id)` | SELECT … WHERE id=? | 특정 가게 찾기 |
| `findAll()` | SELECT * | "가게 목록 보기" |
| `deleteById(id)` | DELETE | 삭제 |
| `count()`, `existsById()` | … | — |

> 🔑 **핵심 깨달음**: `implements`로 구현체를 우리가 안 만든다. 인터페이스만 선언하면 스프링이 **실행 시점에 진짜 SQL을 짜는 구현체(프록시)를 자동 생성**해 빈으로 꽂아준다.
> "무엇을(인터페이스)"만 선언하고 "어떻게(SQL)"는 프레임워크에 위임 — 이게 Spring Data의 핵심.
>
> 콘솔 `RestrauntAdminManagement`에서 손으로 짜던 add/검색 반복문/파일저장이 전부 이걸로 대체됨.

### 쿼리 메서드 (예고)
필요하면 **메서드 이름만** 선언하면 스프링이 이름을 해석해 쿼리를 만든다:
```java
List<Restaurant> findByCategory(String category);   // → SELECT … WHERE category = ?
```

---

## 6. 검증 결과 (실제로 동작 확인)

```bash
./mvnw spring-boot:run      # → "Started KioskApplication" (5초), 톰캣 8080
```
- Hibernate가 콘솔에 `create table restaurant (...)` SQL 출력
- MySQL에서 확인:
```
mysql> USE kiosk; DESCRIBE restaurant;
+----------+--------+------+-----+---------+----------------+
| Field    | Type   | Null | Key | Default | Extra          |
+----------+--------+------+-----+---------+----------------+
| id       | bigint | NO   | PRI | NULL    | auto_increment |   ← @GeneratedValue 동작
| category | varchar(255) | YES | | NULL |                 |
| name     | varchar(255) | YES | | NULL |                 |
+----------+--------+------+-----+---------+----------------+
```
- RestaurantRepository 추가 후에도 기동 성공 → 빈 정상 등록 확인 ✅

---

## 7. 헷갈렸던 점 / 메모

- **getter/setter는 세트가 아니다.** 판단 기준 3가지:
  1. 밖에서 읽어야 하나? → getter (아니면 안 만듦)
  2. 밖에서 바꿔야 하나? → setter (안 바뀌어야 하면 `final` + setter 없음, 예: `serialNumber`)
  3. 바꿀 때 규칙이 있나? → setter 안에 규칙을 넣음 (예: `setMenuQty` → soldOut 자동)
- `localhost:8080` 404는 에러 아님 → 컨트롤러(URL 매핑)가 아직 없을 뿐.
- DB(`kiosk`)는 수동 생성, 테이블은 자동 생성 — 둘을 구분.

---

## 8. 다음 할 일

- **service 층**: `@Service` + **생성자 주입(DI)**으로 Repository를 받아 비즈니스 로직 작성.
- **controller 층**: `@RestController`로 `GET /restaurants`, `POST /restaurants` 등 URL 매핑.
- 더미 데이터 3건 넣어 `findAll()` 응답을 JSON으로 확인.
