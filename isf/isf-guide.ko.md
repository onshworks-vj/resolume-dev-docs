---
title: Resolume 7 ISF 셰이더 가이드
version: ISF 2.0 / Resolume 7.8+
last_updated: 2026-04-30 (ISF 스펙으로 검증)
---

# Resolume용 ISF 셰이더 가이드

ISF (Interactive Shader Format)는 JSON 메타데이터가 앞에 붙은 GLSL 프래그먼트 셰이더 포맷. Resolume Avenue/Arena 7.8+가 네이티브로 로드. **C++ 빌드, SDK 불필요** — `.fs` 파일을 적절한 폴더에 떨어뜨리면 Resolume이 자동 인식. 프래그먼트 셰이더로 표현 가능한 비주얼이라면 ISF가 정답.

오디오 액세스, 호스트 영속 상태, 셰이더 외 로직이 필요하면 [FFGL](../ffgl/ffgl-guide.ko.md) 사용.

---

## 1. ISF 파일 형태

ISF는 단일 `.fs` 파일 — 맨 위에 JSON 메타데이터(GLSL 다중행 주석으로 감쌈), 그 아래에 GLSL 프래그먼트 셰이더 코드:

```glsl
/*{
    "DESCRIPTION": "입력 이미지에 색조 입힘",
    "CREDIT": "your name",
    "ISFVSN": "2.0",
    "CATEGORIES": ["Color"],
    "INPUTS": [
        { "NAME": "inputImage", "TYPE": "image" },
        { "NAME": "tint", "TYPE": "color", "DEFAULT": [1.0, 0.5, 0.5, 1.0] }
    ]
}*/

void main() {
    vec4 src = IMG_THIS_PIXEL(inputImage);
    gl_FragColor = src * tint;
}
```

세 가지 규칙:

1. **JSON 블록은 반드시 맨 앞**, `/*{ ... }*/`로 감쌈.
2. **`IMG_PIXEL` / `IMG_NORM_PIXEL` 사용** — GLSL의 `texture2D` / `texture` 대신.
3. **출력은 `gl_FragColor`** (스펙은 GLSL 1.10 스타일).

---

## 2. 파일 위치

| 플랫폼 | 경로 |
|---|---|
| Resolume 사용자 폴더 | `~/Documents/Resolume Arena/Sources/` 또는 `~/Documents/Resolume Arena/Effects/` |
| Windows 등가 | `%USERPROFILE%\Documents\Resolume Arena\Sources\` (또는 `Effects\`) |
| 크로스앱 ISF 폴더 (mac) | `~/Library/Graphics/ISF/` |
| 크로스앱 ISF 폴더 (win) | `%PROGRAMDATA%\ISF\` |

Resolume의 `Sources/`에 두면 Source로, `Effects/`에 두면 Effect로 표시.

> **크로스앱 `ISF` 폴더**(`~/Library/Graphics/ISF/`, `%PROGRAMDATA%\ISF\`)는 Vidvox/ISF 커뮤니티 컨벤션 — VDMX, MadMapper 등 다른 ISF 호스트가 공유. Resolume도 실제로 읽지만 공식 스캔 경로로 문서화하지는 않음. 확실하게 하려면 Resolume의 제품별 `Sources/Effects` 폴더에 복사본을 둘 것.

> **Source인지 Effect인지는 INPUTS이 결정.** Resolume은 INPUTS를 보고 판단:
> - `inputImage` 없음 → **Source**
> - `inputImage`라는 이름의 `image` 입력 1개 → **Effect**
> - `startImage`와 `endImage` 이미지 입력 2개 + `progress` float → **Transition**

---

## 3. ISF JSON 최상위 필드

| 필드 | 필수 | 설명 |
|---|---|---|
| `ISFVSN` | 예 | 스펙 버전. `"2.0"` 사용 |
| `DESCRIPTION` | 아니오 | Resolume에 표시되는 설명 |
| `CREDIT` | 아니오 | 저작자 |
| `CATEGORIES` | 아니오 | 문자열 배열, 예: `["Color", "Stylize"]` |
| `INPUTS` | 아니오 | 입력 파라미터 객체 배열 (§4) |
| `PASSES` | 아니오 | 다중 패스 렌더링 (§7) |
| `IMPORTED` | 아니오 | 셰이더와 함께 로드할 외부 이미지 |
| `VSN` | 아니오 | 이 셰이더 파일 자체의 버전 |

---

## 4. INPUTS — 파라미터 타입

`INPUTS`의 각 항목은 `NAME`과 `TYPE` 필수. 대부분 `LABEL`, `DEFAULT`, `MIN`, `MAX`, `IDENTITY`도 지원.

| TYPE | GLSL 유니폼 | Resolume UI | DEFAULT 형식 |
|---|---|---|---|
| `event` | `bool` | 모멘터리 버튼 | `false` |
| `bool` | `bool` | 토글 | `true` / `false` |
| `long` | `int` | 드롭다운 — `VALUES`와 `LABELS` 배열 필요 | 정수 |
| `float` | `float` | 슬라이더 — `MIN`/`MAX` 사용 | 숫자 |
| `point2D` | `vec2` | XY 패드 — `MIN`/`MAX`는 2-원소 배열 | `[x, y]` |
| `color` | `vec4` | 컬러 피커 — `MIN`/`MAX`/`DEFAULT`는 4-원소 배열 | `[r, g, b, a]` |
| `image` | `sampler2D` | 이미지 입력 | (없음) |
| `audio` | `sampler2D` | 오디오 파형(이미지로 패킹) | (없음) |
| `audioFFT` | `sampler2D` | FFT 처리된 오디오 | (없음) |

### 예시

```json
{ "NAME": "amount", "TYPE": "float", "MIN": 0.0, "MAX": 2.0, "DEFAULT": 1.0 }

{ "NAME": "center", "TYPE": "point2D",
  "MIN": [0.0, 0.0], "MAX": [1.0, 1.0], "DEFAULT": [0.5, 0.5] }

{ "NAME": "blendMode", "TYPE": "long",
  "VALUES": [0, 1, 2], "LABELS": ["Add", "Multiply", "Screen"], "DEFAULT": 0 }

{ "NAME": "tint", "TYPE": "color",
  "DEFAULT": [1.0, 1.0, 1.0, 1.0] }
```

선언하면 셰이더에서 그 이름의 유니폼으로 사용 가능.

### 기타 입력 속성

- **`LABEL`** — 호스트 UI에 표시되는 이름 (NAME은 GLSL 식별자로 유지).
- **`IDENTITY`** — "이펙트 없음"으로 간주할 값. 사용자가 리셋했을 때 호스트가 깔끔히 바이패스하는 데 사용.
- **`MAX`** on `audio` / `audioFFT` — 오디오 샘플 수 또는 FFT 빈 수의 힌트:

```json
{ "NAME": "audioFFT", "TYPE": "audioFFT", "MAX": 64 }
```

---

## 5. 자동 선언되는 유니폼

ISF가 자동 선언 — 다시 선언하지 말 것:

| 이름 | 타입 | 의미 |
|---|---|---|
| `RENDERSIZE` | `vec2` | 출력 픽셀 크기 |
| `TIME` | `float` | 셰이더 시작 후 경과 초 |
| `TIMEDELTA` | `float` | 직전 프레임 이후 경과 초 (첫 프레임 0) |
| `FRAMEINDEX` | `int` | 프레임 카운터, 0부터 |
| `DATE` | `vec4` | (년, 월, 일, 그날의 초) |
| `PASSINDEX` | `int` | 현재 패스 (PASSES 없으면 0) |
| `isf_FragNormCoord` | `vec2` | 현재 프래그먼트의 정규화 좌표 0..1 (원점: 좌하단) |

---

## 6. 샘플링 함수

`texture2D` 쓰지 말고 다음을 사용:

| 함수 | 시그니처 | 용도 |
|---|---|---|
| `IMG_PIXEL` | `vec4 IMG_PIXEL(image, vec2 pixelCoord)` | 정수 픽셀 좌표로 샘플 |
| `IMG_NORM_PIXEL` | `vec4 IMG_NORM_PIXEL(image, vec2 normCoord)` | 0..1 정규화 좌표로 샘플 |
| `IMG_THIS_PIXEL` | `vec4 IMG_THIS_PIXEL(image)` | 현재 프래그먼트의 픽셀 좌표로 샘플 |
| `IMG_THIS_NORM_PIXEL` | `vec4 IMG_THIS_NORM_PIXEL(image)` | 현재 프래그먼트의 정규화 좌표로 샘플 |
| `IMG_SIZE` | `vec2 IMG_SIZE(image)` | 이미지의 픽셀 크기 |

```glsl
// 그대로 통과
gl_FragColor = IMG_THIS_NORM_PIXEL(inputImage);

// 왼쪽 10px 샘플
vec2 here = gl_FragCoord.xy;
gl_FragColor = IMG_PIXEL(inputImage, here - vec2(10.0, 0.0));
```

---

## 7. PASSES — 다중 패스 셰이더

피드백이나 다운샘플링이 필요한 이펙트엔 `PASSES`로 중간 버퍼에 렌더:

```json
"PASSES": [
    {
        "TARGET": "lastFrame",
        "PERSISTENT": true,
        "FLOAT": true
    },
    { }
]
```

| 필드 | 설명 |
|---|---|
| `TARGET` | 버퍼 이름. `IMG_NORM_PIXEL`로 읽을 수 있는 sampler2D가 됨 |
| `PERSISTENT` | `true`면 프레임 간 버퍼 유지 (피드백) |
| `FLOAT` | `true`면 32비트 float 저장 (HDR / 누적) |
| `WIDTH` | 옵션 표현식. `$WIDTH`, `$HEIGHT`, 그리고 **입력 이름**을 `$inputName`으로 참조 가능 (예: `"$WIDTH * $scale"`) |
| `HEIGHT` | 옵션 — WIDTH와 동일 규칙 |

셰이더 안에서 `PASSINDEX`로 현재 패스 확인:

```glsl
void main() {
    if (PASSINDEX == 0) {
        // "lastFrame"에 기록 — 다음 프레임을 위해 저장
        gl_FragColor = IMG_THIS_NORM_PIXEL(inputImage);
    } else {
        // 최종 패스: 현재와 이전 합성
        vec4 cur  = IMG_THIS_NORM_PIXEL(inputImage);
        vec4 prev = IMG_THIS_NORM_PIXEL(lastFrame);
        gl_FragColor = mix(prev, cur, 0.1); // 모션 트레일
    }
}
```

빈 마지막 패스 `{ }`는 화면에 출력.

---

## 8. IMPORTED 이미지

LUT나 노이즈 텍스처 같은 정적 이미지 번들:

```json
"IMPORTED": {
    "noiseTex": { "PATH": "noise.png" }
}
```

`noiseTex`가 sampler로 사용 가능. `noise.png`를 `.fs` 파일 옆에 둘 것.

---

## 9. Effect / Source / Transition 컨벤션

Resolume은 INPUTS로 셰이더의 종류를 결정:

### Effect (클립에 적용되는 필터)

```json
"INPUTS": [
    { "NAME": "inputImage", "TYPE": "image" },
    { "NAME": "amount", "TYPE": "float", "DEFAULT": 0.5 }
]
```

### Source (제너레이터, 입력 클립 없음)

```json
"INPUTS": [
    { "NAME": "color", "TYPE": "color", "DEFAULT": [1, 1, 1, 1] }
]
```

### Transition (두 클립 사이, A→B)

```json
"INPUTS": [
    { "NAME": "startImage", "TYPE": "image" },
    { "NAME": "endImage",   "TYPE": "image" },
    { "NAME": "progress",   "TYPE": "float", "MIN": 0.0, "MAX": 1.0, "DEFAULT": 0.0 }
]
```

---

## 10. 버텍스 셰이더 (드물게 필요)

99%의 프래그먼트 전용 이펙트는 기본 버텍스 셰이더로 충분. 정말 필요하면 `.fs`와 같은 이름에 `.vs` 확장자. `main()` 첫 줄은 반드시:

```glsl
void main() {
    isf_vertShaderInit();
    // ... 커스텀 로직
}
```

Resolume Wire(노드 기반 patcher, 7.26+에서 활발히 개발 중)는 2D만 다루기 때문에 버텍스 셰이더 미지원. Avenue/Arena는 완전 지원.

---

## 11. 오디오 리액티브

오디오 데이터를 텍스처로 받는 두 가지 입력 타입:

```json
{ "NAME": "audio",    "TYPE": "audio" },
{ "NAME": "audioFFT", "TYPE": "audioFFT" }
```

이미지처럼 샘플. 텍스처 너비 = 샘플 수(파형) 또는 주파수 빈 수(FFT). 높이는 채널 수.

```glsl
// 빈 평균 FFT 진폭
float energy = 0.0;
int bins = int(IMG_SIZE(audioFFT).x);
for (int i = 0; i < bins; i++) {
    float u = float(i) / float(bins);
    energy += IMG_NORM_PIXEL(audioFFT, vec2(u, 0.5)).r;
}
energy /= float(bins);
```

---

## 12. GLSL / Shadertoy → ISF 포팅

자주 바꿔야 하는 것:

| GLSL / Shadertoy | ISF |
|---|---|
| `mainImage(out vec4 fragColor, in vec2 fragCoord)` | `void main()` + `gl_FragColor` |
| `iResolution.xy` | `RENDERSIZE` |
| `iTime` | `TIME` |
| `iFrame` | `FRAMEINDEX` |
| `iChannel0` | `image` 입력 선언, `IMG_NORM_PIXEL`로 샘플 |
| `texture(iChannel0, uv)` | `IMG_NORM_PIXEL(inputImage, uv)` |
| `fragCoord / iResolution.xy` | `isf_FragNormCoord` |

최종 출력은 `gl_FragColor.a = 1.0` (또는 적절한 알파)를 명시 — Resolume 컴포지터의 아티팩트 방지.

---

## 13. 워크플로 팁

1. **저장하면 Resolume이 리로드** — 클립이 셰이더를 사용 중이면 즉시 반영. 편집-저장-확인 루프가 빠름.
2. **ISF 에디터**(https://editor.isf.video) — 브라우저에서 ISF를 즉시 렌더. 가장 빠른 반복 환경. 여기서 동작하면 Resolume에서도 그대로 동작.
3. **에러는 Resolume 로그**. UI엔 GLSL 컴파일 에러 안 표시. `~/Library/Logs/Resolume Arena/` (mac) 또는 `%LOCALAPPDATA%\Resolume Arena\Logs\` (win) 확인.
4. **`.vs`가 정말 필요하기 전엔 프래그먼트 전용**. 멋진 이펙트 대부분이 프래그먼트만으로 가능.
5. **`LABEL` 활용** — `NAME`은 GLSL 식별자(공백 불가), `LABEL`이 사용자에게 보이는 이름.

---

## 14. 자주 하는 실수

1. **`inputImage` 미선언** → Resolume이 Source로 인식. Effect로 만들려면 정확히 `inputImage`라는 이름의 `image` 입력 추가.
2. **빌트인 이름과 충돌** — 입력 이름을 `TIME`, `RENDERSIZE` 등으로 짓지 말 것.
3. **Source인데 `gl_FragColor.a = 1.0` 누락** → 합성 시 가장자리 이상.
4. **JSON 파싱 에러** → 셰이더가 조용히 로드 실패. JSON 검증 필수, 후행 쉼표 금지.
5. **퍼시스턴트 버퍼 안 살아남음** → `PERSISTENT: true` 확인, 매 프레임 그 버퍼로 렌더하는지도 확인.
6. **좌표 원점 혼동** — `isf_FragNormCoord`는 **좌하단 원점**. Shadertoy(좌상단)에서 포팅했다면 `y` 뒤집힘.

---

## 15. 최소 템플릿

### Source: 애니메이션 그라디언트

```glsl
/*{
    "DESCRIPTION": "Time-warped gradient",
    "ISFVSN": "2.0",
    "CATEGORIES": ["Generator"],
    "INPUTS": [
        { "NAME": "speed", "TYPE": "float", "MIN": 0.0, "MAX": 5.0, "DEFAULT": 1.0 }
    ]
}*/
void main() {
    vec2 uv = isf_FragNormCoord;
    float t = TIME * speed;
    gl_FragColor = vec4(
        0.5 + 0.5 * sin(uv.x * 6.0 + t),
        0.5 + 0.5 * sin(uv.y * 6.0 + t * 1.3),
        0.5 + 0.5 * sin((uv.x + uv.y) * 4.0 + t * 0.7),
        1.0
    );
}
```

### Effect: 픽셀화

```glsl
/*{
    "DESCRIPTION": "Pixelate",
    "ISFVSN": "2.0",
    "CATEGORIES": ["Stylize"],
    "INPUTS": [
        { "NAME": "inputImage", "TYPE": "image" },
        { "NAME": "size", "TYPE": "float", "MIN": 1.0, "MAX": 64.0, "DEFAULT": 8.0 }
    ]
}*/
void main() {
    vec2 grid = floor(gl_FragCoord.xy / size) * size + vec2(size * 0.5);
    gl_FragColor = IMG_PIXEL(inputImage, grid);
}
```

### Transition: 디졸브

```glsl
/*{
    "DESCRIPTION": "Crossfade",
    "ISFVSN": "2.0",
    "CATEGORIES": ["Transition"],
    "INPUTS": [
        { "NAME": "startImage", "TYPE": "image" },
        { "NAME": "endImage",   "TYPE": "image" },
        { "NAME": "progress",   "TYPE": "float", "MIN": 0.0, "MAX": 1.0, "DEFAULT": 0.0 }
    ]
}*/
void main() {
    vec4 a = IMG_THIS_NORM_PIXEL(startImage);
    vec4 b = IMG_THIS_NORM_PIXEL(endImage);
    gl_FragColor = mix(a, b, progress);
}
```

---

## 출처

- [docs.isf.video](https://docs.isf.video) — 공식 ISF 문서
- [docs.isf.video/quickstart.html](https://docs.isf.video/quickstart.html) — 빠른 시작
- [docs.isf.video/ref_json.html](https://docs.isf.video/ref_json.html) — JSON 레퍼런스
- [docs.isf.video/ref_variables.html](https://docs.isf.video/ref_variables.html) — 빌트인 변수
- [docs.isf.video/ref_functions.html](https://docs.isf.video/ref_functions.html) — 헬퍼 함수
- [docs.isf.video/ref_converting.html](https://docs.isf.video/ref_converting.html) — 일반 GLSL 포팅
- [github.com/mrRay/ISF_Spec](https://github.com/mrRay/ISF_Spec) — 스펙 원본
- [editor.isf.video](https://editor.isf.video) — 브라우저 에디터
- [resolume.com/support/en/isf](https://resolume.com/support/en/isf) — Resolume ISF 지원 페이지
