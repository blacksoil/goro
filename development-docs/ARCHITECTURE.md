# goro Architecture

`goro` is a from-scratch Go client for Ragnarok Online (pre-renewal, 2008
packet era). It talks the original login/char/map server protocol to a
private server (typically rAthena), loads the original client's data files
(GRF archives, sprites, maps, tables), and renders everything through a
modern GPU pipeline (GoGPU/wgpu, Vulkan-first) instead of the original
DirectX 7 client.

This document explains how the pieces fit together well enough to debug an
issue or add a feature without reading the whole codebase first. It
deliberately stays shallow on rendering internals (GPU pipeline, mesh/texture
formats, shaders, sprite/animation drawing) — see
[`GRAPHICS.md`](GRAPHICS.md) for that.

## Table of contents

1. [Big picture](#big-picture)
2. [Process entry point and lifecycle](#process-entry-point-and-lifecycle)
3. [Configuration](#configuration)
4. [Package dependency layering](#package-dependency-layering)
5. [The shared runtime context (`client.Context`)](#the-shared-runtime-context-clientcontext)
6. [Session, World, and the game/render/input handoff](#session-world-and-the-gamerenderinput-handoff)
7. [Game orchestration: modes](#game-orchestration-modes)
8. [Networking](#networking)
9. [Resource loading (`res`)](#resource-loading-res)
10. [UI system](#ui-system)
11. [Static game data (`db`)](#static-game-data-db)
12. [Audio](#audio)
13. [Input](#input)
14. [Bot / Lua scripting](#bot--lua-scripting)
15. [Standalone tools (`cmd/`)](#standalone-tools-cmd)
16. [Logging (`glog`)](#logging-glog)
17. [Testing conventions](#testing-conventions)
18. [Build and packaging](#build-and-packaging)
19. [Key third-party dependencies](#key-third-party-dependencies)
20. [How to...](#how-to)

## Big picture

```
                          ┌────────────────────┐
                          │        main         │  main.go
                          └─────────┬───────────┘
                                    │ loads config, builds app.Game
                                    ▼
                          ┌────────────────────┐
                          │      render.Run /    │  window + GPU loop
                          │   render.RunHeadless  │  (or headless ticker)
                          └─────────┬───────────┘
                     Update()/Draw()│  ▲ input events
                                    ▼  │
                          ┌────────────────────┐
                          │      app.Game        │  owns all subsystems,
                          │  (client/context.go) │  builds client.Context
                          └─────────┬───────────┘
                                    │ client.Context (read/write handle)
                                    ▼
                          ┌────────────────────┐
                          │     game.Manager      │  finite state machine
                          │   (Mode: Login/World) │  of "modes"
                          └───┬────────┬─────────┘
                              │        │
                 reads/writes │        │ calls into
                              ▼        ▼
                 ┌─────────────┐  ┌─────────────┐   ┌─────────────┐
                 │   session    │  │    world     │   │     ui       │
                 │ (account/char│  │ (map model,  │   │ (windows,    │
                 │  progress)   │  │  actors, cam)│   │  modals)     │
                 └─────────────┘  └─────────────┘   └─────────────┘
                              │        │                  │
                              ▼        ▼                  ▼
                 ┌─────────────────────────────────────────────────┐
                 │   network (packets)   res (GRF/sprites/tables)    │
                 │   audio (bgm/sfx)      render (Frame/GPU)          │
                 │   db (static tables)   input (key/mouse snapshot)  │
                 └─────────────────────────────────────────────────┘
```

`app.Game` is the composition root: it owns one instance of every subsystem
(`config.Config`, `input.State`, `res.Manager`, `session.Session`,
`world.World`, `network.Client`, `audio.BGM`, `game.Manager`, `ui.Manager`)
and, every frame, packages read/write access to all of them into a single
`client.Context` value that is handed down into the currently active `game.Mode`.

Each package's `doc.go` states its responsibility and, importantly, its
**boundary** — what it must *not* do. These comments are load-bearing; they
are the architecture in prose form and are enforced only by convention and
code review, not by the compiler (Go has no sub-package visibility). When
in doubt about where new code belongs, read the relevant `doc.go` first:
`app/doc.go`, `audio/doc.go`, `client/doc.go`, `config/doc.go`, `db/doc.go`,
`game/doc.go`, `input/doc.go`, `network/doc.go`, `render/doc.go`,
`res/doc.go`, `session/doc.go`, `ui/doc.go`, `world/doc.go`.

## Process entry point and lifecycle

`main.go` does exactly four things:

1. `config.LoadConfig(os.Args[1:])` — parse configuration (flags/ini).
2. `glog.Configure(cfg.Log)` — set up the logger, get a `closeLog` deferred func.
3. `app.New(cfg)` — build the `*app.Game`.
4. Depending on `cfg.Headless`, call `render.Run(game, cfg.Window, cfg.Render)`
   (windowed, GPU-backed) or `render.RunHeadless(ctx, game, cfg.Window)`
   (no window, no GPU — used by tests, CI, and profiling).

`app.Game` (`app/app.go`) implements the `render.Game` interface that both
run loops drive:

```go
type Game interface {
    Update() error
    Draw(*Frame)
    Resize(width, height int)
    InputState() *input.State
}
```

It optionally implements several more interfaces the runner probes for via
type assertion (`quitReceiver`, `uiAppReceiver`, `keyboardInputPreparer`,
`overlayDrawer`, `uiOverlayDrawer`, `frameSubmittedReceiver` — see
`render/backend.go`). This is how `render` stays decoupled from `app`: it
only depends on `client.UIApp`/`input` types, never on `app` or `game`
directly (`render` is a leaf package, see
[Package dependency layering](#package-dependency-layering)).

**Frame cycle** (windowed, `render.Run` in `render/backend.go`):

- GoGPU (`github.com/gogpu/gogpu`) owns the OS window, event pump, and GPU
  surface. `render.Run` builds a `gogpu.App`, wires `OnUpdate` →
  `r.update()` → `game.Update()`, and `OnDraw` → `r.draw(ctx)` →
  `screen.BeginFrame(); game.Draw(screen); ...; ` submit to GPU.
- `game.Update()` (`app/app.go:82`) drains input for the frame
  (`defer g.input.EndFrame()`), pumps the network client
  (`g.network.Pump()`), rebuilds the `client.Context` for this frame
  (`g.modes.UpdateContext(g.modeContext())`), then calls
  `g.modes.Update()` — the `game.Manager` state machine (see
  [Game orchestration](#game-orchestration-modes)).
- `game.Draw(screen *render.Frame)` calls `g.modes.Draw(screen)`;
  `DrawOverlay`/`DrawUIOverlay` are separate passes for 2D overlays and UI
  widgets layered on top of the 3D world (see `render/frame.go` — a `Frame`
  batches `WorldCommand`/`WorldMeshCommand`/`UIRectCommand`/etc. rather than
  issuing GPU calls directly; `game`/`ui` never touch the GPU API).
- `FrameSubmitted()` is called once the frame has actually been handed to
  the GPU, letting modes release per-frame-only state.

**Headless mode** (`render/headless.go`, `RunHeadless`): a plain
60 Hz `time.Ticker` loop calling the same `Update`/`Draw*`/`FrameSubmitted`
sequence but against a `render.NewFrame` that is *never* presented (no
window, no GPU context created at all). `app.Game.Draw`/`DrawOverlay`/
`DrawUIOverlay` early-return when `cfg.Headless` is true, so headless runs
still exercise all game logic and packet handling but skip building draw
commands. Headless mode is used for integration tests
(`app/headless_test.go`, `config/headless_test.go`) and for
scripted/CI runs (`--headless` requires `--autologin`-equivalent settings —
see `config/config.go`'s `applyCLI`, which forces `AutoLogin`, disables
audio, and requires a login/password/char-slot when headless).

## Configuration

`config.Config` (`config/config.go`) is a single struct fanning out into
`WindowConfig`, `PacketConfig`, `LoginConfig`, `AudioConfig`, `RenderConfig`,
`NetworkConfig`, `FogConfig`, `GameplayConfig`, `ScriptConfig`,
`glog.LogConfig`, and `ChatShortcuts`.

**Precedence** (`LoadConfig`, low → high), matching the README:

1. Built-in defaults (`defaultConfig()`).
2. `./goro.ini` in the working directory, if present.
3. `<data-dir>/goro.ini`, if `--data-dir` was passed on the CLI (parsed in a
   first, throwaway CLI pass just to discover `--data-dir`/`--config`
   before the real merge).
4. The file passed via `--config <path>` explicitly (`explicit=true` — a
   missing explicit file is an error, unlike the implicit ini files above).
5. Command-line flags, applied last via `applyCLI` → `parseCLI` (Go's
   `flag` package; `--` stops flag parsing, matching Unix convention).

This exact order is pinned by `config/precedence_test.go`
(`TestConfigPrecedence`) — read that test if you need to change precedence
or add a new source. `cfg.ConfigPath` ends up holding whichever file was
actually selected, and that is the file `SaveUserSettings`/`SaveLoginID`
write back to later (`config/config.go:179`, `:210`) — so in-game
"remember my settings" writes land in `--config`'s file if given, else
`<data-dir>/goro.ini`, else `./goro.ini`.

INI parsing is hand-rolled (`applyINI`, `applyConfigValue` — a big
`section.key` switch) rather than using a library; unknown keys are a hard
error, which catches typos in `goro.ini` early. `chatshortcuts` is a
special section keyed by digit (see `config/chat_shortcuts.go`).

`config` must not depend on game runtime state (`config/doc.go`) — it is a
pure leaf package, loaded before anything else exists.

## Package dependency layering

Reading every `doc.go` boundary comment together gives this dependency
direction (lower packages never import higher ones):

```
 leaf / infrastructure   config  input  glog  db  render  res  network  audio
                          (no upward deps; render/res/network/audio may use glog)
                                    │
 shared runtime state     client  session  world
                          (client.Context references the leaves above;
                           session/world hold pure map/account state)
                                    │
 orchestration             game  ui
                          (game composes world+network+res+audio+render+ui;
                           ui may read session/client but not own gameplay)
                                    │
 composition root          app
                          (owns one of everything, builds client.Context)
                                    │
 process entry              main
```

Concretely, per `go.mod` module `github.com/kivutar/goro`, and confirmed by
import lists in the files read above:

- `render` imports `client`, `config`, `input` — but never `game`, `app`,
  `world`, or `session`. It is intentionally generic: `Frame` is just typed
  draw-command buffers (`DrawCommand`, `WorldCommand`, `WorldMeshCommand`,
  `UIRectCommand`, ...) that any caller can populate.
- `network` imports only `glog` — it knows nothing about `game`, `world`,
  or `session`; it exposes parsed packet structs and lets callers decide
  policy (`network/doc.go`).
- `res` has no dependency on `game`/`world`/`render` — it is pure data
  loading/decoding.
- `world` imports `res` (map data references `*res.GAT`/`*res.GND`/`*res.RSW`)
  but not `game`, `network`, `render`, or `ui`.
- `client` imports `audio`, `config`, `input`, `network`, `res`, `session`,
  `world` — it is the "assembly" of leaf/mid-level state into one context
  struct, plus two small interfaces (`UIApp`, `UIManager`, `RuntimeSettings`)
  that let `game`/`ui` talk to the render/runner layer without importing it.
- `game` imports `client`, `db`, `network`, `render`, `res`, `session`,
  `ui` (aliased `gameui`), `world` (aliased `worldstate`) — it is the
  orchestrator that is allowed to reach into everything.
- `app` imports `game`, `ui`, `client`, `config`, `render`, `res`,
  `session`, `world`, `network`, `audio`, `input` — the composition root.

If you're deciding where a new piece of logic belongs: if it needs to touch
GPU/draw calls, it's `render`; if it decodes a wire packet, it's `network`;
if it decodes a game data file, it's `res`; if it's "what's true about the
current map/actors right now," it's `world`; if it's "what's true about my
account/character/inventory," it's `session`; if it's a reusable widget or
window, it's `ui`; everything else — input handling, deciding which packet
handler or skill to run, gluing subsystems together — is `game`.

## The shared runtime context (`client.Context`)

`client/context.go` defines:

```go
type Context struct {
    Config            config.Config
    Input             *input.State
    Resources         *res.Manager
    Session           *session.Session
    World             *world.World
    Network           *network.Client
    Audio             *audio.BGM
    Started           time.Time
    ScreenW, ScreenH  int
    Runtime           RuntimeSettings   // fullscreen/vsync/fps toggles
    RequestQuit       func()
    RequestScreenshot func() (string, error)
    UIApp             UIApp             // set once the renderer exists
    UIManager         UIManager
}
```

It is a **value type**, rebuilt fresh every frame by
`app.Game.modeContext()` (`app/app.go:229`) and pushed into the mode
manager via `UpdateContext`. All the pointer fields (`*session.Session`,
`*world.World`, `*network.Client`, `*audio.BGM`, `*input.State`) are stable
long-lived objects owned by `app.Game` — `Context` is just a bundle of
references, not a copy of the state. So mutating `ctx.Session.PlayerX` from
deep inside a `ui` window genuinely mutates the one true session.

`client.Context` is what nearly every function signature in `game` and
`ui` takes as its first argument (`func (m *WorldMode) Foo(ctx client.Context, ...)`).
This is the mechanism by which `game`/`ui` reach `res`, `network`, `audio`,
`session`, `world`, and the render-facing hooks without importing `app`.

## Session, World, and the game/render/input handoff

- **`session.Session`** (`session/session.go`) is server-authoritative
  account/character state: login/char IDs, guild, inventory, cart, storage,
  stats, skills, hotkeys, quests, party, friends, whisper settings,
  companions. `Session.SelectCharacter` resets nearly everything to a clean
  slate on character select — read that function to see the full list of
  per-character state that must not leak across character switches.
  `EstimatedServerTick`/`SyncServerTick` implement clock sync against the
  map server's tick counter (used for e.g. cast-time/cooldown UI without a
  round-trip per frame).
- **`world.World`** (`world/world.go`) is the current map: `Actors
  map[uint32]Actor`, `Items map[uint32]FloorItem`, `Camera`, and loaded map
  resources (`*res.GAT` collision grid, `*res.GND` ground mesh, `*res.RSW`
  scene/world file, `RSM map[string]*res.RSM` models keyed by path).
  `world.Actor` carries both current state and movement-interpolation
  fields (`FromX/FromY/ToX/ToY/MoveStartTick/MovePath/...`) — movement is
  server-authoritative but client-interpolated between server ticks.
  `MapProperty` encodes PvP/GvG/WoE zone rules with helper predicates
  (`IsPvP`, `IsGvG`, `PlayerCombatEnabled`, `IsAutoReviveRestricted`) used
  throughout `game` for combat gating.
- **`input.State`** (`input/input.go`) is a backend-neutral per-frame
  snapshot: current/previous/just-pressed key and mouse-button maps (both a
  small custom `Key` enum and the full `gpucontext.Key` set via
  `KeyCode = gpucontext.Key`), mouse position/delta/wheel, touch points,
  pending text input runes. `render.wireInput` (in `render/backend.go`)
  feeds OS events from GoGPU into this struct each frame;
  `input.State.EndFrame()` (called by `app.Game.Update`'s deferred call)
  rotates "current" into "previous" so `JustPressed`-style queries work.
  `input` must not know about RO gameplay, rendering, or packets
  (`input/doc.go`) — it is a pure event/state abstraction.
- **`render.Frame`** (`render/frame.go`) is the per-frame command buffer:
  `game`/`ui` call `frame.DrawImage`, `DrawTriangles3D`, `DrawWorldMesh`,
  `DrawWorldBillboard`, etc. to *describe* what to draw; nothing is
  actually rendered until the `render` package's GPU backend consumes the
  buffer after `Draw`/`DrawOverlay`/`DrawUIOverlay` return. This
  command-buffer indirection is exactly why `render` can be swapped for a
  no-op headless discard (`RunHeadless`) without `game`/`ui` changing at
  all — see [GRAPHICS.md](GRAPHICS.md) for what happens to the buffer past
  this handoff.

## Game orchestration: modes

`game.Manager` (`game/manager.go`) is a tiny finite-state machine over a
`Mode` interface:

```go
type Mode interface {
    Name() string
    Enter(client.Context) Mode   // may redirect to a different mode
    Update(client.Context) (Mode, error)
    Draw(client.Context, *render.Frame)
}
```

Optional interfaces a mode may also implement: `overlayMode` (`DrawOverlay`),
`uiOverlayMode` (`DrawUIOverlay`), `frameSubmittedMode` (`FrameSubmitted`),
plus (checked by `app.Game`, not `Manager`) `HandleKeyPress`,
`PrepareTextInput`, `PrepareKeyInput` for OS-level key routing before the
normal update tick.

`Manager.enter(mode)` loops while `Enter` keeps returning a different mode —
this lets a mode refuse entry and redirect (e.g. "no character selected,
go back to character select") without the caller needing special-case
logic. `Manager.Update()` calls the active mode's `Update`; if it returns a
non-nil `Mode`, the manager transitions (`m.enter(next)`) before the next
frame.

There are two top-level modes today: `LoginMode` (`game/login.go` —
connection/login/char-select/char-create flow, ~ the whole pre-map-server
experience) and `WorldMode` (`game/world.go`, `game/world_packets.go`, and
many sibling files — the entire in-map gameplay loop). `WorldMode` is
large; it is split across many files by concern (`actor.go`, `battle.go`,
`camera.go`, `bot*.go`, `effects*.go`, `cart.go`, `friends.go`, `gnd.go`
(ground rendering hookup), `adoption.go`, etc.) but they're all methods on
the same `WorldMode` struct declared in `game/world.go`. When looking for
"where does X happen in-game," grep `game/*.go` for the feature name first —
file names closely track RO feature names (e.g. `chat_room.go`,
`disconnect_dialog.go`, `equipment_view.go`, `emotions.go`).

Mode transitions between `LoginMode` ↔ `WorldMode` happen when: login/char
selection completes (`LoginMode.Update` returns a fresh `WorldMode`) or the
world disconnects/the player logs out (`WorldMode` returns `nil` or a new
`LoginMode`, see `handleNetworkDisconnectErrors` in `game/world_packets.go`).

## Networking

`network.Client` (`network/client.go`) owns one TCP connection at a time
(`Connect`/`Close`), a dedicated goroutine pair (`readLoop`/`writeLoop`), a
bounded outbound queue (`sendCh`, size 256, `Send` returns `"send queue
full"` rather than blocking), and a `*Framer` that turns the raw byte
stream into discrete `Packet{ID uint16, Data []byte}` values.

**Framing** (`network/framer.go`): RO's protocol has fixed-length packets
for most IDs and variable-length ones (length prefix at bytes 2-3) marked
by `-1` in the `LengthTable` (`network/packet.go`,
`PacketLengths2008()` — one big map from opcode to length, one entry per
known packet ID for the pinned 2008 packet version). `Framer.Push` accumulates
bytes and greedily extracts as many whole packets as are buffered;
`resyncToKnownPacket` scans forward for a plausible header if an unknown ID
is hit, so one corrupt/misparsed packet doesn't wedge the connection
forever, though it does return an error for the caller to log.

`Client.Pump()` (called once per frame from `app.Game.Update`) is the
frame-boundary between the async socket goroutines and the synchronous
game loop: it moves anything the reader goroutine has accumulated into
`c.packets`/`c.errs` under the mutex. `DrainPackets()` and `DrainErrors()`
hand the accumulated slices to the caller and reset them — always called
from `game` (`game/world_packets.go:18`, `game/login.go:215`), never from
`render` or `ui`.

**Sending**: `network.Client` exposes one `SendXxx(...)` method per
outbound packet family (`SendWalkToXY`, `SendActionRequest`,
`SendUseSkillToID`, `SendChangeCart`, ...) — see the full list via `grep
"func (c \*Client) Send" network/client.go`. Each builds the wire bytes
(typically via a `BuildXxxPacket` helper in the matching `*_packets.go`
file) and calls `c.enqueue`.

**Receiving / decoding**: each `*_packets.go` file (there is one per
feature area: `login_packets.go`, `map_packets.go`, `actor_packets.go`,
`chat_packets.go`, `equipment_packets.go`, `guild_packets.go`,
`quest_packets.go`, `pet_packets.go`, `companion_packets.go`,
`mail_packets.go`, `friend_packets.go`, `party_packets.go` via
`ranking_packets.go`/`pvp_packets.go`, etc.) exposes `ParseXxx(pkt
network.Packet) (Result, bool, error)` functions: `bool` says "this
packet's ID matched," `error` is a decode failure for a matching ID.
`game/world_packets.go`'s `handleNetworkPacket` is a long chain of:

```go
if result, ok, err := network.ParseSomething(pkt); err != nil {
    glog.Errorf("parse ... 0x%04X: %v", pkt.ID, err)
} else if ok {
    m.applySomething(ctx, result)
    return nil, false
}
```

— i.e. try each known parser in turn until one claims the packet ID. This
linear chain is simple but means parser order can matter if two families
ever claimed the same opcode (they should not — opcodes are unique per
packet version). Adding a new packet type means adding one `ParseXxx`
function to the right `*_packets.go` file and one new arm to this chain
(see [How to: add a new packet type](#how-to)).

**Connection lifecycle**: `network/map_keepalive.go` runs a heartbeat
goroutine on the map connection specifically (`startMapKeepalive`,
`defaultMapKeepaliveInterval = 5s`) and treats silence beyond
`mapReadTimeout = 10s` as a disconnect — tied to the `net.Conn` itself so a
reconnect can't have keepalive goroutines from the old connection
interfering (`c.conn != conn` guards throughout). `network/doc.go`'s
boundary: `network` "should expose parsed protocol facts without making UI
or gameplay policy decisions" — decoding lives here, but *what to do* about
a parsed packet (show a dialog, update HP, play a sound) is `game`'s job.

## Resource loading (`res`)

`res.Manager` (`res/manager.go`) is constructed once
(`res.NewManager(cfg.DataDir)`) and owns the root data directory, the
parsed `clientinfo.xml`/`sclientinfo.xml` (`ClientInfo`, including login
server connection info used to pre-fill the login window), and any opened
GRF archives (`[]*GRF`). `Manager.Find`/`ReadFile`/`ReadFileExact`/
`ReadFirst` resolve a logical RO path (e.g. `data\texture\...\foo.bmp`,
using either slash style) against, in order, loose files under `Root` and
then the loaded GRF archives — mirroring the original client's loose-file-
overrides-archive behavior, which is what lets a server/client pack
"patch" the data directory without repacking the GRF. `clientinfo.xml`
discovery order is exactly what the README documents:
`data/clientinfo.xml`, `data/sclientinfo.xml`, `clientinfo.xml`,
`sclientinfo.xml`, `System/clientinfo.xml`, `System/sclientinfo.xml`.

Beyond file lookup, `Manager` lazily loads and caches a large set of
side-table data the client needs at runtime — accessory names, item
metadata, non-PC resource names, "is this RSW an indoor map" flags, camera
view points per map, message strings (`msgstringtable.txt`), fog
parameters per map, skill resource names/max levels/descriptions, skill
tree positions, song/pet talk lines, quest metadata, world-map entries —
each guarded by its own `xLoaded bool` so the (often slow) parse happens
at most once per process. If you're adding a new piece of static
client-side (not server-sent) RO data that lives in a data file rather than
Go source, this lazy-cache pattern in `res/manager.go` is the place to
follow.

`res` also contains the decoders for every binary/text RO format:
`grf.go`/`grf_decrypt.go`/`grf_writer.go` (GRF archives),
`gat.go` (collision grid), `gnd.go` (ground mesh), `rsw.go` (world/scene),
`rsm.go` (3D models), `gr2.go`+`gr2_*.go` (Granny3D models used by newer
assets, including Bink/Oodle-compressed variants), `act.go`/`image.go`
(sprite actions/frames + palette-indexed images), `pal.go` (palettes),
`imf.go`, `lub.go` (compiled Lua item scripts), `msgstring.go`, `quest.go`,
`pet_talk.go`, `book.go`, `item_resource.go`, `nonpc_resource.go`,
`player_resource.go`, `skill_resource.go`, `clientinfo.go`. Format
internals are covered in [GRAPHICS.md](GRAPHICS.md) where relevant (GAT,
GND, RSW, RSM, GR2, ACT/sprite images); this document only cares that
`res` is the single place those formats are parsed, and that `game`/`ui`
consume the *decoded* Go structs, never raw bytes.

## UI system

`ui` (`ui/doc.go`) holds "client UI composition, modal/menu/window state,
and reusable skinning primitives." It may read `client`/`session`/`network`
data to *display* it, but must not own map drawing, sprite rendering, or
gameplay simulation — those stay in `game`/`render`/`world`.

The dominant pattern, visible in `ui/character_window.go`
(`CharacterWindow`) and repeated across `ui/*_window.go`: a struct embeds a
shared `Window` base (open/close/position/publish plumbing) plus a
`snapshot string` used as a cheap dirty-check. `Update(ctx Context) bool`:

1. Ensures the window/its size exists (`EnsureWindow`).
2. Computes a snapshot of the underlying session/world data
   (`characterWindowSnapshot(ctx.Session)`), and only rebuilds the widget
   tree (`w.body.replace(...)`, `invalidateWindowLayout`) when the snapshot
   actually changed — avoiding rebuilding widget trees every frame for data
   that rarely changes.
3. Delegates to the embedded `Window.Update(ctx)` for generic window
   behavior, then `Publish(ctx)` to hand the (possibly unchanged) widget
   tree to the renderer.

Windows are built on `github.com/gogpu/ui` (`widget`, `state`, `event`,
`geometry`, `primitives` packages) — an immediate/retained-ish hybrid UI
toolkit; `ui/rotheme` provides RO-flavored theming on top of it. There is
one `*Window` type per RO window: inventory, equipment, character,
friends, guild, party, chat room, book/library, card composition/
illustration, homunculus info/skills, quest journal, hotkeys/shortcut
bar, escape/context menus, confirm modals, and more — see the `ui/`
listing for the full set (roughly 60+ files). `ui.Manager`
(`gameui.NewManager()` in `app/app.go`) tracks overlay widgets
(`AddOverlay`/`RemoveOverlay`/`Clear`) that render above the normal window
stack (used for e.g. floating context menus).

`game` owns *when* a window opens/closes and *what* it's told to display
(gameplay policy); `ui` owns *how* it's laid out and styled. A bug where a
window shows stale data almost always means the snapshot/dirty-check logic
in the relevant `ui/*_window.go` file didn't detect a change — check what
fields the snapshot function reads vs. what actually changed.

## Static game data (`db`)

`db` (`db/doc.go`) holds tables "adapted from robr's DB directory" — i.e.
ported from another open RO client's reference data, not scraped at
runtime. Contents: `skills.go` (one `uint16` constant per skill ID plus,
presumably, more, — the biggest file), `skill_info.go`, `skill_tree.go`,
`skill_units.go`, `job.go`, `item_tables.go`, `monster_tables.go`,
`weapon.go`/`weapon_action.go`, `mount.go`, `status.go`/`status_icons.go`,
`emotions.go`, `effects.go`, `hit_sounds.go`, `pet_actions.go`. This is
pure static data + small lookup helpers, safe to import from both `game`
and `ui` (and is — e.g. `ui/character_window.go` imports `db`). If a
number/name/ID used in gameplay logic looks like it should come from the
official client tables rather than be computed, check `db` first before
hardcoding it elsewhere.

## Audio

`audio.BGM` (`audio/bgm.go`) plays background music and, per `sfx.go`,
sound effects, decoding through `godexture` (an MP3-capable audio SDK) and
outputting via `ebitengine/oto/v3`. Note the build tag:
`audio/bgm.go`/`audio/sfx.go` carry `//go:build nofakecgo` and there is a
`audio/bgm_stub.go` providing a no-op/alternate implementation for builds
without that tag — this is part of how the project stays CGO-free and
statically compiled (see [Build and packaging](#build-and-packaging)).
`audio/volume.go` handles volume clamping/scaling shared by both BGM and
SFX paths.

Per `audio/doc.go`: audio "should stay focused on decoding, volume, and
playback; gameplay decisions about which sound to trigger belong in game."
`game` calls into `ctx.Audio` to say *play this file now*; it never asks
`audio` to interpret game state itself.

## Input

Covered above under [Session, World, and the game/render/input
handoff](#session-world-and-the-gamerenderinput-handoff). Key files:
`input/input.go` (the `State` snapshot type and query methods),
`input/keycode.go` (key-code tables/aliases).

## Bot / Lua scripting

`--script <path>` (README) points at a Lua file executed by
`game/bot.go`'s `luaBot`, built on `github.com/yuin/gopher-lua`. Each
`WorldMode.updateBot` tick (every `botTickInterval = 150ms`,
`game/bot.go:18`) loads/reloads the script if the configured path changed,
then calls `bot.tick()`. `bot.registerAPI(ctx, mode)` (see `game/bot.go`
and sibling `game/bot_movement.go`/`game/bot_targeting.go`) exposes a Lua
API surface for scripts to query world/session state and issue movement
and targeting commands; `game/bot_keyboard.go` additionally lets a script
drive keyboard-equivalent input for cases the movement/targeting API
doesn't cover (`updateBotInput`, gated by `keyboardAvailable`). Example
scripts live in `scripts/` (`wasd.lua` — manual keyboard-style control
exposed to a human via the script layer; `loot-and-attack.lua` — an actual
autonomous loop). A malformed or crashing script disables itself
(`m.bot.disabled = true`) rather than crashing the client; check
`glog.Warnf("lua script ... failed")` output first when a script silently
stops doing anything.

## Standalone tools (`cmd/`)

Two small CLI utilities independent of the main client binary, both
thin wrappers around `res` GRF support:

- `cmd/grf-extract` — extracts every file in a `.grf` archive to a
  directory (`res.OpenGRF`, then walk and write files out).
- `cmd/grf-pack` — the inverse: builds a `.grf` archive from a directory,
  via `res/grf_writer.go`.

Useful for inspecting/repacking client data during development without
touching the main game loop at all.

## Logging (`glog`)

Thin wrapper (`glog/logging.go`) around `github.com/charmbracelet/log`
(and bridged into `log/slog` via `slog.SetDefault`). `glog.Configure(cfg)`
sets the level (`debug`/`info`/`warn`/`error`/`fatal`) and, if
`cfg.Log.File` is set, tees output to both stderr and that file
(`io.MultiWriter`). Call sites use `glog.Debugf`/`Infof`/`Warnf`/`Errorf`/
`Fatalf` throughout every package — it's the one cross-cutting dependency
every other package is allowed to take (see the layering diagram; `glog`
is a true leaf).

## Testing conventions

- Nearly every package has a `_test.go` per source file
  (`actor_test.go`, `battle_mode_test.go`, ...) — this project favors many
  small, colocated test files over few large ones. When adding a feature,
  add or extend the matching `_test.go`.
- `app/headless_test.go` and `config/headless_test.go` exercise the
  headless run path end-to-end (no GPU/window needed), which is also how
  this architecture doc's claims about the `Update`/`Draw` cycle were
  verified — read them for a concrete "spin up the whole app and drive N
  frames" example.
- `config/precedence_test.go` (`TestConfigPrecedence`) is the authoritative
  spec for config-source precedence; don't change precedence without
  updating it.
- `res/*_real_test.go` (e.g. `gnd_real_test.go`, `rsw_real_test.go`,
  `item_resource_real_test.go`, `gat_real_test.go`, `msgstring_real_test.go`,
  `accessory_real_test.go`) are gated behind a real, legally-obtained RO
  client data directory: `realDataManager(t)` in `res/real_data_test.go`
  calls `t.Skip(...)` unless the `GORO_DATA_DIR` environment variable is
  set. Run these locally with `GORO_DATA_DIR=/path/to/RO/data go test
  ./res/...` when validating format-decoding changes against actual game
  assets; they are skipped (not failed) in normal CI/dev runs that lack
  the data.
- Standard `go test ./...` (with `CGO_ENABLED=0 -tags nofakecgo` to match
  the real build, see below) runs everything else, including the headless
  app/game integration tests.

## Build and packaging

- **Pure Go, no CGO**: `CGO_ENABLED=0 go build -tags nofakecgo .` is the
  canonical build command (README). The `nofakecgo` build tag selects the
  real audio backend (`audio/bgm.go`/`sfx.go`) over the stub
  (`audio/bgm_stub.go`); omit it and audio compiles to a no-op, which is
  useful for environments where the audio dependency chain is
  unavailable. This is also why cross-compilation and static deployment
  are simple — there is no C toolchain dependency anywhere in the build.
- `internal/appicon` embeds/generates the application icon
  (`icon.png` + `generate.go`/`icon.go`) used both for the window icon
  (`render/backend.go`'s `WithIcon(appicon.Image())`) and for packaging.
- `packaging/linux/` (`goro.desktop`, `goro.png`) and `packaging/windows/`
  (`goro.ico`) hold OS-specific packaging metadata/icons, not build logic —
  actual packaging is presumably driven by external scripts/CI (check
  `.github/` and `scripts/` if you need to reproduce a release build).
- `doc.go` at the repo root documents `package main` itself (Go convention
  for a package-level doc comment when `main.go` has none): "wires
  configuration, the application model, and the renderer. Runtime state
  belongs in app, game, world, or session rather than here" — i.e. don't
  grow `main.go`; if you're tempted to add a field or a loop to it, that
  state belongs in `app.Game` instead.

## Key third-party dependencies

From `go.mod`:

- **`github.com/gogpu/*`** (`gogpu`, `gpucontext`, `gputypes`, `naga`,
  `ui`, `wgpu`, `gg`) — the whole windowing/GPU/2D-canvas/UI-toolkit stack
  this project is built on. `gogpu` is the app/window/event shell,
  `wgpu`/`gpucontext`/`gputypes`/`naga` are the WebGPU-style GPU
  abstraction and shader IR, `gogpu/ui` is the retained-ish widget toolkit
  `ui/*_window.go` is built on, `gogpu/gg` is a 2D vector-graphics/canvas
  library used for UI rasterization. Several are pinned to forked
  `github.com/kivutar/...` replacements in `go.mod`'s `replace` directives
  — check those forks first if you hit a gogpu-stack bug that looks
  upstream-caused.
- **`github.com/charmbracelet/log`** — the logging backend behind `glog`.
- **`github.com/yuin/gopher-lua`** — the embedded Lua VM behind the bot
  scripting layer (`game/bot.go`).
- **`github.com/godexture/*`** (`core`, `sdk`, `codec-mp3`,
  `format-mp3`, `metadata-id3`, pinned via `replace` to
  `github.com/godexture/godec/...`) — the audio decode/playback SDK behind
  `audio/bgm.go`/`sfx.go`.
- **`github.com/ebitengine/oto/v3`** — low-level cross-platform audio
  output (what `godexture` ultimately writes PCM to).
- **`golang.org/x/image`**, **`golang.org/x/text`** — image decoding
  helpers and text/encoding utilities (RO client text is not always UTF-8).

## How to...

**Add a new incoming packet type.** Pick the right `network/*_packets.go`
file by feature area (or create one for a genuinely new feature area).
Add a `ParseXxx(pkt network.Packet) (Result, bool, error)` function there
(look at a neighboring `ParseXxx` in the same file for the binary-layout
convention — little-endian, `encoding/binary`). Add the opcode to
`PacketLengths2008()` in `network/packet.go` if it isn't there yet (fixed
length, or `-1` for variable-length with a length prefix at bytes 2-3).
Then, in `game/world_packets.go`'s `handleNetworkPacket` (or
`game/login.go` for pre-map packets), add an `if result, ok, err :=
network.ParseXxx(pkt); ...` arm that applies the result to
`session`/`world`/`ui` state as appropriate. Write a
`network/xxx_packets_test.go` round-trip test (encode then parse, or parse
a captured byte sequence) alongside the existing `*_packets_test.go` files.

**Add a new outgoing packet.** Add a `BuildXxxPacket(...)  []byte` helper
in the matching `network/*_packets.go` file, then a `func (c *Client)
SendXxx(...) error` method in `network/client.go` that builds and
`enqueue`s it (copy an existing `SendXxx` for the pattern). Call it from
the relevant `game` mode in response to input or a UI action.

**Add a new UI window.** Create `ui/foo_window.go` following the
`CharacterWindow` pattern in `ui/character_window.go`: embed `Window`,
track a dirty-check `snapshot`, build a widget tree in `Update`. Wire it
into whichever `game` mode should own opening/closing it (typically
`WorldMode`, stored as a field alongside the other `*gameui.FooWindow`
fields near `WorldMode`'s UI substructures) — `game` decides *when* it's
visible, `ui` decides *how* it looks.

**Add a new gameplay feature (e.g. a new skill effect, a new
window-triggering packet, a new bot API).** Start in `game/`: find the
sibling file closest to the feature (`battle.go` for combat math,
`effects*.go` for visual/status effects, `bot_targeting.go` for bot
targeting primitives, etc.), or add a new `game/feature_name.go` file. Pull
any static IDs/names from `db` rather than hardcoding them. If the feature
needs new server data, that's a new packet (see above); if it needs new
client asset data, check whether `res.Manager` already loads it before
adding a new lazy-cache field there.

**Debug: "a server packet doesn't seem to do anything."** Turn on
`--net-trace` (`cfg.Network.Trace`, logs every enqueue/read via `glog`) to
confirm the packet is actually arriving and its opcode. Then check
`PacketLengths2008()` has the right entry (wrong length ⇒ the framer either
waits forever for more bytes or misframes the next packet — the "unknown
packet id" error from `Framer.Push`/`resyncToKnownPacket` is the symptom of
the latter). Then confirm a `ParseXxx` for that opcode exists and is wired
into the `handleNetworkPacket` chain in `game/world_packets.go` (or
`game/login.go` if it's pre-map).

**Debug: "a UI window shows stale/wrong data."** Find the window's
`ui/*_window.go` file and check its snapshot function — if it doesn't read
the field that changed, the window won't know to rebuild. Also check that
`game` is actually updating the underlying `session`/`world` field from
the relevant packet handler in the first place (i.e. rule out a networking
bug before assuming it's a UI bug).

**Debug: "a config option isn't taking effect."** Check precedence first —
a lower-priority source can't override a higher one, and
`config/precedence_test.go` documents the exact order. Confirm the flag/ini
key is spelled as `applyConfigValue`/`parseCLI` expect (unknown ini keys
are a hard error, so a typo in `goro.ini` will surface as a startup error,
not silent ignoring — but a typo'd *flag* name is a `flag` package error
too). If it's a value meant to persist across runs (fullscreen, volume,
gameplay toggles), confirm it's actually one of the keys
`SaveUserSettings`/`SaveLoginID` write (`config/config.go`) — not every
`Config` field is currently persisted.

**Run the app locally against a dev server / add a new
map-server-dependent feature.** See `docs/rathena-setup.md` and
`docs/client-setup.md` for standing up a local rAthena + client data
directory; `docs/companions-20080910.md`, `docs/woe-20080910.md`,
`docs/quest-journal.md`, `docs/world-map.md`, etc. document feature-specific
implementation notes and TODOs worth reading before extending those
systems (there is also a repo-root `TODO.md` and several `docs/*-todo.md`
gap-checklists tracking known missing reference-client behavior).
