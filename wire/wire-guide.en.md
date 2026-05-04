---
title: Resolume Wire — Patch Development Guide
version: Wire 7.26 (requires Arena/Avenue 7.4+)
last_updated: 2026-04-30
---

# Resolume Wire — Patch Development Guide

**Resolume Wire** is a node-based visual programming environment that produces patches you can load in Resolume Arena and Avenue as custom **Sources**, **Effects**, or **Mixers** — or run standalone. It's the third authoring path alongside [FFGL](../ffgl/ffgl-guide.en.md) (C++) and [ISF](../isf/isf-guide.en.md) (shader-only). Wire is the "logic + graphics" route: combine generators, math, modulation, ISF shaders, OSC/MIDI input, and audio reactivity in a single patch without writing C++.

> Wire is bundled with the Arena/Avenue installers for free trial use. Producing watermark-free, distributable patches requires a Wire license (€399 separate from Arena/Avenue) — except for licensed Juicebar patches, which their end users can run without owning Wire.

---

## 1. What Wire produces

A Wire **patch** is a self-contained graph saved to disk in one of two formats:

| Extension | Editable? | Use |
|---|---|---|
| `.wired` | ✅ | Source format. Users with Wire can open and edit. |
| `.cwired` | ❌ | Compiled / locked. Loaded by Arena/Avenue (or end-users without a Wire license, if a Juicebar key is provided). |

Patches dropped into `~/Documents/Resolume Arena/Sources/` (or `Effects/`) appear in Resolume's source/effect picker just like ISF and FFGL plugins. The patch's **template** (Source / Effect / Mixer) determines how it's exposed to the host.

---

## 2. Setup and version requirements

| Item | Value |
|---|---|
| Minimum Resolume | 7.4 |
| Latest tested | Wire 7.26 |
| Default install (mac) | `/Applications/Resolume Wire.app` |
| Default install (win) | `C:\Program Files\Resolume Wire\` |
| User patches folder | `~/Documents/Resolume Wire/Patches/` |
| Default OSC port (Wire) | **8081** (Arena/Avenue use 8080) |

Wire installs alongside Arena and Avenue. License Wire separately if you need to remove the watermark on standalone or non-Juicebar distributions.

---

## 3. The three node categories

| Category | Purpose | Examples |
|---|---|---|
| **Input** | Bring data into the patch from outside | `Texture Input`, `Trigger Input`, `Number Input`, `Color In`, `OSC In`, `MIDI In`, `FFT In` |
| **Output** | Send data out of the patch | `Texture Output`, `OSC Out`, `MIDI Out` |
| **Processor** | Compute, transform, mix, modulate | `Add`, `Mix`, `Sample Texture`, `ISF`, `Mesh Render`, `Text Render`, `Kaleidoscope`, `Gaussian Blur` |
| **Comment** | Documentation only — no inlets or outlets | `Comment` (toggle "fill" off for section dividers) |

Input nodes also become the patch's **parameters** — anything you mark for the dashboard appears as a slider/toggle/color in Arena/Avenue's controls panel.

---

## 4. Flow types — three connection styles

Every inlet and outlet has a **flow type** that governs when data crosses the wire. The shape on the node tells you which:

| Shape | Flow | When it transmits |
|---|---|---|
| ⬤ circle | **Signal** | Every frame (frame-rate driven). The default — almost everything is a signal. |
| ▮ rectangle | **Event** | Only when the user triggers the source node. Independent of frame rate. |
| ◆ diamond | **Attribute** | Only when the patch (re)compiles. Used for static configuration, not animatable. |

You **cannot animate Attributes** — that's the point. Convert an attribute to a signal first if you need to drive it from runtime values.

---

## 5. Data types

Wire types its connections. Type-mismatched wires either auto-convert or refuse to connect. Common types:

| Type | Notes |
|---|---|
| `Number` | Single float |
| `Vec2`, `Vec3`, `Vec4` | Component vectors. Pack/unpack with `Make Vec2` / `Unpack` |
| `Color` | First-class color (added in 7.20). Convert with `Color from RGB`, `Color from HSB`. Compatible effects accept it directly: Hue Rotate, Greyscale, AddSubtract, Bright.Contrast |
| `Texture` | GPU texture — what Texture Input/Output ferry around |
| `Array` | List of values |
| `Shape` | Path / 2D shape geometry |
| `Trigger` | Event-flow signal |
| `String` | Text — used by Text Render and OSC addresses |

Wire colors connection wires by data type. The exact palette is set in Preferences → Wire → Color Types and was overhauled in 7.20.

---

## 6. Templates: Source / Effect / Mixer

When you create a new patch, pick a template from the welcome screen. The template determines:

- Whether the patch shows up under Arena/Avenue's Sources or Effects section
- What input/output nodes are pre-wired
- The texture inputs the host will provide

| Template | Pre-wired inputs | Where it appears in Arena/Avenue |
|---|---|---|
| **Source** | None | Sources panel (generators) |
| **Effect** | One `Texture Input` (the clip below) | Effect chain on a clip/layer/group |
| **Mixer** | Two texture inputs (A and B) | Layer mixer / transition |

After pressing **Compile for Resolume**, the resulting `.cwired` (or `.wired`) appears in `~/Documents/Resolume Arena/Sources/` (or `Effects/`) and can be triggered like any built-in source.

---

## 7. Parameters & the Dashboard

Parameters that Arena/Avenue should expose go on the **Dashboard** tab. Each Input node has a "show in dashboard" toggle. Order in the dashboard matches order in Arena/Avenue's parameters panel.

**Parameter groups (Wire 7.21+).** From the Dashboard's cogwheel menu, create input groups. Groups appear as **foldable sections** in Arena/Avenue.

**Presets.** Use the **P** icon to save preset states. Presets bundle into compiled patches. End users see them in Arena/Avenue's preset list. The "Default" slot is reserved for the saved patch state.

---

## 8. ISF inside Wire

The **ISF node** lets you write or load an ISF fragment shader as a node in your patch. The shader's `INPUTS` automatically appear as inlets on the node — float becomes a Number inlet, color becomes a Color inlet, image becomes a Texture inlet.

> **Wire is 2D-only.** You cannot use ISF vertex shaders here. The `.fs` file is fine; a paired `.vs` will be ignored. For 3D mesh rendering use the `Mesh Render` and related nodes.

See the [ISF guide](../isf/isf-guide.en.md) for the JSON metadata format and built-in uniforms.

---

## 9. OSC

Wire is fully bidirectional on OSC. The relevant nodes:

| Node | Purpose |
|---|---|
| `OSC In` | Open a UDP listener on a port |
| `Read OSC` | Receive a value from a specific address |
| `OSC Out` | Open a UDP sender to a host:port |
| `Write OSC` | Send a value to an address |

**Setup.** In Wire → Preferences (Ctrl/Cmd + ,) → OSC, enable **OSC Input** and set the listening port. Default Wire OSC port is **8081**. Toggle **Use Bundles** to pack messages.

**Common patterns.**

- Receive a TouchOSC fader: `OSC In :8081` → `Read OSC /1/fader1` → drives a Number signal.
- Send patch state to Ableton/external apps: any signal → `Write OSC /myparam` → `OSC Out 192.168.1.10:9001`.

---

## 10. MIDI

Mirror nodes for MIDI: `MIDI In`, `Read MIDI Note`, `Read MIDI CC`, `MIDI Out`, `Write MIDI Note`, `Write MIDI CC`. Wire can also act as a host, sending MIDI to other DAWs.

---

## 11. Audio reactivity

Wire receives audio from the host (Arena/Avenue) when the patch runs as a plugin:

- **FFT input** — frequency-domain bins delivered as an array.
- **Audio sample input** — raw waveform.
- Combined with `Sample Texture`, `Average`, `Smooth`, etc. for energy / beat extraction.

Standalone Wire can also tap the system audio device for testing.

---

## 12. Useful UI shortcuts

| Action | Shortcut |
|---|---|
| Pan canvas | hold Space + drag, or middle mouse |
| Zoom | middle-mouse scroll, or Ctrl/⌘ + `+` / `-` |
| Fit to view | Ctrl/⌘ + `0` |
| 100% zoom | Ctrl/⌘ + `1` |
| Select all | Ctrl/⌘ + `A` |
| Select unused nodes | Ctrl/⌘ + Shift + `A` |
| Auto-sort selected | Ctrl/⌘ + `L` |
| Duplicate | Ctrl/⌘ + `D` (or Alt + drag) |
| Rename | Ctrl/⌘ + Shift + `R` |
| Search nodes | Ctrl + F (win) / ⌘ + F (mac) |
| Copy / Paste | Ctrl/⌘ + `C` / `V` (also `X` for cut) |
| Copy For Sharing | right-click → exports patch as forum-pasteable text |
| Delete connection / node | Backspace or Delete |

The right-click **Insert Node Before / After Current** action (added in 7.24) auto-rewires connections — invaluable for inserting filters into long chains.

---

## 13. Panels at a glance

| Panel | What it does |
|---|---|
| **Library** | Searchable node browser. Drag onto canvas, or double-click empty canvas to open the node search popup |
| **Monitor** | Live preview of the selected node's output. Pin to lock a node |
| **Patch** | Patch metadata: name, description, category, resolution, bit depth, license info |
| **Node** | Selected node's parameters (manual inlet values, instance count, color, datatype) |
| **Resources** | External files referenced by the patch (images, ISF files, video). Use **Consolidate** to bundle them next to the patch |
| **Notes** | Free-text notes panel (added 7.24) for in-patch documentation |
| **Log** | Errors, warnings, and `Print` node output |
| **Dashboard** | Reorder/group parameters; toggle which inputs are exposed to Arena/Avenue |
| **Patch Navigator** | Minimap-style fast jump through large patches (added 7.24) |
| **Stats** | Per-node CPU/GPU load — sortable by name, category, load, version |

---

## 14. Saving and distribution

```
File → Save                  # writes a .wired
File → Compile for Resolume  # writes .cwired (locked) into Arena/Avenue's Sources or Effects folder
File → Consolidate           # bundles patch + all resources into a folder
```

**Compile dialog options:**

- **Editable** — when on, recipients with Wire can open and modify. Off → `.cwired` (locked, no resource access).
- **Compile for Juicebar** — generates a license-gated build. End users buy a Juicebar license; Wire injects the watermark unless the license is valid.

**Sharing on forums.** Select nodes → right-click → **Copy For Sharing** → paste the resulting text into a message. The recipient pastes it into Wire to materialize the nodes. Note: external resources (images, ISF files) are **not** included by Copy For Sharing — only the graph.

The **Juicebar marketplace** at [get-juicebar.com/wire](https://get-juicebar.com/wire) is Resolume's first-party storefront for selling Wire patches.

---

## 15. Version timeline (Wire 7.x highlights)

| Version | Notable additions |
|---|---|
| **7.20** | First-class Color type. Color from RGB / HSB, Analogous / Complementary palette nodes, Sample Texture node. Hardware Stats panel (FPS / GPU / CPU / RAM). Node Stats panel (per-node load). Ctrl+F node search. Math: sqrt, radians, pi, phi, tau. Tutorial section overhaul. |
| **7.21** | Parameter Grouping on the Dashboard — groups appear as foldable sections in Arena/Avenue. Transform Widget. |
| **7.23** | Parameter Animation Presets. Performance improvements. New and improved Wire nodes. |
| **7.24** | Patch Navigator panel, Notes panel, Insert Node Before/After Current Node, search popup with previews. New nodes: Kaleidoscope, Gaussian Blur, Cylindrical / Spherical Coordinate tools, Stable Sort. |
| **7.26** | 10-bit colour output, speed fixes. |

---

## 16. Workflow tips

1. **Build inputs first, output last.** Lay down your `Texture Input` (if Effect/Mixer) and `Texture Output` early; everything else slots between them.
2. **Use `Comment` nodes liberally.** Toggle "fill" off and a comment becomes a labeled section divider — your future self thanks you.
3. **`Print` is the debugger.** Pipe a signal into a Print node to read its value in the Log panel.
4. **`Pin` the Monitor** on a key node to see its live output regardless of selection.
5. **Test in Arena/Avenue early.** Compile a draft `.cwired` and load it; the host's framerate and audio context behave differently from Wire's standalone preview.
6. **Consolidate before sharing.** It copies resources into the patch folder and rewrites resource paths, avoiding broken-link reports from collaborators.
7. **Don't fight types.** If a wire won't connect, look for a converter node (`Make Vec2`, `Color from RGB`, `Texture from Color`) rather than monkey-patching with packs/unpacks.
8. **Group early.** Once you have more than ~6 dashboard parameters, create groups (Wire 7.21+) — Arena/Avenue's UI gets cluttered fast otherwise.

---

## 17. Common pitfalls

1. **Patch compiles but doesn't appear in Arena/Avenue.** Check the patch was compiled to the right folder (`Sources` for a Source template, `Effects` for an Effect template). Restart Resolume after copying — scan happens on launch.
2. **Watermark on output.** Either no Wire license, or you didn't enable Juicebar compilation while distributing through Juicebar.
3. **Vertex shader ignored.** Wire's ISF node is fragment-only (Wire is 2D). Move 3D logic to `Mesh Render` nodes.
4. **OSC messages not arriving.** Three usual culprits: firewall (especially Windows), wrong port (Wire defaults to 8081, not Arena's 8080), or sender pointing at `localhost` while Wire is on a different machine. Use Wire's OSC Input log to confirm packets are arriving.
5. **Attribute won't animate.** By design — attributes are compile-time only. Use a signal-flow input.
6. **Parameter not showing in Arena/Avenue.** The Input node's "show in dashboard" toggle is off, or you're checking the wrong product (Source vs Effect template).
7. **Resources missing on someone else's machine.** Forgot to **Consolidate** before sharing. Run File → Consolidate; that copies images/ISF files into the patch folder and rewrites paths.

---

## 18. When to use Wire vs FFGL vs ISF

| Need | Tool |
|---|---|
| One fragment shader, no extra logic | ISF |
| Procedural visuals composed from many primitives + math + audio reactivity | **Wire** |
| Custom OpenGL pipeline / non-shader CPU work / persistent host state | FFGL (C++) |
| Sell to end users who don't own Wire | FFGL or `.cwired` via Juicebar |
| Quick prototype, no build system | ISF or Wire (Wire wins for anything beyond a single shader) |
| Maximum performance with hand-tuned GL | FFGL |

Wire excels when your patch is a system — multiple sources combined, audio-reactive, with parameter groups end users will tweak. For pure shader experiments stay in ISF; for production-grade compiled effects with installer-ready binaries use FFGL.

---

## Sources

- [resolume.com/software/wire](https://resolume.com/software/wire) — product page
- [resolume.com/support/en/wire-introduction](https://resolume.com/support/en/wire-introduction) — official intro
- [resolume.com/support/en/wire-node-anatomy](https://resolume.com/support/en/wire-node-anatomy) — node structure
- [resolume.com/support/en/wire-data-flow](https://resolume.com/support/en/wire-data-flow) — flow types
- [resolume.com/support/en/wire-user-interface](https://resolume.com/support/en/wire-user-interface) — UI reference
- [resolume.com/support/en/wire-osc](https://resolume.com/support/en/wire-osc) — OSC integration
- [resolume.com/support/en/wire-saving-consolidating](https://resolume.com/support/en/wire-saving-consolidating) — file formats and distribution
- [resolume.com/blog/24094](https://resolume.com/blog/24094) — 7.20 Color Types release
- [resolume.com/blog/24836](https://resolume.com/blog/24836) — 7.21 Parameter Grouping
- [resolume.com/blog/30807](https://resolume.com/blog/30807) — 7.23 release
- [get-juicebar.com/wire](https://get-juicebar.com/wire) — Wire patch marketplace
- [github.com/YaNesyTortiK/WirePatches](https://github.com/YaNesyTortiK/WirePatches) — community patches
