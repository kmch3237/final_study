# Day 2 — 회원가입을 실제 DB까지 연결 (수직 슬라이스 완성)

> 2026-05-28 · 폼 → Controller → Service → Mapper → Oracle DB 전 구간을 연결. 가입 버튼을 누르면 `USER_ACCOUNT` + `USER_INFO` 두 테이블에 INSERT 되는 것이 목표. **(성공!)**

---

## 0. 오늘 완성한 전체 흐름

```
enrollForm.jsp (폼: name=loginId, password, ...)
  → MemberController.enroll(Member member)     [Spring이 폼 → Member 자동 바인딩]
  → MemberService.enroll(member)               [@Transactional — 한 묶음]
  → MemberMapper.insertAccount(member)         [USER_ACCOUNT INSERT, 시퀀스로 USER_ID 생성]
  → MemberMapper.insertInfo(member)            [USER_INFO INSERT, 같은 USER_ID 재사용]
  → Oracle DB (USER_ACCOUNT, USER_INFO) ✅
```

핵심: **하나의 가입 = 두 테이블 INSERT**. 이걸 한 트랜잭션으로 묶는 게 오늘의 큰 주제.

---

## 1. 테이블 설계 (노션 명세서 → Oracle)

회원가입은 **2개 테이블**(1:1, `USER_ID` 공유). `USER_INFO.USER_ID`가 `USER_ACCOUNT`를 가리키는 **FK**.

```sql
-- 번호표 기계 (USER_ID 자동 채번)
CREATE SEQUENCE SEQ_USER_ID START WITH 1 INCREMENT BY 1 NOCACHE;

CREATE TABLE USER_ACCOUNT (
    USER_ID     NUMBER          PRIMARY KEY,
    LOGIN_ID    VARCHAR2(50)    NOT NULL UNIQUE,
    PASSWORD    VARCHAR2(500)   NOT NULL,
    NICKNAME    VARCHAR2(50)    NOT NULL UNIQUE,
    CREATED_AT  DATE   DEFAULT SYSDATE  NOT NULL
);

CREATE TABLE USER_INFO (
    USER_ID     NUMBER          PRIMARY KEY,
    EMAIL       VARCHAR2(100)   NOT NULL UNIQUE,
    NAME        VARCHAR2(50)    NOT NULL,
    PHONE       CHAR(11)        NOT NULL,
    GENDER      CHAR(1)         NOT NULL,
    BIRTHDATE   DATE            NOT NULL,
    CONSTRAINT USER_INFO_USER_ID_FK FOREIGN KEY (USER_ID) REFERENCES USER_ACCOUNT(USER_ID),
    CONSTRAINT USER_INFO_GENDER_CK  CHECK (GENDER IN ('M','F'))
);
```

> ⚠️ `USER_ID`(숫자 PK, 시퀀스) ≠ `LOGIN_ID`(로그인 문자열). 둘 다 "아이디"라 헷갈림. 폼에서 받는 건 `LOGIN_ID`, `USER_ID`는 시퀀스가 자동 부여(폼에 없음).

---

## 2. ⭐ 이름 3계층 매핑 (오늘 제일 중요)

```
폼 name  ──(Spring 자동 바인딩)──▶  VO 필드  ──(MyBatis가 SQL에서 연결)──▶  DB 컬럼
"loginId"                          loginId                              LOGIN_ID
```

| 구간 | 규칙 |
|---|---|
| 폼 `name` ↔ VO 필드 | **반드시 글자까지 일치** (Spring이 이름 보고 setter 호출) |
| VO 필드 ↔ DB 컬럼 | 꼭 같을 필요 없음 (mapper SQL에서 `#{loginId}` ↔ `LOGIN_ID` 연결) |

→ 이걸 어겨서 오늘 제일 크게 막힘 (아래 트러블슈팅 ③).

---

## 3. VO (Member) — Lombok

```java
package com.doit.app.model;

import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;

@Getter @Setter @NoArgsConstructor   // getter/setter/기본생성자 자동
public class Member {
    private Long   userId;            // PK (시퀀스), 폼엔 없음
    private String loginId, password, nickname, name, email, phone, gender, birthdate;
}
```

- `@Setter` → Spring이 폼값 담을 때 필요 / `@Getter` → MyBatis가 값 꺼낼 때 필요
- `@NoArgsConstructor` → Spring·MyBatis가 `new Member()`로 빈 객체 만든 뒤 setter로 채움
- 필드명은 **소문자 카멜** (`loginId`, `birthdate`) — 폼 name과 똑같이!

---

## 4. ⭐ @Mapper(인터페이스) vs Service(클래스) — 구현을 누가 짜느냐

| | Mapper | Service |
|---|---|---|
| 실제 구현을 누가 | **MyBatis가 자동** (mapper.xml SQL) | **우리가 직접** 작성 |
| 그래서 형태 | 인터페이스만 (Impl 만들면 안 됨!) | 클래스 필요 (`ServiceImpl`) |

> 비유: 인터페이스 = **메뉴판**, 클래스 = **주방**. Mapper는 주방(요리)을 MyBatis가 대신 해주니 메뉴판만 있으면 됨. Service는 로직(요리)을 우리가 하니 주방이 필요.
> → ❌ `MemberMapperImpl` 만들지 않는다. (mapper.xml이 곧 그 구현)

```java
@Mapper
public interface MemberMapper {
    int insertAccount(Member member);   // 테이블 2개 → insert 2개
    int insertInfo(Member member);
}
```

---

## 5. mapper.xml + ⭐ selectKey (NEXTVAL / CURRVAL)

두 테이블을 **같은 USER_ID**로 잇는 비결:

```xml
<mapper namespace="com.doit.app.mapper.MemberMapper">   <!-- 인터페이스 전체경로, 대소문자까지 일치 -->

  <insert id="insertAccount" parameterType="com.doit.app.model.Member">
    <selectKey keyProperty="userId" resultType="Long" order="BEFORE">
      SELECT SEQ_USER_ID.NEXTVAL FROM DUAL    <!-- 새 번호 뽑아 member.userId에 저장 -->
    </selectKey>
    INSERT INTO USER_ACCOUNT (USER_ID, LOGIN_ID, PASSWORD, NICKNAME)
    VALUES (#{userId}, #{loginId}, #{password}, #{nickname})
  </insert>

  <insert id="insertInfo" parameterType="com.doit.app.model.Member">
    INSERT INTO USER_INFO (USER_ID, EMAIL, NAME, PHONE, GENDER, BIRTHDATE)
    VALUES (#{userId}, #{email}, #{name}, #{phone}, #{gender},
            TO_DATE(#{birthdate}, 'YYYY-MM-DD'))    <!-- 문자열 → DATE 변환 -->
  </insert>
</mapper>
```

- `selectKey order="BEFORE"` = INSERT 전에 SELECT 실행 → 결과를 `keyProperty`(userId)에 set.
- `insertInfo`는 **selectKey 없음!** insertAccount가 채워둔 `member.userId`를 그대로 재사용. (또 NEXTVAL 뽑으면 번호가 달라져 FK 깨짐 = "번호표 다시 뽑기" 실수)
- 날짜는 `TO_DATE(#{birthdate}, 'YYYY-MM-DD')` 로 변환 (DATE 컬럼).

---

## 6. ⭐ Service + @Transactional (계좌이체 비유)

회원가입 = INSERT 2번. 계정만 들어가고 정보가 실패하면 **유령 회원**. 이걸 막는 게 `@Transactional`.

> 비유: **계좌이체** — 출금 + 입금이 한 묶음. 하나 실패하면 둘 다 취소. 가입도 정보 실패하면 계정도 자동 롤백.

| | sp14 (순수 JDBC) | 지금 (@Transactional) |
|---|---|---|
| 시작 | `setAutoCommit(false)` | 어노테이션 한 줄 |
| 성공 | `commit()` 직접 | 자동 |
| 실패 | `catch` 에서 `rollback()` | 예외 터지면 자동 |

```java
public interface MemberService { void enroll(Member member); }   // 업무 메서드 1개

@Service
public class MemberServiceImpl implements MemberService {
    private final MemberMapper memberMapper;
    public MemberServiceImpl(MemberMapper memberMapper) {   // 생성자 주입
        this.memberMapper = memberMapper;
    }
    @Override @Transactional
    public void enroll(Member member) {
        memberMapper.insertAccount(member);   // 출금
        memberMapper.insertInfo(member);      // 입금
    }
}
```

### 헷갈렸던 것: 메서드 개수 ≠ DB 작업 개수
- Service는 **업무 단위** `enroll` **1개**. 그 안에서 mapper insert 2개를 호출.
- xml에 INSERT 2개를 한 `<insert>`에 합치면 안 됨 → **`<insert>` 1개 = SQL 1개**.

### 의존성 주입 2가지
| | 필드 주입 `@Autowired` | 생성자 주입 (권장) |
|---|---|---|
| `final` 가능 | ❌ | ✅ |
| Spring 권장 | 비권장 | ✅ |

> Lombok `@RequiredArgsConstructor` 쓰면 `final` 필드용 생성자 자동 생성.

---

## 7. Controller 연결 (폼 → 객체 바인딩)

```java
@PostMapping("enroll")
public String enroll(Member member) {     // 파라미터로 Member 쓰면 Spring이 폼값 자동 바인딩
    try {
        service.enroll(member);
    } catch (Exception e) {
        log.info("enroll : ", e);           // @Slf4j 로 log 객체 자동
    }
    return "redirect:/member/login";
}
```

- 커스텀 객체(`Member`)는 **`@ModelAttribute` 생략 가능** (Spring이 자동으로 모델 어트리뷰트 취급).
- `Member`(대문자)=클래스(설계도), `member`(소문자)=객체(변수). `enroll(...)`엔 변수 `member`를 넘김.
- GET `enrollForm()`=폼 화면만, POST `enroll()`=실제 처리. (역할 분리)

---

## 8. 🐛 오늘 막힌 것 & 해결 (트러블슈팅 로그 — 오늘의 하이라이트)

| 증상 | 원인 | 해결 |
|---|---|---|
| `The blank final field memberMapper may not have been initialized` | **STS에 Lombok이 설치 안 됨** (Gradle 캐시에 jar는 있지만 IDE엔 미적용) → `@RequiredArgsConstructor`가 생성자 생성 못 함 | STS 폴더에 `lombok.jar` 복사 + `SpringToolSuite4.ini` 끝에 `-javaagent:...lombok.jar` 추가 → **STS 재시작** |
| `ORA-02289: 시퀀스가 존재하지 않습니다` | DDL을 앱 계정(**scott**)이 아닌 **다른 스키마**에 실행함 (스키마 불일치) | SQL Developer에서 **scott/tiger 접속**(service name=xe) 만들고, 거기서 시퀀스+테이블 재생성 |
| `ORA-17004: 열 유형이 부적합합니다 (1111)`<br>`property='nickName' ... setting null` | VO 필드가 `nickName`/`birthDate`(**대문자**)인데 폼 name은 `nickname`/`birthdate`(소문자) → **바인딩 실패 → null** → Oracle이 null의 컬럼 타입을 못 정함 | VO·mapper를 **소문자(`nickname`,`birthdate`)로 통일** (폼과 일치) |
| `ORA-12899: PHONE 값이 너무 큼 (실제 12, 최대 11)` | `PHONE CHAR(11)`인데 하이픈 포함 12자 입력 | 연락처를 **하이픈 없이 11자리 숫자**(`01012345678`)로 입력 |

---

## 9. 오늘의 핵심 6가지
1. **이름 3계층 일치**: 폼 `name` = VO 필드(대소문자까지!). 어긋나면 null → 에러.
2. **@Mapper는 Impl을 안 만든다** (MyBatis 자동). Service는 Impl 클래스 필요.
3. **selectKey + NEXTVAL/CURRVAL**: 첫 테이블에서 USER_ID 생성 → 둘째는 재사용 (새로 뽑으면 FK 깨짐).
4. **@Transactional**: 여러 INSERT를 한 묶음으로 (다 성공 or 다 롤백). 계좌이체.
5. **Lombok은 IDE에도 설치**해야 함 (Gradle 의존성만으론 STS 에디터가 모름).
6. **DB 객체는 앱이 접속하는 스키마(scott)에** 만들어야 함. 타입·길이도 명세서대로(`CHAR(11)` 등).

---

## ▶️ 다음(Day 3) 할 것
- **로그인** 실제 DB 연결: `LOGIN_ID`+`PASSWORD`로 조회 → 세션 처리.
- (여유되면) 가입 시 비밀번호 암호화, 폼 유효성 검사, 중복 아이디/닉네임 체크.
