# Day 13 — 출석기록 페이지 신규 + 리팩터링 4건 (2026-06-12)

D-Day 시연 후 보강 작업 이어서. **출석기록 페이지(`/owner/attendance/history`) 신규** — Day 10~12 출석체크는 "처리 대기 + 처리 단계" 화면, 이건 "처리된 결과 보기" 화면. `ATTENDANCE` 테이블에 INSERT된 예약만 INNER JOIN으로 필터 + 완료/노쇼 카운트는 서브쿼리 2개. + 코드 정리 4건(메서드명 통일, 데드 메서드 5개 제거, 들여쓰기, 아이디찾기 LOGIN_ID null 버그 수정).

---

## 1. 출석기록 페이지 — 혼자 다시 짤 때 흐름

### 1-A. 설계 결정 — "기록"이 뭔지 명확히 하기

우리 시스템 = 사장이 까먹으면 **방탈출 종료 30분 후 자동 출석 처리** 스케줄러 동작.

→ "기록" 정의 후보 3개:

| 후보 | 의미 | 채택? |
|---|---|---|
| 시간 지난 모든 예약 | OPEN_AT < SYSDATE | ❌ 자동 처리 시스템 있으니까 다 처리됨. 의미 없음 |
| 처리 완료된 예약만 | ATTENDANCE 테이블 INSERT된 것 | ✅ 수동/자동 구분 없이 "기록" 의미 깔끔 |
| 완료/노쇼만 (대기 제외) | ATTEND_STATUS_ID IN (1,2) | △ 위와 동일하지만 조건이 디테일 |

**채택 = ②**. JOIN으로 ATTENDANCE 행 있는 것만 골라내면 됨. 자동 처리든 수동이든 일단 ATTENDANCE 행이 박혀 있으면 "처리된 것".

### 1-B. 디자인 결정 — 별도 페이지 vs 한 페이지 두 섹션

| 관점 | 별도 페이지 (채택) | 한 페이지 두 섹션 |
|---|---|---|
| 사이드바 | "출석체크" + "출석기록" 두 메뉴 | "출석체크" 하나 |
| 페이징 | 각자 `?page=1` 독립 | 한 곳만 가능 |
| 학원 패턴 일관성 | `themeManage` 패턴 그대로 | 새 패턴 |
| 답하기 쉬움 | "list 페이지 분리 컨벤션" | 설명 더 길어짐 |

→ 별도 페이지. **`themeManage` 패턴 100% 복붙**해서 변형만.

### 1-C. 학원 7단계 — 한 번 더 반복

회원가입(Day 2) → 카페등록(Day 5) → 테마등록(Day 5) → 매니저임명(Day 9) → 출석체크(Day 10) → 2단계 분리(Day 11) → 자동처리·아이디찾기(Day 12) → **출석기록(Day 13)**.

```
DTO → Mapper IF → mapper.xml → Service IF → ServiceImpl → Controller → JSP
```

### 1-D. 단계별 코드 + 생각 방법

#### Step 1. DTO 필드 추가 (`AttendanceListDTO.java`)

```java
// 기존 필드들 그대로 두고 아래만 추가
private int doneCount;
private int noshowCount;
```

**왜 새 필드?** 출석체크 화면은 `FN_GET_ATTEND_STATUS` 함수가 "출석 완료/노쇼/미등록" 문자열 반환 → 한 예약에 한 상태. 기록 화면은 **한 예약에 5명 파티원이면 "완료 4명, 노쇼 1명"**처럼 카운트 표시가 더 정보량 많음. 그래서 카운트 두 개를 새 필드로.

**왜 statusName 안 쓰고?** statusName은 리더 1명 기준. 파티원 전체 통계가 안 보임.

#### Step 2. Mapper IF (`AttendanceMapper.java`)

```java
public List<AttendanceListDTO> selectHistoryByOwnerUserId(Map<String, Object> map);
public List<AttendanceListDTO> selectHistoryByManagerUserId(Map<String, Object> map);
public int dataCountHistoryByOwnerUserId(Map<String, Object> map);
public int dataCountHistoryByManagerUserId(Map<String, Object> map);
```

**왜 4개?** Day 10 출석체크 패턴 그대로 — 역할별 SELECT 2개 + 페이징용 COUNT 2개. 학원 룰 **"의도 다른 SQL은 메서드 분리"**. 매니저용은 V_ACTIVE_MANAGER JOIN이 추가되니 다른 메서드.

#### Step 3. mapper.xml — 핵심 SQL

```xml
<!-- 사장: 본인 카페에서 처리된 예약만 -->
<select id="selectHistoryByOwnerUserId" parameterType="map"
        resultType="com.noexit.app.model.AttendanceListDTO">
    SELECT VRA.RESERVATION_ID, VRA.OPEN_AT, VRA.CAFE_ID, VRA.CAFE_NAME,
           VRA.ROOM_ID, VRA.ROOM_NAME, VRA.LEADER_ID,
           UA.NICKNAME AS LEADER_NICKNAME,
           VRA.TOTAL_MEMBER,
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
      JOIN ATTENDANCE A ON A.RESERVATION_ID = VRA.RESERVATION_ID  -- 처리된 것만
     WHERE C.USER_ID = #{ownerUserId}
     ORDER BY VRA.OPEN_AT DESC
     OFFSET #{offset} ROWS FETCH FIRST #{size} ROWS ONLY
</select>
```

**SQL 추론 순서:**

1. **베이스** = Day 10 `selectListByOwnerUserId` 그대로 복붙
2. **필터 변경** = `FN_GET_ATTEND_STATUS(...)` 제거 + `OPEN_AT <= SYSDATE+1/24` 조건 제거 + **`JOIN ATTENDANCE A`** 추가 (INNER JOIN이라 처리된 예약만 자동 필터)
3. **카운트 컬럼 추가** = 서브쿼리 2개 (완료/노쇼). 한 행 = 한 예약 단위라 SCALAR 서브쿼리로 카운트 박는 게 직관적

**왜 서브쿼리? GROUP BY 안 쓰고?**
- GROUP BY 쓰면 SELECT에 그룹 키 다 명시해야 하고 한 예약당 한 행 보장이 SQL이 복잡해짐 (PIVOT까지)
- 서브쿼리 2개면 사람이 읽었을 때 "완료 카운트, 노쇼 카운트" 즉각 이해
- 학원 룰: **명료한 SQL > 똑똑한 SQL**

**왜 INNER JOIN ATTENDANCE A?**
- 처리 안 된 예약은 ATTENDANCE 행이 없음 → INNER JOIN 하면 자동으로 빠짐
- WHERE EXISTS 써도 되지만 학원에서 안 배움. 학원 패턴 일관

**ATTEND_STATUS_ID 값:** `1 = 출석 완료, 2 = 노쇼` (Day 9 본인이 만든 SP/INSERT 패턴)

#### Step 4. Service IF (`AttendanceService.java`)

```java
public List<AttendanceListDTO> selectHistoryByRole(Map<String, Object> map, String role);
public int dataCountHistoryByRole(Map<String, Object> map, String role);
```

**왜 byRole 하나로?** Day 10 패턴(`selectAttendListByRole`, `dataCountByRole`) 그대로. 컨트롤러가 role 분기 안 하게 만들고 Service 안에서 분기. 컨트롤러 코드 깔끔.

#### Step 5. ServiceImpl — 학원 try/catch 패턴

```java
@Override
public List<AttendanceListDTO> selectHistoryByRole(Map<String, Object> map, String role) {
    List<AttendanceListDTO> list = null;
    try {
        if ("OWNER".equals(role)) {
            list = mapper.selectHistoryByOwnerUserId(map);
        } else {
            list = mapper.selectHistoryByManagerUserId(map);
        }
    } catch (Exception e) {
        log.info("selectHistoryByRole : ", e);
    }
    return list;
}
```

**SELECT 패턴** = `result=null; try { ... } catch { log.info } return result;` — throw 없음.
**INSERT/UPDATE/DELETE 패턴** = 같지만 catch 안에 `throw e;`. 트랜잭션 롤백용.

#### Step 6. Controller — `attendance()` 페이징 패턴 복붙

```java
@GetMapping("/history")
public String history(@RequestParam(name = "page", defaultValue = "1") int currentPage,
                      HttpSession session, HttpServletRequest req, Model model) {
    String redirect = AuthUtil.checkStaff(session);  // 사장+매니저 통과
    if (redirect != null) return redirect;

    User loginUser = (User) session.getAttribute("loginUser");
    String role = (String) session.getAttribute("role");

    try {
        int size = 10;
        Map<String, Object> map = new HashMap<>();
        Long userId = loginUser.getUserId();
        map.put("ownerUserId", userId);
        map.put("managerUserId", userId);

        int dataCount = attendanceService.dataCountHistoryByRole(map, role);
        int totalPage = (dataCount != 0) ? paginateUtil.pageCount(dataCount, size) : 0;
        currentPage = Math.min(currentPage, totalPage);

        int offset = (currentPage - 1) * size;
        if (offset < 0) offset = 0;
        map.put("offset", offset);
        map.put("size", size);

        List<AttendanceListDTO> historyList = attendanceService.selectHistoryByRole(map, role);

        String paging = paginateUtil.paging(currentPage, totalPage,
                            req.getContextPath() + "/owner/attendance/history");

        model.addAttribute("historyList", historyList);
        model.addAttribute("paging", paging);
        model.addAttribute("dataCount", dataCount);
        // currentPage, size 등 동일
    } catch (Exception e) {
        log.info("history : ", e);
    }
    return "owner/attendanceHistory";
}
```

**왜 `checkStaff`?** 사장+매니저 둘 다 통과. Day 10 출석체크 메서드와 동일 권한 (`checkOwner` 아님).

**왜 `ownerUserId`/`managerUserId` 둘 다 박음?** Service가 role 보고 둘 중 하나만 골라 SQL에 바인딩. 매핑 누락 방지.

#### Step 7. JSP — attendance.jsp 헤더 + manager.jsp 12분할

```jsp
<div class="row fw-bold border-bottom py-2 m-0">
    <div class="col-1">시간</div>
    <div class="col-2">카페</div>
    <div class="col-2">테마</div>
    <div class="col-2">예약자</div>
    <div class="col-1">인원</div>
    <div class="col-2">출석</div>
    <div class="col-2">노쇼</div>
</div>
<c:forEach var="r" items="${historyList}">
    <div class="row align-items-center border-bottom py-2 m-0 text-muted">
        <div class="col-1 fw-bold">
            <fmt:formatDate value="${r.openAt}" pattern="MM-dd"/><br>
            <fmt:formatDate value="${r.openAt}" pattern="HH:mm"/>
        </div>
        <div class="col-2">${r.cafeName}</div>
        <div class="col-2">${r.roomName}</div>
        <div class="col-2">${r.leaderNickname}</div>
        <div class="col-1">${r.totalMember}명</div>
        <div class="col-2"><span class="status-done">${r.doneCount}명</span></div>
        <div class="col-2"><span class="status-noshow">${r.noshowCount}명</span></div>
    </div>
</c:forEach>
<c:if test="${empty historyList}">
    <div class="text-center py-3">처리된 출석 기록이 없습니다.</div>
</c:if>
${dataCount == 0 ? "" : paging}
```

**컬럼 합계 = 12 (Bootstrap row 12분할):** `1+2+2+2+1+2+2 = 12`.

**날짜는 2줄(MM-dd / HH:mm):** col-1 좁아도 안 잘림. Day 10 attendance.jsp 패턴 그대로.

#### + 사이드바 메뉴 한 줄 (`ownerSide.jsp`)

```jsp
<li><a href="${pageContext.request.contextPath }/owner/attendance/history">
    <svg ... class="bi bi-clock-history" ...>...</svg>출석기록
</a></li>
```

`<c:if test="${role == 'OWNER'}">` **바깥**에 둠 → 사장+매니저 둘 다 보임. 매니저도 본인 카페 기록 봐야 하니까.

### 1-E. 함정 모음 (혼자 짤 때 막힐 만한 곳)

| 함정 | 증상 | 해결 |
|---|---|---|
| ATTEND_STATUS_ID 값 헷갈림 | 완료/노쇼 카운트가 둘 다 0 또는 뒤집힘 | Day 9 본인 SP 확인: `1=완료, 2=노쇼` |
| INNER JOIN 안 하고 LEFT JOIN | 처리 안 된 예약도 같이 나옴 (대기 화면이랑 중복) | INNER JOIN ATTENDANCE A |
| 서브쿼리 NULL 걱정 | 아무것도 처리 안 된 예약이면 COUNT=0 | COUNT(*)는 항상 숫자 반환. NVL 불필요 |
| 매니저 SQL에 `JOIN CAFE` 남김 | 사장 카페만 보여서 매니저는 빈 화면 | `JOIN V_ACTIVE_MANAGER VAM` 으로 교체 |
| `OPEN_AT <= SYSDATE+1/24` 조건 안 지움 | 미래 예약은 안 보임 (기록인데도) | history는 시간 필터 X. INNER JOIN ATTENDANCE만 |
| 사이드바 메뉴 `c:if role=='OWNER'` 안에 둠 | 매니저는 메뉴 안 보임 | c:if 밖에 둬야 함 |

---

## 2. 아이디찾기 LOGIN_ID null 버그 수정

### 증상
사용자가 "아이디 찾기" 폼에서 이름+이메일 입력 → 메일은 발송됨 → 메일 본문: **"회원님의 아이디는 null 입니다"** 😱

### 원인 추적 흐름

1. 컨트롤러: `mailService.sendUserIdMail(dto.getEmail(), dto.getLoginId());`
2. 서비스: `dto = userMapper.findByNameAndEmail(user);`
3. **매퍼 SQL (버그):**
   ```sql
   SELECT * FROM USER_INFO WHERE NAME = #{name} AND EMAIL = #{email}
   ```

### 핵심 통찰

| 테이블 | 가진 컬럼 |
|---|---|
| USER_ACCOUNT | USER_ID, **LOGIN_ID**, PASSWORD, NICKNAME |
| USER_INFO | USER_ID, EMAIL, NAME, PHONE, GENDER, BIRTHDATE |

**USER_INFO엔 LOGIN_ID가 없음** → `SELECT *`로 USER_INFO만 가져왔는데 User 모델로 매핑 → loginId는 그냥 null.

### 해결 (학원 JOIN 패턴 — 같은 파일에 이미 있음)

```sql
SELECT UA.USER_ID, UA.LOGIN_ID, UA.NICKNAME, UI.NAME, UI.EMAIL
  FROM USER_ACCOUNT UA
  JOIN USER_INFO UI ON UA.USER_ID = UI.USER_ID
 WHERE UI.NAME = #{name}
   AND UI.EMAIL = #{email}
```

**왜 SELECT * 지양?** 학원 룰: **컬럼 명시 = 의도 명확 + 안전 + 인덱스 활용 가능**. 어떤 컬럼이 어디서 오는지 한눈에 보임.

**바로 아래 `findByLoginIdAndName`은 이미 JOIN 패턴이었음** — 비대칭을 통일.

---

## 3. 리팩터링 3건 — 코드 청소

### 3-A. `UserService.selectByLoginId` → `findByLoginId` 통일

**문제:** 서비스 메서드명 `selectByLoginId(String)`인데 내부에서 `userMapper.findByLoginId(loginId)` 호출. **서비스와 매퍼의 같은 의미 메서드명이 달랐음.**

**더 큰 혼란 원인:** 매퍼에 `selectByLoginId(User)` (로그인용, PW 비교)랑 `findByLoginId(String)` (단순 조회) 두 개 따로 있음. 서비스가 `selectByLoginId`라는 이름으로 매퍼의 `findByLoginId`를 부르니까 의미 두 번 꼬임.

**해결:** 서비스를 `findByLoginId`로 rename. 매퍼는 그대로(`selectByLoginId(User)`는 로그인 PW 비교용으로 의도 다른 메서드라 분리 유지).

| 레이어 | 메서드명 | 용도 |
|---|---|---|
| `UserMapper.selectByLoginId(User)` | 로그인 (PW 비교) | 그대로 |
| `UserMapper.findByLoginId(String)` | 단순 조회 | 그대로 |
| `UserService.findByLoginId(String)` | wrapper | ✅ rename (was: selectByLoginId) |

**호출처 한 곳:** `OwnerManagerController.managerEnroll`에서 매니저 임명할 사람 찾을 때.

**학원 룰:** **메서드명 = 의도**. 같은 의도면 같은 이름.

### 3-B. AttendanceMapper 데드 메서드 5개 제거

```java
// 삭제한 것
public void mergeAttendance(Long reservationId, Long staffId);
public void mergeAttendDetail(AttendItemDTO item);
public void updateAttendanceStaff(AttendItemDTO item);
public int selectAttendDetailExists(AttendItemDTO item);
public void updateAttendDetail(AttendItemDTO item);
```

**왜 위험?** xml에 SQL이 없음 → 호출하는 순간 `BindingException`. 빌드는 통과해서 잠복해있다 누가 손대면 폭발.

**왜 생긴 것?** Day 10에 upsert 패턴으로 짜려다 selectKey + 분기로 우회. 인터페이스 시그니처만 남음.

**룰:** **죽은 코드 = 지뢰**. 보이면 즉시 제거.

### 3-C. `AttendanceServiceImpl.finalizeAttendance()` 들여쓰기 정리

로직 동일. for 루프 닫는 위치와 catch 블록 들여쓰기가 어긋나 있던 것 수동으로 정리. STS `Ctrl+Shift+F`로도 가능하지만 본인 스타일이 섞여있어서 수동.

---

## 4. 면접 톤 Q&A

### Q. 출석체크와 출석기록을 왜 페이지를 따로 뺐나요?
> 두 화면은 시간 필터와 데이터 출처가 완전히 다릅니다. 출석체크는 "시작 1시간 전~지금" 사이 처리 대기 중인 예약, 출석기록은 ATTENDANCE 테이블에 INSERT된 모든 처리 완료 예약. 같은 페이지에 두면 페이징 파라미터가 충돌하고 UX도 혼란스러워서 별도 페이지로 뺐고, 사이드바 메뉴를 추가해 학원에서 배운 list 페이지 분리 컨벤션을 따랐습니다.

### Q. 처리된 예약만 어떻게 골라냈나요?
> `VW_RESERVATION_ALL`에 `JOIN ATTENDANCE A ON A.RESERVATION_ID = VRA.RESERVATION_ID`를 INNER JOIN으로 걸어서, ATTENDANCE 테이블에 행이 있는 예약만 자동으로 남게 했습니다. WHERE EXISTS도 가능하지만 학원에서 배운 JOIN 패턴이 가독성 좋아 그쪽을 선택했습니다.

### Q. 완료/노쇼 카운트는 왜 서브쿼리 두 개로 뽑았나요?
> 한 예약에 5명 파티원이면 "완료 4명, 노쇼 1명"처럼 두 카운트가 따로 표시되어야 해서, SELECT에 SCALAR 서브쿼리 2개를 박았습니다. GROUP BY로도 가능하지만 SELECT 컬럼에 그룹 키를 다 적어야 하고 PIVOT까지 가야 해서 복잡해집니다. 서브쿼리는 사람이 읽을 때 즉시 의미가 보입니다.

### Q. 사장과 매니저는 같은 화면인데 데이터가 다릅니다. 어떻게 분기했나요?
> Day 10 출석체크와 동일한 패턴입니다. 세션 `role` 값으로 Service 안에서 메서드를 분리하고, 사장은 `JOIN CAFE`로 본인이 소유한 카페, 매니저는 본인이 Day 9에 만든 `V_ACTIVE_MANAGER` 뷰 JOIN으로 임명받은 카페만 필터합니다. SQL을 두 개로 분리한 이유는 "의도가 다른 SQL은 메서드도 분리"라는 학원 룰을 따른 것입니다.

### Q. 자동 출석 처리 기능과 화면이 어떻게 통합되나요?
> 사장이 처리를 까먹으면 방탈출 종료 30분 후 스케줄러가 자동으로 ATTENDANCE에 INSERT합니다. 기록 화면은 ATTENDANCE 행 존재 여부만 보고 INNER JOIN으로 필터하기 때문에, 수동/자동을 구분하지 않고 둘 다 "처리됨"으로 표시됩니다. 사장 입장에서 "내가 처리한 건지, 자동으로 된 건지"는 운영상 동일한 결과라 굳이 구분할 필요가 없다고 판단했습니다.

### Q. 아이디찾기 메일에 LOGIN_ID가 null로 갔다고 하셨는데?
> `findByNameAndEmail` SQL이 `SELECT * FROM USER_INFO`였는데, USER_INFO 테이블에는 LOGIN_ID 컬럼이 없습니다. User 모델로 매핑하면 loginId는 그대로 null이 되고, 그 null이 메일 본문에 박혀 발송됐습니다. USER_ACCOUNT와 JOIN해서 UA.LOGIN_ID를 명시적으로 SELECT하도록 수정했고, 동시에 `SELECT *`를 컬럼 명시 패턴으로 바꿔서 같은 종류 버그가 재발하지 않게 했습니다.

### Q. 데드 코드 정리는 왜 별도 커밋으로 분리하셨나요?
> 의도가 다른 변경을 같은 커밋에 섞으면 PR 리뷰 시 어디가 기능이고 어디가 청소인지 구분이 안 됩니다. 출석기록은 신규 기능, 아이디찾기는 버그픽스, rename은 일관성, 데드 메서드 제거는 청소라서 4개 커밋으로 분리했습니다. 의미 단위 커밋이 코드 히스토리에서 변경 의도를 추적하기 쉽습니다.

---

## 5. 시연 흐름 (Day 13 추가분)

Day 10~12 시연 후 이어서:

11. ★ 사이드바 [출석기록] → 방금 처리한 예약 + 자동 처리된 예약 다 보임
12. ★ 각 행: 카페/테마/예약자/완료N명/노쇼N명 카운트 확인
13. ★ 매니저 로그인 (user0018) → [출석기록] → 본인 임명 카페만 (V_ACTIVE_MANAGER 활용)
14. ★ 아이디찾기 폼에서 본인 이름+이메일 입력 → 메일에 정상 LOGIN_ID 박혀 발송

---

## 6. 다음 (남은 todo)

- 🔴 **테마 이미지 경로 정리** — Day 10 저장 위치(`webapp/uploads/theme`) → 오늘 코드는 `src/main/resources/static/dist/images`로 바뀌었음. 옛 파일 이전 or 경로 통일 필요
- 🟡 **페이징 가드** — `dataCount == 0`일 때 `currentPage`가 0이 되는 케이스 (출석체크/테마관리/매니저관리/출석기록 모두 동일)
- 🟡 **카페 등록/수정 풀스택** — 테마 mode 패턴 그대로 적용
- 🟢 **이메일 인증/발송** — 비번찾기 흐름 보강 (학원 WebApp30/36/37 참고)

---

## 7. 핵심 패턴 카드 (앞으로 반복)

### 카드 1: 학원 7단계 (이번이 8번째 반복)
회원가입(Day 2) → 카페등록(Day 5) → 테마등록(Day 5) → 매니저임명(Day 9) → 출석체크(Day 10) → 2단계 분리(Day 11) → 자동처리·아이디찾기(Day 12) → **출석기록(Day 13)**

```
DTO → Mapper IF → mapper.xml → Service IF → ServiceImpl → Controller → JSP
```

### 카드 2: SELECT 패턴 (학원 NoticeServiceImpl)
```java
T result = null;
try { result = mapper.x(); }
catch (Exception e) { log.info("x : ", e); }
return result;
```
**throw 없음.** UI에 빈 리스트만 가게.

### 카드 3: INSERT/UPDATE 패턴
위와 동일하지만 catch 안에 `throw e;` (트랜잭션 롤백).

### 카드 4: 페이징 5줄
```java
int dataCount = service.dataCount(map);
int totalPage = (dataCount != 0) ? paginateUtil.pageCount(dataCount, size) : 0;
currentPage = Math.min(currentPage, totalPage);
int offset = Math.max(0, (currentPage - 1) * size);
map.put("offset", offset); map.put("size", size);
```

### 카드 5: 역할 분기
```java
if ("OWNER".equals(role)) { /* 사장 메서드 */ }
else { /* 매니저 메서드 */ }
```
**null 안전 비교** = 상수 먼저 (`role.equals("OWNER")`는 role==null이면 NPE).
