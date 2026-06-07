# Day 10 (D-Day) — 출석체크 풀스택 + 테마 등록/수정 통합 (파일 업로드 + mode 패턴)

## 📌 오늘 한 작업 한 줄 요약
D-Day 작업. **출석체크 SELECT/INSERT 풀스택** (`FN_GET_ATTEND_STATUS` 함수 + `V_ACTIVE_MANAGER` 뷰 활용, ATTENDANCE + ATTENDANCE_DETAIL 2테이블 트랜잭션, 노쇼 시 `SP_INSERT_NOSHOW` 추가 호출) + **테마 등록/수정을 학원 sb02 mode 패턴으로 통합** (`themeWriteForm.jsp` 하나로 write/update 분기, 파일 업로드, 새 이미지 없으면 기존 imagePath 유지).

---

## 1. 출석체크 풀스택 — 다른 분 함수/뷰 활용 패턴

### 1-A. 협업 자산 그대로 활용 (의도 분리)

| 자산 | 누구 | 용도 |
|---|---|---|
| `VW_RESERVATION_ALL` 뷰 | 다른 분(JY) | 예약+카페+테마+파티장+인원 한 번에 |
| `FN_GET_ATTEND_STATUS(reservationId, userId)` 함수 | 다른 분(yuhwon) | "출석 완료" / "노쇼" / "출석 미등록" 반환 |
| `V_ACTIVE_MANAGER` 뷰 | 학생 본인 (Day 9) | MANAGER_HISTORY MAX(MANAGER_HISTORY_ID) + REG_EVENT_ID=1 |
| `SP_INSERT_NOSHOW` 프로시저 | 학생 본인 (Day 9) | MANNER_HISTORY INSERT(REASON_ID=1, SCORE=-1) + OUT 매너온도 |

**룰**: 다른 분 코드는 **호출만**, 절대 수정 X. 다른 분이 미래에 변경해도 호출자 코드에 영향 없음.

### 1-B. role 분기 = 메서드 분리 (학원 룰 "의도 다른 SQL은 메서드 분리")

```java
// Owner.attendance() 컨트롤러
if ("OWNER".equals(role)) {
    attendList = attendanceService.selectListByOwnerUserId(loginUser.getUserId());
} else {  // MANAGER
    attendList = attendanceService.selectListByManagerUserId(loginUser.getUserId());
}
```

```xml
<!-- 사장: 본인 카페 조회 -->
<select id="selectListByOwnerUserId" parameterType="Long" resultType="...AttendanceListDTO">
    SELECT VRA.RESERVATION_ID, VRA.OPEN_AT, VRA.CAFE_NAME, VRA.ROOM_NAME,
           VRA.LEADER_ID, UA.NICKNAME AS LEADER_NICKNAME, VRA.TOTAL_MEMBER,
           FN_GET_ATTEND_STATUS(VRA.RESERVATION_ID, VRA.LEADER_ID) AS STATUS_NAME
      FROM VW_RESERVATION_ALL VRA
      JOIN CAFE         C  ON VRA.CAFE_ID   = C.CAFE_ID
      JOIN USER_ACCOUNT UA ON VRA.LEADER_ID = UA.USER_ID
     WHERE C.USER_ID = #{ownerUserId}
       <![CDATA[ AND VRA.OPEN_AT < SYSDATE ]]>
     ORDER BY VRA.OPEN_AT DESC
</select>

<!-- 매니저: V_ACTIVE_MANAGER JOIN으로 임명받은 카페만 -->
<select id="selectListByManagerUserId" ...>
    ...
      JOIN V_ACTIVE_MANAGER VAM ON VAM.CAFE_ID = VRA.CAFE_ID
      ...
     WHERE VAM.USER_ID = #{managerUserId}
       <![CDATA[ AND VRA.OPEN_AT < SYSDATE ]]>
</select>
```

**핵심 결정**: IN 절 안 배웠다 → JOIN으로 매니저 필터. 학원 패턴 100% 일치.

### 1-C. 출석/노쇼 INSERT — @Transactional + 2테이블 + 노쇼 분기

```java
@Override
@Transactional
public AttendanceListDTO attend(AttendanceListDTO dto) throws Exception {
    try {
        mapper.insertAttendance(dto);       // 1) ATTENDANCE INSERT, selectKey로 attendanceId 박힘
        mapper.insertAttendDetail(dto);     // 2) ATTENDANCE_DETAIL INSERT, 같은 attendanceId 재사용

        if (dto.getAttendStatusId() == 2) { // 3) 노쇼면 매너온도 깎기
            Manner manner = new Manner();
            manner.setUserId(dto.getLeaderId());
            mapper.insertNoshow(manner);    // SP_INSERT_NOSHOW 호출 (Day 9 만든 프로시저)
        }
    } catch (Exception e) {
        log.info("attend : ", e);
        throw e;                            // 트랜잭션 롤백
    }
    return dto;
}
```

**selectKey 재사용 패턴 (Day 2 회원가입 패턴과 동일)**:
- `insertAttendance` → `<selectKey order="BEFORE">SELECT ATTENDANCE_SEQ.NEXTVAL</selectKey>` → dto.attendanceId 박힘
- `insertAttendDetail` → 같은 dto의 attendanceId 사용 → ATTENDANCE_DETAIL.ATTENDANCE_ID 박힘 → FK 연결

### 1-D. JSP `<c:choose>` 분기 + 학원 manager.jsp form 패턴

```jsp
<c:choose>
    <c:when test="${r.statusName == '출석 완료'}">
        <div class="col-2"><span class="status-done">출석</span></div>
        <div class="col-2">-</div>
    </c:when>
    <c:when test="${r.statusName == '노쇼'}">
        <div class="col-2"><span class="status-noshow">노쇼</span></div>
        <div class="col-2">-</div>
    </c:when>
    <c:otherwise>
        <div class="col-2"><span class="status-wait">대기</span></div>
        <div class="col-2">
            <form id="attendForm-${r.reservationId}" action=".../owner/attendance/attend" method="post">
                <input type="hidden" name="reservationId"  value="${r.reservationId}">
                <input type="hidden" name="leaderId"       value="${r.leaderId}">
                <input type="hidden" name="attendStatusId" value="1">
                <button type="button" onclick="attendOk('attendForm-${r.reservationId}')">출석</button>
            </form>
            <form id="noshowForm-${r.reservationId}" ...>
                <input type="hidden" name="attendStatusId" value="2">
                <button type="button" onclick="noshowOk('noshowForm-${r.reservationId}')">노쇼</button>
            </form>
        </div>
    </c:otherwise>
</c:choose>
```

**JS = manager.jsp의 `deactOk` 패턴 그대로**:
```javascript
function attendOk(formId){
    if(confirm('출석 처리 하시겠습니까?')) document.getElementById(formId).submit();
}
```

→ form 동적 생성(`document.createElement('form')`) 패턴 안 씀. JSP에 미리 form 박는 학원 룰.

### 1-E. SYSDATE 비교 시 CDATA

```xml
<![CDATA[
AND VRA.OPEN_AT < SYSDATE
]]>
```

`<`는 XML 예약 문자 → `&lt;` 또는 CDATA. 학원 myReservation/themeMapper의 `<![CDATA[ ... ]]>` 패턴과 일관성.

---

## 2. 테마 등록/수정 — 학원 sb02 mode 패턴 통합

### 2-A. URL/뷰 1세트로 통합 (write/update 분기)

```
사이드바 [테마 관리]
   ↓
GET /owner/theme/manage  → themeManage.jsp (본인 카페 테마 리스트)
   ↓
   ├─ [테마 등록] 버튼 → /owner/theme/write?mode=write
   └─ 각 행 [수정]     → /owner/theme/write?mode=update&roomId=X
   ↓
themeWriteForm.jsp (mode에 따라 빈 폼 / 채워진 폼)
   ↓
POST /owner/theme/write (mode 받아서 insert/update 분기)
   ↓ redirect:/owner/theme/manage
```

**`writeForm()` 진입 시점**:
```java
model.addAttribute("mode", mode);
if ("update".equals(mode) && roomId != null) {
    model.addAttribute("dto", themeService.getThemeById(roomId));
}
return "theme/themeWriteForm";
```

**`write()` 처리**:
```java
if ("update".equals(mode)) {
    themeService.themeUpdate(dto);
} else {
    themeService.themeInsert(dto);
}
return "redirect:/owner/theme/manage";
```

### 2-B. JSP mode 분기 — c:choose / c:if

| 영역 | 분기 |
|---|---|
| `<title>` | `<c:choose><c:when test="${mode == 'update'}">수정</c:when><c:otherwise>등록</c:otherwise></c:choose>` |
| input value | `value="${dto.themeName}"` — write 모드 dto null이면 빈 문자열 |
| select selected | `<c:if test="${dto.cafeId == cafe.cafeId}">selected</c:if>` |
| 이미지 필수 | `<c:if test="${mode != 'update'}">required</c:if>` — 수정 시 새 이미지 없어도 OK |
| 버튼 라벨 | "등록" / "수정" 분기 |
| **themeId hidden** | `<c:if test="${mode == 'update'}">` 로 감싸야 함 (등록 시 빈값 → long 변환 에러!) |

**버그 1건 잡음**: 등록 모드에서 `<input type="hidden" name="themeId" value="">` 가 ThemeDTO.themeId(long primitive)에 바인딩 실패 → 400 Bad Request. **`c:if mode==update`로 hidden을 감싸서 등록 시 아예 안 보내게 해결**.

### 2-C. 파일 업로드 — 학원 sb02 패턴 그대로

```java
@PostConstruct
public void init() {
    uploadPath = servletContext.getRealPath("/uploads/theme");
    File file = new File(uploadPath);
    if (!file.exists()) file.mkdirs();
}

private String saveThemeImage(ThemeDTO dto) throws Exception {
    List<MultipartFile> files = dto.getThemeImageFile();
    if (files == null || files.isEmpty()) return null;
    MultipartFile mf = files.get(0);
    if (mf == null || mf.isEmpty()) return null;

    String extension = ...lastIndexOf(".")...;
    UUID uuid = UUID.randomUUID();
    long uniqueNumber = Math.abs(uuid.getMostSignificantBits());
    String saveFilename = System.currentTimeMillis() + String.valueOf(uniqueNumber) + extension;

    mf.transferTo(new File(uploadPath + File.separator + saveFilename));
    return saveFilename;
}
```

- 경로: `webapp/uploads/theme/` (sb02의 `uploads/notice` 패턴 그대로 — `notice → theme`만 다름)
- 파일명: `시간밀리초 + UUID 숫자 + 확장자` → 중복 0%

### 2-D. 수정 시 이미지 처리 — 새 파일 없으면 기존 유지

```java
@Override
public int themeUpdate(ThemeDTO dto) throws Exception {
    String saveFilename = saveThemeImage(dto);
    if (saveFilename != null) {
        dto.setImagePath(saveFilename);       // 새 이미지 박힘
    } else {
        ThemeDTO old = themeMapper.getThemeById(dto.getThemeId());
        dto.setImagePath(old.getImagePath()); // 기존 imagePath 그대로 박기
    }
    result = themeMapper.themeUpdate(dto);
    ...
}
```

**이유**: `ROOM_IMG VARCHAR2(200) NOT NULL` 제약. 수정 시 새 파일 안 받으면 dto.imagePath = null → UPDATE 시 NOT NULL 위반.

→ **MyBatis `<if>` 동적 쿼리 안 쓰고 자바에서 처리**. 학원 패턴 (Map/dynamic SQL 우회).

---

## 3. 면접 톤 정리 (오늘 핵심 개념)

### Q. 출석 처리에서 트랜잭션을 왜 쓰셨나요?
> ATTENDANCE 행은 INSERT됐는데 ATTENDANCE_DETAIL이 실패하면, "처리됐다는 헤더만 있고 실제 출석/노쇼 상태가 없는 깨진 상태"가 됨. `@Transactional` + `throw e`로 두 INSERT를 묶어서 원자성 보장.

### Q. FN_GET_ATTEND_STATUS는 본인이 만든 건가요?
> 아니요, 다른 팀원(yuhwon)이 만든 PL/SQL 함수. ATTENDANCE 행 있는지 + ATTENDANCE_DETAIL.ATTEND_STATUS_ID로 "출석 완료"/"노쇼"/"출석 미등록" 반환. **다른 분 자산을 호출만 하고 코드는 안 건드림** — 협업 룰.

### Q. 사장과 매니저는 어떻게 다른 카페 데이터를 보여주나요?
> 세션의 `role` 값으로 컨트롤러에서 분기, 매퍼는 의도 다른 SQL 2개로 분리. 사장은 `CAFE.USER_ID`로, 매니저는 본인이 Day 9에 만든 `V_ACTIVE_MANAGER` 뷰 JOIN으로. 학원 룰 "의도 다른 SQL은 메서드 분리".

### Q. 학원에서 안 배운 IN 절을 피한 이유?
> 학원에서 배운 SQL은 단순 WHERE + JOIN 위주. 본인이 모르는 패턴 쓰면 면접 때 설명 못 함. 매니저 필터를 `OR ... IN (SELECT ...)` 대신 `JOIN V_ACTIVE_MANAGER`로 풀어서 학원 패턴 유지.

### Q. write.jsp 하나로 등록과 수정을 통합한 이유는?
> 학원 sb02 패턴. 같은 폼 컴포넌트를 두 파일로 중복 관리하면 한 쪽 수정 누락 위험. mode 파라미터로 분기하면 form 필드가 항상 동기화됨. JSP는 `<c:choose>`/`<c:if>`로 분기, 컨트롤러는 mode 받아서 service.insert vs service.update 호출.

### Q. 등록 모드에서 `themeId` 빈값 문제는 어떻게 해결?
> ThemeDTO.themeId가 long primitive라 빈 문자열 바인딩 실패(400). JSP에서 `<c:if test="${mode == 'update'}">`로 hidden 자체를 감싸서 **등록 시 아예 안 보내게**. 등록 시엔 매퍼 `<selectKey order="BEFORE">`가 ROOM_SEQ에서 themeId 박음.

### Q. 수정 시 이미지 안 바꾸면 어떻게 처리?
> ROOM_IMG가 NOT NULL이라 dto.imagePath null로 UPDATE 가면 위반. `themeUpdate` 안에서 새 파일 없으면 **DB에서 기존 행 다시 조회해서 imagePath 가져와 박기**. MyBatis `<if>` 동적 SQL 회피 + 자바 코드로 처리.

---

## 4. 새로 등록한 코칭 룰

### [[feedback-git-minimal-diff]]
**원본 코드 줄은 절대 건드리지 말고 새 줄만 추가**. git diff/PR에 AI 손댄 티 안 나게 + 학생 본인 스타일 보존. 예: 한 줄 `if-return`을 두 줄로 풀어 쓰는 식의 통째 교체 금지. 단, 학생 본인이 자기 일관성을 위해 명시적으로 요청한 변경은 OK.

---

## 5. 시연 흐름 (D-Day 발표용, 10분)

1. 회원가입 (유효성 9가지 빨간 글씨 — Day 9 commit 88f3183)
2. 사장 로그인 (`user0007 / 12341234`) — role 파생 로직 (countCafe > 0 → OWNER)
3. 사이드바 [테마 관리] → 본인 카페 테마 리스트
4. ★ [테마 등록] → 이미지 업로드 → uploads/theme 폴더 + ROOM.ROOM_IMG 확인
5. ★ [수정] → 채워진 폼 확인 → 이미지 빼고 수정 → 기존 imagePath 유지 확인
6. [매니저 관리] → 매니저 임명 (V_ACTIVE_MANAGER 뷰)
7. ★ [출석체크] → VW_RESERVATION_ALL + FN_GET_ATTEND_STATUS → 시간 끝난 예약
8. ★ [출석] → ATTENDANCE + ATTENDANCE_DETAIL INSERT → 행 상태 "출석"
9. ★ [노쇼] → 위 + SP_INSERT_NOSHOW → MANNER_HISTORY 행 + 매너온도 깎임
10. 로그아웃 → 매니저 로그인 (`user0018`) → 출석체크 → 임명받은 카페만 보임

★ = 오늘 작업

---

## 6. 다음 (선택)

- 카페 등록/수정 풀스택 (테마 패턴 그대로) — 시간 남으면
- 이메일 인증/발송 (학원 WebApp30/36/37 참고)
- 다른 폼 유효성 검증 (로그인/카페등록 등 — 회원가입은 이미 9가지 완성)
