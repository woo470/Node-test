# PNP 접합 시뮬레이터 — 작업 인수인계 보고서

작성일: 2026-07-09
작성 목적: 다음 작업자(AI 포함)가 맥락 없이도 프로젝트 현황을 파악하고 이어서 작업할 수 있도록 함

---

## 1. 프로젝트 개요

반도체 P-N 접합/트랜지스터(PNP, NPN) 동작 원리를 배우는 3D 인터랙티브 교육용 웹 시뮬레이터. Three.js 기반 3D 그래픽으로 P형/N형 반도체 블록, 전극(+/-), 도선을 직접 배치해 회로를 구성하고 전류 흐름 여부를 시각적으로 확인하는 방식.

### 파일 구조

```
C:\Users\Owner\Desktop\
├─ pnp 접합 시뮬레이터 (2)\                     ← 프로젝트 루트
│  ├─ pnp 접합 시뮬레이터\                       ← 실제 사이트 폴더 (여기서 index.html을 열어야 함)
│  │  ├─ index.html                          ← 소개/랜딩 페이지
│  │  ├─ pn3.html                            ← 3D 시뮬레이터 워크스페이스 (v10)
│  │  ├─ 사진에셋.png                          ← 랜딩 페이지 히어로 이미지 (기존 자산)
│  │  └─ screenshots\                        ← 실제 연결 사진 4종 (아래 3장 참고)
│  │     ├─ pn_junction.png
│  │     ├─ np_junction.png
│  │     ├─ pnp_junction.png
│  │     └─ npn_junction.png
│  └─ screenshots\                           ← 위와 동일한 사진의 사본 (작업 중 생성된 부산물, 사이트에서 참조 안 함)
├─ pnp 시뮬레이터 백업 (디자인 변경 전)\             ← 블루프린트 스타일 시도 전 원본 백업
│  ├─ index.html
│  └─ pn3.html
└─ PNP_접합_시뮬레이터_인수인계_보고서.md            ← 이 문서
```

**중요**: 브라우저에서 열 때는 반드시 `pnp 접합 시뮬레이터 (2)\pnp 접합 시뮬레이터\index.html` 경로여야 함. 바깥쪽 `screenshots` 폴더는 사이트가 참조하지 않는 사본이므로, 사진을 새로 찍었을 때는 반드시 **안쪽** `pnp 접합 시뮬레이터\screenshots\`에도 복사해야 실제 사이트에 반영됨 (이 문제로 한 번 혼선이 있었음, 4-3절 참고).

---

## 2. 사용 기술 스택

| 용도 | 기술 |
|---|---|
| 3D 렌더링 | Three.js r0.160.0 (CDN import map, `three`, `three/addons/OrbitControls`, `RoundedBoxGeometry`, `EffectComposer`, `UnrealBloomPass`) |
| 회로 판정 로직 | 순수 JavaScript, BFS(너비 우선 탐색) 기반 그래프 탐색 |
| 폰트 | Google Fonts `Inter`, `Pretendard` |
| 브라우저 자동화(캡처용) | Python `playwright` (sync API), 시스템에 설치된 실제 Chrome 브라우저를 `channel="chrome"`으로 구동 (playwright 번들 Chromium은 사내망 SSL 인터셉션으로 다운로드 실패하여 우회) |
| 이미지 후처리 | Python `Pillow`(PIL) — 스크린샷 크롭 + 2배 업스케일 |
| 셸 | Windows PowerShell |

Playwright/Pillow는 `pip install playwright`, `pip install pillow`로 로컬 Python(3.13, `C:\Users\Owner\AppData\Local\Programs\Python\Python313`)에 설치됨. `playwright install chromium`은 네트워크 인증서 문제(`SELF_SIGNED_CERT_IN_CHAIN`, 사내 보안 에이전트로 추정 — 바탕화면에 Genian NAC 아이콘 존재)로 실패했고, 대신 이미 설치되어 있던 `C:\Program Files\Google\Chrome\Application\chrome.exe`를 Playwright가 직접 구동하도록 `channel="chrome"` 옵션을 사용해 우회함.

---

## 3. `pn3.html` 핵심 로직 요약 (다음 작업자가 알아야 할 것)

### 3-1. 판정 알고리즘 (`checkStrictPhysics`, 약 492번째 줄 부근)

- 씬에 배치된 모든 오브젝트(P, N, PLUS, MINUS, WIRE)를 순회하며, 모든 PLUS↔MINUS 쌍에 대해 BFS로 경로 탐색
- 인접 판정: `GRID_SIZE(2)` 이내 유클리드 거리
- 경로를 따라가며 P/N 타입을 만날 때마다 연속 중복은 무시하고 시퀀스 문자열을 누적 (`P,N`, `N,P,N` 등)
- `VALID` 배열에 있는 시퀀스로 PLUS→MINUS 도달 시에만 "전류 흐름"으로 판정

```js
// 490번째 줄
const VALID = ['P,N', 'P,N,P', 'N,P,N'];
```

- **원래 코드는 `'N,P'`도 VALID에 포함되어 있었음** — 이번 작업에서 제거함 (4-2절 참고)
- `isValidPrefix()`는 여전히 `'N,P'`를 NPN(`N,P,N`)으로 가는 중간 경로로는 허용함 (탐색이 막히지 않도록). 즉 `N,P`가 **최종 성공 상태로는 안 되지만, N,P,N의 중간 단계로는 그대로 통과**되는 구조 — 이 둘을 혼동해서 다시 `N,P`를 VALID에 넣으면 원래 버그가 재발하니 주의.

### 3-2. 성공 HUD 분기 (약 555번째 줄 `updateLogicState`)

```js
if (physicsResult.type === 'P,N') { title = 'PN 접합 성공'; ... }
else if (physicsResult.type === 'P,N,P') { title = 'PNP 접합 성공'; ... }
else { title = 'NPN 접합 성공'; ... }  // N,P,N만 여기 도달 가능 (N,P는 애초에 physicsResult가 안 나옴)
```

`N,P` 분기는 삭제됨 (죽은 코드였음).

### 3-3. UI 조작 방식 (자동화/재현용 참고)

- 좌측 사이드바에서 부품 타입 클릭(`selectType(type, el)`, `window`에 노출됨) → 고스트 메시가 마우스를 따라다님
- 3D 바닥 평면에 클릭하면 `spawnReal(type, pos)`가 그 위치에 실제 오브젝트 생성 후 자동으로 선택 해제
- 도선(WIRE)도 동일하게 "사이드바에서 WIRE 선택 → 원하는 칸 클릭"을 반복하면 개별 배치 가능 (초록색 화살표 드래그 기능도 있지만, 자동화 스크립트에서는 안 씀 — 매번 사이드바를 다시 클릭해 `hideArrows()`를 강제로 태워서 화살표 히트테스트를 피함)
- `window.resetScene()`으로 전체 초기화 가능

---

## 4. 이번 세션에서 수행한 작업 (시간순)

### 4-1. 접합 유형 확인 및 실제 연결 사진 캡처

요청: "코드에 접합 종류가 몇 개인지", "실제로 연결한 사진을 찍어서 사이트에 올리고 싶다"

**방법**: Python + Playwright로 실제 Chrome을 띄워 `pn3.html`을 열고, 사이드바 버튼 클릭과 3D 바닥 클릭을 좌표 기반으로 자동화해서 4가지 회로(PN, NP, PNP, NPN)를 직접 조립함. 3D 카메라(원근 투영, FOV 50°, 카메라 위치 `(d,d,d)` where `d = distance/√3`, 타겟 원점)를 수학적으로 역산해 월드 좌표 → 스크린 좌표 변환 함수를 직접 구현했고, 디버그 마커(초록 점)를 격자 교차점에 겹쳐 그려서 투영 공식이 정확한지 먼저 검증한 뒤 진행함.

핵심 투영 공식 (재사용 가능):
```python
import math
FOV_DEG = 50
F = 1 / math.tan(math.radians(FOV_DEG) / 2)
FORWARD = (-1/math.sqrt(3),)*3
RIGHT = (1/math.sqrt(2), 0, -1/math.sqrt(2))
CAMUP = (-0.40824829, 0.81649658, -0.40824829)

def make_projector(distance, vw, vh):
    d = distance / math.sqrt(3)
    cam = (d, d, d)
    aspect = vw / vh
    def world_to_screen(x, z, y=0):
        vx, vy, vz = x - cam[0], y - cam[1], z - cam[2]
        x_view = vx*RIGHT[0] + vy*RIGHT[1] + vz*RIGHT[2]
        y_view = vx*CAMUP[0] + vy*CAMUP[1] + vz*CAMUP[2]
        z_view = -(vx*FORWARD[0] + vy*FORWARD[1] + vz*FORWARD[2])
        ndc_x = (x_view / -z_view) * F / aspect
        ndc_y = (y_view / -z_view) * F
        return (ndc_x + 1) / 2 * vw, (1 - ndc_y) / 2 * vh
    return world_to_screen
```
(카메라 거리는 시뮬레이터의 3단계 줌 레벨 `[8, 20, 35]` 중 하나와 일치시켜야 함. 페이지 로드 후 카메라가 초기 위치에서 해당 거리로 서서히 줌되는 애니메이션이 있으므로 약 2.5초 대기 후 좌표 계산해야 함.)

각 회로 배치 좌표 (GRID_SIZE=2 간격, 일직선 배치):
- PN: `PLUS(-6,0) → WIRE(-4,0) → P(-2,0) → N(0,0) → WIRE(2,0) → MINUS(4,0)`
- NP: `PLUS(-6,0) → WIRE(-4,0) → N(-2,0) → P(0,0) → WIRE(2,0) → MINUS(4,0)`
- PNP: `PLUS(-6,0) → WIRE(-4,0) → P(-2,0) → N(0,0) → P(2,0) → WIRE(4,0) → MINUS(6,0)`
- NPN: `PLUS(-6,0) → WIRE(-4,0) → N(-2,0) → P(0,0) → N(2,0) → WIRE(4,0) → MINUS(6,0)`

각 회로 조립 후 성공 HUD가 실제로 뜨는지(`#success-hud`에 `active` 클래스, `.hud-title` 텍스트 일치) 확인 → 전체 화면 스크린샷 → 회로 부분만 바운딩 박스로 크롭 후 2배 업스케일(Pillow `LANCZOS`) → `screenshots/` 폴더에 저장.

### 4-2. NP 접합 물리 버그 발견 및 수정

사용자 친구가 "NP가 뭔가 이상하다"고 지적. 확인 결과 **원본 코드가 실제 반도체 물리와 모순**되어 있었음:

- 원본: `NP`(역방향 바이어스, +전극이 N형에 연결)인데도 `PN`과 동일하게 "성공, 전류 흐름"으로 표시
- 실제 물리: 역방향 바이어스는 공핍층이 넓어져 전류가 **차단**되어야 정상
- **+/-를 바꾸면 NP도 성공하지 않냐**는 질문이 있었으나, +/-를 바꾸면 그 순간 코드상 시퀀스가 `P,N`(즉 "PN")으로 바뀌는 것이므로 "NP인데 성공하는" 상태는 물리적으로 존재할 수 없음을 확인·설명함.

**수정 내용** (`pn3.html`):
1. `const VALID = ['P,N', 'N,P', 'P,N,P', 'N,P,N'];` → `const VALID = ['P,N', 'P,N,P', 'N,P,N'];` (`'N,P'` 제거)
2. `updateLogicState()` 안의 `else if (physicsResult.type === 'N,P') { ... 'NP 접합 성공' ... }` 죽은 분기 삭제
3. NP 스크린샷을 "전류 차단" 상태(스파크 없음, HUD 안 뜸)로 재캡처
4. `index.html`의 NP 카드 설명 문구를 "역방향 바이어스 — 전류 차단"으로 정정, 태그 색상도 보라색(성공 느낌) → 회색(차단 느낌)으로 변경

이 과정에서 **바깥쪽 `screenshots` 폴더에만 새 사진을 저장하고 실제 사이트가 참조하는 안쪽 `pnp 접합 시뮬레이터\screenshots` 폴더에 복사를 빠뜨린 실수**가 있었음 → 사용자가 지적해서 바로 동기화함. **향후 스크린샷을 다시 찍을 때는 반드시 두 위치(또는 최소한 사이트 폴더 안쪽)에 저장할 것.**

### 4-3. `index.html` 콘텐츠 개편

- 기존 SVG 일러스트 기반 "접합 유형 소개"(PN/NP/PNP 3종, `Junction Types` 배지) 섹션 **전체 삭제**
- 새 섹션 "실제 연결 사진"(`Simulator Capture` 배지) 추가: 실제 캡처 사진 4장 + 각 회로의 연결 순서·결과를 설명하는 짧은 문구, 4열 그리드(`photo-grid` 클래스, 1200px 이하에서 2열로 반응형)
- "v9.0 Release" 뱃지 문구 삭제 (최종 요청)

### 4-4. 디자인 변경 시도 및 최종 확정

1. **1차 시도 — 블루프린트/회로도 스타일**: 밝은 종이색 배경 + 모눈종이 격자 + 잉크 네이비 텍스트 + 레드 포인트, 모노스페이스 폰트(IBM Plex Mono). `index.html`, `pn3.html` UI(사이드바/HUD/컨텍스트메뉴) 전부 적용 → **사용자가 "안 어울린다"며 반려** → `pnp 시뮬레이터 백업 (디자인 변경 전)\` 백업본으로 즉시 원복.
2. **2차 시도 — 미니멀 다크 플랫 스타일 (현재 최종 상태)**: 어두운 톤은 유지하되 그라데이션·블러(`backdrop-filter`)·네온 글로우(`box-shadow`/`text-shadow` glow)를 전부 제거하고 단색 위주로 재설계. **사용자 승인 완료, 현재 적용된 최종 디자인.**

#### 최종 디자인 스펙 (미니멀 다크)

```css
--bg: #0b0d12;           /* 배경 (단일 색상, 그라데이션 없음) */
--bg-alt: #14171d;       /* 카드/사이드바/패널 배경 */
--text-primary: #f4f4f5;
--text-secondary: #9299a3;
--border: #26292f;       /* 모든 테두리 공통 */
--accent-color: #00e5a0; /* 민트그린 — 버튼/뱃지/로고점/HUD 테두리 등 유일한 포인트 컬러 */
```

- 버튼: 단색 배경(`--accent-color`) + hover 시 밝은 민트로 전환, 그라데이션 스윕 없음
- 카드: `1px solid var(--border)`, hover 시 테두리만 `--accent-color`로 전환 + 살짝 위로 이동, box-shadow 글로우 없음
- 태그(PN/NP/PNP/NPN): 배경 없이 색상 테두리 + 같은 색 텍스트만 사용 (PN=파랑 `#4db8ff`, NP=회색 `--text-secondary`, PNP=주황 `#ff9d4d`, NPN=민트 `--accent-color`)
- `pn3.html`도 동일 팔레트로 사이드바/홈버튼/컨텍스트메뉴/성공 HUD 통일. **3D 씬 자체(블록 재질, 파티클, 조명, bloom 포스트프로세싱)는 손대지 않음** — 이건 UI 디자인이 아니라 시뮬레이터 기능이므로 의도적으로 유지.

---

## 5. 백업 위치

| 백업 대상 | 위치 |
|---|---|
| 블루프린트 스타일 적용 **전** 원본 (그라데이션 버전) | `C:\Users\Owner\Desktop\pnp 시뮬레이터 백업 (디자인 변경 전)\index.html`, `pn3.html` |

주의: 이 백업은 "그라데이션 원본"이며, 현재 사이트에 적용된 "미니멀 다크" 버전과는 다름. 미니멀 다크 버전은 별도로 백업해두지 않았으므로, 만약 이 상태를 보존하고 싶다면 지금 시점에 별도 백업을 만들어두는 것을 권장함.

---

## 6. 향후 작업 시 주의사항 (다음 작업자용 체크리스트)

- [ ] 스크린샷을 다시 찍어야 할 경우, 반드시 **`pnp 접합 시뮬레이터\screenshots\`**(안쪽, 사이트가 실제 참조하는 경로)에 저장할 것. 바깥쪽 `screenshots` 폴더는 사이트와 무관한 사본임.
- [ ] `pn3.html`의 `VALID` 배열에 다시 `'N,P'`를 추가하면 NP 역방향 바이어스 버그가 재발함 — 절대 추가하지 말 것.
- [ ] Playwright로 3D 시뮬레이터를 재현/자동화할 때는 카메라가 페이지 로드 후 목표 거리로 서서히 줌되는 애니메이션이 있으므로 약 2~2.5초 대기 후 좌표 계산할 것.
- [ ] `pn3.html`의 UI 오버레이(사이드바/HUD 등)와 Three.js 3D 렌더링 로직은 완전히 분리되어 있음 — 디자인만 바꿀 때는 `<style>` 블록만 건드리면 되고 `<script type="module">` 내부(회로 판정/렌더링 로직)는 그대로 둘 것.
- [ ] 현재 적용된 "미니멀 다크" 디자인이 마음에 들면 별도로 백업본을 만들어 둘 것 (현재 유일한 백업은 그 이전 그라데이션 버전뿐).
