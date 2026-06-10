# Day 12 — 출석체크 자동처리 + 아이디/비밀번호 찾기 (2026-06-10)

D-Day 시연 후 출석체크 흐름 완성 + 미구현이던 회원 도메인 보강.

---

## 1. 출석체크 마무리 — 4가지 흐름

### 요구사항
1. **1시간 전 필터** — 방탈출 시작 1시간 전부터 페이지에 노출
2. **개별 임시저장** — 파티 3명 중 1명만 와도 1명만 체크 가능 (세션에 누적)
3. **최종확인** — 모든 파티원 체크 끝나야 DB INSERT
4. **자동처리** — 종료 30분 후에도 사장이 안 누르면 자동 출석 처리

---

## 2. 핵심 설계 결정 — DB 스케줄러로 자동처리

### 옵션 비교

| 옵션 | 장점 | 단점 |
|---|---|---|
| Spring `@Scheduled` | 학원에서 본 적 없음 | 사용자 접속 여부 무관 |
| 페이지 진입 시 호출 | 단순 | "사장이 들어와야 처리됨" → 비즈니스 요구와 안 맞음 |
| **Oracle `DBMS_SCHEDULER`** ✅ | 사용자 무관, **책임 분리** (UI=Spring / 시간 기반=DB) | DBMS_SCHEDULER 처음 사용 |

### `SP_AUTO_ATTEND_ALL` 프로시저
```sql
CREATE OR REPLACE PROCEDURE SP_AUTO_ATTEND_ALL
IS
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
        INSERT INTO ATTENDANCE (ATTENDANCE_ID, RESERVATION_ID, USER_ID)
        VALUES (ATTENDANCE_SEQ.NEXTVAL, rec.RESERVATION_ID, NULL)
        RETURNING ATTENDANCE_ID INTO V_ATTENDANCE_ID;

        INSERT INTO ATTENDANCE_DETAIL (DETAIL_ID, ATTENDANCE_ID, USER_ID, ATTEND_STATUS_ID)
        SELECT ATTENDANCE_DETAIL_SEQ.NEXTVAL, V_ATTENDANCE_ID, T.USER_ID, 1
          FROM ( /* 파티장 + 활성 파티원 */ ) T;
    END LOOP;
    COMMIT;
END;
```

**포인트**
- 종료 시간 = `OPEN_AT + DURATION/1440` (분 → 일 변환, 1일 = 1440분)
- `ATTENDANCE.USER_ID = NULL` → 자동처리 표시 (수동은 사장/매니저 USER_ID)
- `UNIQUE(RESERVATION_ID)` 제약 덕분에 동시 호출되어도 중복 INSERT 차단
- `VW_RESERVATION_ALL`(yuhwon 분 자산) 호출만 함 — 다른 분 코드 안 건드림

### `JOB_AUTO_ATTEND` 스케줄러 잡
```sql
DBMS_SCHEDULER.CREATE_JOB (
    job_name        => 'JOB_AUTO_ATTEND',
    job_type        => 'PLSQL_BLOCK',
    job_action      => 'BEGIN SP_AUTO_ATTEND_ALL; END;',
    repeat_interval => 'FREQ=MINUTELY; INTERVAL=1',
    enabled         => TRUE
);
```

---

## 3. 출석체크 풀스택 — 세션 draft + 최종확인 트랜잭션

### 새 DTO — `AttendForm` (sb02 패턴)
학원 `sb02_FileUploadNotice`에서 `Notice` DTO 안에 `List<MultipartFile>` 필드를 두고 Service에서 순회하는 패턴 → 그대로 적용:

```java
@Getter @Setter @NoArgsConstructor
public class AttendForm {
    private Long reservationId;
    private List<Long> userIds;
    private List<Long> attendStatusIds;
}
```

JSP의 `<input name="userIds">` × N + `<select name="attendStatusIds">` × N → Spring이 자동으로 `List<Long>`에 모아 바인딩.

### Service — `saveDraft` (세션 누적)
```java
public void saveDraft(AttendForm form, HttpSession session) throws Exception {

    try {
        List<AttendItemDTO> drafts = (List<AttendItemDTO>) session.getAttribute("attendDraft");
        if (drafts == null) drafts = new ArrayList<>();

        // 같은 예약의 이전 draft 제거
        List<AttendItemDTO> newDrafts = new ArrayList<>();
        for (AttendItemDTO d : drafts) {
            if (!form.getReservationId().equals(d.getReservationId())) {
                newDrafts.add(d);
            }
        }

        // 이번 폼 항목 누적 (미정은 스킵)
        for (int i = 0; i < form.getUserIds().size(); i++) {
            Long statusId = form.getAttendStatusIds().get(i);
            if (statusId == null) continue;
            AttendItemDTO item = new AttendItemDTO();
            item.setReservationId(form.getReservationId());
            item.setUserId(form.getUserIds().get(i));
            item.setAttendStatusId(statusId);
            newDrafts.add(item);
        }

        session.setAttribute("attendDraft", newDrafts);

    } catch (Exception e) {
        log.info("saveDraft : ", e);
        throw e;
    }
}
```

**왜 Map 안 썼나** — 학원 패턴에 Map 안 나옴. List + for + contains 만으로 처리.

### Service — `finalizeAttendance` (트랜잭션 묶음 INSERT)
```java
@Transactional(rollbackFor = Exception.class)
public void finalizeAttendance(HttpSession session, Long staffUserId) throws Exception {

    try {
        List<AttendItemDTO> drafts = (List<AttendItemDTO>) session.getAttribute("attendDraft");
        if (drafts == null || drafts.isEmpty()) return;

        // 중복 없는 reservationId 모으기
        List<Long> resIds = new ArrayList<>();
        for (AttendItemDTO d : drafts) {
            if (!resIds.contains(d.getReservationId())) {
                resIds.add(d.getReservationId());
            }
        }

        for (Long reservationId : resIds) {

            // 스케줄러가 먼저 박은 경우 스킵
            if (mapper.selectAttendanceExists(reservationId) > 0) continue;

            AttendItemDTO head = new AttendItemDTO();
            head.setReservationId(reservationId);
            head.setUserId(staffUserId);
            mapper.insertAttendance(head);     // selectKey로 attendanceId 박힘

            for (AttendItemDTO it : drafts) {
                if (!reservationId.equals(it.getReservationId())) continue;
                it.setAttendanceId(head.getAttendanceId());
                mapper.insertAttendDetailByUser(it);

                if (it.getAttendStatusId() != null && it.getAttendStatusId() == 2L) {
                    Manner m = new Manner();
                    m.setUserId(it.getUserId());
                    mapper.callInsertNoshow(m);
                }
            }
        }

        session.removeAttribute("attendDraft");

    } catch (Exception e) {
        log.info("finalizeAttendance : ", e);
        throw e;
    }
}
```

**포인트**
- `selectKey BEFORE`로 `attendanceId`가 head 객체에 박힘 → 자식 INSERT 재사용 (회원가입 USER_ID 패턴과 동일)
- `@Transactional(rollbackFor = Exception.class)` + `throw e` → 노쇼 SP 실패하면 ATTENDANCE도 같이 롤백
- `selectAttendanceExists > 0` 체크 = 스케줄러와의 UNIQUE 충돌 사전 방지

### Controller — 1줄 위임 (sp16 패턴)
```java
@PostMapping("/saveDraft")
public String saveDraft(AttendForm form, HttpSession session) {
    String redirect = AuthUtil.checkStaff(session);
    if (redirect != null) return redirect;

    try {
        attendanceService.saveDraft(form, session);
    } catch (Exception e) {
        log.info("saveDraft : ", e);
    }
    return "redirect:/owner/attendance";
}
```

비즈니스 0줄, Service 호출 1줄. sp16의 `BoardController.writeSubmit` 결.

---

## 4. 진행 상태 분기 — partialList

### 문제
세션에 draft 1명 박혀도 `doneList`에 추가됨 → "입력 완료" 표시 잘못. 사용자 요구는 "1명만 박힌 거면 미완료 상태로".

### 해결 — TOTAL_MEMBER와 draft 카운트 비교
```java
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
    else                     partialList.add(rid);
}
```

JSP 분기:
```jsp
<c:choose>
    <c:when test="${r.statusName == '출석 완료'}">출석 + 버튼 없음</c:when>
    <c:when test="${r.statusName == '노쇼'}">노쇼 + 버튼 없음</c:when>
    <c:otherwise>
        <c:choose>
            <c:when test="${doneList.contains(r.reservationId)}">입력 완료 + [다시 입력]</c:when>
            <c:when test="${partialList.contains(r.reservationId)}">출석 미완료 + [이어서 입력]</c:when>
            <c:otherwise>대기 + [출석체크확인]</c:otherwise>
        </c:choose>
    </c:otherwise>
</c:choose>
```

→ 최종확인 후엔 statusName이 '출석 완료'로 바뀌어 첫 분기 → **[다시 입력] 자체가 사라짐 = 수정 불가**.

---

## 5. 재진입 시 이전 선택값 selected 복원

### 모델 — `AttendCrew`에 필드 추가
```java
private Long userId;
private String nickname;
private String position;
// 이전 선택값 (null = 미정)
private Long attendStatusId;
```

### Controller — `check()`에서 draft 매칭
```java
List<AttendCrew> crewList = attendanceService.selectCrewByReservationId(reservationId);

List<AttendItemDTO> drafts = (List<AttendItemDTO>) session.getAttribute("attendDraft");
if (drafts != null && crewList != null) {
    for (AttendCrew c : crewList) {
        for (AttendItemDTO d : drafts) {
            if (reservationId.equals(d.getReservationId())
                && c.getUserId().equals(d.getUserId())) {
                c.setAttendStatusId(d.getAttendStatusId());
                break;
            }
        }
    }
}
```

### JSP — EL `eq` 비교로 selected
```jsp
<select name="attendStatusIds" class="form-select">
    <option value="" ${empty c.attendStatusId ? 'selected' : ''}>미정</option>
    <option value="1" ${c.attendStatusId eq 1 ? 'selected' : ''}>출석</option>
    <option value="2" ${c.attendStatusId eq 2 ? 'selected' : ''}>노쇼</option>
</select>
```

---

## 6. 아이디 찾기 / 비밀번호 변경

### 아이디 찾기
- 폼: 이름 + 이메일
- 흐름: `findByNameAndEmail` 매칭 → `MailService.sendUserIdMail` → 가입 이메일로 LOGIN_ID 발송
- Ajax 응답: `response.getWriter().print("SUCCESS"/"NOT_FOUND")` — 학원 `idCheck` 패턴 유지

### 비밀번호 변경 — 인증번호 3단계
```
[1단계] 이름 + 아이디 → /user/findPwAuth
   Service.sendAuthCode :
     ① findByLoginIdAndName 검증
     ② 6자리 랜덤 코드 생성
     ③ 세션에 authCode + authCodeLoginId 저장
     ④ MailService.sendAuthCodeMail → 메일 발송
   ⇒ "SUCCESS" / "NOT_FOUND"

[2단계] 인증번호 입력 → /user/verifyCode
   Service.verifyAuthCode :
     세션의 코드 + loginId 둘 다 일치 → authCodeVerified=true
   ⇒ JSON {"status":"success"}

[3단계] 새 비번 → /user/resetPw
   Service.resetPassword :
     ① authCodeVerified 체크 (인증 안 한 사람 차단)
     ② findByLoginId 조회
     ③ UPDATE PASSWORD = CRYPTPACK.ENCRYPT(...)
     ④ 세션 정리
   ⇒ JSON {"status":"success"} → 로그인 페이지 이동
```

### 보안 보호 장치 — 3중
1. `authCodeLoginId` 저장 → 폼에서 LOGIN_ID 바꿔도 매칭 실패
2. `authCodeVerified` 플래그 → 인증 안 한 사람의 비번 변경 차단
3. `CRYPTPACK.ENCRYPT` → 비번 평문 저장 X (회원가입과 동일)

### 6자리 코드 생성
```java
String authCode = String.valueOf(100000 + new Random().nextInt(900000));
```
100000 ~ 999999 → 항상 6자리 보장.

---

## 7. 학원 패턴 적용 사례

| 우리 코드 | 학원 출처 |
|---|---|
| `try { mapper.x(); } catch { log.info; throw e; }` | sp16 `BoardServiceImpl.insertBoard` |
| Controller 1줄 위임 | sp16 `BoardController.writeSubmit` |
| DTO 안 List + Service 순회 | sb02 `NoticeServiceImpl.insertNoticeFile` |
| `selectKey BEFORE` → 자식 INSERT 재사용 | 학생 본인 회원가입 풀스택 |
| `response.getWriter().print("SUCCESS"/"NOT_FOUND")` | 학생 idCheck (학원 Servlet 패턴) |
| `CRYPTPACK.ENCRYPT(?, '12341234')` | 학원 PL/SQL 패키지 |

---

## 8. 면접 Q&A

**Q1. 자동처리를 왜 DB 스케줄러로 했나요? Spring @Scheduled가 더 익숙하지 않나요?**
A. 비즈니스 요구가 "사장이 안 누르면 자동 처리"였습니다. Spring @Scheduled를 쓰려면 애플리케이션이 떠 있어야 하는데, 그건 사실상 "페이지 진입 시 호출"과 같은 결입니다. DBMS_SCHEDULER는 DB 안에서 돌기 때문에 애플리케이션 의존성이 없고, **책임 분리** 관점에서도 깔끔합니다 — UI 액션은 Spring, 시간 기반 처리는 DB.

**Q2. ATTENDANCE 테이블의 UNIQUE 제약을 어떻게 활용했나요?**
A. `UNIQUE(RESERVATION_ID)` 덕분에 사장이 막 누른 순간 스케줄러가 동시에 처리해도 DB 레벨에서 중복 INSERT가 차단됩니다. 그래서 Service에서 `selectAttendanceExists` 사전 체크로 충돌을 우아하게 회피하고, 만약 race condition으로 통과해도 DB 제약이 마지막 안전망 역할을 합니다.

**Q3. 세션에 임시저장한 이유는?**
A. 출석체크는 즉석에서 끝나는 작업이라 DB 임시테이블까지 둘 필요는 없다고 판단했습니다. 또 세션이 끊겨도 다시 체크하면 되는 정도의 작업입니다. 학원 게시판에서 본 세션 패턴과 결이 같아 인지 부하도 낮습니다.

**Q4. ATTENDANCE.USER_ID를 왜 NULL로 박았나요?**
A. 스키마에 `USER_ID NUMBER`로 NOT NULL이 아닙니다. 컬럼 주석에 "카페관계자번호"라고 적혀 있어서, 사람이 처리하면 USER_ID, 스케줄러가 자동 처리하면 NULL로 의미를 구분했습니다. 운영자가 "이건 사람이 안 처리한 거구나"를 한눈에 파악할 수 있습니다.

**Q5. Map을 안 썼다고 했는데, 그룹핑은 어떻게 했나요?**
A. List에 reservationId를 중복 없이 모으는 1차 순회, 그 후 예약별로 draft 항목을 다시 필터링하는 2차 순회로 처리했습니다. O(n²)지만 출석체크는 한 번에 처리하는 예약이 많아야 몇 건이라 부하는 무시할 만했고, **학원에서 본 List + for + contains 패턴**으로 유지하는 게 가독성에 더 좋다고 판단했습니다.

**Q6. 비밀번호 인증번호 흐름에서 가장 신경 쓴 보안 포인트는?**
A. 1단계에서 입력한 LOGIN_ID를 세션에 저장하고, 3단계에서 다시 비교했습니다. 이게 없으면 사용자가 1단계에선 본인 아이디로 인증받고 3단계에서 남의 아이디를 입력해 비번을 바꿀 수 있습니다. 또 `authCodeVerified` 플래그로 2단계를 건너뛰고 3단계로 바로 못 들어오게 막았습니다.

**Q7. selectKey BEFORE를 어떻게 응용했나요?**
A. 회원가입에서 USER_ACCOUNT INSERT 후 같은 USER_ID로 USER_INFO INSERT 하는 패턴을 그대로 출석체크에 적용했습니다. ATTENDANCE INSERT 시 selectKey로 attendanceId가 dto에 박히고, 그 dto를 그대로 ATTENDANCE_DETAIL INSERT에 재사용합니다. 한 번 익힌 패턴을 도메인을 바꿔서 반복 적용한 사례입니다.

**Q8. partialList 분기는 왜 필요했나요?**
A. 처음엔 세션에 draft가 있으면 doneList에 추가하는 로직이었는데, 시연 중 "1명만 박혔는데 입력 완료로 뜬다"는 피드백을 받았습니다. 비즈니스 의미상 모든 파티원이 체크돼야 입력 완료입니다. draft 개수와 TOTAL_MEMBER를 비교해서 미완료(partial)와 완료(done)를 분리했습니다.

---

## 9. 시연 흐름 (10단계)

1. user0007/12341234 로그인
2. `/owner/attendance` → 1시간 이내 시작 예약만 노출
3. [출석체크 확인] → attendanceCheck 페이지 (파티원 N명)
4. 1명만 "출석" 선택, 나머지 "미정" → [확인]
5. 리스트 돌아옴 → **"출석 미완료" + [이어서 입력]**
6. [이어서 입력] → 이전 선택값 selected로 표시
7. 나머지 채우기 → [확인] → "입력 완료" + [다시 입력]
8. [최종 확인 (1건)] → ATTENDANCE + ATTENDANCE_DETAIL INSERT + (노쇼면) MANNER_HISTORY 차감
9. 페이지 갱신 → statusName='출석 완료' → **[다시 입력] 사라짐 = 수정 불가**
10. 자동처리 시연: 종료+30분 지난 예약 만들기 → 1분 안에 스케줄러가 처리 → ATTENDANCE.USER_ID=NULL 박힘

---

## 10. 코칭 계약 회고

- 빠른 진행 모드 유지. 코드 보여주고 OK 받으면 적용 패턴 안정화.
- 학원 패턴 우선 룰 강화: "내가 배운 것들로만"이라는 학생 본인 명시 → sp16 + sb02 모범 코드를 GitHub `Spring_Boot_Study2`에서 직접 가져와 패턴 매칭.
- 다른 팀원 코드(View/Function)는 호출만, 학생 본인 영역만 수정.
- 줄끝 주석 → 위 줄로 옮기는 본인 스타일 일관 적용.
