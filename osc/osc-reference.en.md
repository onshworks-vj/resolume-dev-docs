---
title: Resolume Arena / Avenue 7 — OSC Reference
version: Resolume 7.20+ (verified up to 7.26)
last_updated: 2026-04-30 (verified)
---

# Resolume OSC Reference (Arena & Avenue 7)

This is a practical, up-to-date reference for controlling **Resolume Arena and Avenue 7.x** via Open Sound Control. All paths below are valid in 7.20+ and verified against the official `resolume.com/support/en/osc` page and the actively-maintained Bitfocus Companion module (`bitfocus/companion-module-resolume-arena`).

> **Why this exists.** Most OSC tutorials online still use the Resolume 4/5 namespace (`/layer1/clip3/...`). That namespace is **gone** in Resolume 7. The current namespace is hierarchical and starts with `/composition`.

---

## 1. Setup

Enable OSC in **Preferences → OSC**:

| Setting | Description | Default |
|---|---|---|
| Incoming OSC port | UDP port Resolume listens on | **7000** |
| Outgoing OSC host | Where Resolume sends feedback | `127.0.0.1` |
| Outgoing OSC port | UDP port for feedback | **7001** |
| OSC output preset | Selects which messages are sent out | "Output All OSC Messages" for full feedback |

**Bidirectional rule.** Resolume sends a query response back to **the port the sender used**, not to the configured outgoing port. If you want to receive responses to `?` queries, send from the same UDP socket you're listening on.

---

## 2. Address Conventions

A Resolume 7 OSC address looks like:

```
/composition/layers/1/clips/3/connect
```

Three rules to remember:

1. **Always start with `/composition`.** There is no shortcut namespace.
2. **Indices are 1-based.** Layer 1, clip 1, deck 1, etc. There is no layer 0.
3. **Triggers need a value.** Sending an integer `1` triggers, then `0` releases. Resolume treats the rising edge as the trigger event.

### Value modifiers

Any settable parameter accepts these prefixes as the **first OSC argument** (string), followed by the value:

| Prefix | Meaning | Example |
|---|---|---|
| (none) | Absolute set | `0.5` |
| `+` | Increment | `"+", 1` |
| `-` | Decrement | `"-", 1` |
| `*` | Multiply | `"*", 2` |
| `?` | Query current value (no value arg) | `"?"` |

> **Caveat — relative modifiers with floats.** As of Resolume 7.x the relative `+` / `-` / `*` modifiers are unreliable for **float** parameters; only **integer** arguments are honored consistently. If you need fractional increments on opacity, etc., compute the absolute value on the client side and send that instead. ([Companion issue #9](https://github.com/bitfocus/companion-module-resolume-arena/issues/9), Resolume forum t=21959.)

> **Tip.** The fastest way to discover an address is to enter **Shortcuts → Edit OSC** in Resolume, then click the control you want. The Shortcuts panel shows the exact address.

---

## 3. Composition

| Address | Type | Description |
|---|---|---|
| `/composition/master` | <span class="tag float">float 0–1</span> | Composition master output level |
| `/composition/video/opacity` | <span class="tag float">float 0–1</span> | Composition video opacity |
| `/composition/audio/volume` | <span class="tag float">float 0–1</span> | Composition audio volume |
| `/composition/speed` | <span class="tag float">float 0–2</span> | Master playback speed (1.0 = normal) |
| `/composition/direction` | <span class="tag int">int</span> | Playback direction (`0` backwards, `1` forwards, `2` pause, `3` random) |
| `/composition/disconnectall` | <span class="tag trigger">trigger</span> | Stop all playing clips |
| `/composition/connectnextcolumn` | <span class="tag trigger">trigger</span> | Trigger next column |
| `/composition/connectprevcolumn` | <span class="tag trigger">trigger</span> | Trigger previous column |
| `/composition/connectspecificcolumn` | <span class="tag int">int</span> | Trigger column by index (1-based) |
| `/composition/selectnextdeck` | <span class="tag trigger">trigger</span> | Move selection to next deck |
| `/composition/selectprevdeck` | <span class="tag trigger">trigger</span> | Move selection to previous deck |
| `/composition/selectspecificdeck` | <span class="tag int">int</span> | Select deck by index |
| `/composition/name` | <span class="tag string">string</span> | Composition name (read/write) |
| `/composition/recorder/record` | <span class="tag bool">bool</span> | Toggle recording (Arena) |

### Tempo

| Address | Type | Description |
|---|---|---|
| `/composition/tempocontroller/tempo` | <span class="tag float">float</span> | BPM (range 2.0–500.0) |
| `/composition/tempocontroller/tempotap` | <span class="tag trigger">trigger</span> | Tap tempo |
| `/composition/tempocontroller/resync` | <span class="tag trigger">trigger</span> | Re-sync the playhead |
| `/composition/tempocontroller/tempopush` | <span class="tag trigger">trigger</span> | Nudge phase forward |
| `/composition/tempocontroller/tempopull` | <span class="tag trigger">trigger</span> | Nudge phase backward |
| `/composition/tempocontroller/tempoinc` | <span class="tag trigger">trigger</span> | Increase BPM by 1 |
| `/composition/tempocontroller/tempodec` | <span class="tag trigger">trigger</span> | Decrease BPM by 1 |
| `/composition/tempocontroller/tempodividetwo` | <span class="tag trigger">trigger</span> | Halve BPM |
| `/composition/tempocontroller/tempomultiplytwo` | <span class="tag trigger">trigger</span> | Double BPM |
| `/composition/tempocontroller/metronome` | <span class="tag bool">bool</span> | Toggle audible metronome |

### Crossfader (composition-level only)

In Resolume the crossfader sits at the composition level — there is **no per-layer crossfader address**. Layers opt into A or B via `crossfadergroup`.

| Address | Type | Description |
|---|---|---|
| `/composition/crossfader/phase` | <span class="tag float">float -1..1</span> | A/B crossfader position |
| `/composition/crossfader/sidea` | <span class="tag trigger">trigger</span> | Snap to side A |
| `/composition/crossfader/sideb` | <span class="tag trigger">trigger</span> | Snap to side B |
| `/composition/crossfader/curve` | <span class="tag int">int</span> | Curve shape choice |
| `/composition/crossfader/behaviour` | <span class="tag int">int</span> | Behaviour mode |
| `/composition/crossfader/video/mixer/blendmode` | <span class="tag int">int</span> | Blend mode for the crossfader |

### Dashboard links

Resolume exposes 8 user-mappable composition-level shortcuts:

```
/composition/dashboard/link1   …   /composition/dashboard/link8
```

These also exist per-layer and per-clip (`/composition/layers/{L}/dashboard/link{N}`, `…/clips/{C}/dashboard/link{N}`).

---

## 4. Layers

`{L}` = layer index (1-based).

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/select` | <span class="tag trigger">trigger</span> | Select layer |
| `/composition/layers/{L}/clear` | <span class="tag trigger">trigger</span> | Stop the active clip on this layer |
| `/composition/layers/{L}/bypassed` | <span class="tag bool">bool</span> | Bypass layer |
| `/composition/layers/{L}/solo` | <span class="tag bool">bool</span> | Solo layer |
| `/composition/layers/{L}/master` | <span class="tag float">float 0–1</span> | Layer master |
| `/composition/layers/{L}/name` | <span class="tag string">string</span> | Layer name (read/write) |
| `/composition/layers/{L}/video/opacity` | <span class="tag float">float 0–1</span> | Layer opacity |
| `/composition/layers/{L}/audio/volume` | <span class="tag float">float 0–1</span> | Layer volume |
| `/composition/layers/{L}/audio/pan` | <span class="tag float">float</span> | Layer audio pan |
| `/composition/layers/{L}/speed` | <span class="tag float">float</span> | Layer playback speed |
| `/composition/layers/{L}/direction` | <span class="tag int">int</span> | `0` backwards / `1` forwards / `2` pause / `3` random |
| `/composition/layers/{L}/transition/duration` | <span class="tag float">float</span> | Crossfade duration in seconds |
| `/composition/layers/{L}/connectnextclip` | <span class="tag trigger">trigger</span> | Trigger next clip on layer |
| `/composition/layers/{L}/connectprevclip` | <span class="tag trigger">trigger</span> | Trigger previous clip on layer |
| `/composition/layers/{L}/connectspecificclip` | <span class="tag int">int</span> | Trigger clip by index on this layer |
| `/composition/layers/{L}/autopilot` | <span class="tag int">int</span> | Autopilot mode (Off / Next clip / etc.) |
| `/composition/layers/{L}/faderstart` | <span class="tag bool">bool</span> | Layer fader-start enabled |
| `/composition/layers/{L}/ignorecolumntrigger` | <span class="tag bool">bool</span> | Skip column triggers for this layer |
| `/composition/layers/{L}/playmode` | <span class="tag int">int</span> | Layer play mode |
| `/composition/layers/{L}/crossfadergroup` | <span class="tag int">int</span> | Assign layer to crossfader side (Off / A / B) |
| `/composition/layers/{L}/maskmode` | <span class="tag int">int</span> | Layer mask mode |

### Video mixer (per layer)

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/video/mixer/blendmode` | <span class="tag int">int</span> | Blend mode index for the clip layer (Add, Lighten, etc.) |
| `/composition/layers/{L}/video/transition/mixer/blendmode` | <span class="tag int">int</span> | Blend mode used during clip transitions |

---

## 5. Clips

`{L}` = layer, `{C}` = column index.

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/connect` | <span class="tag trigger">trigger</span> | Trigger this clip (most common operation) |
| `/composition/layers/{L}/clips/{C}/select` | <span class="tag trigger">trigger</span> | Select clip without triggering |
| `/composition/layers/{L}/clips/{C}/name` | <span class="tag string">string</span> | Clip name (read/write) |
| `/composition/layers/{L}/clips/{C}/video/opacity` | <span class="tag float">float</span> | Per-clip opacity override |
| `/composition/layers/{L}/clips/{C}/audio/volume` | <span class="tag float">float</span> | Per-clip volume |

### Transport (playhead) per clip

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/transport/position` | <span class="tag float">float 0–1</span> | Playhead position normalized |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/speed` | <span class="tag float">float</span> | Per-clip speed |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playdirection` | <span class="tag int">int</span> | `0` backwards / `1` forwards / `2` pause / `3` random |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playmode` | <span class="tag int">int</span> | Play mode (loop / once / bounce / random) |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/playmodeaway` | <span class="tag int">int</span> | Play mode when clip is not connected |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/syncmode` | <span class="tag int">int</span> | BPM sync mode |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/dividetwo` | <span class="tag trigger">trigger</span> | Halve playback speed |
| `/composition/layers/{L}/clips/{C}/transport/position/behaviour/multiplytwo` | <span class="tag trigger">trigger</span> | Double playback speed |

### Cuepoints per clip

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/transport/cuepoints/setparams/set{N}` | <span class="tag trigger">trigger</span> | Store current playhead as cuepoint N |
| `/composition/layers/{L}/clips/{C}/transport/cuepoints/jumpparams/jump{N}` | <span class="tag trigger">trigger</span> | Jump to stored cuepoint N |

### Other clip parameters

| Address | Type | Description |
|---|---|---|
| `/composition/layers/{L}/clips/{C}/autopilot` | <span class="tag int">int</span> | Per-clip autopilot |
| `/composition/layers/{L}/clips/{C}/beatsnap` | <span class="tag int">int</span> | Beat snap value |
| `/composition/layers/{L}/clips/{C}/cliptarget` | <span class="tag int">int</span> | Clip target |
| `/composition/layers/{L}/clips/{C}/cliptriggerstyle` | <span class="tag int">int</span> | Trigger style |
| `/composition/layers/{L}/clips/{C}/faderstart` | <span class="tag bool">bool</span> | Fader-start enabled |
| `/composition/layers/{L}/clips/{C}/ignorecolumntrigger` | <span class="tag bool">bool</span> | Skip column trigger |
| `/composition/layers/{L}/clips/{C}/transporttype` | <span class="tag int">int</span> | Transport type |

> **Per-clip audio volume** is implementation-dependent. Some Resolume builds expose `/composition/layers/{L}/clips/{C}/audio/volume`; on others audio volume is layer-only. Verify in **Shortcuts → Edit OSC** before relying on it.

---

## 6. Columns

`{C}` = column index.

| Address | Type | Description |
|---|---|---|
| `/composition/columns/{C}/connect` | <span class="tag trigger">trigger</span> | Fire entire column (every layer) |
| `/composition/columns/{C}/name` | <span class="tag string">string</span> | Column name (read/write) |
| `/composition/columns/{C}/selected` | <span class="tag bool">bool</span> | Read-only feedback: is this column selected |

> **No "select column" action.** Resolume does not expose an OSC address to select a column without connecting it. Use `connect` to fire the column (which also selects it), or query `selected` with `?` to read the current selection.

---

## 7. Decks

`{D}` = deck index.

| Address | Type | Description |
|---|---|---|
| `/composition/decks/{D}/select` | <span class="tag trigger">trigger</span> | Select deck |
| `/composition/decks/{D}/name` | <span class="tag string">string</span> | Deck name (read/write) |

---

## 8. Layer Groups

Resolume 7 introduced **layer groups**. The naming differs by protocol — this isn't an alias situation, it's two distinct namespaces:

| Protocol | Namespace |
|---|---|
| **OSC** | `/composition/groups/{G}/...` |
| **REST / WebSocket** | `/composition/layergroups/{G}/...` |

If you mix protocols (e.g. send OSC and listen on the WebSocket), expect to see both forms. The Bitfocus Companion module uses both: `groups` for OSC sends and `layergroups` for REST API calls.

`{G}` = group index.

| Address (OSC) | Type | Description |
|---|---|---|
| `/composition/groups/{G}/select` | <span class="tag trigger">trigger</span> | Select group |
| `/composition/groups/{G}/bypassed` | <span class="tag bool">bool</span> | Bypass entire group |
| `/composition/groups/{G}/solo` | <span class="tag bool">bool</span> | Solo group |
| `/composition/groups/{G}/master` | <span class="tag float">float</span> | Group master |
| `/composition/groups/{G}/speed` | <span class="tag float">float</span> | Group speed |
| `/composition/groups/{G}/direction` | <span class="tag int">int</span> | `0` backwards / `1` forwards / `2` pause / `3` random |
| `/composition/groups/{G}/video/opacity` | <span class="tag float">float</span> | Group opacity |
| `/composition/groups/{G}/audio/volume` | <span class="tag float">float</span> | Group volume |
| `/composition/groups/{G}/columns/{C}/connect` | <span class="tag trigger">trigger</span> | Trigger column within group |
| `/composition/groups/{G}/columns/{C}/select` | <span class="tag trigger">trigger</span> | Select column within group |
| `/composition/groups/{G}/disconnectlayers` | <span class="tag trigger">trigger</span> | Stop all clips in group |

---

## 9. Effects

Effects are accessible by walking down the parameter tree. An effect's parameters appear under whichever entity it is attached to (composition, group, layer, or clip):

```
/composition/video/effects/{N}/...
/composition/layers/{L}/video/effects/{N}/...
/composition/layers/{L}/clips/{C}/video/effects/{N}/...
/composition/groups/{G}/video/effects/{N}/...
```

`{N}` is the effect's slot index in the effect chain (1-based). Each effect exposes:

| Sub-address | Type | Description |
|---|---|---|
| `/bypassed` | <span class="tag bool">bool</span> | Effect bypass |
| `/mix` | <span class="tag float">float 0–1</span> | Wet/dry mix |
| `/{paramName}` | <span class="tag float">float</span> | Effect-specific parameters (camelCase) |

> **Discover effect addresses.** The reliable way is: in Resolume, **right-click the effect parameter → Shortcuts → Edit OSC → click the parameter**. The address shown is exact and tracks the current effect order.

### Effect parameter types you'll encounter

- `float` (most parameters) — normalized 0–1
- `int` (option choices) — discrete enum
- `bool` (toggles)
- `color` — usually exposed as separate `r`, `g`, `b`, `a` floats or as a single hex int

---

## 10. Source / Generator parameters

Native Resolume sources (Solid, Gradients, etc.) and FFGL/ISF generators expose their parameters under the clip's `video/source/` subtree:

```
/composition/layers/{L}/clips/{C}/video/source/{paramName}
```

The parameter names match what you see in the Resolume UI (camelCase, no spaces).

---

## 11. Querying state (`?`)

Send an OSC message with a single string argument `"?"` to read the current value:

```
/composition/layers/1/video/opacity   "?"
```

Resolume replies on the same UDP socket (port-matched) with the current value as the appropriate type.

---

## 12. Receiving updates

If you set the OSC output preset to **Output All OSC Messages**, Resolume continuously broadcasts:

- Every parameter change (user-driven or automated)
- Tempo, beat, and phase
- Activity feedback (clip connects, selections)

Listen on the configured outgoing port (default 7001) and parse OSC messages — same address scheme as for sending.

---

## 13. Examples

### Trigger clip in column 3 of layer 2

```
/composition/layers/2/clips/3/connect   1
```

### Set composition master to 75%

```
/composition/master   0.75
```

### Increase layer 1 opacity by 10%

The relative modifiers don't work reliably with float values in Resolume 7 (see §2). Compute the absolute target on the client side:

```
# After querying the current value as 0.4
/composition/layers/1/video/opacity   0.5
```

### Tap tempo, then re-sync

```
/composition/tempocontroller/tempotap    1
/composition/tempocontroller/resync      1
```

### Read current opacity of layer 2

```
/composition/layers/2/video/opacity   "?"
```

### Stop everything

```
/composition/disconnectall   1
```

---

## 14. Common gotchas

1. **The trigger pattern is `1` then `0`.** Many "didn't fire" issues are because only `1` was sent. Companion sends `1`, waits ~50ms, sends `0`. Replicate this in your own controllers.
2. **`/layer1/...` style addresses don't work.** That's the Resolume 4/5 namespace. Migrate to `/composition/layers/1/...`.
3. **Indices are 1-based, not 0.** Sending to layer `0` does nothing.
4. **Use `selectedlayer` for context-relative.** Resolume exposes `/composition/selectedlayer/...` and `/composition/selectedclip/...` if you want to operate on whatever the user has selected.
5. **OSC output preset matters.** If your "Output All OSC Messages" preset isn't selected, you won't see most feedback messages.
6. **Effect slot indices change when reordering.** Don't hard-code effect slot numbers if your operator reorders effects mid-show. Use `/by-id/...` via the REST/WebSocket API for stable IDs.

---

## 15. Beyond OSC: REST and WebSocket API

For state-rich control (thumbnails, structure changes, stable IDs), Resolume exposes a REST and WebSocket API on the same port:

- REST: `http://<host>:<port>/api/v1/composition`
- WebSocket: `ws://<host>:<port>/api/v1`
- Default port: **8080** for Arena/Avenue, **8081** for Wire (set in Preferences → Webserver)
- API doc: `http://<host>:<port>/api/v1/` (Swagger UI served live)
- Introduced in **Resolume 7.8**.

### Parameter actions (WebSocket)

Lowercase verbs: `subscribe`, `unsubscribe`, `set`, `get`, `reset`, `trigger`.

```json
{ "action": "set", "parameter": "/composition/layers/1/video/opacity", "value": 0.5 }
```

`value` is required for `set` only. Parameter paths use either `/parameter/by-id/{id}` (stable across reorders) or the same logical path as OSC.

### Structural actions (WebSocket)

A separate action family for adding/removing entities, with `path` + optional `body`:

```json
{ "action": "post",   "id": "my_layer", "path": "/composition/layers/add" }
{ "action": "remove", "path": "/composition/layers/3" }
```

The response echoes the `id` and includes `error: null` on success.

> Use the REST/WebSocket API when you need: stable parameter IDs that survive reordering, clip thumbnails, structural edits (add/remove layers, columns, effects), or change subscriptions.

---

## Sources

- [resolume.com/support/en/osc](https://resolume.com/support/en/osc) — official OSC support page
- [resolume.com/support/en/restapi](https://resolume.com/support/en/restapi) — REST API
- [resolume.com/support/en/websocket-api](https://resolume.com/support/en/websocket-api) — WebSocket API
- [resolume.com/docs/restapi/](https://resolume.com/docs/restapi/) — Swagger reference
- [github.com/bitfocus/companion-module-resolume-arena](https://github.com/bitfocus/companion-module-resolume-arena) — most actively maintained third-party reference
