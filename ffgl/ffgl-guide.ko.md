---
title: Resolume 7 FFGL 플러그인 개발 가이드
version: FFGL 2.3 / Resolume 7.3+ (master 브랜치는 7.20+ 검증)
last_updated: 2026-04-30 (SDK 소스로 검증)
---

# FFGL 플러그인 개발 가이드

이 가이드는 Resolume Arena/Avenue 7용 **FFGL (FreeFrame GL) 플러그인** 작성을 다룸. FFGL은 Resolume이 네이티브 이펙트/소스/믹서에 사용하는 C++ / OpenGL 플러그인 포맷 — 고성능 경로. C++ 없이 셰이더만 짜고 싶다면 [ISF 가이드](../isf/isf-guide.ko.md) 참고.

---

## 1. 플러그인 종류

Resolume이 호스팅하는 FFGL 플러그인은 세 종류, 각각 SDK의 베이스 클래스가 대응:

| 종류 | C++ 베이스 | 입력 | 용도 |
|---|---|---|---|
| `FF_SOURCE` | `ffglqs::Source` | 0 | 제너레이터, 절차적 콘텐츠 (Gradients, Particles, Text) |
| `FF_EFFECT` | `ffglqs::Effect` | 1 | 클립에 적용되는 필터 (color, distort, blur) |
| `FF_MIXER` | `ffglqs::Mixer` | 2 | A/B 레이어 사이의 트랜지션 / 블렌드 모드 |

`CFFGLPluginInfo`(§4)에서 종류를 지정 — `ProcessOpenGL`이 받는 입력 텍스처 수를 결정.

---

## 2. SDK와 툴체인

공식 SDK: **`github.com/resolume/ffgl`**.

- **master 브랜치** — 최신 기능과 버그. **Resolume 7.3.1 이상** 필요.
- **`v2.2` 태그** — 가장 최근 안정 릴리스. 구버전 Resolume 호환이 중요하면 이쪽.

### 버전별 추가 사항

| FFGL | 필요 Resolume | 주요 추가 |
|---|---|---|
| 2.0 | 7.0.3+ | 초기 Resolume 7 지원, FF_TYPE_OPTION/FF_TYPE_BUFFER |
| 2.1 | 7.1+ | 임베디드 썸네일, GL 상태 RAII 바인딩, 파일 파라미터 |
| 2.2 | 7.2+ | 파라미터 그룹(7.3+), top-left 텍스처 오리엔테이션(7.3.1+), 호스트 로그 훅 |
| 2.3 | 7.4+ | 동적 디스플레이 이름, value-change 이벤트, 동적 옵션 요소(7.4.1+) |
| master | 7.3.1+ | glew 기반 OpenGL 4.6 (이전엔 glload) |

### 플랫폼 요구사항

- **macOS**: 최신 Xcode. **Universal (Apple Silicon + Intel) 빌드 필수** — Resolume 7.11+가 Apple Silicon에서 네이티브로 동작하므로 x86_64 전용 플러그인은 로드 안 됨.
- **Windows**: Visual Studio 2017 이상.
- **CMake**가 점진적으로 표준 빌드 시스템화되는 중. 기존 `build/osx/FFGLPlugins.xcodeproj`와 `build/windows/FFGLPlugins.sln`도 계속 유지됨.

---

## 3. 빠른 시작 — 예제 복제

가장 빠른 길은 예제를 복제하는 것.

```bash
git clone --recursive https://github.com/resolume/ffgl.git
cd ffgl
```

시작 템플릿 선택:

- **Effect** → `source/plugins/AddSubtract` 복사
- **Source** → `source/plugins/Gradients` 복사
- **Mixer** → `source/plugins/Add` 복사

### macOS

1. `build/osx/FFGLPlugins.xcodeproj` 열기.
2. 기존 타겟 우클릭 → **Duplicate** → 이름 변경.
3. **Build Settings → Packaging → Info.plist** — `FFGLPlugin-Info.plist`로 변경 (자동 생성된 `xx copy-Info.plist`는 제거).
4. Finder에서 `source/plugins/<YourName>` 아래 소스 폴더 복제, 파일명 변경.
5. 새 폴더를 Xcode로 드래그 → 새 타겟에만 추가.
6. 클래스 이름과 `PluginInfo` 필드 교체 (§4).
7. 빌드 (Cmd+B). 결과: `binaries/debug/<YourName>.bundle`.
8. `.bundle`을 `~/Documents/Resolume Arena/Extra Effects/`에 복사 후 재시작.

### Windows

1. `build/windows/`에서 `.vcxproj`와 `.vcxproj.filters` 둘 다 복제 후 이름 변경.
2. `FFGLPlugins.sln` 열기 → 솔루션 우클릭 → **Add → Existing Project** → 새 `.vcxproj` 추가.
3. 새 프로젝트의 소스 목록에서 원본 `.cpp`/`.h` 제거.
4. `source/plugins/<YourName>` 아래 소스 폴더 복제, 파일명 변경.
5. 새 소스를 Solution Explorer의 새 프로젝트 노드로 드래그.
6. F5로 바로 빌드하고 싶으면 프로젝트 우클릭 → **Set as Startup Project**.
7. 결과: `binaries/x64/Debug/<YourName>.dll`.
8. `%USERPROFILE%\Documents\Resolume Arena\Extra Effects\`에 복사 후 재시작.

> `glew` 링커 에러가 나오면 프로젝트 속성에 `deps/glew.props`를 추가.

---

## 4. `CFFGLPluginInfo` 선언

모든 플러그인 파일은 정적 `CFFGLPluginInfo` 하나를 선언. Resolume이 스캔 시 읽는 매니페스트.

```cpp
static CFFGLPluginInfo PluginInfo(
    PluginFactory< MyPlugin >,    // 팩토리 콜백
    "AB01",                       // 4자 고유 ID (전역 유일)
    "My Plugin",                  // Resolume에 표시되는 이름
    2,                            // FFGL API major
    1,                            // FFGL API minor
    1,                            // 플러그인 major
    0,                            // 플러그인 minor
    FF_EFFECT,                    // 종류: FF_EFFECT, FF_SOURCE, FF_MIXER
    "UI에 표시되는 설명",
    "Author / About 텍스트"
);
```

**중요:** **4자 고유 ID**는 컴포지션에 플러그인을 영속화할 때 사용. 두 플러그인이 같은 ID를 쓰면 하나가 조용히 숨겨짐. 자기만의 prefix(예: 이니셜)를 정해두고, 이미 배포된 플러그인의 ID는 절대 바꾸지 말 것.

Resolume 자체 예제는 `RE`(Resolume Effect), `RS`(Resolume Source), `RM`(Resolume Mixer) 같은 prefix를 씀 — 어떤 4자든 OK.

---

## 5. 플러그인 라이프사이클

최소 Effect 플러그인:

```cpp
#pragma once
#include <FFGLSDK.h>

class MyPlugin : public CFFGLPlugin
{
public:
    MyPlugin();

    FFResult InitGL( const FFGLViewportStruct* vp ) override;
    FFResult ProcessOpenGL( ProcessOpenGLStruct* pGL ) override;
    FFResult DeInitGL() override;

    FFResult SetFloatParameter( unsigned int idx, float value ) override;
    float    GetFloatParameter( unsigned int idx ) override;

private:
    ffglex::FFGLShader     shader;
    ffglex::FFGLScreenQuad quad;
    float                  brightness;
};
```

**호출 순서**

1. **Constructor** — `SetParamInfof(...)`로 파라미터 선언. 여기서 GL은 만지지 말 것.
2. **`InitGL`** — 셰이더 컴파일, GL 객체 생성. 클립에 들어갈 때 한 번 호출.
3. **`ProcessOpenGL`** — 매 프레임 호출. 입력 텍스처 바인딩, 유니폼 설정, 그리기.
4. **`DeInitGL`** — GL 자원 해제. 클립을 떠날 때 호출.

### `ProcessOpenGL` 규칙

- 호스트는 **GL 컨텍스트가 기본 상태로 복원**되길 기대. SDK의 RAII 헬퍼(`ScopedShaderBinding`, `Scoped2DTextureBinding`, `ScopedSamplerActivation`)를 써서 보장.
- Effect는 입력 텍스처가 `pGL->inputTextures[0]`. Mixer는 dest=`[0]`, src=`[1]`. Source는 입력 없음.
- 텍스처 좌표는 FFGL에서 **non-normalized**. 프레임마다 `GetMaxGLTexCoords(...)`로 활성 영역을 받아 유니폼으로 전달.
- **프리멀티플라이드 알파:** Resolume의 비디오 파이프라인은 프리멀티플라이드 색상으로 동작. RGB에 작용하는 이펙트라면 **언프리멀티플라이 → 적용 → 리프리멀티플라이** 패턴 필수 (`AddSubtract` 예제 참고).

```glsl
// 언프리멀티플라이
if( color.a > 0.0 ) color.rgb /= color.a;
// ... 이펙트 ...
// 리프리멀티플라이, 클램프
color.rgb = clamp( color.rgb * color.a, vec3(0.0), vec3(color.a) );
fragColor = color;
```

---

## 6. 파라미터 타입

생성자에서 `SetParamInfof(idx, "Name", FF_TYPE_*)`로 선언. Resolume이 자동으로 UI와 바인딩.

| 상수 | UI 표현 | 비고 |
|---|---|---|
| `FF_TYPE_STANDARD` | 슬라이더 0–1 | float 기본 |
| `FF_TYPE_BOOLEAN` | 토글 | 0 / 1 |
| `FF_TYPE_EVENT` | 버튼 | 모멘터리 — 1로 펄스 후 0 복귀 |
| `FF_TYPE_INTEGER` | 정수 슬라이더 | `SetParamRange()`로 min/max |
| `FF_TYPE_OPTION` | 드롭다운 | `SetParamElementInfo(idx, n, "Label", value)`로 항목 정의 |
| `FF_TYPE_TEXT` | 텍스트 입력 | `SetTextParameter`/`GetTextParameter` 사용 |
| `FF_TYPE_FILE` | 파일 선택 | 초기값은 FFGL 2.2 + Resolume 7.2 필요 |
| `FF_TYPE_BUFFER` | 내부용 | 호스트 제공 데이터(예: FFT) |

### 컬러 컴포넌트

단일 컬러 피커를 만들려면 **연속된 3개(RGB) 또는 4개(RGBA) 파라미터**를 다음 타입으로 선언:

| 상수 | 채널 |
|---|---|
| `FF_TYPE_RED` | R |
| `FF_TYPE_GREEN` | G |
| `FF_TYPE_BLUE` | B |
| `FF_TYPE_ALPHA` | A (HSB 다음에도 페어링) |
| `FF_TYPE_HUE` | H |
| `FF_TYPE_SATURATION` | S |
| `FF_TYPE_BRIGHTNESS` | B (Brightness, Blue 아님) |

Resolume이 연속된 RGB/RGBA, HSB/HSBA 블록을 자동으로 감지해 단일 컬러 피커로 표시.

### 위치 컴포넌트 (컬러와 별개)

위치 채널은 컬러 컴포넌트가 아님 — Resolume은 연속된 `XPOS`/`YPOS`를 2D 위치 피커로 묶음:

| 상수 | 채널 |
|---|---|
| `FF_TYPE_XPOS` | x 위치 |
| `FF_TYPE_YPOS` | y 위치 |

```cpp
// 생성자에서
SetParamInfof( PT_RED,   "Tint Red",   FF_TYPE_RED );
SetParamInfof( PT_GREEN, "Tint Green", FF_TYPE_GREEN );
SetParamInfof( PT_BLUE,  "Tint Blue",  FF_TYPE_BLUE );
// → 단일 RGB 컬러 피커로 표시됨
```

### 파라미터 그룹 (FFGL 2.2+ / Resolume 7.3+)

여러 파라미터를 접을 수 있는 섹션으로 묶음:

```cpp
SetParamGroup( PT_HUE, "Tint" );
SetParamGroup( PT_SAT, "Tint" );
SetParamGroup( PT_BRI, "Tint" );
```

---

## 7. `ffglquickstart` 단축 레이어

SDK에 포함된 `source/lib/ffglquickstart/`는 더 높은 추상화 — 선언한 파라미터로부터 프래그먼트 셰이더를 자동 생성하고 매 프레임 값을 전달. `ffglqs::Effect`, `ffglqs::Source`, `ffglqs::Mixer`를 상속:

```cpp
#include <FFGLSDK.h>

class MyEffect : public ffglqs::Effect
{
public:
    MyEffect()
    {
        AddParam( ffglqs::Param::Create( "Brightness", 0.5f ) );
    }
};
```

셰이더 쪽엔 같은 이름의 유니폼만 선언하면 됨 — `ffglquickstart`가 보일러플레이트 자동 주입.

완전한 제어가 필요하거나 비표준 레이아웃이면 그냥 `CFFGLPlugin`을 사용.

---

## 8. 유용한 SDK 유틸리티

`ffglex` (FFGLSDK.h로 자동 포함):

| 헬퍼 | 용도 |
|---|---|
| `FFGLShader` | vert+frag 컴파일, 유니폼 찾기, 값 설정 |
| `FFGLScreenQuad` | 위치/UV 포함된 풀스크린 쿼드 |
| `ScopedShaderBinding` | 셰이더 RAII 바인딩 |
| `Scoped2DTextureBinding` | 텍스처 RAII 바인딩 |
| `ScopedSamplerActivation` | 활성 텍스처 유닛 RAII |
| `FFGLLog::LogToHost` | Resolume 로그 파일에 메시지 (FFGL 2.2+) |
| `GetMaxGLTexCoords` | 입력 텍스처의 활성 s,t 영역 반환 |

`ffglqs` (quickstart):

| 헬퍼 | 용도 |
|---|---|
| `Param::Create` | 표준 0–1 파라미터 |
| `ParamRange::Create` | min/max 지정 가능 |
| `ParamOption::Create` | 드롭다운 |
| `ParamText::Create` | 텍스트 입력 |
| `ParamTrigger::Create` | 모멘터리 버튼 |
| `ParamFFT::Create` | 호스트 오디오 FFT 버퍼 수신 |

---

## 9. 오디오 리액티브

소리에 반응하려면 `FFT` 파라미터를 선언. 호스트가 주파수 도메인 데이터를 매 프레임 채워줌:

```cpp
auto fftParam = ffglqs::ParamFFT::Create( "Audio FFT", 16 );
AddParam( fftParam );
```

16개 빈이 호스트 오디오 분석에 매핑됨. `Update()`에서 빈 값을 읽어 셰이더에 `uniform float[16]` 등으로 전달.

---

## 10. Value-change 이벤트 (master 브랜치 / Resolume 7.4+)

플러그인이 자기 파라미터를 직접 바꾸고 호스트에 알림. 랜덤화 버튼이나 제너러티브 이펙트에 유용. 플러그인이 내부 값을 갱신한 뒤 `RaiseParamEvent`로 호스트에 알리면, 호스트가 `GetFloatParameter`로 다시 읽어감:

```cpp
brightness = newValue;
RaiseParamEvent( PT_BRIGHTNESS, FF_EVENT_FLAG_VALUE );
// Resolume이 파라미터를 다시 읽어 UI/OSC 구독자 업데이트
```

`PT_BRIGHTNESS`는 자체 정의한 파라미터 인덱스(`SetParamInfof`에 넘긴 enum 슬롯). `source/plugins/Events/FFGLEvents.cpp`에 randomize, 동적 옵션 리스트, display name 업데이트의 완전한 예제.

> 이 API는 `CFFGLPluginManager`에 있고 master 브랜치에서 사용 가능 (value 이벤트는 Resolume 7.4+, 동적 옵션은 7.4.1+). FFGL 2.3 자체가 추가한 건 동적 display name뿐.

---

## 11. 임베디드 썸네일 (FFGL 2.1+)

Source 플러그인은 **160×120 RGBA** 미리보기를 임베드 가능 — Resolume이 소스 피커에 표시. 별도 파일이 아니라 `CFFGLPluginInfo` 옆에 정적 `CFFGLThumbnailInfo`로 선언:

```cpp
static const unsigned char thumbPixels[ 160 * 120 * 4 ] = {
    /* 160x120 RGBA, row-major */
};

static CFFGLThumbnailInfo ThumbnailInfo( 160, 120, thumbPixels );
```

픽셀 데이터는 컴파일 시점에 실제 PNG로부터 생성 가능 (`CustomThumbnail` 예제는 이미지에서 헤더를 생성하는 한 가지 방법을 보여줌). 전체 패턴은 `source/plugins/CustomThumbnail/` 참고.

---

## 12. Top-left 텍스처 오리엔테이션 (FFGL 2.2+ / Resolume 7.3.1+)

Resolume은 내부적으로 bottom-left를 쓰지만 요청 시 flip. 베이스 생성자에 `true`를 넘기고 매 프레임 오리엔테이션을 조회하면 더블 flip을 피할 수 있음:

```cpp
class MyPlugin : public CFFGLPlugin
{
public:
    MyPlugin() : CFFGLPlugin( /*supportTopLeftTextureOrientation=*/ true ) {}

    FFResult ProcessOpenGL( ProcessOpenGLStruct* pGL ) override
    {
        bool topLeft = GetTextureOrientation() == TextureOrientation::TOP_LEFT;
        // UV 조정
    }
};
```

`TextureOrientation`은 `CFFGLPluginManager`의 scoped enum이며 값은 `TOP_LEFT`, `BOTTOM_LEFT`.

---

## 13. 설치 경로 (릴리스 빌드)

공식 FFGL SDK 빠른 시작이 가리키는 위치는 **제품명 없는 형태**:

| 플랫폼 | 경로 (SDK README 기준) |
|---|---|
| macOS | `~/Documents/Resolume/Extra Effects/` |
| Windows | `%USERPROFILE%\Documents\Resolume\Extra Effects\` |

반면 Resolume의 디렉토리 가이드는 사용자 파일을 **제품별**로 정리(`Resolume Arena` / `Resolume Avenue`):

| 플랫폼 | 제품별 경로 |
|---|---|
| macOS | `~/Documents/Resolume Arena/Extra Effects/` |
| Windows | `%USERPROFILE%\Documents\Resolume Arena\Extra Effects\` |

실제로는 대부분의 설치에서 두 형태 모두 스캔됨. 제품별 폴더는 Resolume이 첫 실행 시 만드는 폴더. 듀얼 호스트 플러그인은 제품별 폴더 사용 — 같은 `.bundle`/`.dll`을 `Resolume Arena/Extra Effects/`와 `Resolume Avenue/Extra Effects/` 양쪽에 복사.

> Resolume은 시작 시에만 스캔 — 복사 후 재시작 필수.

---

## 14. 디버깅

1. **`FFGLLog::LogToHost("...")`** — Resolume 로그(`~/Library/Logs/Resolume Arena/` 또는 `%LOCALAPPDATA%\Resolume Arena\Logs\`)에 출력. FFGL 2.2+.
2. **디버거 어태치** — Visual Studio: Debug → Attach to Process → `Arena.exe`. Xcode: Debug → Attach to Process.
3. **디버그 빌드 검증** — FFGL 2.2가 컨텍스트 상태 검증을 추가. GL 상태 복원 누락 시 SDK가 알려줌.
4. **개발 중엔 호출마다 `glGetError()`** — FFGL 자체는 호출하지 않음.

---

## 15. 자주 하는 실수

1. **플러그인이 안 보임** — 고유 ID 충돌, 잘못된 타겟 아키텍처(Apple Silicon 누락), Release 설치 경로에 Debug 빌드.
2. **검은 출력** — `Scoped*` 바인딩 누락으로 잘못된 텍스처가 활성. RAII 헬퍼 사용.
3. **두 번째 클립에서 크래시** — 자원을 `InitGL` 대신 생성자에서 할당. GL 할당은 모두 `InitGL`로.
4. **컬러가 이상함** — 셰이더에서 RGB 언프리멀티플라이/리프리멀티플라이 누락.
5. **리사이즈 시 UV 깨짐** — `MaxUV`를 `vec2(1.0)`으로 하드코딩. 프레임마다 `GetMaxGLTexCoords`로 가져와야 함.
6. **로드되는데 셰이더 컴파일 실패** — Resolume 로그 확인. GLSL 410 vs 330 불일치가 가장 흔함.

---

## 16. 배포

Resolume 공식 플러그인 마켓플레이스가 있음. 서드파티 배포 형식:

- 단일 `.bundle`(mac) 또는 `.dll`(win) + README zip.
- Resolume `.rfx`/`.rsx` 아카이브 (메타데이터 포함 zip).
- Mac은 Universal 빌드 (Apple Silicon + Intel).
- macOS 코드 사이닝 권장 (notarization은 필수는 아니지만 점점 기대됨).

---

## 출처

- [github.com/resolume/ffgl](https://github.com/resolume/ffgl) — 공식 FFGL SDK
- [github.com/resolume/ffgl/wiki](https://github.com/resolume/ffgl/wiki) — wiki: 튜토리얼과 API 노트
- `FFGL.h` 헤더 주석 — 각 FFGL 버전 추가사항의 정본
- [github.com/cyrilcode/ffglplugin-examples](https://github.com/cyrilcode/ffglplugin-examples) — 커뮤니티 예제
- [github.com/flyingrub/ffgl/tree/more](https://github.com/flyingrub/ffgl/tree/more) — 공식 README가 가리키는 추가 예제
