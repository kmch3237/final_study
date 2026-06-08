# Day 11 — 시연 후 보강 (2026-06-09)

D-Day 시연 후 출석체크 흐름 개선 + 정적 리소스 매핑 + 회원가입 UX 보강.

---

## 1. 출석체크 흐름을 2단계로 분리

### 기존 (Day 10)
- 출석체크 페이지에서 예약 행마다 `[출석]` / `[노쇼]` 버튼 → **버튼 누르면 즉시 INSERT**
- 한 예약당 파티장(LEADER) 1명만 처리

### 변경 후
1. 출석체크 페이지: 예약 행마다 `[출석체크확인]` 버튼만
2. `[출석체크확인]` 클릭 → `/owner/attendance/check?reservationId=X` 새 페이지
3. 파티원 N명 리스트 (파티장 + 멤버 전부) → 각자 `select` 박스로 출석/노쇼 선택
4. `[확인]` → 세션에 임시 저장 (`attendDraft` List) → 출석체크 페이지로 redirect
5. 출석체크 페이지 하단 `[최종 확인 (N건)]` → INSERT 실행 + 세션 비움

### 핵심 패턴

**(1) DB 자산 재활용 (협업)**
- jsc 분의 `VW_PARTY_ACTIVE_MEMBER` 뷰 = 파티별 활성 멤버 (HOST + MEMBER)
- yuhwon 분의 `VW_RESERVATION_ALL` 뷰 = 예약-파티-룸 통합
- → 둘을 JOIN해서 `reservationId`로 파티원 전체 조회

**(2) 세션 임시 저장 (List 패턴)**
```java
// Map 안 배워서 List<AttendanceListDTO>로
session.setAttribute("attendDraft", draftList);
```
- Controller에서 `userIds[]` `attendStatusIds[]` 배열로 받음
- 같은 reservationId 다시 입력 시 기존 draft 제거 후 새로 추가

**(3) 예약별 그룹핑 INSERT (Service)**
```java
@Transactional
public void attendAll(List<AttendanceListDTO> list, Long staffUserId) throws Exception {
    AttendanceListDTO head = null;
    for (AttendanceListDTO dto : list) {
        // 새 예약이면 ATTENDANCE 1행 INSERT
        if (head == null || !head.getReservationId().equals(dto.getReservationId())) {
            head = new AttendanceListDTO();
            head.setReservationId(dto.getReservationId());
            head.setUserId(staffUserId);
            mapper.insertAttendance(head);  // selectKey가 attendanceId 박음
        }
        // 매번 ATTENDANCE_DETAIL INSERT (USER_ID = 파티원)
        AttendanceListDTO det = new AttendanceListDTO();
        det.setAttendanceId(head.getAttendanceId());
        det.setLeaderId(dto.getUserId());
        det.setAttendStatusId(dto.getAttendStatusId());
        mapper.insertAttendDetail(det);
    }
}
```
- 한 ATTENDANCE에 N개 DETAIL 묶음
- `@Transactional`로 중간 실패 시 전체 롤백
- 노쇼는 `TRG_NOSHOW` 트리거가 매너온도 자동 -1 (Day 9의 SP_INSERT_NOSHOW 호출 불필요)

### 트러블 - VW_PARTY_ACTIVE_MEMBER 컴파일 에러 (ORA-04063)
- jsc 분 뷰가 의존 함수(`FN_GET_USER_AGE` 등) 부재로 컴파일 에러
- **다른 분 뷰는 못 건드림** → 학생 본인 매퍼에 raw 테이블 직접 JOIN으로 우회

```xml
<select id="selectCrewByReservationId" parameterType="Long"
        resultType="com.noexit.app.model.AttendCrew">
    SELECT USER_ID, NICKNAME, POSITION
      FROM (
        SELECT P.USER_ID, UA.NICKNAME, 'HOST' AS POSITION
          FROM PARTY P
          JOIN USER_ACCOUNT UA ON UA.USER_ID = P.USER_ID
          JOIN RESERVATION R   ON R.PARTY_ID = P.PARTY_ID
         WHERE R.RESERVATION_ID = #{reservationId}
        UNION ALL
        SELECT PA.USER_ID, UA.NICKNAME, 'MEMBER' AS POSITION
          FROM PARTY_MEMBER PM
          JOIN PARTY_APPLY  PA ON PA.APPLY_ID = PM.APPLY_ID
          JOIN USER_ACCOUNT UA ON UA.USER_ID = PA.USER_ID
          JOIN RESERVATION  R  ON R.PARTY_ID = PA.PARTY_ID
         WHERE R.RESERVATION_ID = #{reservationId}
           AND NOT EXISTS (
               SELECT 1 FROM PARTY_KICK PK WHERE PK.MEMBER_ID = PM.MEMBER_ID
           )
      )
     ORDER BY DECODE(POSITION, 'HOST', 0, 1), USER_ID
</select>
```

### 학원에서 안 배운 JS 회피
첫 시도: JS로 동적 ID(`statusText${st.index}`) + `getElementById` → 학생이 어려워함
바꿈: **select + form submit** (학원 themeWriteForm 패턴)
- `<select>` + `<option>` (학원 기본)
- `<button type="submit" onclick="return confirm(...)">` (학원 패턴)
- `required` HTML 속성으로 검증
- → JS 함수 거의 0

### 면접용 한 줄
> "출석체크 흐름을 즉시 INSERT → 임시 저장 후 최종 확인 2단계로 분리했습니다. 세션 List를 사용해 한 예약 안 N명의 파티원 상태를 모아 한 트랜잭션에 INSERT하도록 `@Transactional` 메서드를 작성했고, 노쇼 시 매너온도는 다른 분의 DB 트리거가 자동 처리되도록 협업했습니다."

---

## 2. 정적 리소스 매핑 (WebConfig)

### 문제
- 학원 sb02 패턴으로 `webapp/uploads/theme/`에 이미지 업로드는 성공
- 그러나 브라우저에서 `/uploads/theme/X.jpg` 요청 시 **404**
- Spring Boot embedded Tomcat은 `webapp/uploads/`를 자동 서빙하지 않음

### 해결 - `WebMvcConfigurer.addResourceHandlers`
```java
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

    private final ServletContext servletContext;

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/uploads/**")
                .addResourceLocations("file:///C:/escapeRoom/noExit/src/main/webapp/uploads/");
    }
}
```

### 학습 포인트
- `addResourceHandler("/uploads/**")` = 브라우저가 요청할 URL 패턴
- `addResourceLocations("file:///...")` = 실제 파일 시스템 경로 (file: 프로토콜)
- `ServletContext.getRealPath`도 가능하지만 STS 환경에 따라 다른 경로 반환 가능성 → 절대경로가 안전
- 한 번 등록하면 다른 분 업로드 기능 (예: 카페 이미지)도 자동 혜택

### 면접용 한 줄
> "Spring Boot가 webapp 외부 폴더의 업로드 파일을 자동 서빙하지 않아서, `WebMvcConfigurer`를 구현한 설정 클래스에서 `/uploads/**` URL을 절대경로로 매핑하는 ResourceHandler를 등록했습니다."

---

## 3. 회원가입 비밀번호 검증 메시지 위치 분리

### 기존
- 모든 검증 메시지(아이디/비번/닉네임/이메일/연락처/성별/생년월일)가 단일 `#error` span에 표시
- 비밀번호 입력 칸과 시각적으로 떨어져 있어 어느 필드가 잘못됐는지 즉시 확인 어려움

### 변경
- `<input id="password">` 바로 아래 `<span id="pwError">` 추가
- `<input id="passwordConfirm">` 바로 아래 `<span id="pwConfirmError">` 추가
- JS 검증 메시지를 해당 span으로 분리 (`$("#pwError").html(...)`, `$("#pwConfirmError").html(...)`)
- 가입 버튼 누를 때 둘 다 reset

### 학원 패턴 유지
- alert 안 씀, 빨간 글씨로 안내 (강사님 표준)
- `.css({color: "red", display: "inline"})` 그대로

---

## 4. 알아두면 좋은 - 다른 분 페이지 이미지 표시 (참고)

`theme/list`, `theme/info`는 JStrong 분 작품. 학생 영역 아니라 직접 수정 X.
**그분에게 다음 한 줄 수정 요청 시 이미지 표시됨**:

### themelist.jsp 라인 201
```javascript
// 기존
+ "<div class='theme-image'>" + "<span>" + item.imagePath + "</span></div>"

// 변경 (학생 업로드 + 더미 둘 다 처리)
+ "<div class='theme-image'><img src='" + (item.imagePath && item.imagePath.charAt(0) === '/' ? '${path}' + item.imagePath : '${path}/uploads/theme/' + item.imagePath) + "' style='width:100%; height:auto;'></div>"
```

### themeinfo.jsp 라인 187
```jsp
<!-- 기존 -->
<span>${dto.imagePath }</span>

<!-- 변경 -->
<img src="${pageContext.request.contextPath}/uploads/theme/${dto.imagePath}" style="width:100%;">
```

전제: 학생이 만든 `WebConfig`가 `/uploads/**`를 매핑하고 있어야 동작.

---

## 5. 협업 룰 재확인
- **다른 분 파일은 호출만, 수정 X** (Day 10 룰)
  - 다른 분 뷰/함수 SQL에서 `FROM`/`JOIN`으로 이름만 사용 OK
  - 다른 분 Java 클래스 `@Autowired`로 주입해서 메서드 호출 OK
- **본인 파일은 자유**, 단 git diff 최소화 위해 그대로 둘 줄은 절대 안 건드림
- 신규 클래스는 자유 작성 (WebConfig, AttendCrew DTO 등)
