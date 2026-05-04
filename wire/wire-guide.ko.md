---
title: Resolume Wire — 패치 개발 가이드
version: Wire 7.26 (Arena/Avenue 7.4+ 필요)
last_updated: 2026-04-30
---

# Resolume Wire — 패치 개발 가이드

**Resolume Wire**는 노드 기반 비주얼 프로그래밍 환경. 결과물은 패치(patch)로, Resolume Arena/Avenue에 커스텀 **Source**, **Effect**, **Mixer**로 로드하거나 단독 실행 가능. [FFGL](../ffgl/ffgl-guide.ko.md)(C++)과 [ISF](../isf/isf-guide.ko.md)(셰이더 전용) 옆의 세 번째 저작 경로. Wire는 "로직 + 그래픽" 라우트 — 제너레이터, 수학, 모듈레이션, ISF 셰이더, OSC/MIDI 입력, 오디오 리액티브를 한 패치에서 C++ 없이 결합.

> Wire는 Arena/Avenue 인스톨러에 무료 트라이얼로 동봉. 워터마크 없는 배포 가능한 패치를 만들려면 Wire 라이선스(€399, Arena/Avenue와 별도) 필요. 단, Juicebar 라이선스 패치는 최종 사용자가 Wire 없이 실행 가능.

---

## 1. Wire가 만들어내는 것

Wire **패치**는 디스크에 두 형식 중 하나로 저장되는 자기 완결적 그래프:

| 확장자 | 편집 가능 | 용도 |
|---|---|---|
| `.wired` | ✅ | 소스 형식. Wire가 있는 사용자가 열어 수정 가능. |
| `.cwired` | ❌ | 컴파일/잠김. Arena/Avenue가 로드 (또는 Juicebar 키만 있으면 Wire 없이도). |

`~/Documents/Resolume Arena/Sources/`(또는 `Effects/`)에 떨어뜨린 패치는 ISF/FFGL 플러그인처럼 Resolume의 소스/이펙트 피커에 나타남. 패치의 **템플릿**(Source / Effect / Mixer)이 호스트에 어떻게 노출될지 결정.

---

## 2. 셋업과 버전 요구사항

| 항목 | 값 |
|---|---|
| 최소 Resolume | 7.4 |
| 최신 검증 | Wire 7.26 |
| 기본 설치 (mac) | `/Applications/Resolume Wire.app` |
| 기본 설치 (win) | `C:\Program Files\Resolume Wire\` |
| 사용자 패치 폴더 | `~/Documents/Resolume Wire/Patches/` |
| Wire 기본 OSC 포트 | **8081** (Arena/Avenue는 8080) |

Wire는 Arena/Avenue와 함께 설치됨. 단독 또는 비-Juicebar 배포에서 워터마크를 없애려면 Wire 라이선스 별도 구매.

---

## 3. 세 가지 노드 카테고리

| 카테고리 | 용도 | 예시 |
|---|---|---|
| **Input** | 외부 데이터를 패치로 가져오기 | `Texture Input`, `Trigger Input`, `Number Input`, `Color In`, `OSC In`, `MIDI In`, `FFT In` |
| **Output** | 패치 데이터를 외부로 보내기 | `Texture Output`, `OSC Out`, `MIDI Out` |
| **Processor** | 계산, 변형, 믹싱, 모듈레이션 | `Add`, `Mix`, `Sample Texture`, `ISF`, `Mesh Render`, `Text Render`, `Kaleidoscope`, `Gaussian Blur` |
| **Comment** | 문서 전용 — 인렛/아웃렛 없음 | `Comment` ("fill" 토글 끄면 섹션 구분선) |

Input 노드는 동시에 패치의 **파라미터** — Dashboard에 노출하면 Arena/Avenue 컨트롤 패널에 슬라이더/토글/컬러로 표시됨.

---

## 4. Flow 타입 — 세 가지 연결 스타일

모든 인렛/아웃렛에는 데이터가 언제 와이어를 건너는지를 결정하는 **flow 타입**이 있음. 노드의 모양으로 구분:

| 모양 | Flow | 전송 시점 |
|---|---|---|
| ⬤ 원 | **Signal** | 매 프레임 (프레임레이트 기반). 기본 — 거의 모든 게 시그널. |
| ▮ 사각 | **Event** | 사용자가 트리거할 때만. 프레임레이트와 무관. |
| ◆ 다이아몬드 | **Attribute** | 패치 (재)컴파일 시에만. 정적 설정용, 애니메이션 불가. |

**Attribute는 애니메이션 불가** — 그게 포인트. 런타임 값으로 구동하려면 먼저 시그널로 변환할 것.

---

## 5. 데이터 타입

Wire는 연결에 타입이 있음. 타입 불일치 와이어는 자동 변환되거나 연결 거부됨. 자주 쓰는 타입:

| 타입 | 메모 |
|---|---|
| `Number` | 단일 float |
| `Vec2`, `Vec3`, `Vec4` | 컴포넌트 벡터. `Make Vec2` / `Unpack`으로 패킹/언패킹 |
| `Color` | 1급 컬러 타입 (7.20에서 추가). `Color from RGB`, `Color from HSB`로 변환. 호환 이펙트는 직접 받음: Hue Rotate, Greyscale, AddSubtract, Bright.Contrast |
| `Texture` | GPU 텍스처 — Texture Input/Output이 운반 |
| `Array` | 값 리스트 |
| `Shape` | 패스 / 2D 셰이프 지오메트리 |
| `Trigger` | 이벤트 플로 시그널 |
| `String` | 텍스트 — Text Render와 OSC 주소에 사용 |

Wire는 데이터 타입별로 와이어 색상을 다르게 표시. 정확한 팔레트는 Preferences → Wire → Color Types에서 설정 (7.20에서 정비됨).

---

## 6. 템플릿: Source / Effect / Mixer

새 패치 만들 때 시작 화면에서 템플릿 선택. 템플릿이 결정하는 것:

- Arena/Avenue의 Sources / Effects 중 어디 표시될지
- 어떤 입출력 노드가 미리 배선될지
- 호스트가 제공할 텍스처 입력 수

| 템플릿 | 미리 배선된 입력 | Arena/Avenue 위치 |
|---|---|---|
| **Source** | 없음 | Sources 패널 (제너레이터) |
| **Effect** | `Texture Input` 1개 (아래 클립) | 클립/레이어/그룹의 이펙트 체인 |
| **Mixer** | 텍스처 입력 2개 (A와 B) | 레이어 믹서 / 트랜지션 |

**Compile for Resolume**을 누르면 결과 `.cwired`(또는 `.wired`)가 `~/Documents/Resolume Arena/Sources/`(또는 `Effects/`)에 생성되며 빌트인 소스처럼 트리거됨.

---

## 7. 파라미터와 Dashboard

Arena/Avenue에 노출할 파라미터는 **Dashboard** 탭에 배치. 각 Input 노드에 "show in dashboard" 토글이 있음. Dashboard 순서가 Arena/Avenue 파라미터 패널 순서와 일치.

**파라미터 그룹 (Wire 7.21+).** Dashboard 톱니바퀴 메뉴에서 입력 그룹 생성. Arena/Avenue에서 **접을 수 있는 섹션**으로 표시됨.

**프리셋.** **P** 아이콘으로 프리셋 상태 저장. 컴파일된 패치에 프리셋 번들됨. 최종 사용자는 Arena/Avenue 프리셋 리스트에서 봄. "Default" 슬롯은 저장된 패치 상태용으로 예약.

---

## 8. Wire 안의 ISF

**ISF 노드**로 ISF 프래그먼트 셰이더를 패치 안의 노드로 작성/로드. 셰이더의 `INPUTS`이 노드의 인렛으로 자동 노출 — float은 Number 인렛, color는 Color 인렛, image는 Texture 인렛.

> **Wire는 2D 전용.** ISF 버텍스 셰이더 사용 불가. `.fs`는 OK, 짝지은 `.vs`는 무시됨. 3D 메시 렌더링은 `Mesh Render` 등의 노드 사용.

JSON 메타데이터 형식과 빌트인 유니폼은 [ISF 가이드](../isf/isf-guide.ko.md) 참조.

---

## 9. OSC

Wire는 OSC를 완전 양방향 지원. 관련 노드:

| 노드 | 용도 |
|---|---|
| `OSC In` | 포트 UDP 리스너 열기 |
| `Read OSC` | 특정 주소 값 수신 |
| `OSC Out` | host:port로 UDP 송신 열기 |
| `Write OSC` | 주소로 값 송신 |

**셋업.** Wire → Preferences (Ctrl/Cmd + ,) → OSC에서 **OSC Input** 활성, 수신 포트 설정. Wire 기본 OSC 포트 **8081**. **Use Bundles** 토글로 메시지 패킹.

**자주 쓰는 패턴.**

- TouchOSC 페이더 수신: `OSC In :8081` → `Read OSC /1/fader1` → Number 시그널 구동.
- 패치 상태를 Ableton/외부 앱으로 송신: 시그널 → `Write OSC /myparam` → `OSC Out 192.168.1.10:9001`.

---

## 10. MIDI

MIDI도 동일 패턴: `MIDI In`, `Read MIDI Note`, `Read MIDI CC`, `MIDI Out`, `Write MIDI Note`, `Write MIDI CC`. Wire가 호스트로서 다른 DAW에 MIDI 송신도 가능.

---

## 11. 오디오 리액티브

플러그인으로 동작 중일 때 Wire가 호스트(Arena/Avenue)로부터 오디오 수신:

- **FFT 입력** — 주파수 도메인 빈, 배열로 전달.
- **오디오 샘플 입력** — 원시 파형.
- `Sample Texture`, `Average`, `Smooth` 등과 결합해 에너지/비트 추출.

단독 Wire는 시스템 오디오 디바이스를 탭해 테스트 가능.

---

## 12. 자주 쓰는 UI 단축키

| 동작 | 단축키 |
|---|---|
| 캔버스 패닝 | Space + 드래그, 또는 휠 버튼 |
| 줌 | 휠 스크롤, 또는 Ctrl/⌘ + `+` / `-` |
| 화면에 맞춤 | Ctrl/⌘ + `0` |
| 100% 줌 | Ctrl/⌘ + `1` |
| 모두 선택 | Ctrl/⌘ + `A` |
| 사용 안 한 노드 선택 | Ctrl/⌘ + Shift + `A` |
| 자동 정렬 | Ctrl/⌘ + `L` |
| 복제 | Ctrl/⌘ + `D` (또는 Alt + 드래그) |
| 이름 변경 | Ctrl/⌘ + Shift + `R` |
| 노드 검색 | Ctrl + F (win) / ⌘ + F (mac) |
| 복사 / 붙여넣기 | Ctrl/⌘ + `C` / `V` (자르기는 `X`) |
| 공유용 복사 | 우클릭 → 포럼에 붙여넣을 수 있는 텍스트로 출력 |
| 연결/노드 삭제 | Backspace 또는 Delete |

우클릭 **Insert Node Before / After Current** (7.24에서 추가)로 연결 자동 재배선 — 긴 체인 중간에 필터를 넣을 때 필수.

---

## 13. 패널 한눈에

| 패널 | 역할 |
|---|---|
| **Library** | 검색 가능한 노드 브라우저. 캔버스로 드래그하거나, 빈 캔버스를 더블클릭해 검색 팝업 열기 |
| **Monitor** | 선택된 노드 출력의 라이브 미리보기. 핀 고정 가능 |
| **Patch** | 패치 메타데이터: 이름, 설명, 카테고리, 해상도, 비트뎁스, 라이선스 정보 |
| **Node** | 선택 노드의 파라미터 (인렛 값, 인스턴스 수, 색상, 데이터타입) |
| **Resources** | 패치가 참조하는 외부 파일(이미지, ISF, 비디오). **Consolidate**로 패치 옆에 번들 |
| **Notes** | 자유 메모 패널 (7.24에서 추가) — 패치 내 문서화 |
| **Log** | 에러, 경고, `Print` 노드 출력 |
| **Dashboard** | 파라미터 순서/그룹화, Arena/Avenue 노출 토글 |
| **Patch Navigator** | 큰 패치를 미니맵 스타일로 빠르게 탐색 (7.24에서 추가) |
| **Stats** | 노드별 CPU/GPU 부하 — 이름/카테고리/부하/버전으로 정렬 |

---

## 14. 저장과 배포

```
File → Save                  # .wired 작성
File → Compile for Resolume  # .cwired (잠김) → Arena/Avenue의 Sources/Effects 폴더에 작성
File → Consolidate           # 패치 + 모든 리소스를 폴더로 번들
```

**컴파일 다이얼로그 옵션:**

- **Editable** — 켜면 Wire가 있는 수신자가 열어 수정 가능. 끄면 `.cwired` (잠김, 리소스 접근 불가).
- **Compile for Juicebar** — 라이선스 게이트 빌드 생성. 최종 사용자는 Juicebar 라이선스 구매; Wire는 라이선스가 유효하지 않으면 워터마크 주입.

**포럼 공유.** 노드 선택 → 우클릭 → **Copy For Sharing** → 결과 텍스트를 메시지에 붙여넣기. 수신자가 Wire에 붙여넣으면 노드가 재구성됨. 단, 외부 리소스(이미지, ISF 파일)는 Copy For Sharing에 **포함되지 않음** — 그래프만.

**Juicebar 마켓플레이스** [get-juicebar.com/wire](https://get-juicebar.com/wire) — Wire 패치 판매용 Resolume 공식 스토어프론트.

---

## 15. 버전 타임라인 (Wire 7.x 하이라이트)

| 버전 | 주요 추가 |
|---|---|
| **7.20** | 1급 Color 타입 도입. Color from RGB / HSB, Analogous / Complementary 팔레트 노드, Sample Texture 노드. Hardware Stats 패널 (FPS / GPU / CPU / RAM). Node Stats 패널 (노드별 부하). Ctrl+F 노드 검색. 수학: sqrt, radians, pi, phi, tau. 튜토리얼 섹션 정비. |
| **7.21** | Dashboard 파라미터 그룹화 — 그룹이 Arena/Avenue에서 접을 수 있는 섹션으로 표시. Transform Widget. |
| **7.23** | 파라미터 애니메이션 프리셋. 성능 개선. 새/개선된 Wire 노드. |
| **7.24** | Patch Navigator 패널, Notes 패널, Insert Node Before/After Current Node, 검색 팝업 미리보기. 새 노드: Kaleidoscope, Gaussian Blur, Cylindrical / Spherical Coordinate 도구, Stable Sort. |
| **7.26** | 10비트 컬러 출력, 속도 수정. |

---

## 16. 워크플로 팁

1. **입력 먼저, 출력 마지막.** `Texture Input`(Effect/Mixer)과 `Texture Output`을 먼저 깔고, 나머지를 사이에 끼움.
2. **`Comment` 노드를 적극 활용.** "fill" 끄면 라벨 달린 섹션 구분선이 됨 — 미래의 자신이 고마워함.
3. **`Print`가 디버거.** 시그널을 Print에 연결해 Log 패널에서 값 확인.
4. **Monitor에 핀을 박아라.** 핵심 노드를 핀 고정하면 선택과 무관하게 라이브 출력 확인 가능.
5. **Arena/Avenue에서 일찍 테스트.** 초안 `.cwired` 컴파일해 로드 — 호스트의 프레임레이트와 오디오 컨텍스트가 Wire 단독 미리보기와 다름.
6. **공유 전에 Consolidate.** 리소스를 패치 폴더에 복사하고 경로를 재작성해 협업자의 broken-link 보고를 방지.
7. **타입과 싸우지 말 것.** 와이어가 안 연결되면 monkey-patching 대신 변환 노드(`Make Vec2`, `Color from RGB`, `Texture from Color`)를 찾을 것.
8. **그룹은 일찍.** Dashboard 파라미터가 6개를 넘으면 그룹 생성 (Wire 7.21+) — 안 그러면 Arena/Avenue UI가 금방 어수선해짐.

---

## 17. 자주 하는 실수

1. **컴파일은 됐는데 Arena/Avenue에 안 보임.** 올바른 폴더에 컴파일됐는지 확인 (Source 템플릿이면 `Sources`, Effect면 `Effects`). 복사 후 Resolume 재시작 — 스캔은 시작 시에만.
2. **출력에 워터마크.** Wire 라이선스가 없거나, Juicebar로 배포하면서 Juicebar 컴파일을 활성화하지 않음.
3. **버텍스 셰이더 무시됨.** Wire의 ISF 노드는 프래그먼트 전용 (Wire는 2D). 3D 로직은 `Mesh Render` 노드로.
4. **OSC 메시지 안 옴.** 보통 세 가지: 방화벽(특히 Windows), 잘못된 포트(Wire 기본 8081, Arena 8080 아님), 또는 송신자가 `localhost`인데 Wire는 다른 머신에 있음. Wire의 OSC Input 로그로 패킷 도착 확인.
5. **Attribute 애니메이션 안 됨.** 의도된 동작 — attribute는 컴파일 타임 전용. signal-flow 입력 사용.
6. **파라미터가 Arena/Avenue에 안 보임.** Input 노드의 "show in dashboard" 토글 꺼져 있거나, 잘못된 제품 확인 중 (Source vs Effect 템플릿).
7. **다른 머신에서 리소스 누락.** 공유 전 **Consolidate** 안 함. File → Consolidate 실행 — 이미지/ISF 파일을 패치 폴더에 복사하고 경로 재작성.

---

## 18. Wire vs FFGL vs ISF — 언제 무엇을

| 필요 | 도구 |
|---|---|
| 프래그먼트 셰이더 하나, 추가 로직 없음 | ISF |
| 여러 프리미티브 + 수학 + 오디오 리액티브로 구성된 절차적 비주얼 | **Wire** |
| 커스텀 OpenGL 파이프라인 / 셰이더 외 CPU 작업 / 호스트 영속 상태 | FFGL (C++) |
| Wire 없는 최종 사용자에게 판매 | FFGL 또는 Juicebar 통한 `.cwired` |
| 빌드 시스템 없는 빠른 프로토타입 | ISF 또는 Wire (단일 셰이더 이상이면 Wire) |
| 손튜닝된 GL로 최대 성능 | FFGL |

Wire는 패치가 시스템일 때 강력 — 여러 소스 결합, 오디오 리액티브, 최종 사용자가 조작할 파라미터 그룹. 순수 셰이더 실험은 ISF에 머물러도 됨; 인스톨러 동봉용 프로덕션급 컴파일된 이펙트는 FFGL.

---

## 출처

- [resolume.com/software/wire](https://resolume.com/software/wire) — 제품 페이지
- [resolume.com/support/en/wire-introduction](https://resolume.com/support/en/wire-introduction) — 공식 소개
- [resolume.com/support/en/wire-node-anatomy](https://resolume.com/support/en/wire-node-anatomy) — 노드 구조
- [resolume.com/support/en/wire-data-flow](https://resolume.com/support/en/wire-data-flow) — flow 타입
- [resolume.com/support/en/wire-user-interface](https://resolume.com/support/en/wire-user-interface) — UI 레퍼런스
- [resolume.com/support/en/wire-osc](https://resolume.com/support/en/wire-osc) — OSC 통합
- [resolume.com/support/en/wire-saving-consolidating](https://resolume.com/support/en/wire-saving-consolidating) — 파일 형식과 배포
- [resolume.com/blog/24094](https://resolume.com/blog/24094) — 7.20 Color Types 릴리스
- [resolume.com/blog/24836](https://resolume.com/blog/24836) — 7.21 파라미터 그룹화
- [resolume.com/blog/30807](https://resolume.com/blog/30807) — 7.23 릴리스
- [get-juicebar.com/wire](https://get-juicebar.com/wire) — Wire 패치 마켓플레이스
- [github.com/YaNesyTortiK/WirePatches](https://github.com/YaNesyTortiK/WirePatches) — 커뮤니티 패치
