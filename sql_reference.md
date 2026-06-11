# SQL 레퍼런스 — 컬럼 사전 + 학생 담당 쿼리 총정리

영문 컬럼명 보면 머릿속에 한국어로 즉시 떠오르게 만드는 것이 목표.
학생 담당 도메인(User / Cafe / Theme / Manager / Attendance) 중심.

---

# Part A — 영문 컬럼을 한국어로 "읽는" 법

## A-1. 머릿속 매핑 룰 5개

### 룰 1: 접미사 `_ID` = "PK/FK (번호)"
- `USER_ID` = "회원 번호"
- `CAFE_ID` = "카페 번호"
- `ROOM_ID` = "테마 번호" (테이블이 ROOM이지만 화면에선 "테마")

### 룰 2: 접미사 `_AT` = "시각 (시점)"
- `CREATED_AT` = "생성된 시각"
- `OPEN_AT` = "오픈된 시각" = "예약 시간"
- `_AT` 보이면 DATE/TIMESTAMP, **TO_DATE 또는 fmt:formatDate 필요**

### 룰 3: 접미사 `_NO`, `_CODE` = "특정 번호 문자열"
- `BR_NO` = Business Registration Number = "사업자등록번호"
- `POSTAL_CODE` = "우편번호"
- 보통 `CHAR` 또는 `VARCHAR2` 자릿수 고정

### 룰 4: 접미사 `_NAME`, `_DESC` = "이름/설명 (텍스트)"
- `CAFE_NAME` = "카페 이름"
- `ROOM_DESC` = "테마 설명"

### 룰 5: `STATUS_ID`, `EVENT_ID`, `REASON_ID` = "코드 테이블 참조 (FK)"
- `REG_EVENT_ID` = "등록 이벤트 코드" (1=등록, 2=해제 — REG_EVENT 테이블 참조)
- `ATTEND_STATUS_ID` = "출석 상태 코드" (1=완료, 2=노쇼)
- 폼에서 직접 1, 2 보내면 안 됨 → **셀렉트박스로 코드 테이블 조회해 옵션 채움**

## A-2. 자주 헷갈리는 짝 — 같은 단어 다른 의미

| 컬럼 | 진짜 의미 | 헷갈리지 말 것 |
|---|---|---|
| `USER_ID` | 시퀀스로 박은 숫자 PK (1, 2, 3…) | LOGIN_ID 아님! 화면 표시용 X |
| `LOGIN_ID` | 사용자가 로그인할 때 치는 문자열 (test01) | USER_ID 아님! 길이 6~15 |
| `PASSWORD` | 암호화된 문자열 (VARCHAR2 500) | 평문 아님 — CRYPTPACK.ENCRYPT 필수 |
| `NICKNAME` | 화면에 닉네임으로 표시 | NAME 아님 |
| `NAME` | 실제 본명 (USER_INFO에 있음) | NICKNAME 아님 |
| `PHONE` | 하이픈 없는 11자리 | 하이픈 포함 X (`010-1234-5678` ❌) |
| `BIRTHDATE` | DATE 타입 | 문자열 아님 — **TO_DATE 필수** |
| `BR_NO` | 하이픈 없는 10자리 사업자번호 | `123-45-67890` ❌ |
| `ROOM_ID` (테이블) | 테마 번호 | 화면·폴더는 "theme"인데 DB는 "ROOM" |
| `CAFE.USER_ID` | 카페 소유 사장의 USER_ID | 카페에 들어가는 USER_ID 아님 |
| `OPEN_AT` | 예약 시작 시간 (방탈출 시작 시각) | 카페 오픈 시각 아님 |

## A-3. 자주 보는 패턴 약어

| 약어 | 풀이 | 예시 |
|---|---|---|
| `VRA` | View Reservation All — 예약-카페-테마-파티 통합 뷰 | 다른 분(yuhwon) 자산 |
| `VAM` | V_ACTIVE_MANAGER — 활성 매니저 뷰 (학생 본인 작성) | Day 9 |
| `UA` | USER_ACCOUNT | 로그인 정보 |
| `UI` | USER_INFO | 개인 정보 (이메일/이름/전화/성별/생일) |
| `MH` | MANAGER_HISTORY | 매니저 임명/해제 이력 |
| `AD` | ATTENDANCE_DETAIL | 출석 상세 (한 명당 한 줄) |

---

# Part B — 학생 담당 테이블 사전

**보는 순서 = 학생이 매일 쓰는 빈도순.**

## B-1. USER_ACCOUNT (회원가입 핵심)

> "로그인할 때 필요한 정보만 모은 테이블"

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `USER_ID` | NUMBER PK | 회원 번호 (시퀀스) | USER_ACCOUNT_SEQ.NEXTVAL |
| `LOGIN_ID` | VARCHAR2(15) UNIQUE | 로그인 아이디 (문자열) | 6~15자 |
| `PASSWORD` | VARCHAR2(500) | 비밀번호 (암호화) | `CRYPTPACK.ENCRYPT(평문, '12341234')` |
| `NICKNAME` | VARCHAR2 UNIQUE | 닉네임 | 2~10자 |
| `CREATED_AT` | DATE DEFAULT SYSDATE | 가입 시각 | **단수형! CREATE_AT** ← 함정 |

## B-2. USER_INFO (회원 상세)

> "USER_ACCOUNT와 1:1. 개인 정보만 분리."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `USER_ID` | NUMBER PK + FK | 회원 번호 | USER_ACCOUNT.USER_ID 와 같은 값 |
| `EMAIL` | VARCHAR2 UNIQUE | 이메일 | |
| `NAME` | VARCHAR2 | 실명 | NICKNAME과 다름 |
| `PHONE` | CHAR(11) | 핸드폰 번호 | 하이픈 X |
| `GENDER` | CHAR(1) | 성별 | M/F 만 |
| `BIRTHDATE` | DATE NOT NULL | 생년월일 | **TO_DATE 필수** |

## B-3. CAFE (방탈출 카페)

> "사장(USER)이 운영하는 카페. USER_ID FK가 사장."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `CAFE_ID` | NUMBER PK | 카페 번호 (시퀀스) | CAFE_SEQ.NEXTVAL |
| `USER_ID` | NUMBER NOT NULL FK | **카페 소유주 USER_ID** | 세션 loginUser.userId |
| `BR_NO` | CHAR(10) UNIQUE | 사업자등록번호 | 하이픈 X |
| `CAFE_NAME` | VARCHAR2 | 카페 이름 | |
| `PHONE` | VARCHAR2(50) | 카페 전화번호 | 하이픈 X |
| `POSTAL_CODE` | CHAR(5) | 우편번호 | |
| `ADDRESS` | VARCHAR2 | 주소 | |
| `ADDRESS_DETAIL` | VARCHAR2 nullable | 상세주소 | |
| `CREATED_AT` | DATE | 카페 등록 시각 | |

⚠️ **영업시간/소개/대표자 컬럼 없음** — 폼에 받지 말 것.

## B-4. ROOM (= 테마 — 테이블명 주의!)

> "한 카페에 N개의 방. 화면·폴더는 'theme'인데 DB는 'ROOM'."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `ROOM_ID` | NUMBER PK | 테마 번호 | ROOM_SEQ.NEXTVAL |
| `CAFE_ID` | NUMBER FK | 어느 카페의 테마인지 | **폼에서 받지 말 것! 세션 사장 카페 조회** |
| `ROOM_NAME` | VARCHAR2 | 테마 이름 | |
| `GENRE_ID` | NUMBER FK | 장르 코드 | ROOM_GENRE 테이블 (공포/SF/스릴러 등) |
| `DURATION` | NUMBER | 소요시간 (분) | |
| `DIFFICULTY` | NUMBER FK | 난이도 코드 (1~5) | LEVEL 테이블 |
| `HORROR_LEVEL` | NUMBER FK | 공포도 코드 (1~5) | LEVEL 테이블 |
| `ACTIVITY_LEVEL` | NUMBER FK | 활동성 코드 (1~5) | LEVEL 테이블 |
| `MIN_PLAYERS` | NUMBER(2) | 최소 인원 | ≥ 1 |
| `MAX_PLAYERS` | NUMBER(3) | 최대 인원 | ≥ MIN |
| `ROOM_DESC` | VARCHAR2 nullable | 테마 설명 | |
| `ROOM_IMG` | VARCHAR2(200) NOT NULL | 이미지 파일명 | **NOT NULL → 수정 시 빈값 X** |
| `PRICE` | NUMBER(6) | 가격 | |
| `IS_ADULT` | NUMBER FK | 성인여부 코드 | COMMON 테이블 (Y/N) |

## B-5. MANAGER_HISTORY (매니저 임명·해제 이력)

> "현재 상태 X. 임명/해제 이벤트를 누적해 쌓음."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `MANAGER_HISTORY_ID` | NUMBER PK | 이력 번호 (시퀀스) | MANAGER_HISTORY_SEQ.NEXTVAL |
| `CAFE_ID` | NUMBER FK | 어느 카페의 매니저인지 | |
| `USER_ID` | NUMBER FK | 임명될/해제될 사람 USER_ID | |
| `REG_EVENT_ID` | NUMBER FK | 등록 이벤트 코드 | **1=등록, 2=해제** (REG_EVENT 테이블) |
| `CREATED_AT` | DATE | 이력 발생 시각 | |

→ 활성 매니저 = `MAX(MANAGER_HISTORY_ID)` 행이 `REG_EVENT_ID = 1`

## B-6. ATTENDANCE (출석체크 헤더)

> "한 예약당 한 행. 누가 처리했는지 기록."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `ATTENDANCE_ID` | NUMBER PK | 출석 헤더 번호 | ATTENDANCE_SEQ.NEXTVAL |
| `RESERVATION_ID` | NUMBER FK | 어떤 예약의 출석체크인지 | |
| `USER_ID` | NUMBER FK nullable | **누가 처리했는지** (사장/매니저) | **NULL = 스케줄러 자동처리** |

## B-7. ATTENDANCE_DETAIL (파티원별 출석 상태)

> "한 예약의 파티원 N명에 대해 각자 한 행."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `DETAIL_ID` | NUMBER PK | 상세 번호 | ATTENDANCE_DETAIL_SEQ.NEXTVAL |
| `ATTENDANCE_ID` | NUMBER FK | 어느 출석 헤더에 속하는지 | |
| `USER_ID` | NUMBER FK | 그 파티원 USER_ID | |
| `ATTEND_STATUS_ID` | NUMBER FK | **출석 상태 코드** | **1=완료, 2=노쇼** |

## B-8. MANNER_HISTORY (매너온도 이력)

> "노쇼 같은 이벤트마다 SCORE 누적."

| 컬럼 | 타입 | 한국어 의미 | 비고 |
|---|---|---|---|
| `MANNER_HISTORY_ID` | NUMBER PK | 이력 번호 | MANNER_HISTORY_SEQ.NEXTVAL |
| `USER_ID` | NUMBER FK | 누구의 매너온도인지 | |
| `REASON_ID` | NUMBER FK | 이유 코드 | **1=노쇼** (REASON 테이블) |
| `SCORE` | NUMBER | 점수 가감 | 노쇼 = -1 |

매너온도 = `36.5 + SUM(SCORE)` (다른 분 함수 `FN_GET_MANNER`)

---

# Part C — 학생이 짠 쿼리 총정리 (도메인별)

## C-1. UserMapper (회원가입·로그인·아이디찾기·비번찾기) — 7개

### ① 회원가입 — USER_ACCOUNT INSERT (selectKey 패턴)
```xml
<insert id="insertAccount" parameterType="com.noexit.app.model.User">
    <selectKey keyProperty="userId" resultType="Long" order="BEFORE">
        SELECT USER_ACCOUNT_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO USER_ACCOUNT (USER_ID, LOGIN_ID, PASSWORD, NICKNAME)
    VALUES (#{userId}, #{loginId}
           , CRYPTPACK.ENCRYPT(#{password}, '12341234')
           , #{nickname})
</insert>
```
**한국어로 읽기:** "회원 번호를 미리 시퀀스에서 뽑고, USER_ACCOUNT 테이블에 (회원번호, 로그인아이디, 암호화된비번, 닉네임) 박는다."

**핵심:** `order="BEFORE"` = INSERT 전에 시퀀스 값을 keyProperty(`userId`)에 박음 → 두 번째 INSERT(USER_INFO)에서 같은 userId 재사용 가능.

### ② 회원가입 — USER_INFO INSERT (같은 USER_ID 재사용)
```xml
<insert id="insertInfo" parameterType="com.noexit.app.model.User">
    INSERT INTO USER_INFO (USER_ID, EMAIL, NAME, PHONE, GENDER, BIRTHDATE)
    VALUES (#{userId}, #{email}, #{name}, #{phone}, #{gender}
           , TO_DATE(#{birthDate}, 'YYYY-MM-DD'))
</insert>
```
**한국어로 읽기:** "위에서 받은 회원번호로 USER_INFO에 (회원번호, 이메일, 이름, 폰, 성별, 생일) 박기. 생일은 문자열 → 날짜 변환."

**함정:**
- `TO_DATE` 빠뜨리면 `ORA-01861` (생일이 DATE 타입이라).
- `selectKey` 두 번 쓰면 안 됨! 두 번째는 첫 번째 시퀀스 값 그대로 사용.

### ③ 아이디 중복확인 (Ajax)
```xml
<select id="countByLoginId" parameterType="string" resultType="int">
    SELECT COUNT(*) FROM USER_ACCOUNT WHERE LOGIN_ID = #{loginId}
</select>
```
**한국어로 읽기:** "이 로그인아이디 쓰는 회원 수 세기." 0이면 사용 가능, 1 이상이면 중복.

### ④ 로그인 (PW 비교는 SQL에서)
```xml
<select id="selectByLoginId" parameterType="com.noexit.app.model.User"
        resultType="com.noexit.app.model.User">
    SELECT USER_ID, LOGIN_ID, NICKNAME, CREATED_AT
      FROM USER_ACCOUNT
     WHERE LOGIN_ID = #{loginId}
       AND PASSWORD = CRYPTPACK.ENCRYPT(#{password}, '12341234')
</select>
```
**한국어로 읽기:** "로그인아이디 + 비번(암호화된 거 비교) 둘 다 맞는 회원 1명 가져오기." null이면 로그인 실패.

**중요:** 비번을 자바에서 `.equals` 비교 X. SQL에서 `CRYPTPACK.ENCRYPT`로 비교 — 학원 패턴 (B 옵션).

### ⑤ 매니저 임명용 — 단순 USER 조회 (PW 비교 없음)
```xml
<select id="findByLoginId" parameterType="string" resultType="com.noexit.app.model.User">
    SELECT USER_ID, LOGIN_ID, NICKNAME, CREATED_AT
      FROM USER_ACCOUNT
     WHERE LOGIN_ID = #{loginId}
</select>
```
**왜 ④랑 따로?** ④는 PW도 비교(로그인용), ⑤는 LOGIN_ID로 USER만 찾기(매니저 임명용). 학원 룰 **"의도 다른 SQL은 메서드 분리"**.

### ⑥ 역할 파생 — 사장인지 확인
```xml
<select id="countCafeByUserId" parameterType="long" resultType="int">
    SELECT COUNT(*) FROM CAFE WHERE USER_ID = #{userId}
</select>
```
**한국어로 읽기:** "이 USER가 소유한 카페 수." > 0 이면 OWNER(사장), 아니면 USER(일반).

### ⑦ 아이디찾기 — JOIN으로 LOGIN_ID 가져오기 (Day 13 버그픽스)
```xml
<select id="findByNameAndEmail" parameterType="com.noexit.app.model.User"
        resultType="com.noexit.app.model.User">
    SELECT UA.USER_ID, UA.LOGIN_ID, UA.NICKNAME, UI.NAME, UI.EMAIL
      FROM USER_ACCOUNT UA
      JOIN USER_INFO UI ON UA.USER_ID = UI.USER_ID
     WHERE UI.NAME = #{name}
       AND UI.EMAIL = #{email}
</select>
```
**왜 JOIN?** LOGIN_ID는 USER_ACCOUNT, NAME/EMAIL은 USER_INFO. 두 테이블 합쳐야 함.

### ⑧ 비번찾기 — 아이디+이름 검증
```xml
<select id="findByLoginIdAndName" parameterType="com.noexit.app.model.User"
        resultType="com.noexit.app.model.User">
    SELECT UA.USER_ID, UA.LOGIN_ID, UA.NICKNAME, UI.NAME, UI.EMAIL
      FROM USER_ACCOUNT UA
      JOIN USER_INFO UI ON UA.USER_ID = UI.USER_ID
     WHERE UA.LOGIN_ID = #{loginId}
       AND UI.NAME = #{name}
</select>
```

### ⑨ 비번 변경
```xml
<update id="updatePassword" parameterType="com.noexit.app.model.User">
    UPDATE USER_ACCOUNT
       SET PASSWORD = CRYPTPACK.ENCRYPT(#{password}, '12341234')
     WHERE USER_ID = #{userId}
</update>
```

---

## C-2. CafeMapper — 2개

### ① 사장의 카페 목록 (테마 등록 폼의 카페 셀렉트박스용)
```xml
<select id="selectByUserId" parameterType="long" resultType="com.noexit.app.model.Cafe">
    SELECT CAFE_ID, USER_ID, BR_NO, CAFE_NAME, PHONE,
           POSTAL_CODE, ADDRESS, ADDRESS_DETAIL, CREATED_AT
      FROM CAFE
     WHERE USER_ID = #{userId}
     ORDER BY CAFE_ID
</select>
```
**한국어로 읽기:** "이 사장(USER_ID)이 소유한 카페들."

### ② 카페 등록
```xml
<insert id="insertCafe" parameterType="com.noexit.app.model.Cafe">
    <selectKey keyProperty="cafeId" resultType="long" order="BEFORE">
        SELECT CAFE_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO CAFE (CAFE_ID, USER_ID, BR_NO, CAFE_NAME, PHONE
                    , POSTAL_CODE, ADDRESS, ADDRESS_DETAIL)
    VALUES (#{cafeId}, #{userId}, #{brNo}, #{cafeName}, #{phone}
          , #{postalCode}, #{address}, #{addressDetail})
</insert>
```

---

## C-3. ManagerMapper — 4개

### ① 활성 매니저 목록 (사장 본인 카페들의)
```xml
<select id="selectActiveByOwnerUserId" parameterType="map"
        resultType="com.noexit.app.model.Manager">
    SELECT *
      FROM V_ACTIVE_MANAGER
     WHERE OWNER_USER_ID = #{ownerUserId}
     ORDER BY CREATED_AT DESC
     OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
</select>
```
**한국어로 읽기:** "**활성 매니저 뷰**에서 이 사장이 소유한 카페들의 매니저 목록 (페이징)."

**핵심:** `V_ACTIVE_MANAGER` = 본인이 만든 뷰. MAX(MANAGER_HISTORY_ID) + REG_EVENT_ID=1로 활성만 뽑음. **복잡한 로직을 뷰에 캡슐화**.

### ② 매니저 페이징 카운트
```xml
<select id="dataCount" parameterType="map" resultType="int">
    SELECT NVL(COUNT(*), 0)
      FROM V_ACTIVE_MANAGER
     WHERE OWNER_USER_ID = #{ownerUserId}
</select>
```

### ③ 매니저 임명
```xml
<insert id="insertEnroll" parameterType="com.noexit.app.model.Manager">
    INSERT INTO MANAGER_HISTORY (MANAGER_HISTORY_ID, CAFE_ID, USER_ID, REG_EVENT_ID)
    VALUES (MANAGER_HISTORY_SEQ.NEXTVAL, #{cafeId}, #{userId}, 1)
</insert>
```
**한국어로 읽기:** "이력 테이블에 (이력번호, 카페번호, USER번호, **등록=1**) 박기."

### ④ 매니저 해제
```xml
<insert id="insertDeact" parameterType="com.noexit.app.model.Manager">
    INSERT INTO MANAGER_HISTORY (MANAGER_HISTORY_ID, CAFE_ID, USER_ID, REG_EVENT_ID)
    VALUES (MANAGER_HISTORY_SEQ.NEXTVAL, #{cafeId}, #{userId}, 2)
</insert>
```
**핵심:** 해제도 DELETE/UPDATE 아닌 **새 행 INSERT (REG_EVENT_ID=2)**. 이력 누적식. 마지막 행이 1이면 활성, 2면 해제.

### ⑤ 본인이 활성 매니저인지 확인 (역할 파생용)
```xml
<select id="countActiveByUserId" parameterType="long" resultType="int">
    SELECT COUNT(*) FROM V_ACTIVE_MANAGER WHERE USER_ID = #{userId}
</select>
```
**한국어로 읽기:** "이 USER가 어떤 카페든 활성 매니저로 등록돼 있는지." > 0 이면 매니저 권한 있음.

---

## C-4. ThemeMapper (학생 영역만) — 5개

### ① 테마 등록 (selectKey 패턴)
```xml
<insert id="themeInsert" parameterType="com.noexit.app.model.ThemeDTO">
    <selectKey keyProperty="themeId" resultType="long" order="BEFORE">
        SELECT ROOM_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO ROOM (
        ROOM_ID, CAFE_ID, ROOM_NAME, GENRE_ID, DURATION,
        DIFFICULTY, HORROR_LEVEL, ACTIVITY_LEVEL,
        MIN_PLAYERS, MAX_PLAYERS, ROOM_DESC, ROOM_IMG, PRICE, IS_ADULT
    ) VALUES (
        #{themeId}, #{cafeId}, #{themeName}, #{genreId}, #{duration},
        #{difficulty}, #{horrorLevel}, #{activityLevel},
        #{minPlayers}, #{maxPlayers}, #{themeDesc}, #{imagePath}, #{price}, #{isAdult}
    )
</insert>
```

### ② 테마 단일 조회 (수정 폼 채우기용)
```xml
<select id="getThemeById" parameterType="long"
        resultType="com.noexit.app.model.ThemeDTO">
    SELECT * FROM ROOM WHERE ROOM_ID = #{themeId}
</select>
```

### ③ 테마 수정
```xml
<update id="themeUpdate" parameterType="com.noexit.app.model.ThemeDTO">
    UPDATE ROOM
       SET CAFE_ID = #{cafeId}, ROOM_NAME = #{themeName},
           GENRE_ID = #{genreId}, DURATION = #{duration},
           DIFFICULTY = #{difficulty}, HORROR_LEVEL = #{horrorLevel},
           ACTIVITY_LEVEL = #{activityLevel},
           MIN_PLAYERS = #{minPlayers}, MAX_PLAYERS = #{maxPlayers},
           ROOM_DESC = #{themeDesc}, ROOM_IMG = #{imagePath},
           PRICE = #{price}, IS_ADULT = #{isAdult}
     WHERE ROOM_ID = #{themeId}
</update>
```
**함정:** `ROOM_IMG NOT NULL` → 새 이미지 안 받으면 자바에서 기존 imagePath 박아 넣어야 함. (`<if>` 동적 SQL 안 씀)

### ④ 사장 본인 테마 목록 (페이징)
```xml
<select id="selectListByOwnerUserId" parameterType="map"
        resultType="com.noexit.app.model.ThemeDTO">
    SELECT R.ROOM_ID AS THEME_ID, R.ROOM_NAME AS THEME_NAME,
           R.CAFE_ID, C.CAFE_NAME, R.ROOM_IMG AS IMAGE_PATH,
           R.GENRE_ID, RG.GENRE_NAME, R.DURATION, R.PRICE
      FROM ROOM R
      JOIN CAFE C ON R.CAFE_ID = C.CAFE_ID
      JOIN ROOM_GENRE RG ON R.GENRE_ID = RG.GENRE_ID
     WHERE C.USER_ID = #{ownerUserId}
     ORDER BY R.ROOM_ID DESC
     OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
</select>
```

### ⑤ 테마 페이징 카운트
```xml
<select id="dataCount" parameterType="map" resultType="int">
    SELECT NVL(COUNT(*), 0)
      FROM ROOM R
      JOIN CAFE C ON R.CAFE_ID = C.CAFE_ID
     WHERE C.USER_ID = #{ownerUserId}
</select>
```

---

## C-5. AttendanceMapper — 가장 복잡 (10개)

### ① 사장의 처리 대기 출석 목록 (시작 1시간 전 ~ 지금)
```xml
<select id="selectListByOwnerUserId" parameterType="map"
        resultType="com.noexit.app.model.AttendanceListDTO">
    SELECT VRA.RESERVATION_ID, VRA.OPEN_AT, VRA.CAFE_ID, VRA.CAFE_NAME,
           VRA.ROOM_ID, VRA.ROOM_NAME, VRA.LEADER_ID,
           UA.NICKNAME AS LEADER_NICKNAME,
           VRA.TOTAL_MEMBER,
           FN_GET_ATTEND_STATUS(VRA.RESERVATION_ID, VRA.LEADER_ID) AS STATUS_NAME
      FROM VW_RESERVATION_ALL VRA
      JOIN CAFE C ON VRA.CAFE_ID = C.CAFE_ID
      JOIN USER_ACCOUNT UA ON VRA.LEADER_ID = UA.USER_ID
     WHERE C.USER_ID = #{ownerUserId}
       AND VRA.CANCEL_ID IS NULL
       <![CDATA[ AND VRA.OPEN_AT <= SYSDATE + 1/24 ]]>
     ORDER BY VRA.OPEN_AT DESC
     OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
</select>
```
**한국어로 읽기:**
- "**예약 통합뷰**(VRA)에서 본인 카페(`C.USER_ID = #{ownerUserId}`)의 예약을 뽑되,"
- "취소 안 됐고 (`CANCEL_ID IS NULL`),"
- "시작이 **지금부터 +1시간 이내** 또는 이미 지난 것만"  ← (시연 시 이거 때문에 빈 화면 자주 남)

**함수 호출:** `FN_GET_ATTEND_STATUS` = 다른 분(yuhwon) 함수. "출석 완료"/"노쇼"/"출석 미등록" 문자열 반환.

### ② 매니저의 처리 대기 출석 목록
```xml
<select id="selectListByManagerUserId" parameterType="map" resultType="...">
    ... 위와 거의 동일 ...
      JOIN V_ACTIVE_MANAGER VAM ON VAM.CAFE_ID = VRA.CAFE_ID  -- 차이: CAFE 대신 VAM
      ...
     WHERE VAM.USER_ID = #{managerUserId}
       ... 나머지 동일 ...
</select>
```
**핵심 차이:** `JOIN CAFE` → `JOIN V_ACTIVE_MANAGER`. **본인이 매니저로 등록된 카페만** 필터.

### ③ 페이징 카운트 (사장/매니저 각각)
- `dataCountByOwnerUserId` — `JOIN CAFE`
- `dataCountByManagerUserId` — `JOIN V_ACTIVE_MANAGER`

### ④ 한 예약의 파티원 전체 (호스트 + 멤버)
```xml
<select id="selectCrewByReservationId" parameterType="Long"
        resultType="com.noexit.app.model.AttendCrew">
    SELECT USER_ID, NICKNAME, POSITION
    FROM (
        SELECT P.USER_ID, UA.NICKNAME, 'HOST' AS POSITION
          FROM PARTY P
          JOIN USER_ACCOUNT UA ON UA.USER_ID = P.USER_ID
          JOIN RESERVATION R ON R.PARTY_ID = P.PARTY_ID
         WHERE R.RESERVATION_ID = #{reservationId}
        UNION ALL
        SELECT PA.USER_ID, UA.NICKNAME, 'MEMBER' AS POSITION
          FROM PARTY_MEMBER PM
          JOIN PARTY_APPLY PA ON PA.APPLY_ID = PM.APPLY_ID
          JOIN USER_ACCOUNT UA ON UA.USER_ID = PA.USER_ID
          JOIN RESERVATION R ON R.PARTY_ID = PA.PARTY_ID
         WHERE R.RESERVATION_ID = #{reservationId}
           AND NOT EXISTS (
               SELECT 1 FROM PARTY_KICK PK WHERE PK.MEMBER_ID = PM.MEMBER_ID
           )
    )
    ORDER BY DECODE(POSITION, 'HOST', 0, 1), USER_ID
</select>
```
**한국어로 읽기:**
- "이 예약의 파티장(HOST) + 파티원(MEMBER) 합치기 (UNION ALL)"
- "단 파티원 중 KICK 당한 사람은 제외 (NOT EXISTS)"
- "정렬: HOST 먼저(0), 나머지 USER_ID 순"

### ⑤ ATTENDANCE 헤더 INSERT (selectKey)
```xml
<insert id="insertAttendance" parameterType="com.noexit.app.model.AttendanceListDTO">
    <selectKey keyProperty="attendanceId" resultType="Long" order="BEFORE">
        SELECT ATTENDANCE_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO ATTENDANCE (ATTENDANCE_ID, RESERVATION_ID, USER_ID)
    VALUES (#{attendanceId}, #{reservationId}, #{userId})
</insert>
```

### ⑥ ATTENDANCE_DETAIL INSERT (파티원 1명당)
```xml
<insert id="insertAttendDetailByUser" parameterType="...AttendItemDTO">
    INSERT INTO ATTENDANCE_DETAIL (DETAIL_ID, ATTENDANCE_ID, USER_ID, ATTEND_STATUS_ID)
    VALUES (ATTENDANCE_DETAIL_SEQ.NEXTVAL, #{attendanceId}, #{userId}, #{attendStatusId})
</insert>
```

### ⑦ 자동처리된 ATTENDANCE_ID 조회 (Day 12.5 통합 패턴)
```xml
<select id="selectAttendanceIdByReservationId" parameterType="Long" resultType="Long">
    SELECT ATTENDANCE_ID FROM ATTENDANCE WHERE RESERVATION_ID = #{reservationId}
</select>
```
**용도:** 스케줄러가 미리 박아둔 ATTENDANCE 행 있는지 확인 → 있으면 ID 재사용해서 DETAIL만 채움.

### ⑧ 노쇼 처리 (프로시저 호출)
```xml
<update id="callInsertNoshow" parameterType="com.noexit.app.model.Manner"
        statementType="CALLABLE">
    { CALL SP_INSERT_NOSHOW(
        #{userId,  mode=IN,  jdbcType=NUMERIC},
        #{newTemp, mode=OUT, jdbcType=NUMERIC}
    ) }
</update>
```
**한국어로 읽기:** "노쇼 프로시저 호출. USER_ID 넣고, 갱신된 매너온도를 newTemp로 받기."

**핵심:** `statementType="CALLABLE"` + `{CALL ...}` 형식. IN/OUT 명시. Map 대신 `Manner` VO에 두 필드 박아 객체로 처리 (학원 룰).

### ⑨ 출석기록 — 사장 (Day 13 신규)
```xml
<select id="selectHistoryByOwnerUserId" parameterType="map" resultType="...">
    SELECT VRA.RESERVATION_ID, VRA.OPEN_AT, VRA.CAFE_ID, VRA.CAFE_NAME,
           VRA.ROOM_ID, VRA.ROOM_NAME, VRA.LEADER_ID,
           UA.NICKNAME AS LEADER_NICKNAME, VRA.TOTAL_MEMBER,
           (SELECT COUNT(*) FROM ATTENDANCE_DETAIL AD
              JOIN ATTENDANCE A2 ON A2.ATTENDANCE_ID = AD.ATTENDANCE_ID
             WHERE A2.RESERVATION_ID = VRA.RESERVATION_ID
               AND AD.ATTEND_STATUS_ID = 1) AS DONE_COUNT,
           (SELECT COUNT(*) FROM ATTENDANCE_DETAIL AD
              JOIN ATTENDANCE A2 ON A2.ATTENDANCE_ID = AD.ATTENDANCE_ID
             WHERE A2.RESERVATION_ID = VRA.RESERVATION_ID
               AND AD.ATTEND_STATUS_ID = 2) AS NOSHOW_COUNT
      FROM VW_RESERVATION_ALL VRA
      JOIN CAFE C ON VRA.CAFE_ID = C.CAFE_ID
      JOIN USER_ACCOUNT UA ON VRA.LEADER_ID = UA.USER_ID
      JOIN ATTENDANCE A ON A.RESERVATION_ID = VRA.RESERVATION_ID
     WHERE C.USER_ID = #{ownerUserId}
     ORDER BY VRA.OPEN_AT DESC
     OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
</select>
```
**한국어로 읽기:** "처리된 예약(ATTENDANCE INNER JOIN)만 + 완료 N명·노쇼 N명 서브쿼리로 카운트."

### ⑩ 출석기록 — 매니저
사장 SQL에서 `JOIN CAFE` → `JOIN V_ACTIVE_MANAGER`로 교체.

---

## C-6. PL/SQL — 본인 작성 자산 (noExit_kmc.sql)

### ① SP_INSERT_NOSHOW — 노쇼 시 매너온도 1점 차감
```sql
CREATE OR REPLACE PROCEDURE SP_INSERT_NOSHOW(
    P_USER_ID  IN  NUMBER
  , P_NEW_TEMP OUT NUMBER
) IS
BEGIN
    INSERT INTO MANNER_HISTORY (MANNER_HISTORY_ID, USER_ID, REASON_ID, SCORE)
    VALUES (MANNER_HISTORY_SEQ.NEXTVAL, P_USER_ID, 1, -1);

    P_NEW_TEMP := FN_GET_MANNER(P_USER_ID);
END;
```
**한국어로 읽기:**
1. MANNER_HISTORY에 (이력번호, USER_ID, **이유=1=노쇼**, **점수=-1**) INSERT
2. 갱신된 매너온도를 다른 분(JY) 함수 `FN_GET_MANNER`로 조회해서 OUT 반환

### ② V_ACTIVE_MANAGER — 활성 매니저 뷰
```sql
CREATE OR REPLACE VIEW V_ACTIVE_MANAGER AS
SELECT MH.MANAGER_HISTORY_ID, MH.CAFE_ID, MH.USER_ID,
       MH.CREATED_AT,
       C.USER_ID AS OWNER_USER_ID,   -- 사장 검색용
       C.CAFE_NAME,
       UA.LOGIN_ID, UA.NICKNAME,
       UI.PHONE
  FROM MANAGER_HISTORY MH
  JOIN CAFE C ON C.CAFE_ID = MH.CAFE_ID
  JOIN USER_ACCOUNT UA ON UA.USER_ID = MH.USER_ID
  LEFT JOIN USER_INFO UI ON UI.USER_ID = MH.USER_ID
 WHERE MH.MANAGER_HISTORY_ID IN (
       SELECT MAX(MANAGER_HISTORY_ID)
         FROM MANAGER_HISTORY
        GROUP BY CAFE_ID, USER_ID
   )
   AND MH.REG_EVENT_ID = 1;
```
**한국어로 읽기:**
1. `(카페별, USER별) 가장 최근 이력(MAX(MANAGER_HISTORY_ID))`을 그룹별로 뽑고
2. 그게 등록 이벤트(REG_EVENT_ID=1) 이면 활성 매니저로 인정
3. 카페 정보 + USER 정보 같이 붙임 (편의)
4. **OWNER_USER_ID 컬럼** = 사장이 본인 매니저들 조회할 때 쓰는 핵심 키

### ③ SP_AUTO_ATTEND_ALL — 30분 후 자동 출석 처리
```sql
CREATE OR REPLACE PROCEDURE SP_AUTO_ATTEND_ALL IS
    V_ATTENDANCE_ID NUMBER;
BEGIN
    FOR rec IN (
        SELECT VRA.RESERVATION_ID
          FROM VW_RESERVATION_ALL VRA
          JOIN ROOM R ON VRA.ROOM_ID = R.ROOM_ID
         WHERE VRA.CANCEL_ID IS NULL
           AND VRA.OPEN_AT + (R.DURATION + 30) / 1440 < SYSDATE
           AND NOT EXISTS (
               SELECT 1 FROM ATTENDANCE A
                WHERE A.RESERVATION_ID = VRA.RESERVATION_ID
           )
    ) LOOP
        -- ATTENDANCE INSERT (USER_ID NULL = 자동처리 표시)
        INSERT INTO ATTENDANCE (ATTENDANCE_ID, RESERVATION_ID, USER_ID)
        VALUES (ATTENDANCE_SEQ.NEXTVAL, rec.RESERVATION_ID, NULL)
        RETURNING ATTENDANCE_ID INTO V_ATTENDANCE_ID;

        -- 파티원 전체 출석=1로 INSERT
        INSERT INTO ATTENDANCE_DETAIL (DETAIL_ID, ATTENDANCE_ID, USER_ID, ATTEND_STATUS_ID)
        SELECT ATTENDANCE_DETAIL_SEQ.NEXTVAL, V_ATTENDANCE_ID, T.USER_ID, 1
          FROM ( ... 파티장 + 활성 파티원 ... ) T;
    END LOOP;
    COMMIT;
END;
```
**한국어로 읽기:**
1. **종료된 지 30분 이상 지났는데(`OPEN_AT + (DURATION+30)/1440 < SYSDATE`)** 출석 처리 안 된 예약 찾기
2. ATTENDANCE INSERT (`USER_ID=NULL` = "사람이 안 했음" 표시)
3. 모든 파티원을 출석(`ATTEND_STATUS_ID=1`)로 ATTENDANCE_DETAIL INSERT

**1/1440 의미:** Oracle DATE 산술에서 1 = 1일. `1/1440` = 1분. `(DURATION + 30) / 1440` = "소요시간+30분"을 일 단위로 환산.

### ④ DBMS_SCHEDULER 잡 등록 — 1분마다 자동 실행
```sql
DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'JOB_AUTO_ATTEND',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN SP_AUTO_ATTEND_ALL; END;',
    start_date      => SYSDATE,
    repeat_interval => 'FREQ=MINUTELY; INTERVAL=1',
    enabled         => TRUE
);
```
**한국어로 읽기:** "JOB_AUTO_ATTEND 라는 잡을 등록. 1분마다 SP_AUTO_ATTEND_ALL 프로시저 실행."

---

# Part D — 빈출 패턴 모음

## 패턴 1: selectKey 시퀀스 패턴 (회원가입·카페·테마)
```xml
<insert id="...">
    <selectKey keyProperty="xxxId" resultType="long" order="BEFORE">
        SELECT XXX_SEQ.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO XXX (XXX_ID, ...) VALUES (#{xxxId}, ...)
</insert>
```
**언제 쓰나:** 새 행 INSERT + 다른 INSERT/UPDATE에서 같은 ID 재사용 필요할 때.

**시퀀스 명명 컨벤션 (팀 표준):** `{TABLE_NAME}_SEQ` (예: USER_ACCOUNT_SEQ, CAFE_SEQ, ROOM_SEQ, MANAGER_HISTORY_SEQ, ATTENDANCE_SEQ, ATTENDANCE_DETAIL_SEQ, MANNER_HISTORY_SEQ)

## 패턴 2: 페이징 (Oracle 12c+)
```xml
SELECT ... 
  FROM ... 
 WHERE ...
 ORDER BY ... 
 OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
```
**Map 파라미터:** `offset` (= (currentPage-1)*size), `size` (= 페이지당 행 수)

**페이징 카운트는 별도 SELECT:** `SELECT NVL(COUNT(*), 0) FROM ... WHERE ...`

## 패턴 3: role 분기 (사장/매니저)
```xml
<!-- 사장 -->
JOIN CAFE C ON VRA.CAFE_ID = C.CAFE_ID
WHERE C.USER_ID = #{ownerUserId}

<!-- 매니저 -->
JOIN V_ACTIVE_MANAGER VAM ON VAM.CAFE_ID = VRA.CAFE_ID
WHERE VAM.USER_ID = #{managerUserId}
```
**Service에서 분기, Mapper는 두 메서드.**

## 패턴 4: 부등호는 CDATA로 감싸기
```xml
<![CDATA[ AND VRA.OPEN_AT <= SYSDATE + 1/24 ]]>
```
**왜?** xml은 `<`를 태그 시작으로 해석. CDATA 안에선 그냥 문자.

## 패턴 5: 이력 누적식 INSERT (매니저 임명/해제)
```xml
INSERT INTO MANAGER_HISTORY (..., REG_EVENT_ID) VALUES (..., 1)  -- 등록
INSERT INTO MANAGER_HISTORY (..., REG_EVENT_ID) VALUES (..., 2)  -- 해제
```
**핵심:** UPDATE 안 하고 INSERT. 활성 = MAX(이력ID) 행이 1.

## 패턴 6: 코드 테이블 FK
```sql
SELECT * FROM ROOM_GENRE  -- 셀렉트박스 옵션 채우기
SELECT * FROM LEVEL       -- 1~5 점수
SELECT * FROM COMMON      -- Y/N
```
**폼에서 직접 1, 2 보내지 말 것!** 셀렉트박스로 옵션을 코드 테이블에서 동적 조회.

## 패턴 7: CRYPTPACK 암호화 (학원 패키지)
```sql
-- 저장 시
CRYPTPACK.ENCRYPT(#{password}, '12341234')

-- 로그인 비교 시 (학원 패턴 B)
WHERE PASSWORD = CRYPTPACK.ENCRYPT(#{password}, '12341234')

-- 조회 시 (학원 패턴 A)
SELECT CRYPTPACK.DECRYPT(PASSWORD, '12341234')
```

## 패턴 8: 프로시저 호출 (CALLABLE + Manner VO)
```xml
<update id="callXxx" parameterType="...XxxVO" statementType="CALLABLE">
    { CALL SP_XXX(
        #{inParam,  mode=IN,  jdbcType=NUMERIC},
        #{outParam, mode=OUT, jdbcType=NUMERIC}
    ) }
</update>
```

---

# Part E — "한국어로 쿼리 떠올리기" 훈련 5단계

쿼리가 막힐 때 머릿속 순서:

### 1️⃣ "무엇을 가져오고 싶지?"
한국어 한 문장으로. 예: "사장 본인 카페에서 처리 완료된 예약 목록"

### 2️⃣ "그 정보는 어느 테이블에 있지?"
- 처리 완료 = ATTENDANCE 테이블
- 예약 정보 = RESERVATION (또는 통합 뷰 VW_RESERVATION_ALL)
- 카페 = CAFE
- 사장 = CAFE.USER_ID

### 3️⃣ "테이블끼리 어떻게 연결되지?"
- VW_RESERVATION_ALL ↔ CAFE: `VRA.CAFE_ID = C.CAFE_ID`
- VW_RESERVATION_ALL ↔ ATTENDANCE: `VRA.RESERVATION_ID = A.RESERVATION_ID`

### 4️⃣ "조건은 뭐지?"
- 사장 본인 = `C.USER_ID = #{ownerUserId}`
- 처리 완료 = `JOIN ATTENDANCE A` (INNER JOIN이면 자동 필터)

### 5️⃣ "정렬·페이징은?"
- 최신순 = `ORDER BY VRA.OPEN_AT DESC`
- 페이징 = `OFFSET ... ROWS FETCH FIRST ... ROWS ONLY`

→ 5단계 거치면 위 출석기록 SQL 그대로 나옴.

---

# Part F — 빠른 참고 카드

## 학생 본인 영역 시퀀스 8개
| 시퀀스 | 어디서 쓰나 |
|---|---|
| USER_ACCOUNT_SEQ | 회원가입 |
| CAFE_SEQ | 카페 등록 |
| ROOM_SEQ | 테마 등록 |
| MANAGER_HISTORY_SEQ | 매니저 임명/해제 |
| ATTENDANCE_SEQ | 출석 헤더 INSERT |
| ATTENDANCE_DETAIL_SEQ | 출석 상세 INSERT |
| MANNER_HISTORY_SEQ | 노쇼 매너온도 차감 |

## 본인이 만든 PL/SQL 자산 3개
| 이름 | 용도 |
|---|---|
| SP_INSERT_NOSHOW | 노쇼 -1점 + 매너온도 반환 |
| V_ACTIVE_MANAGER | 활성 매니저 뷰 |
| SP_AUTO_ATTEND_ALL + JOB_AUTO_ATTEND | 30분 후 자동 출석 처리 + 스케줄러 |

## 본인이 호출하는 다른 분 자산 3개
| 이름 | 누구 | 용도 |
|---|---|---|
| VW_RESERVATION_ALL | yuhwon | 예약-카페-테마-파티 통합 |
| FN_GET_ATTEND_STATUS | yuhwon | 출석 상태 문자열 |
| FN_GET_MANNER | JY | 매너온도 계산 |

## 본인이 자주 쓰는 코드값
| 코드 | 의미 |
|---|---|
| `REG_EVENT_ID = 1` | 매니저 등록 |
| `REG_EVENT_ID = 2` | 매니저 해제 |
| `ATTEND_STATUS_ID = 1` | 출석 완료 |
| `ATTEND_STATUS_ID = 2` | 노쇼 |
| `REASON_ID = 1` | 노쇼 사유 |
| `ATTENDANCE.USER_ID IS NULL` | 자동처리 표시 |
| `CRYPTPACK.ENCRYPT(..., '12341234')` | 비밀번호 암호화 (학원 키) |

---

# 마지막 — 헷갈리면 이 파일 한 번 훑기

쿼리 짤 때 막히면:
1. **Part B** 테이블 사전에서 컬럼 확인
2. **Part C** 학생 본인이 짠 비슷한 쿼리 찾기 (회원가입 패턴, 페이징 패턴 등)
3. **Part D** 패턴 모음 — 시퀀스/페이징/role 분기 등 복붙
4. **Part E** 5단계 — 한국어로 먼저 떠올린 후 영문으로
