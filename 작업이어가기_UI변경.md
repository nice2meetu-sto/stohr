# 로컬 작업 이어가기 안내서 — UI/레이아웃 변경분 (2026-07-09)

> 클라우드에서 작업한 최근 UI 변경 4가지를 **로컬 Claude Code 앱에서 이어서** 작업하기 위한 안내서입니다.
> 대상 파일: **`index.html`** (단일 파일, CSS·JS 전부 인라인)
> 브랜치: **`claude/zealous-ptolemy-dye8z8`**
> ※ 행 번호는 편집하면 바뀌니, **"찾을 문자열"**(셀렉터/텍스트)로 검색하는 걸 추천합니다.

---

## 0. 로컬에서 시작하는 법

```bash
git fetch origin claude/zealous-ptolemy-dye8z8
git checkout claude/zealous-ptolemy-dye8z8
git pull origin claude/zealous-ptolemy-dye8z8
```
- `index.html`을 편집기/브라우저로 열어 작업 → 저장 → 브라우저 새로고침으로 확인
- 로컬 서버 권장: `python -m http.server` → `http://localhost:8000/index.html`
- (참고 문서: 같은 폴더의 **`HANDOVER.md`** — 전체 구조·데이터·개발 규칙)

---

## 1. 메뉴바 → 상단 중앙 플로팅 캡슐 네비게이션

**무엇**: 왼쪽 세로 사이드바를 Safari 주소창처럼 **상단 중앙에 떠 있는 알약형 가로 메뉴**로 변경. 스크롤해도 상단에 붙어있음(`sticky`).

**위치**: `index.html` `<style>` 끝부분, 주석 `/* ── 플로팅 상단 중앙 네비게이션 … */` 아래 블록 (현재 약 310~331행)

**찾을 문자열**: `플로팅 상단 중앙 네비게이션` 또는 `#admin-sidebar.sidebar{`

**현재 코드 (핵심)**:
```css
.admin-layout{display:block}
#admin-sidebar.sidebar{
  position:sticky; top:60px; z-index:90; bottom:auto;
  width:fit-content; max-width:calc(100% - 24px);
  margin:14px auto 20px; padding:5px;
  display:flex; gap:4px; flex-wrap:nowrap; overflow-x:auto;
  background:rgba(255,255,255,.82);
  backdrop-filter:saturate(180%) blur(20px);        /* 반투명 블러 */
  border:1px solid rgba(0,0,0,.06); border-radius:999px;  /* 알약 모양 */
  box-shadow:0 6px 24px rgba(0,0,0,.10);
}
#admin-sidebar .sidebar-item{ border-radius:999px; padding:8px 16px; white-space:nowrap; … }
#admin-sidebar .sidebar-item.active{ background:var(--main-lt); color:var(--main); font-weight:700; }
```

**조정 팁**:
- 상단 여백/붙는 위치: `top:60px`, `margin:14px auto 20px`
- 캡슐 둥글기: `border-radius:999px`, 그림자: `box-shadow`, 투명도: `background:rgba(...,.82)`
- 항목 간격/크기: `.sidebar-item`의 `padding:8px 16px`, `gap:4px`
- **원래 세로 사이드바로 되돌리려면**: 이 블록(310~331행)만 삭제하면 됨(그 위 기본 사이드바 CSS가 되살아남)

**관련 커밋**: `0990484`

---

## 2. 콘텐츠 최대 폭 1300px

**무엇**: 사이드바가 없어진 만큼 본문을 **가운데 정렬 + 최대 폭 1300px**로 제한.

**위치**: 위 플로팅 블록 안, 현재 약 329행

**찾을 문자열**: `.admin-body{margin:0 auto; max-width:`

**현재 코드**:
```css
.admin-body{margin:0 auto; max-width:1300px; padding:0 24px 44px; min-width:0}
```

**조정 팁**: `max-width:1300px` 숫자만 바꾸면 전체 콘텐츠 폭이 바뀝니다. (데이터베이스 탭은 전체화면이라 영향 없음 — `.db-mode .admin-body{max-width:none!important}`로 예외 처리됨)

**관련 커밋**: `49bb67f`

---

## 3. 출처 칼럼 너비 100px 고정 (모든 테이블 통일)

**무엇**: 모든 표의 **'출처' 헤더를 `width:100px` + 중앙정렬**로 통일.

**위치**: 총 **5곳** (현재 약 459 / 493 / 525 / 622 / 745행)
각각 마일리지조회·소멸예정·항공사별상세·개인별상세·데이터베이스 테이블

**찾을 문자열**: `width:100px;text-align:center">출처</th>`

**현재 코드 (5곳 동일)**:
```html
<th style="width:100px;text-align:center">출처</th>
```

**조정 팁**:
- 특정 테이블만 넓히려면 해당 행의 `100px`만 수정
- 전부 한 번에 바꾸려면 위 문자열을 전체 치환

**관련 커밋**: `3e9fbbc` (그 전 `cebcd9b`에서 DB만 130px 했다가 통일하며 원복)

---

## 4. 출처 배지 항상 한 줄로 표시 (글자 잘림 해결)

**무엇**: 한국어는 글자 사이 줄바꿈이 되어, 컬럼이 좁아지면 "주식회사"가 쪼개져 잘렸음.
→ 출처 배지에 **`white-space:nowrap`**을 넣어 **항상 한 줄**로 표시(잘림 방지).

**위치**: `<style>` 내 `.src-badge` 규칙 (현재 약 118행)

**찾을 문자열**: `.src-badge{`

**현재 코드**:
```css
.src-badge{font-size:10px;padding:2px 6px;border-radius:4px;background:var(--main-lt);color:var(--main);white-space:nowrap;display:inline-block}
```
(추가된 부분: `white-space:nowrap;display:inline-block`)

**원리**: `nowrap`이면 배지 최소폭이 "주식회사" 한 줄 폭으로 잡혀, 컬럼이 그 아래로는 안 눌려 잘리지 않음. 넘치면 표가 가로 스크롤됨.

**관련 커밋**: `5d9f7d2`

---

## 5. (관련) 개인현황 '목록' 화면만 폭 1040px

**무엇**: 개인현황 **목록 화면**을 마일리지 조회와 동일한 **1040px**로. **세부내역(이름 클릭 후 표)은 기존 너비 유지**.

**찾을 문자열**: `id="personal-list-view"` (현재 약 534행)

**현재 코드**:
```html
<div id="personal-list-view" style="max-width:1040px;margin:0 auto">
```
- 목록(`personal-list-view`)에만 적용해서, 세부내역(`personal-detail-view`)은 영향 없음.

**관련 커밋**: `d81abf4`

---

## 6. 작업 규칙 (필수)

- **단일 파일 원칙**: `index.html`은 CSS/JS 전부 인라인. 외부 파일로 분리 금지.
- **커밋/푸시 전**: `git fetch` → 필요 시 `git rebase origin/claude/zealous-ptolemy-dye8z8` (원격에 직접 편집분이 있을 수 있음).
- **행 번호보다 문자열 검색**: 편집하면 행이 밀리니 위의 "찾을 문자열"로 검색.
- 그 외 백엔드/데이터/KST 등 전반 규칙은 **`HANDOVER.md`** 참고.

---

## 7. 행 배경색 / 글씨색 규칙 (getRowBgClass) — ★최종

**무엇**: 표의 각 행 배경색·글씨색을 정하는 **단일 함수 `getRowBgClass(r)`**. 모든 테이블이 공유.
**설계**: **배경은 '구분(category)' 기준으로만** 정하고, **글씨 연하게는 '미적립'일 때만** 별도로 얹음(배경과 분리).

**위치**: JS `function getRowBgClass(r)` / CSS는 `.row-negative`·`.row-void`·`.row-muted` 등
**찾을 문자열**: `function getRowBgClass` , `.row-muted`

**현재 로직**:
```js
function getRowBgClass(r) {
  let bg = '';
  if (r.mileage < 0 || r.category_raw === '사용') bg = 'row-negative';            // 음수·사용 → 연한 빨강
  else if (r.category_raw === '불용')             bg = 'row-void';                // 불용 → 연한 회색
  else if (r.category_raw === '미적립' || r.is_valid_raw === '미적립') bg = '';   // 미적립 → 흰색
  else if (isValidRow(r)) {                                                       // 유효+적립 · 임박 → 유효기간 색
    const d = getDaysLeft(r.expire_date);
    if (d !== null && d >= 0) {
      if (d <= 30)       bg = 'row-exp30';
      else if (d <= 90)  bg = 'row-exp60';
      else if (d <= 180) bg = 'row-exp90';
    }
  }
  // 글씨 연하게: 미적립만 (무효는 기본 글씨색)
  if (r.is_valid_raw === '미적립' || r.category_raw === '미적립')
    bg = (bg + ' row-muted').trim();
  return bg;
}
```

**색상표**:
| 조건 | 배경 | 글씨 |
|---|---|---|
| 음수(mileage<0) 또는 구분=사용 | 연한 빨강 `rgba(163,45,45,.07)` (`row-negative`) | 사용액 빨간 글씨(유지) |
| 구분=불용 | 연한 회색 `rgba(136,135,128,.08)` (`row-void`) | 기본 |
| 미적립 | 흰색 | **연하게** (`row-muted`) |
| 무효 | (배경은 위 구분 따라) | **기본**(연하게 안 함) |
| 유효+적립 · 임박(30/90/180일) | 유효기간 색 `row-exp30/60/90` | 기본 |
| 그 외 | 흰색 | 기본 |

**관련 CSS**:
```css
.row-negative{background:rgba(163,45,45,.07)}   /* 음수·사용 : 연빨강 */
.row-void{background:rgba(136,135,128,.08)}      /* 불용 : 연회색 */
.data-table tr.row-muted td{color:var(--text-hint)!important}  /* 미적립 : 글씨 연하게 */
/* .row-exp30=var(--warn30-bg) / .row-exp60=var(--warn60-bg) / .row-exp90=#FDF8EC (유효기간 색) */
```

**조정 팁**:
- 색 진하기: 각 `rgba(...)`의 **마지막 알파값**(.07/.08 등)을 키우면 진해짐
- "글씨 연하게" 대상 바꾸기: `row-muted` 조건(`미적립 …`)에 `무효` 등을 넣거나 빼면 됨
- 배경 기준 바꾸기: `getRowBgClass` 위쪽 `if/else if` 순서·조건만 수정 (수식·필터 로직과 무관)

**관련 커밋**: `820b4dc` → `fd6e1c7` → `b12d73b` → `a7f6c3d` → **`e7f451a`(최종)**

---

## 최근 커밋 요약 (20:20 이후)

| 커밋 | 내용 |
|---|---|
| `0990484` | 사이드바 → 상단 중앙 플로팅 캡슐 네비 |
| `49bb67f` | 콘텐츠 최대 폭 1160 → 1300px |
| `3e9fbbc` | 모든 테이블 출처 100px 통일 |
| `5d9f7d2` | 출처 배지 nowrap(글자 잘림 해결) |
| `d81abf4` | 개인현황 목록만 폭 1040px |
| `e7f451a` | 행 배경색/글씨색 규칙 최종(배경=구분, 미적립만 글씨 연하게) |

*로컬에서 이어받은 뒤, 위 "찾을 문자열"로 각 위치를 열어 조정하시면 됩니다.*
