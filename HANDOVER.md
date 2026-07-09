# 서울관광재단 국외출장 마일리지 관리 시스템 — 인수인계 문서

> 클라우드(Claude Code Web) 작업분을 로컬 개발로 이어받기 위한 정리 문서입니다.
> 이 문서 하나로 프로젝트 구조·데이터·개발 규칙·이력·배포 방법을 파악할 수 있습니다.

---

## 1. 프로젝트 개요

**서울관광재단의 국외출장 적립 항공 마일리지를 조회·관리하는 웹 애플리케이션**입니다.

- **프론트엔드**: `index.html` (단일 파일 SPA, CSS·JS 전부 인라인)
- **백엔드**: Google Apps Script 웹앱 (`doGet`)
- **데이터베이스**: Google Sheets (`재단_db`, `주식회사_db`, `국외출장_db`)
- **부가 앱**: `budgetcal.html` (국외출장 여비 계산기, 독립 실행 파일)

데이터 흐름: **브라우저(index.html) ⇄ fetch(GET) ⇄ Apps Script ⇄ Google Sheets**

---

## 2. 파일 구조

| 파일 | 설명 |
|---|---|
| `index.html` | 마일리지 관리 앱 본체. CSS(`<style>`)·JS(`<script>`) 전부 인라인. 약 4,000행 |
| `budgetcal.html` | 여비 계산기(독립 HTML). index.html에서 `href="budgetcal.html"` 상대경로로 연결 |
| `design-system.css` | 공용 디자인 토큰/컴포넌트 CSS **(참고용 마스터 사본)**. ⚠️ index.html은 자체 인라인 스타일을 쓰므로 이 파일을 링크하지 않음 |
| `HANDOVER.md` | 이 문서 |

**외부 CDN 의존성** (index.html `<head>`):
- Pretendard 폰트 (`jsdelivr`)
- `xlsx` 0.18.5 — 엑셀 내보내기
- `Chart.js` 4.4.1 — 항공사별 도넛 차트

---

## 3. 로컬 개발 환경 세팅

특별한 빌드 도구 없이 **파일만 열면 됩니다.**

1. 저장소를 클론/다운로드 (`index.html`, `budgetcal.html`을 **같은 폴더**에 유지)
2. `index.html`을 브라우저로 열기 (더블클릭 또는 로컬 서버)
   - CDN 라이브러리·Apps Script 통신 때문에 **인터넷 연결 필요**
   - 로컬 서버 권장: `python -m http.server` 후 `http://localhost:8000/index.html`
     (일부 브라우저는 `file://`에서 fetch/CORS 제약이 있을 수 있음)
3. 편집 → 저장 → 브라우저 새로고침으로 확인
4. 배포는 Git push (GitHub Pages 등 정적 호스팅)

> ⚠️ **단일 파일 원칙**: index.html은 CSS/JS를 전부 내부에 담은 self-contained 구조입니다. 외부 파일로 분리하지 마세요(로컬/오프라인/다운로드 배포 호환성 때문).

---

## 4. 데이터 구조 (Google Sheets)

### 4-1. `재단_db` / `주식회사_db` (동일 구조, 25열)
프론트 `COL` 상수, 백엔드 `COL` 상수와 **인덱스가 반드시 일치**해야 합니다.

| 열 | 필드 | 비고 |
|---|---|---|
| A | 순번 | 수식 `=ROW()-1` |
| B~Q | 부서·품의번호·출장자·날짜·국가·항공사·항공편명 등 | |
| R | **사용/적립(mileage)** | 실제 마일 수치. 미적립은 텍스트 `-` |
| S | **잔여(available)** | 합산 기준 값. 미적립은 텍스트 `-` |
| T | 유효기간(expire_date) | |
| U | **유효여부(is_valid)** | ⚠️ **시트 수식 자동계산** |
| V | **구분(category)** | ⚠️ **시트 수식 자동계산** |
| W | 연도 | |
| X | 상태(status) | 퇴직 등 |
| Y | 비고(memo) | |

**U·V열 수식 (buildRow/applyFormulas가 입력):**
```
유효여부(U) = IFS(T="","사용", T="-","미적립", T>=TODAY(),"유효", T<TODAY(),"무효") …
구분(V)     = IFS(S="","확인", S="-","미적립", S=0,"사용", (S>0,U="무효"),"불용", (S>0,U="유효"),"적립" …)
```

### 4-2. `국외출장_db` (30열)
`COL_TRIP` 상수 참조. 예산·집행·항공/일비/식비/숙박 등 + 재단_db에서 자동연동되는 회색 항목.

### 4-3. 상태값 정리
- **is_valid_raw**: `유효` / `무효` / `미적립` / `사용` / `확인`
- **category_raw**: `적립` / `사용` / `불용` / `미적립` / `확인`
- **ID 규칙**: `F-<행번호>`(재단), `C-<행번호>`(주식회사), `T-<행번호>`(출장)

### 4-4. 핵심 계산 규칙
- **사용 가능 마일**: `isCalcSource(재단/주식회사) && is_valid_raw='유효' && category_raw='적립'` 인 행의 **잔여(available)** 합
- **미적립**: R·S열에 텍스트 `-` 저장 → 수식이 미적립으로 판정 → 화면에서 `-` 표시

---

## 5. 프론트엔드 구조 (index.html)

### 5-1. 화면(사이드바 섹션)
| 섹션 | id | 핵심 함수 |
|---|---|---|
| 🔍 마일리지 조회 | `section-mileage` | `doUserSearch`, `renderUserSummary` / 검색 전 대시보드(캘린더+통계+링크) |
| ⚠️ 소멸 예정 | `section-expire` | `renderExpireSection`→`renderExpireBanners`+`renderOverviewTable`/`renderOverviewPage` |
| ✈️ 항공사별 | `section-airline` | `renderAirlineCards`, `showAirlineDetail` (도넛=Chart.js) |
| 👤 개인 현황 | `section-personal` | `renderPersonalCards`, `showPersonalDetail`, `renderPersonalDetailTable` |
| 🌏 출장 현황 | `section-trip-dash` | `renderTripDash` + 출장 캘린더 |
| 🗄️ 데이터베이스(관리자) | `#tab-database` | `renderDbTable`, `applyDbFilters`, `submitForm`, `submitBulkEdit` |

### 5-2. 자주 쓰는 공용 함수
- **데이터 로드**: `loadData(silent)`, `loadTrips(silent)` — fetch + localStorage 캐시
- **날짜**: `getKSTToday()`, `getDaysLeft(expire)`, `parseLocalDate("YYYY-MM-DD")`
- **판정/포맷**: `isValidRow(r)`(=유효), `isCalcSource(r)`(=재단/주식회사), `fmt`, `fmtNum`, `fmtMileage`, `fmtAvailable`
- **행 배경색**: `getRowBgClass(r)` — **모든 테이블이 공유하는 단일 색칠 로직**
- **소멸 구간**: `EXPIRE_RANGE = {30:[0,30], 90:[31,90], 180:[91,180]}`

### 5-3. 로딩 방식 (localStorage 캐시)
- 재방문: 캐시(`sto_mileage_data_v1`, `sto_mileage_trips_v1`)를 **즉시 표시** → 백그라운드에서 `loadData(true)`/`loadTrips(true)`로 조용히 갱신
- 첫 방문: `getAll`·`getTrips` 병렬 로드
- `refreshAllData()`(상단 새로고침 버튼)로 수동 갱신

---

## 6. 백엔드 (Google Apps Script)

- Google Sheets → **확장 프로그램 → Apps Script**에 코드 존재 (이 저장소 밖)
- `doGet(e)` 하나로 조회+CUD 모두 처리 (`action` 파라미터로 분기): `getAll`, `getTrips`, `add`, `update`, `delete`, `updateAvailable`, `updateMemo`, `updateCategory`, `updateCategoryAndStatus`, `addTrip`, `updateTrip`, `deleteTrip`, `lookupTripDoc`
- 주요 함수: `getAllData`, `mapMainRow`, `buildRow`(행 배열 생성), `applyFormulas`(U/V/순번 수식), `applyCellFormats`(서식), `updateRow`, `getTrips`/`buildTripRow`/`applyTripFormulas`
- 응답: `{ success, data / error }` JSON

### 6-1. 배포 방법 (백엔드 수정 시)
1. Apps Script 편집기에서 코드 수정 → **저장(Ctrl+S)**
2. **배포 → 배포 관리 → ✏️수정 → 버전 "새 버전" → 배포**
   - 기존 배포를 "수정"으로 올리면 **URL 유지** → index.html 변경 불필요
   - "새 배포"로 만들면 **새 URL** → `index.html`의 `APPS_SCRIPT_URL`(약 1068행) 교체 필요
3. 접근 권한은 **"누구나"**

현재 값:
- `APPS_SCRIPT_URL` = `…/macros/s/AKfycbzkyb1ZRF…/exec` (index.html 1068행)
- 연결된 시트: `GOOGLE_SHEET_URL` (1069행)

---

## 7. ⚠️ 개발 시 반드시 지켜야 할 가이드라인

> 아래는 실제로 버그를 겪고 고친 항목들입니다. **재발 주의.**

### (1) 날짜는 무조건 KST 기준 파싱
- `new Date("2026-04-08")`는 **UTC 자정**으로 해석 → KST(UTC+9)에서 **하루 밀림**.
- 날짜 문자열은 반드시 **`parseLocalDate("YYYY-MM-DD")`** 사용 (로컬 자정 생성).
- 오늘 날짜는 **`getKSTToday()`**. D-day는 `getDaysLeft()`.
- 캘린더/소멸 계산에서 `new Date(문자열)` 직접 사용 금지.

### (2) Apps Script 이중 디코딩 금지
- GAS는 `e.parameter.xxx`를 **이미 URL 디코딩**해서 줌.
- 여기에 `decodeURIComponent()`를 **한 번 더** 하면, 값에 `%` 등이 있을 때 **`URI malformed`** 오류 발생.
- 반드시 **`JSON.parse(e.parameter.data)`** 처럼 추가 디코딩 없이 사용.

### (3) 미적립은 텍스트 `-`
- 프론트: 폼의 "미적립" 체크 시 mileage/available를 `'-'`로 전송.
- 백엔드 `buildRow`: `data.mileage === '-' ? '-' : (parseFloat(...)||0)`.
  `parseFloat('-')`=NaN → 셀에 `0`/`#NUM!` 들어가므로 **문자 `-`는 그대로 통과**시킬 것.

### (4) 유효여부(U)·구분(V)은 시트 수식이다
- 저장 시 `is_valid:'', category:''`로 보내고 **시트 수식이 자동 계산**.
- 이 값을 임의로 텍스트로 덮어쓰지 말 것 (전용 함수 `updateCategory`/`updateCategoryAndStatus` 제외 — 퇴직·불용 처리용).
- **합산·필터 조건**(유효+적립 등)은 신중히. 규칙 변경 시 화면 통계가 바뀜.

### (5) 행 배경색은 `getRowBgClass` 한 곳에서만
우선순위: **사용(음수) → 잔액0/무효/불용(회색) → 유효기간 임박(exp색) → 기본**
- 모든 테이블이 이 함수를 공유. 색 규칙 바꾸려면 여기만 수정.
- (참고: 과거 만료행 취소선 `row-void`은 제거됨 — 만료 시 시트에서 불용→회색 처리되므로 불필요)

### (6) 테이블 너비/정렬 관례
- 헤더: `<th style="width:NNpx;text-align:center|right">`
- 셀: `class="nowrap"` + 필요 시 `max-width:NNpx;overflow:hidden;text-overflow:ellipsis` + `title="전체값"`(툴팁)
- **너비 미지정 컬럼(예: 출장명)은 남은 공간을 자동 배분** (`.data-table`가 `width:100%`, 좌우 대칭 그리드면 가운데 경계가 정확히 50%에 맞음)

### (7) 커밋/푸시 전 원격 동기화
- GitHub 웹에서 **직접 편집한 커밋**이 원격에 있을 수 있음.
- push 전 `git fetch` → 필요 시 `git rebase origin/<branch>` 후 push (지금까지 문구 수정 등 몇 건 이런 식으로 병합됨).

### (8) budgetcal.html 상대경로 유지
- 여비계산기 링크는 `href="budgetcal.html"` (절대 URL 아님).
- **index.html과 항상 같은 폴더**에 함께 있어야 열림. 다운로드/이전 시 세트로 이동.

### (9) 캐시 주의
- localStorage 캐시로 재방문 시 **직전 데이터가 1~2초 보였다가 최신화**됨(정상).
- 데이터 즉시 반영이 필요하면 상단 '새로고침' 버튼 사용.

---

## 8. 주요 개발 이력 (요약)

시간순 핵심 변경 사항:

1. **여비계산기 분리** — index.html에 있던 계산기(calc) 섹션 제거 → 별도 `budgetcal.html`
2. **통합 현황 탭 제거** — 전체 적립·사용 내역 테이블을 '소멸 예정' 탭으로 이전, 기간 배너 클릭=필터/재클릭=해제 토글
3. **마일리지 조회 첫 화면 대시보드** — 이름 검색 전 출장 캘린더 + 통계 카드 + 외부 링크 버튼 4개 표시
4. **소멸 예정 구간 필터** — 카드가 겹치지 않는 구간(0–30 / 31–90 / 91–180일)만 표시하도록 `EXPIRE_RANGE` 도입
5. **항공사별 상세** — 유효+적립 행만 표시(카드 합계와 일치)
6. **행 색칠 통합** — `getRowBgClass` 단일화, 취소선 제거, 불용도 회색
7. **데이터베이스 개선** — 유효여부 필터 순서, '확인필요'(유효여부 또는 구분='확인') 필터, '미적립(-)' 체크박스, 일괄수정 실패 시 실제 오류 메시지 노출
8. **로딩 속도** — getAll·getTrips 병렬화 → localStorage stale-while-revalidate 캐시
9. **KST 날짜 버그 수정** — `parseLocalDate` 도입(캘린더 시작일 하루 밀림, D-day 하루 오차 해결)
10. **백엔드 수정** — 이중 디코딩(`decodeURIComponent`) 제거, 미적립 `-` 텍스트 저장, `applyFormulas` 복구
11. **테이블 너비/정렬 정비** — 소멸예정·데이터베이스·개인별·항공사별 테이블 컬럼 폭·정렬 조정, 대시보드 좌우 50:50 정렬
12. **디자인 시스템 추출** — `design-system.css`(색·이징·그림자·hover·애니메이션 토큰), 캘린더 카드 hover 효과

---

## 9. 알려진 이슈 / 개선 아이디어

- **첫 로딩 속도**: 근본 원인은 Apps Script 응답 지연. 프론트 캐시로 재방문은 빠르나, **첫 방문/실제 응답**을 빠르게 하려면 백엔드 `CacheService`(예: `getAll` 2분 캐시 + 수정 시 `cache.remove`) 적용 권장. (CacheService 항목당 100KB 제한 → 데이터 크면 분할 저장 필요)
- **GET 방식 저장**: 데이터 저장을 URL 쿼리(GET)로 전송. 데이터가 매우 커지면 URL 길이 한계 가능 → 필요 시 POST 전환 검토.
- **미적립 행 잔여**: 프론트에서 `parseFloat('-')||0`으로 0 처리되나, 화면 표시는 category 수식('미적립')으로 `-` 노출되어 문제없음. 합산에서도 제외됨.

---

## 10. 빠른 참조 (자주 여는 위치)

| 대상 | 위치 |
|---|---|
| Apps Script URL | index.html `const APPS_SCRIPT_URL` (~1068행) |
| 사이드바 섹션 목록 | `switchSection()` 배열 |
| 행 색칠 규칙 | `function getRowBgClass` |
| 소멸 구간 정의 | `const EXPIRE_RANGE` |
| 로딩/캐시 | `loadData`, `loadTrips`, 하단 `// 초기 실행` IIFE |
| 마일리지 대시보드(캘린더·통계·링크) | `id="mil-dashboard"` |
| DB 테이블 렌더 | `renderDbTable`, `applyDbFilters` |
| 저장(추가/수정/일괄) | `submitForm`, `submitNewMultiTraveler`, `submitBulkEdit` |

---

*본 문서는 클라우드 작업 세션 내용을 압축·정리한 것입니다. 구조·규칙이 바뀌면 이 문서도 함께 갱신해 주세요.*
