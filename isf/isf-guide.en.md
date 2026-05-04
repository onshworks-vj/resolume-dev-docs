---
title: ISF Shader Guide for Resolume 7
version: ISF 2.0 / Resolume 7.8+
last_updated: 2026-04-30 (verified against ISF spec)
---

# ISF Shader Guide for Resolume

ISF (Interactive Shader Format) is a JSON-prefixed GLSL fragment-shader format that Resolume Avenue / Arena 7.8+ loads natively. **No C++ build, no SDK** — drop a `.fs` file into the right folder and Resolume picks it up. ISF is the right choice for visuals expressible as a fragment shader.

For native effects with audio access, persistent host state, or non-shader logic, use [FFGL](../ffgl/ffgl-guide.en.md) instead.

---

## 1. What an ISF file looks like

An ISF file is a single `.fs` file with a JSON metadata block at the top wrapped in GLSL multi-line comment delimiters, then GLSL fragment shader code:

```glsl
/*{
    "DESCRIPTION": "Tints the input image",
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

Three rules:

1. **The JSON block must come first** and be wrapped in `/*{ ... }*/`.
2. **Use `IMG_PIXEL` / `IMG_NORM_PIXEL`** instead of GLSL's `texture2D` / `texture` for input images.
3. **Write to `gl_FragColor`** as the final output (the spec is GLSL 1.10-style).

---

## 2. Where to put files

| Platform | Path |
|---|---|
| Resolume user folder | `~/Documents/Resolume Arena/Sources/` or `~/Documents/Resolume Arena/Effects/` |
| Windows equivalent | `%USERPROFILE%\Documents\Resolume Arena\Sources\` (or `Effects\`) |
| Cross-app ISF folder (mac) | `~/Library/Graphics/ISF/` |
| Cross-app ISF folder (win) | `%PROGRAMDATA%\ISF\` |

If you place an ISF in Resolume's `Sources/` folder, it appears as a Source. In `Effects/`, it appears as an Effect.

> **The cross-app `ISF` folders** (`~/Library/Graphics/ISF/` and `%PROGRAMDATA%\ISF\`) are the Vidvox/ISF community convention used by VDMX, MadMapper, and other ISF hosts. Resolume reads them in practice but does not document them as official scan paths — drop a copy into Resolume's per-product `Sources/Effects` folder if you want to be sure.

> **The convention is what makes it a Source vs an Effect.** Resolume looks at your `INPUTS` to decide:
> - No `inputImage` → **Source**
> - One `image` input named `inputImage` → **Effect**
> - Two image inputs named `startImage` and `endImage` plus a `progress` float → **Transition**

---

## 3. ISF JSON top-level fields

| Field | Required | Description |
|---|---|---|
| `ISFVSN` | yes | Spec version. Use `"2.0"` |
| `DESCRIPTION` | no | Human description shown in Resolume |
| `CREDIT` | no | Attribution |
| `CATEGORIES` | no | Array of strings, e.g. `["Color", "Stylize"]` |
| `INPUTS` | no | Array of input parameter objects (§4) |
| `PASSES` | no | Multi-pass rendering setup (§7) |
| `IMPORTED` | no | External images to load alongside the shader |
| `VSN` | no | Version of *this* shader file |

---

## 4. INPUTS — parameter types

Each entry in `INPUTS` requires `NAME` and `TYPE`. Most accept `LABEL`, `DEFAULT`, `MIN`, `MAX`, `IDENTITY`.

| TYPE | GLSL uniform type | UI in Resolume | DEFAULT format |
|---|---|---|---|
| `event` | `bool` | Momentary button | `false` |
| `bool` | `bool` | Toggle | `true` / `false` |
| `long` | `int` | Dropdown — needs `VALUES` and `LABELS` arrays | integer |
| `float` | `float` | Slider — uses `MIN`/`MAX` | number |
| `point2D` | `vec2` | XY pad — `MIN`/`MAX` are 2-element arrays | `[x, y]` |
| `color` | `vec4` | Color picker — `MIN`/`MAX`/`DEFAULT` are 4-element arrays | `[r, g, b, a]` |
| `image` | `sampler2D` | Image input | (n/a) |
| `audio` | `sampler2D` | Audio waveform packed as image | (n/a) |
| `audioFFT` | `sampler2D` | FFT-processed audio | (n/a) |

### Examples

```json
{ "NAME": "amount", "TYPE": "float", "MIN": 0.0, "MAX": 2.0, "DEFAULT": 1.0 }

{ "NAME": "center", "TYPE": "point2D",
  "MIN": [0.0, 0.0], "MAX": [1.0, 1.0], "DEFAULT": [0.5, 0.5] }

{ "NAME": "blendMode", "TYPE": "long",
  "VALUES": [0, 1, 2], "LABELS": ["Add", "Multiply", "Screen"], "DEFAULT": 0 }

{ "NAME": "tint", "TYPE": "color",
  "DEFAULT": [1.0, 1.0, 1.0, 1.0] }
```

Once declared, the input is available as a uniform in the shader by the name you gave it.

### Other input attributes

- **`LABEL`** — display name in the host's UI (NAME stays as the GLSL identifier).
- **`IDENTITY`** — the value that should be treated as "no effect" (used by hosts to bypass the parameter cleanly when the user resets it).
- **`MAX`** on `audio` / `audioFFT` — sets the desired number of audio samples or FFT bins. Hosts honor this as a hint:

```json
{ "NAME": "audioFFT", "TYPE": "audioFFT", "MAX": 64 }
```

---

## 5. Built-in uniforms

ISF declares these automatically — don't redeclare them:

| Name | Type | Meaning |
|---|---|---|
| `RENDERSIZE` | `vec2` | Output dimensions in pixels |
| `TIME` | `float` | Seconds since shader start |
| `TIMEDELTA` | `float` | Seconds since last frame (0 on first frame) |
| `FRAMEINDEX` | `int` | Frame counter starting at 0 |
| `DATE` | `vec4` | (year, month, day, seconds-of-day) |
| `PASSINDEX` | `int` | Current pass (0 if no PASSES) |
| `isf_FragNormCoord` | `vec2` | Current fragment in normalized 0..1 (origin bottom-left) |

---

## 6. Built-in functions for sampling

Don't use `texture2D`. Use these:

| Function | Signature | Purpose |
|---|---|---|
| `IMG_PIXEL` | `vec4 IMG_PIXEL(image, vec2 pixelCoord)` | Sample at integer pixel coords |
| `IMG_NORM_PIXEL` | `vec4 IMG_NORM_PIXEL(image, vec2 normCoord)` | Sample at 0..1 normalized coords |
| `IMG_THIS_PIXEL` | `vec4 IMG_THIS_PIXEL(image)` | Sample at the current fragment's pixel coord |
| `IMG_THIS_NORM_PIXEL` | `vec4 IMG_THIS_NORM_PIXEL(image)` | Sample at the current fragment's normalized coord |
| `IMG_SIZE` | `vec2 IMG_SIZE(image)` | Image dimensions in pixels |

```glsl
// Identity passthrough
gl_FragColor = IMG_THIS_NORM_PIXEL(inputImage);

// Read 10px to the left
vec2 here = gl_FragCoord.xy;
gl_FragColor = IMG_PIXEL(inputImage, here - vec2(10.0, 0.0));
```

---

## 7. PASSES — multi-pass shaders

For effects that need feedback or downsampling, declare `PASSES` to render into intermediate buffers.

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

| Field | Description |
|---|---|
| `TARGET` | Buffer name. Becomes a sampler2D you can read with `IMG_NORM_PIXEL` |
| `PERSISTENT` | If `true`, buffer survives between frames (feedback) |
| `FLOAT` | If `true`, 32-bit float storage (HDR / accumulation) |
| `WIDTH` | Optional expression. Supports `$WIDTH`, `$HEIGHT`, and **input names** as `$inputName` (e.g. `"$WIDTH * $scale"`) |
| `HEIGHT` | Optional expression — same rules as WIDTH |

In the shader, `PASSINDEX` tells you which pass you're in:

```glsl
void main() {
    if (PASSINDEX == 0) {
        // Write to "lastFrame" — store this frame for next time
        gl_FragColor = IMG_THIS_NORM_PIXEL(inputImage);
    } else {
        // Final pass: blend current with previous
        vec4 cur  = IMG_THIS_NORM_PIXEL(inputImage);
        vec4 prev = IMG_THIS_NORM_PIXEL(lastFrame);
        gl_FragColor = mix(prev, cur, 0.1); // motion trails
    }
}
```

The empty `{ }` final pass writes to the screen.

---

## 8. IMPORTED images

Bundle a static image alongside the shader (e.g. for LUTs or noise textures):

```json
"IMPORTED": {
    "noiseTex": { "PATH": "noise.png" }
}
```

`noiseTex` is then available as a sampler in the shader. Place `noise.png` next to your `.fs` file.

---

## 9. Effect / Source / Transition conventions

Resolume decides what your shader is by the inputs you declare:

### Effect (filter on a clip)

```json
"INPUTS": [
    { "NAME": "inputImage", "TYPE": "image" },
    { "NAME": "amount", "TYPE": "float", "DEFAULT": 0.5 }
]
```

### Source (generator, no input clip)

```json
"INPUTS": [
    { "NAME": "color", "TYPE": "color", "DEFAULT": [1, 1, 1, 1] }
]
```

### Transition (between two clips, A→B over time)

```json
"INPUTS": [
    { "NAME": "startImage", "TYPE": "image" },
    { "NAME": "endImage",   "TYPE": "image" },
    { "NAME": "progress",   "TYPE": "float", "MIN": 0.0, "MAX": 1.0, "DEFAULT": 0.0 }
]
```

---

## 10. Vertex shaders (rarely needed)

Default vertex shader is fine for 99% of fragment-only effects. If you do need one, name it the same as your `.fs` with a `.vs` extension. The first line of `main()` must be:

```glsl
void main() {
    isf_vertShaderInit();
    // ... your custom logic
}
```

Resolume Wire (the node-based patcher, actively developed in 7.26+) does **not** support vertex shaders — it operates in 2D only. Avenue/Arena fully support them.

---

## 11. Audio reactivity

Two input types receive audio data as textures:

```json
{ "NAME": "audio",    "TYPE": "audio" },
{ "NAME": "audioFFT", "TYPE": "audioFFT" }
```

Sample them like any image. The width of the texture is the number of samples (waveform) or frequency bins (FFT); the height is the channel count.

```glsl
// Average FFT amplitude across bins
float energy = 0.0;
int bins = int(IMG_SIZE(audioFFT).x);
for (int i = 0; i < bins; i++) {
    float u = float(i) / float(bins);
    energy += IMG_NORM_PIXEL(audioFFT, vec2(u, 0.5)).r;
}
energy /= float(bins);
```

---

## 12. Converting GLSL / Shadertoy to ISF

Common edits when porting:

| GLSL / Shadertoy | ISF |
|---|---|
| `mainImage(out vec4 fragColor, in vec2 fragCoord)` | `void main()` writing to `gl_FragColor` |
| `iResolution.xy` | `RENDERSIZE` |
| `iTime` | `TIME` |
| `iFrame` | `FRAMEINDEX` |
| `iChannel0` | declare an `image` input, sample with `IMG_NORM_PIXEL` |
| `texture(iChannel0, uv)` | `IMG_NORM_PIXEL(inputImage, uv)` |
| `fragCoord / iResolution.xy` | `isf_FragNormCoord` |

Final output should write `gl_FragColor.a = 1.0` (or the proper alpha) so Resolume's compositor doesn't show artifacts.

---

## 13. Workflow tips

1. **Resolume reloads on save** if the shader is open in a clip. Edit-save-see immediately.
2. **The ISF Editor** (https://editor.isf.video) renders ISF in-browser and is the fastest iteration loop. Tested shaders work in Resolume verbatim.
3. **Errors go to Resolume's log.** GLSL compile errors aren't shown in the UI; check `~/Library/Logs/Resolume Arena/` (mac) or `%LOCALAPPDATA%\Resolume Arena\Logs\` (win).
4. **Keep shaders pure-fragment** until you genuinely need a `.vs`. Most cool effects are fragment-only.
5. **Use `LABEL`** to give human-readable names — `NAME` becomes the GLSL identifier (no spaces), `LABEL` is what the user sees.

---

## 14. Common pitfalls

1. **`inputImage` not declared** → Resolume thinks your shader is a Source. Add an `image` input named exactly `inputImage` to make it an Effect.
2. **Naming clash with built-ins** — don't name an input `TIME`, `RENDERSIZE`, etc.
3. **Forgot `gl_FragColor.a = 1.0`** in a Source — Resolume's compositor may produce strange edges.
4. **JSON parse error** → entire shader silently fails to load. Validate JSON; trailing commas are not allowed.
5. **Persistent buffer not surviving** → check `PERSISTENT: true` and that you're rendering to it every frame.
6. **Wrong coordinate origin** — `isf_FragNormCoord` is **bottom-left origin**. If you ported from Shadertoy (top-left), `y` is flipped.

---

## 15. Minimal templates

### Source: animated gradient

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

### Effect: pixelate

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

### Transition: dissolve

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

## Sources

- [docs.isf.video](https://docs.isf.video) — official ISF documentation
- [docs.isf.video/quickstart.html](https://docs.isf.video/quickstart.html) — quickstart guide
- [docs.isf.video/ref_json.html](https://docs.isf.video/ref_json.html) — JSON reference
- [docs.isf.video/ref_variables.html](https://docs.isf.video/ref_variables.html) — built-in variables
- [docs.isf.video/ref_functions.html](https://docs.isf.video/ref_functions.html) — helper functions
- [docs.isf.video/ref_converting.html](https://docs.isf.video/ref_converting.html) — converting from raw GLSL
- [github.com/mrRay/ISF_Spec](https://github.com/mrRay/ISF_Spec) — spec source
- [editor.isf.video](https://editor.isf.video) — browser editor
- [resolume.com/support/en/isf](https://resolume.com/support/en/isf) — Resolume's ISF support page
