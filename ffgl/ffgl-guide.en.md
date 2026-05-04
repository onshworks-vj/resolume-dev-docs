---
title: FFGL Plugin Development Guide for Resolume 7
version: FFGL 2.3 / Resolume 7.3+ (master verified against 7.20+)
last_updated: 2026-04-30 (verified against SDK source)
---

# FFGL Plugin Development Guide

This guide covers writing **FFGL (FreeFrame GL) plugins** for Resolume Arena and Avenue 7. FFGL is the C++ / OpenGL plugin format Resolume uses for native effects, sources, and mixers — it's the high-performance route. For shader-only work without C++, see the [ISF guide](../isf/isf-guide.en.md) instead.

---

## 1. Plugin types

Resolume can host three FFGL plugin types, each backed by a base class in the SDK:

| Type | C++ base | Inputs | Use case |
|---|---|---|---|
| `FF_SOURCE` | `ffglqs::Source` | 0 | Generators, procedural content (Gradients, Particles, Text) |
| `FF_EFFECT` | `ffglqs::Effect` | 1 | Filters applied to a clip (color, distort, blur) |
| `FF_MIXER` | `ffglqs::Mixer` | 2 | Transitions / blend modes between the A and B layers |

The plugin type is set in `CFFGLPluginInfo` (see §4) and determines which inputs the host provides via `ProcessOpenGL`.

---

## 2. SDK and toolchain

The official SDK is **`github.com/resolume/ffgl`**.

- **Master branch** — latest features and bugs. Plugins built from master require **Resolume 7.3.1+**.
- **`v2.2` tag** — most recent stable release; safest if you target older Resolume installs.

### Versions and what they unlocked

| FFGL | Required Resolume | Notable additions |
|---|---|---|
| 2.0 | 7.0.3+ | Initial Resolume 7 support; FF_TYPE_OPTION, FF_TYPE_BUFFER |
| 2.1 | 7.1+ | Embedded thumbnails, RAII GL state bindings, file params |
| 2.2 | 7.2+ | Param groups (7.3+), top-left texture orientation (7.3.1+), host log hook |
| 2.3 | 7.4+ | Dynamic param display names, value-change events, dynamic option elements (7.4.1+) |
| master | 7.3.1+ | OpenGL 4.6 via glew (was glload) |

### Platform requirements

- **macOS**: Xcode (current); build **Universal (Apple Silicon + Intel)** because Resolume 7.11+ runs natively on Apple Silicon and won't load x86_64-only plugins.
- **Windows**: Visual Studio 2017 or newer.
- **CMake** is the build system going forward; `build/osx/FFGLPlugins.xcodeproj` and `build/windows/FFGLPlugins.sln` are still maintained.

---

## 3. Quickstart — duplicate an example

The fastest path to a working plugin is to clone an example.

```bash
git clone --recursive https://github.com/resolume/ffgl.git
cd ffgl
```

Pick a starting template:

- **Effect** → copy `source/plugins/AddSubtract`
- **Source** → copy `source/plugins/Gradients`
- **Mixer** → copy `source/plugins/Add`

### macOS

1. Open `build/osx/FFGLPlugins.xcodeproj`.
2. Right-click an existing target → **Duplicate**, rename it.
3. **Build Settings → Packaging → Info.plist** — change to `FFGLPlugin-Info.plist` (the auto-created `xx copy-Info.plist` is unwanted; trash it).
4. In Finder, duplicate the source folder under `source/plugins/<YourName>` and rename the `.cpp` / `.h` files.
5. Drag the new folder into Xcode and add to your new target only.
6. Replace class names and `PluginInfo` fields (§4).
7. Build (Cmd+B). The artifact is `binaries/debug/<YourName>.bundle`.
8. Copy the `.bundle` to `~/Documents/Resolume Arena/Extra Effects/` (or `Resolume Avenue`) and restart.

### Windows

1. Duplicate a `.vcxproj` and matching `.vcxproj.filters` in `build/windows/`. Rename both.
2. Open `FFGLPlugins.sln` → Solution → **Add → Existing Project** → your new `.vcxproj`.
3. Remove the original `.cpp`/`.h` from the new project's source list.
4. Duplicate the source folder under `source/plugins/<YourName>` and rename files.
5. Drag the new sources onto your new project node in the Solution Explorer.
6. Right-click your project → **Set as Startup Project** if you want F5 to build it directly.
7. The output is `binaries/x64/Debug/<YourName>.dll`.
8. Copy to `%USERPROFILE%\Documents\Resolume Arena\Extra Effects\` and restart.

> If `glew` linker errors appear, add `deps/glew.props` to your project's property pages.

---

## 4. The `CFFGLPluginInfo` declaration

Every plugin file declares a single static `CFFGLPluginInfo`. This is the manifest Resolume reads at scan time.

```cpp
static CFFGLPluginInfo PluginInfo(
    PluginFactory< MyPlugin >,    // Factory callback
    "AB01",                       // Unique 4-char ID (must be globally unique)
    "My Plugin",                  // Display name shown in Resolume
    2,                            // FFGL API major version
    1,                            // FFGL API minor version
    1,                            // Your plugin major version
    0,                            // Your plugin minor version
    FF_EFFECT,                    // Plugin type: FF_EFFECT, FF_SOURCE, or FF_MIXER
    "Description shown in UI",    // Description
    "Author / About text"         // About
);
```

**Critical:** the **4-character unique ID** is what Resolume uses to persist the plugin in compositions. If you reuse an ID across two plugins, one will silently shadow the other. Pick a prefix for yourself (e.g. your initials) and don't change IDs of released plugins.

Resolume's own examples use prefixes like `RE` (Resolume Effect), `RS` (Resolume Source), `RM` (Resolume Mixer) — feel free to use any 4 chars.

---

## 5. The plugin lifecycle

A minimal Effect plugin:

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

**Lifecycle order**

1. **Constructor** — declare parameters with `SetParamInfof(...)`. Don't touch GL here.
2. **`InitGL`** — compile shaders, create GL objects. Called once when the plugin enters a clip.
3. **`ProcessOpenGL`** — called every frame. Bind input textures, set uniforms, draw.
4. **`DeInitGL`** — release GL resources. Called when the plugin leaves the clip.

### `ProcessOpenGL` rules

- The host expects you to return the **GL context in its default state**. Use the SDK's RAII helpers (`ScopedShaderBinding`, `Scoped2DTextureBinding`, `ScopedSamplerActivation`) to guarantee that.
- For Effects, input texture is at `pGL->inputTextures[0]`. For Mixers, dest is `[0]` and source is `[1]`. For Sources, no inputs.
- Texture coordinates are **non-normalized** in FFGL. Use `GetMaxGLTexCoords(...)` to fetch the active region of the input texture and pass it as a uniform.
- **Premultiplied alpha:** Resolume's video pipeline operates on premultiplied colors. If your effect operates on RGB, **un-premultiply first, apply, then re-premultiply** (see `AddSubtract` example).

```glsl
// Un-premultiply
if( color.a > 0.0 ) color.rgb /= color.a;
// ... your effect ...
// Re-premultiply, clamp
color.rgb = clamp( color.rgb * color.a, vec3(0.0), vec3(color.a) );
fragColor = color;
```

---

## 6. Parameter types

Declared in the constructor with `SetParamInfof(idx, "Name", FF_TYPE_*)`. Resolume binds these to its UI automatically.

| Constant | UI shown | Notes |
|---|---|---|
| `FF_TYPE_STANDARD` | Slider 0–1 | Default for floats |
| `FF_TYPE_BOOLEAN` | Toggle | 0 / 1 |
| `FF_TYPE_EVENT` | Button | Momentary — value pulses to 1 then back to 0 |
| `FF_TYPE_INTEGER` | Integer slider | Use `SetParamRange()` to set min/max |
| `FF_TYPE_OPTION` | Dropdown | Define choices with `SetParamElementInfo(idx, n, "Label", value)` |
| `FF_TYPE_TEXT` | Text input | Use `SetTextParameter` / `GetTextParameter` |
| `FF_TYPE_FILE` | File picker | Initial value supported in FFGL 2.2 + Resolume 7.2 |
| `FF_TYPE_BUFFER` | Internal | For host-supplied data (e.g. FFT) |

### Color components

For a single color picker, declare a **group of 3 (RGB) or 4 (RGBA) consecutive parameters** with these types:

| Constant | Channel |
|---|---|
| `FF_TYPE_RED` | R |
| `FF_TYPE_GREEN` | G |
| `FF_TYPE_BLUE` | B |
| `FF_TYPE_ALPHA` | A (also pairs after HSB) |
| `FF_TYPE_HUE` | H |
| `FF_TYPE_SATURATION` | S |
| `FF_TYPE_BRIGHTNESS` | B (brightness, not blue) |

Resolume detects consecutive RGB/RGBA or HSB/HSBA blocks and shows them as a single color picker.

### Position components (separate from color)

Position channels are NOT color components — Resolume groups consecutive `XPOS`/`YPOS` as a 2D position picker:

| Constant | Channel |
|---|---|
| `FF_TYPE_XPOS` | x position |
| `FF_TYPE_YPOS` | y position |

```cpp
// In constructor
SetParamInfof( PT_RED,   "Tint Red",   FF_TYPE_RED );
SetParamInfof( PT_GREEN, "Tint Green", FF_TYPE_GREEN );
SetParamInfof( PT_BLUE,  "Tint Blue",  FF_TYPE_BLUE );
// → shown as one RGB color picker
```

### Param groups (FFGL 2.2+, Resolume 7.3+)

Group several params under a collapsable section:

```cpp
SetParamGroup( PT_HUE,   "Tint" );
SetParamGroup( PT_SAT,   "Tint" );
SetParamGroup( PT_BRI,   "Tint" );
```

---

## 7. The `ffglquickstart` shortcut layer

The SDK ships a higher-level wrapper at `source/lib/ffglquickstart/` that auto-builds a fragment shader from your declared params and ships values to it on each frame. Inherit from `ffglqs::Effect`, `ffglqs::Source`, or `ffglqs::Mixer`:

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

The shader you write only needs to declare matching uniforms — `ffglquickstart` auto-injects the boilerplate.

For full control or unusual layouts, drop down to plain `CFFGLPlugin`.

---

## 8. Useful SDK utilities

In `ffglex` (auto-included via `FFGLSDK.h`):

| Helper | Purpose |
|---|---|
| `FFGLShader` | Compile vert+frag, find uniforms, set values |
| `FFGLScreenQuad` | Pre-baked full-screen quad with positions and UVs |
| `ScopedShaderBinding` | RAII bind a shader; restores previous on scope exit |
| `Scoped2DTextureBinding` | RAII bind a texture |
| `ScopedSamplerActivation` | RAII active texture unit |
| `FFGLLog::LogToHost` | Send messages to Resolume's log file (FFGL 2.2+) |
| `GetMaxGLTexCoords` | Returns the active s,t region of an input texture |

In `ffglqs` (quickstart):

| Helper | Purpose |
|---|---|
| `Param::Create` | Standard 0–1 param |
| `ParamRange::Create` | Param with custom min/max |
| `ParamOption::Create` | Dropdown |
| `ParamText::Create` | Text input |
| `ParamTrigger::Create` | Momentary button |
| `ParamFFT::Create` | Receives audio FFT buffer from host |

---

## 9. Audio reactivity

To respond to sound, declare an `FFT` param. The host fills it with frequency-domain audio data each frame:

```cpp
// In constructor
auto fftParam = ffglqs::ParamFFT::Create( "Audio FFT", 16 );
AddParam( fftParam );
```

The 16 bins map to the host's audio analysis. Read the bin values in `Update()` and pass them to your shader as a `uniform float[16]` or similar.

---

## 10. Value-change events (master branch / Resolume 7.4+)

Plugins can change their own parameters and notify the host. Useful for randomize buttons or generative effects. The plugin updates its internal value, then signals the host with `RaiseParamEvent` so the host re-reads via `GetFloatParameter`:

```cpp
// After computing a new value internally
brightness = newValue;
RaiseParamEvent( PT_BRIGHTNESS, FF_EVENT_FLAG_VALUE );
// Resolume re-reads the param and updates the UI / OSC subscribers
```

`PT_BRIGHTNESS` here is your own parameter index (the same enum slot you passed to `SetParamInfof`). See `source/plugins/Events/FFGLEvents.cpp` for a complete example covering randomize, dynamic option lists, and display-name updates.

> This API lives on `CFFGLPluginManager` and is available on the master branch (Resolume 7.4+ for value events, 7.4.1+ for dynamic option elements). FFGL 2.3 specifically only added dynamic display names.

---

## 11. Embedded thumbnails (FFGL 2.1+)

Source plugins can embed a **160×120 RGBA** preview that Resolume shows in the source picker. Unlike a separate file, the thumbnail is declared as a static `CFFGLThumbnailInfo` next to your `CFFGLPluginInfo`:

```cpp
static const unsigned char thumbPixels[ 160 * 120 * 4 ] = {
    /* 160x120 RGBA, row-major */
};

static CFFGLThumbnailInfo ThumbnailInfo( 160, 120, thumbPixels );
```

The pixels can be generated at compile time from a real PNG (the `CustomThumbnail` example shows one approach using a header generated from an image). See `source/plugins/CustomThumbnail/` for the full pattern.

---

## 12. Top-left texture orientation (FFGL 2.2+ / Resolume 7.3.1+)

Resolume internally uses bottom-left orientation but can flip on demand. Opt your plugin into top-left and avoid double-flips by passing `true` to the base constructor and querying orientation each frame:

```cpp
class MyPlugin : public CFFGLPlugin
{
public:
    MyPlugin() : CFFGLPlugin( /*supportTopLeftTextureOrientation=*/ true ) {}

    FFResult ProcessOpenGL( ProcessOpenGLStruct* pGL ) override
    {
        bool topLeft = GetTextureOrientation() == TextureOrientation::TOP_LEFT;
        // adjust UVs accordingly
    }
};
```

`TextureOrientation` is a scoped enum on `CFFGLPluginManager` with values `TOP_LEFT` and `BOTTOM_LEFT`.

---

## 13. Installation paths (release builds)

The official FFGL SDK quickstart points to a **product-agnostic** location:

| Platform | Path (per SDK README) |
|---|---|
| macOS | `~/Documents/Resolume/Extra Effects/` |
| Windows | `%USERPROFILE%\Documents\Resolume\Extra Effects\` |

Resolume's own directory list, however, organizes user files **per product** (`Resolume Arena` / `Resolume Avenue`):

| Platform | Per-product location |
|---|---|
| macOS | `~/Documents/Resolume Arena/Extra Effects/` |
| Windows | `%USERPROFILE%\Documents\Resolume Arena\Extra Effects\` |

In practice both forms are scanned on most installs. The product-named folder is what Resolume creates the first time you launch it. Use that for dual-host plugins (drop the same `.bundle` / `.dll` into both `Resolume Arena/Extra Effects/` and `Resolume Avenue/Extra Effects/`).

> Resolume scans on launch; a restart is required after copying.

---

## 14. Debugging

1. **`FFGLLog::LogToHost("...")`** — appears in Resolume's log (`~/Library/Logs/Resolume Arena/` or `%LOCALAPPDATA%\Resolume Arena\Logs\`). FFGL 2.2+.
2. **Attach debugger** — Visual Studio: Debug → Attach to Process → `Arena.exe`. Xcode: Debug → Attach to Process.
3. **Validation in debug builds** — FFGL 2.2 added context-state validation. If you forget to restore GL state, the SDK will tell you what to fix.
4. **`glGetError()` after each call** during development — FFGL itself does not.

---

## 15. Common pitfalls

1. **Plugin doesn't appear** — wrong unique ID (collision), wrong target architecture (Apple Silicon), or built as Debug into a Release Resolume install path.
2. **Black output** — Often a missing `Scoped*` binding leaving the wrong texture active. Use the RAII helpers.
3. **Crash on second clip** — Resources allocated in the constructor instead of `InitGL`. Move all GL allocation into `InitGL`.
4. **Colors wrong** — Forgot to un-premultiply / re-premultiply RGB in the fragment shader.
5. **UVs wrong on resize** — Hardcoded `vec2(1.0)` for `MaxUV`. Fetch with `GetMaxGLTexCoords` each frame.
6. **Plugin loads but shaders don't compile** — Check Resolume log; you'll see GLSL compiler errors. Most common: GLSL 410 vs 330 mismatch.

---

## 16. Distribution

Resolume runs an official plugin marketplace. Third-party plugins are typically distributed as:

- A single `.bundle` (mac) or `.dll` (win) zipped with a README.
- Or a Resolume `.rfx` / `.rsx` archive (zip with metadata).
- Universal builds for Mac (Apple Silicon + Intel).
- Code signing recommended on macOS (notarization not strictly required for plugins but increasingly expected).

---

## Sources

- [github.com/resolume/ffgl](https://github.com/resolume/ffgl) — official FFGL SDK
- [github.com/resolume/ffgl/wiki](https://github.com/resolume/ffgl/wiki) — wiki: tutorials and API notes
- `FFGL.h` header — authoritative comments on each FFGL version's additions
- [github.com/cyrilcode/ffglplugin-examples](https://github.com/cyrilcode/ffglplugin-examples) — community examples
- [github.com/flyingrub/ffgl/tree/more](https://github.com/flyingrub/ffgl/tree/more) — extra examples linked from the official README
