---
title: Resolume Development Docs
version: Resolume Arena / Avenue 7.20+
last_updated: 2026-04-30
---

# Resolume 7 Development Documentation

A developer-oriented documentation set for **Resolume Arena and Avenue 7.x** covering OSC control, FFGL plugin development, and ISF shader authoring. Written against the current state of Resolume 7.20+ and verified against official sources and actively-maintained third-party references.

> **Why this exists.** Most online Resolume tutorials still reference the Resolume 4/5 era — wrong OSC namespaces, deprecated SDK paths, and outdated shader patterns. These docs are written from current sources and the FFGL master branch.

## Languages

- **English** — this index → see the docs below
- **한국어** — [README.ko.md](README.ko.md)

## Documents

### OSC Control

Control Resolume from any OSC-capable application (TouchOSC, Companion, Max/MSP, your own code). Covers the current `/composition/...` namespace, value modifiers, querying state, and the related REST/WebSocket APIs.

- [osc/osc-reference.en.md](osc/osc-reference.en.md)

### FFGL Plugin Development (C++)

Build native effects, sources, and mixers in C++ using the official FFGL SDK. Covers SDK setup, plugin lifecycle, parameter types, color components, audio reactivity, and the most common pitfalls.

- [ffgl/ffgl-guide.en.md](ffgl/ffgl-guide.en.md)

### ISF Shader Authoring

Write Resolume effects, sources, and transitions as JSON-prefixed GLSL fragment shaders. No build system, no SDK — drop a `.fs` file and it shows up in Resolume.

- [isf/isf-guide.en.md](isf/isf-guide.en.md)

### Wire Patch Development

Build Sources / Effects / Mixers as node-based patches in Resolume Wire. Combine procedural visuals, audio reactivity, ISF shaders, and OSC/MIDI logic without writing code. Compile to `.wired` / `.cwired` for Arena/Avenue.

- [wire/wire-guide.en.md](wire/wire-guide.en.md)

## Layout

```
resolume-dev-docs/
├── index.html                    ← GitHub Pages entry (= README.en.html)
├── README.{en,ko}.{md,html}      ← index, both formats
├── osc/
│   └── osc-reference.{en,ko}.{md,html}
├── ffgl/
│   └── ffgl-guide.{en,ko}.{md,html}
├── isf/
│   └── isf-guide.{en,ko}.{md,html}
├── wire/
│   └── wire-guide.{en,ko}.{md,html}
└── assets/
    └── style.css                 ← shared styles for the HTML site
```

Markdown is the canonical source. The `.html` files are pre-built and committed so the site can be served directly via **GitHub Pages** (no build step). When you edit Markdown, regenerate the matching HTML before committing.

## Picking the right tool

| You want to... | Use |
|---|---|
| Trigger clips / change parameters from a controller | OSC |
| Build complex tour control surfaces, get state feedback, manipulate structure | REST/WebSocket API (in OSC doc §15) |
| Write a new visual effect or generator quickly, only fragment-shader logic | ISF |
| Need access to host audio data, want feedback buffers with full GL control | ISF (audio inputs + PASSES) |
| Compose multiple sources, math, audio reactivity, OSC/MIDI logic visually | **Wire** |
| Build a high-performance native effect, custom GL pipeline, or anything beyond a single fragment shader | FFGL |
| Distribute to non-technical users with a polished UX | FFGL (compiled binary) or Wire `.cwired` via Juicebar |

## Notes on freshness

- **OSC**: the namespace shown here is verified against Resolume 7.20+, the actively-maintained Bitfocus Companion module source, and Resolume's static OSC list. The `/composition/...` paths have been stable since Resolume 7.0; what's shifted across point releases is mostly added paths (groups, layer groups, transport sub-paths), not breaking changes.
- **FFGL**: the master SDK branch is documented here, with class names, FF_TYPE_* constants, and lifecycle methods verified against the headers in `github.com/resolume/ffgl`. If you target older Resolume installs, use the `v2.2` tag.
- **ISF**: the spec hasn't moved in years. ISF v2.0 is current and Resolume's implementation tracks it directly. Function names verified against the VVISF-GL reference parser.
- **Wire**: documented against Wire 7.26 and the official Resolume support pages. Patch file formats (`.wired` / `.cwired`), node categories, flow types, and recent additions (Color type 7.20, Parameter Grouping 7.21, Patch Navigator 7.24) reflect the current release.

> **Verification.** This documentation has been cross-checked against multiple independent sources: Resolume's official support pages, the FFGL SDK source, the ISF spec repository, the Bitfocus Companion module, and several community OSC implementations. See each doc's Sources section for citations.

## Contributing / updating

When Resolume ships a new release that adds OSC paths or FFGL features:

1. Edit the relevant `.md` file (English first, then mirror to Korean).
2. Update the version line in the frontmatter.
3. Commit.
