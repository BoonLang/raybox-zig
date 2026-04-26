# Raybox-Zig Implementation Plan

Place this file at the root of `raybox-zig` as `AGENTS.md` or `RAYBOX_ZIG_IMPLEMENTATION_PLAN.md`.

This repo exists to build the Zig/Sokol/Raybox equivalent of the current Rust Boon playground:

> All upstream Boon playground examples must compile, run, and render through Raybox in both a native window and a browser canvas.

The implementation must not become a generic engine experiment. Keep the stack fixed, implement the required Boon playground behavior, and verify every imported example.

---

## 0. Non-negotiable outcome

The final app is:

```text
boon-playground-raybox
```

It has one UI renderer and one preview renderer:

```text
Boon source project
  -> boon-zig compile/run
  -> DocumentSnapshot or SceneSnapshot
  -> Raybox document adapter
  -> Clay layout
  -> Sokol renderer
  -> native window or browser canvas
```

Native and browser must render the same playground shell and the same Boon preview:

```text
native:
  zig build run-playground

browser:
  zig build run-playground -Dtarget=wasm32-emscripten
```

The browser preview must not use HTML/DOM rendering. The native preview must not use a separate UI implementation. Both targets use the same Raybox adapter and renderer.

The Boon examples stay Boon programs. Do not manually rewrite examples as Zig widgets.

---

## 1. Fixed v0 technology stack

Use exactly this stack for the first complete implementation:

```text
Language:
  Zig

Platform/app:
  sokol_app through sokol-zig

GPU:
  sokol_gfx through sokol-zig

Browser:
  wasm32-emscripten + WebGL2

Layout:
  Clay

Shaders:
  sokol-shdc annotated GLSL -> generated Zig shader module

Text:
  FontStash + stb_truetype dynamic bitmap glyph atlas

Assets:
  sokol_fetch

Rendering:
  CPU-generated 2D geometry + Sokol batches
```

Do not add these before the definition of done passes:

```text
SDL
raylib
raygui
Dear ImGui
Slang
direct WebGPU
Dawn
wgpu-native
freestanding Wasm
MSDF text
general 3D modeling
3D printing/export
general polygon boolean library
```

This is not because those are bad; it is because this implementation must first finish the Boon playground and the physical TodoMVC example without backend churn.

---

## 2. Source-of-truth corpus

The source of truth is the upstream Rust Boon playground repository:

```text
https://github.com/BoonLang/boon
```

Import examples from:

```text
playground/frontend/src/examples/
```

Generate a manifest. The manifest is authoritative. No example may be silently skipped.

### 2.1 Current known single-file example groups

The current Rust playground embeds main, other, and debug examples. The generated importer must discover the actual files, but this list is a sanity check.

```text
minimal
hello_world
interval
interval_hold
counter
complex_counter
counter_hold
fibonacci
layers
shopping_list
pages
todo_mvc

temperature_converter
crud
timer
flight_booker
circle_drawer
cells
cells_dynamic
latest
text_interpolation_update
then
when
while

list_retain_reactive
list_map_external_dep
list_map_block
list_retain_count
list_object_state
list_retain_remove
filter_checkbox_bug
checkbox_test
chained_list_remove_bug
while_function_call
button_hover_test
button_hover_to_click_test
switch_hold_test
```

### 2.2 Current multi-file example

`todo_mvc_physical` is a hard gate. It is not optional and not a placeholder.

The current Rust playground treats it as an 8-file multi-file project:

```text
RUN.bn
BUILD.bn
Generated/Assets.bn
Theme/Theme.bn
Theme/Professional.bn
Theme/Glassmorphism.bn
Theme/Neobrutalism.bn
Theme/Neumorphism.bn
```

Also import its supporting directories:

```text
assets/icons/
docs/
todo_mvc_physical.expected
README.md
```

The imported project must preserve relative paths exactly.

### 2.3 Manifest format

Generate:

```text
fixtures/corpus_manifest.json
src/playground/generated_example_registry.zig
```

Each entry must include:

```json
{
  "name": "todo_mvc_physical",
  "kind": "multi_file",
  "entry_file": "RUN.bn",
  "files": [
    "RUN.bn",
    "BUILD.bn",
    "Generated/Assets.bn",
    "Theme/Theme.bn",
    "Theme/Professional.bn",
    "Theme/Glassmorphism.bn",
    "Theme/Neobrutalism.bn",
    "Theme/Neumorphism.bn"
  ],
  "expected": "todo_mvc_physical.expected",
  "parser": "NOT_STARTED",
  "runtime": "NOT_STARTED",
  "renderer": "NOT_STARTED",
  "native_window": "NOT_STARTED",
  "browser_canvas": "NOT_STARTED",
  "notes": ""
}
```

Allowed statuses:

```text
NOT_STARTED
PARTIAL
BLOCKED
DONE
```

A release requires every imported example to be `DONE` for parser, runtime, renderer, native window, and browser canvas.

---

## 3. Repository layout

Use this structure:

```text
raybox-zig/
  AGENTS.md
  build.zig
  build.zig.zon

  assets/
    fonts/
      Inter-Regular.ttf
      JetBrainsMono-Regular.ttf
      NotoSans-Regular.ttf

  fixtures/
    corpus_manifest.json
    expected_action_schema.json
    physical_visual_spec.json

  examples/
    upstream/
      minimal/
      hello_world/
      ...
      todo_mvc_physical/

  src/
    main.zig

    app/
      app.zig
      frame_stats.zig
      input.zig
      time.zig

    boon_adapter/
      boon_runtime_host.zig
      document_snapshot.zig
      document_adapter.zig
      style_resolver.zig
      module_project.zig

    playground/
      playground_app.zig
      example_registry.zig
      project_files.zig
      source_editor.zig
      preview_panel.zig
      expected_runner.zig
      semantic_tree.zig
      persistence.zig
      router.zig

    raybox/
      raybox.zig
      ui.zig
      theme.zig
      widgets.zig
      input_model.zig
      custom_commands.zig

    clay/
      clay_bridge.zig
      clay_impl.c
      clay_config.h
      clay.h

    render/
      renderer.zig
      batcher.zig
      geometry.zig
      rounded_rect.zig
      border.zig
      shadows.zig
      images.zig
      svg_minimal.zig
      colors.zig
      oklch.zig

    text/
      font_manager.zig
      fontstash_backend.zig
      text_measure.zig
      fontstash.h
      stb_truetype.h

    assets/
      asset_loader.zig
      data_uri.zig

    shaders/
      ui.glsl

    tools/
      import_upstream.zig
      generate_example_registry.zig
      verify_corpus.zig
      verify_examples.zig
      verify_physical_visual.zig
```

Raybox must not implement the Boon parser/runtime. It consumes `boon-zig`.

Support two development modes:

```text
-Dboon_zig_path=../boon-zig
```

and a pinned git dependency in `build.zig.zon`.

If the required `boon-zig` API is not present, add a small interface proposal in `src/boon_adapter/boon_runtime_host.zig` and fail clearly at compile time. Do not duplicate runtime semantics inside Raybox.

---

## 4. Build commands

Implement these commands:

```bash
zig build run-playground
zig build run-playground -Dtarget=wasm32-emscripten

zig build import-upstream
zig build generate-example-registry
zig build verify-corpus

zig build verify-examples-native
zig build verify-examples-web

zig build verify-physical-native
zig build verify-physical-web

zig build test
```

`run-playground -Dtarget=wasm32-emscripten` must build a web app and serve or open it using the sokol-zig Emscripten link step.

For web builds, use Sokol's Emscripten/WebGL2 path. Do not attempt `wasm32-freestanding`.

---

## 5. App lifecycle

`src/main.zig` contains only Sokol callbacks and delegates to `PlaygroundApp`.

```zig
pub fn init() void {
    app.init();
}

pub fn frame() void {
    app.frame();
}

pub fn event(ev: *const sapp.Event) void {
    app.event(ev);
}

pub fn cleanup() void {
    app.deinit();
}
```

`PlaygroundApp` owns:

```zig
pub const PlaygroundApp = struct {
    allocator: std.mem.Allocator,

    raybox: Raybox,
    renderer: Renderer,
    clay: ClayBridge,
    fonts: FontManager,
    assets: AssetLoader,

    input: InputState,
    project: ProjectFiles,
    examples: ExampleRegistry,
    runtime: BoonRuntimeHost,
    preview: PreviewPanel,
    editor: SourceEditor,
    persistence: PersistStore,

    frame_stats: FrameStats,
};
```

Per-frame order:

```text
1. sfetch_dowork
2. collect input events
3. update playground shell state
4. if run requested, build/compile/run Boon project
5. begin Clay frame
6. declare playground shell UI
7. declare preview UI from DocumentSnapshot/SceneSnapshot
8. end Clay frame
9. render Clay commands through Sokol
10. update frame stats
```

---

## 6. Project model and multi-file build

All examples are represented as projects, even single-file examples.

```zig
pub const ProjectFile = struct {
    path: []const u8,
    contents: []const u8,
    generated: bool = false,
};

pub const Project = struct {
    name: []const u8,
    entry_file: []const u8,
    files: []ProjectFile,
    assets_root: ?[]const u8,
};
```

### 6.1 Module resolution

Match the Rust playground behavior for multi-file examples:

```text
- Entry file is not imported as a module.
- BUILD.bn is not imported as a module.
- Every other .bn file becomes an importable module.
- Module name is the basename without .bn.
```

Examples:

```text
Theme/Theme.bn          -> module Theme
Theme/Professional.bn   -> module Professional
Theme/Glassmorphism.bn  -> module Glassmorphism
Generated/Assets.bn     -> module Assets
```

Functions must be callable as:

```text
Theme/material(...)
Professional/get(...)
Assets/icon()
```

### 6.2 Build file execution

If a project contains `BUILD.bn`, `Run` must execute it before compiling the entry file.

For `todo_mvc_physical`, implement enough build-host builtins to execute `BUILD.bn`:

```text
Directory/entries()
File/read_text()
File/write_text(path, text)
Url/encode()
Text/join_lines()
List/retain()
List/sort_by()
List/map()
Log/info()
Log/error()
Build/succeed()
Build/fail()
FLUSH
FLUSHED
```

The build script must run against an in-memory project VFS with access to imported `assets/icons/`.

Verification for `todo_mvc_physical`:

```text
1. Run BUILD.bn.
2. Confirm Generated/Assets.bn is written.
3. Confirm generated output is semantically equivalent to the checked-in Generated/Assets.bn.
4. Use the generated file for RUN.bn compilation.
```

The generated assets file must expose:

```text
Assets/icon().checkbox_active
Assets/icon().checkbox_completed
```

---

## 7. Runtime bridge contract

Raybox receives snapshots, not AST nodes.

```zig
pub const RuntimeOutput = union(enum) {
    document: DocumentSnapshot,
    scene: SceneSnapshot,
    diagnostics: []Diagnostic,
};

pub const DocumentSnapshot = struct {
    revision: u64,
    root: ValueId,
    values: []const Value,
    links: []const LinkBinding,
    timers: []const TimerBinding,
    route: []const u8,
};

pub const SceneSnapshot = struct {
    revision: u64,
    root: ValueId,
    values: []const Value,
    lights: []const LightValue,
    geometry: GeometryValue,
    links: []const LinkBinding,
    timers: []const TimerBinding,
    route: []const u8,
};

pub const Value = union(enum) {
    none,
    number: f64,
    bool: bool,
    text: []const u8,
    list: []const ValueId,
    record: RecordValue,
    element: ElementNode,
};
```

The Boon runtime owns:

```text
lexing
parsing
module resolution
evaluation
reactive graph
LATEST/HOLD/THEN/WHEN/LINK
timers
router state
persistence
event ordering
```

Raybox owns:

```text
layout
hit testing
text input rendering
focus visuals
mouse/keyboard capture
semantic UI tree
rendering
mapping UI events back to Boon LINK ports
```

Unsupported runtime features must produce explicit diagnostics. The final definition of done requires all imported examples to pass.

---

## 8. Playground shell

The playground shell is also Raybox UI, not DOM.

Required panels:

```text
example sidebar
file tabs
source editor
toolbar
preview panel
diagnostic/output panel
debug frame stats overlay
```

Toolbar actions:

```text
Run
Clear State
Reset Example
Toggle Layout: split / preview-only / code-only
Preview Size
```

Example sidebar sections:

```text
Main
Other
Debug
Multi-file
```

Source editor requirements:

```text
monospace font
line numbers
cursor
selection
copy/paste
scrolling
keyboard editing
basic syntax coloring
error span highlighting
multi-file tabs
dirty file indicator
```

This editor does not need to be a full IDE. It must edit `.bn` files well enough to run playground examples in native and browser.

---

## 9. Clay bridge

Create one Clay context for the playground shell and preview.

```zig
pub const ClayBridge = struct {
    memory: []u8,
    context: *c.Clay_Context,

    pub fn init(allocator: Allocator, fonts: *FontManager) !ClayBridge;
    pub fn beginFrame(self: *ClayBridge, input: InputState, width: f32, height: f32, dt: f32) void;
    pub fn endFrame(self: *ClayBridge) c.Clay_RenderCommandArray;
};
```

Frame sequence:

```text
Clay_SetLayoutDimensions
Clay_SetPointerState
Clay_UpdateScrollContainers
Clay_BeginLayout
declare Raybox UI
Clay_EndLayout
```

Clay text measurement must call `FontManager.measure`. Clay strings are not guaranteed to be null-terminated, so always pass explicit slices.

Do not expose Clay directly to the Boon adapter. The adapter emits Raybox semantic UI nodes.

---

## 10. Raybox UI API

Raybox exposes a semantic API to the playground and Boon adapter.

```zig
pub const Ui = struct {
    pub fn beginRoot(self: *Ui, id: StableId, opts: RootOptions) void;
    pub fn panel(self: *Ui, id: StableId, opts: PanelOptions, body: fn (*Ui) void) void;
    pub fn row(self: *Ui, id: StableId, opts: LayoutOptions, body: fn (*Ui) void) void;
    pub fn column(self: *Ui, id: StableId, opts: LayoutOptions, body: fn (*Ui) void) void;
    pub fn label(self: *Ui, id: StableId, text: []const u8, opts: TextOptions) void;
    pub fn button(self: *Ui, id: StableId, text: []const u8, opts: ButtonOptions) bool;
    pub fn textInput(self: *Ui, id: StableId, state: *TextInputState, opts: TextInputOptions) TextInputEvent;
    pub fn checkbox(self: *Ui, id: StableId, checked: bool, opts: CheckboxOptions) CheckboxEvent;
    pub fn select(self: *Ui, id: StableId, state: *SelectState, opts: SelectOptions) SelectEvent;
    pub fn slider(self: *Ui, id: StableId, value: f64, opts: SliderOptions) SliderEvent;
    pub fn svgSurface(self: *Ui, id: StableId, opts: SvgOptions, body: fn (*Ui) void) SvgEvent;
};
```

Use stable IDs everywhere. For list items, prefer Boon-provided identity such as todo item ID. Do not use transient list index IDs when the item has a durable ID.

---

## 11. Boon element mapping

The Boon adapter must support these values and elements.

### 11.1 Root values

```text
Document/new(root: value)      -> document preview root
Scene/new(root, lights, geometry) -> physical scene preview root
plain number/text/bool root    -> debug/value label
list root                      -> vertical child list
record root                    -> debug/value panel unless recognized as element/style
```

### 11.2 Element mapping

```text
Element/stripe                 -> row or column container
Scene/Element/stripe           -> row or column container with physical style
Element/container              -> panel/block
Scene/Element/block            -> panel/block with physical style
Element/stack                  -> absolute/layered container
Element/text                   -> text node
Scene/Element/text             -> text node with relief/depth support
Element/paragraph              -> wrapped text block
Element/link                   -> text/link button
Element/label                  -> label wrapper and hit target
Element/button                 -> button
Scene/Element/button           -> raised physical button
Element/text_input             -> text input
Scene/Element/text_input       -> recessed physical text input
Element/checkbox               -> checkbox
Scene/Element/checkbox         -> recessed physical checkbox
Element/select                 -> select/dropdown
Element/slider                 -> slider
Element/svg                    -> bounded SVG-like surface
Element/svg_circle             -> circle primitive inside SVG surface
Hidden[text: ...]              -> semantic label, not visible text
Reference[element: ...]        -> semantic accessible label reference
NoElement                      -> no rendered node
```

### 11.3 Style mapping

Support these style fields immediately:

```text
width
height
size
Fill
minimum
maximum
padding
gap
align row/column
background color
background url
font size
font weight
font style
font color
line underline/strike
rounded_corners
border width/color
borders per side
disabled
hovered
focus
scrollbars
transform/move_closer
transform/move_further
move.closer
move.further
move.up
move.down
move.left
move.right
rotate
depth
material
relief
spring_range
```

### 11.4 Colors

Implement OKLCH to sRGB conversion.

Support:

```text
Oklch[lightness: L]
Oklch[lightness: L, chroma: C, hue: H]
hex colors
rgba colors
named text values used by examples
```

All internal renderer colors are premultiplied RGBA.

---

## 12. Physical TodoMVC is a hard gate

`todo_mvc_physical` must render and interact nicely. Do not defer its physical features.

### 12.1 Required project behavior

The example must:

```text
load as a multi-file project
run BUILD.bn
generate Generated/Assets.bn
compile RUN.bn with module files
render Scene/new(...)
show theme buttons
show mode toggle
add todos
toggle todos
filter todos
clear completed
edit todo on double click
show/delete button on hover
switch themes
switch light/dark mode
persist state
clear state
```

Its `.expected` file must pass natively and in browser.

### 12.2 Required physical UI projection

Use a 2.5D physical projection in WebGL2. It must be visually attractive and deterministic.

Implement these style effects:

```text
depth:
  controls bevel strength, shadow strength, and highlight strength

move closer:
  raises the element above its parent; casts a drop shadow

move further:
  recesses the element into its parent; creates an inset shadow

relief: Raised
  text or element appears lifted

relief: Carved[wall: N]
  text or element appears cut into the surface

material.color:
  base color

material.gloss:
  controls highlight strength and surface softness

material.shine:
  adds a secondary clearcoat highlight

material.metal:
  adds subtle colored highlight tint; keep small

material.glow:
  emits focus/active glow

material.transparency:
  makes surfaces translucent while preserving readability

material.refraction:
  approximated by glass highlight and tinted translucent surface
```

Use this visibility-preserving transparency mapping:

```text
if material.transparency is absent:
    alpha = 1.0
else:
    alpha = clamp(1.0 - material.transparency * 0.55, 0.35, 0.9)
```

This keeps Glassmorphism visibly glassy without making the app unreadable.

### 12.3 Required lighting projection

Support at least:

```text
Light/directional
Light/ambient
Light/spot target: FocusedElement
```

Projection rules:

```text
Directional light:
  determines drop-shadow direction, bevel highlight side, and bevel shadow side.

Ambient light:
  softens contrast and adjusts shadow opacity.

Spot light on FocusedElement:
  produces a subtle glow around the focused input/control.
```

Use theme light values. Do not ignore `Theme/lights()`.

### 12.4 Required physical element recipes

#### Scene/Element/stripe and Scene/Element/block

For any element with material/depth/move:

```text
1. draw drop shadow if z > 0
2. draw rounded body
3. draw bevel highlight and bevel shadow
4. draw border/focus/glow if present
5. draw children
```

#### Scene/Element/button

Buttons are raised.

```text
rest:
  body + bevel + small shadow

hovered:
  stronger highlight and slightly higher z if spring_range.extend > 0

pressed:
  lower z and reduced shadow if press/active state is available
```

#### Scene/Element/text_input

Text inputs are recessed.

```text
1. outer rounded body
2. inner cutout well based on padding
3. inner shadow
4. glossy/tinted floor
5. placeholder or text
6. caret and selection
7. focus glow when focused
```

#### Scene/Element/checkbox

Checkboxes are recessed controls.

```text
1. outer rounded/pill body
2. inner well
3. icon from background url or checkmark vector
4. hover/focus glow
```

#### Scene/Element/text with relief

For `relief: Raised`:

```text
draw shadow/offset text layer
draw main text
draw highlight text layer if depth >= 2
```

For `relief: Carved[wall: N]`:

```text
draw dark inset offset
draw main text slightly recessed
draw light upper-edge highlight
```

### 12.5 Theme visual requirements

All four themes must look distinct.

#### Professional

```text
soft rounded edges
subtle shadows
low/medium gloss
warm header color
polished but conservative visual style
```

#### Glassmorphism

```text
translucent panels
visible background through surfaces
bright glass highlights
soft blue/purple tint
selected Glass button has visible outline
```

#### Neobrutalism

```text
high contrast
harder shadows
bold accent colors
square/sharp corners except pill-specific controls
strong outline/edge presence
```

#### Neumorphism

```text
low contrast monochrome surfaces
soft shadows and highlights
gentle rounded edges
controls still readable
```

### 12.6 Physical visual verification

Implement:

```bash
zig build verify-physical-native
zig build verify-physical-web
```

The verifier must run `todo_mvc_physical.expected` and also capture deterministic preview screenshots at:

```text
1000x900 @ scale 1.0
1000x900 @ scale 2.0
```

Required screenshots:

```text
Professional Light initial
Professional Light with two todos
Professional Dark with one todo completed
Glassmorphism Dark
Neobrutalism Light
Neumorphism Light
```

The verifier must also inspect the semantic/render tree. Do not use OCR.

Required assertions:

```text
title "todos" is visible and large
new todo input starts focused
input is visibly recessed: has inner shadow command
main card has non-zero drop shadow command
buttons have raised physical commands
checkboxes have recessed physical commands
theme buttons are visible
selected theme button has outline or selected visual
mode toggle changes label between Dark mode and Light mode
Glass theme surfaces have alpha < 1.0
Neobrutalism main card corner radius is 0 or visually sharp
Neumorphism shadow blur radius is greater than Professional's
at least one glow command exists for focused input
checkbox_completed icon renders after completion
```

If screenshots differ between native and browser, record image diffs and fail unless the difference is below the documented tolerance.

---

## 13. Renderer implementation

### 13.1 Renderer command handling

Render Clay commands:

```text
CLAY_RENDER_COMMAND_TYPE_RECTANGLE
CLAY_RENDER_COMMAND_TYPE_BORDER
CLAY_RENDER_COMMAND_TYPE_TEXT
CLAY_RENDER_COMMAND_TYPE_IMAGE
CLAY_RENDER_COMMAND_TYPE_SCISSOR_START
CLAY_RENDER_COMMAND_TYPE_SCISSOR_END
CLAY_RENDER_COMMAND_TYPE_OVERLAY_COLOR_START
CLAY_RENDER_COMMAND_TYPE_OVERLAY_COLOR_END
CLAY_RENDER_COMMAND_TYPE_CUSTOM
```

Raybox custom commands:

```zig
pub const CustomKind = enum(u32) {
    shadow,
    inner_shadow,
    cutout_rounded_rect,
    bevel,
    glow,
    caret,
    selection,
    svg_circle,
    svg_path,
};
```

### 13.2 Renderer resources

```zig
pub const Renderer = struct {
    pass_action: sg.PassAction,

    solid_pipeline: sg.Pipeline,
    image_pipeline: sg.Pipeline,
    text_pipeline: sg.Pipeline,
    blur_h_pipeline: sg.Pipeline,
    blur_v_pipeline: sg.Pipeline,

    vertex_buffer: sg.Buffer,
    index_buffer: sg.Buffer,

    white_texture: sg.Image,
    glyph_atlas_texture: sg.Image,

    batcher: Batcher,
    shadows: ShadowCache,
};
```

Vertex format:

```zig
pub const Vertex2D = extern struct {
    pos: [2]f32,
    uv: [2]f32,
    color: u32, // premultiplied RGBA8
};
```

Flush batch when:

```text
pipeline changes
texture changes
scissor changes
vertex capacity exceeded
index capacity exceeded
```

### 13.3 Premultiplied alpha

Use premultiplied alpha everywhere.

Blend mode:

```text
src_rgb = one
dst_rgb = one_minus_src_alpha
src_alpha = one
dst_alpha = one_minus_src_alpha
```

Every CPU color conversion must premultiply before writing vertex colors.

### 13.4 Geometry functions

Implement CPU geometry:

```zig
pub fn emitRect(batch: *Batcher, rect: Rect, color: ColorPremul) void;

pub fn emitRoundedRect(
    batch: *Batcher,
    rect: Rect,
    radius: CornerRadius,
    color: ColorPremul,
) void;

pub fn emitBorder(
    batch: *Batcher,
    rect: Rect,
    radius: CornerRadius,
    width: BorderWidth,
    color: ColorPremul,
) void;

pub fn emitImageQuad(
    batch: *Batcher,
    rect: Rect,
    uv: Rect,
    color: ColorPremul,
) void;

pub fn emitCutoutRoundedRect(
    batch: *Batcher,
    outer: Rect,
    inner: Rect,
    outer_radius: CornerRadius,
    inner_radius: CornerRadius,
    color: ColorPremul,
) void;
```

Rounded rectangles are CPU-generated triangle geometry. Use:

```text
segments per quarter circle: 12
antialias fringe: 1 physical pixel
```

Independent borders must support:

```zig
pub const BorderWidth = struct {
    left: f32,
    top: f32,
    right: f32,
    bottom: f32,
};
```

### 13.5 Shadows

Implement cached shadows for rounded rectangles.

```zig
pub const ShadowCmd = struct {
    rect: Rect,
    radius: CornerRadius,
    blur_radius: f32,
    spread: f32,
    offset: Vec2,
    color: ColorPremul,
    cache_id: u64,
};
```

Shadow generation:

```text
1. render rounded-rect alpha mask to offscreen texture
2. blur horizontally
3. blur vertically
4. cache final shadow texture
5. composite behind element
```

Movement does not invalidate the cache. Shape, radius, blur radius, spread, and device scale invalidate the cache.

### 13.6 Shaders

Use a single shader file first:

```text
src/shaders/ui.glsl
```

Programs:

```text
@program solid  solid_vs  solid_fs
@program image  image_vs  image_fs
@program text   text_vs   text_fs
@program blur_h blur_vs   blur_h_fs
@program blur_v blur_vs   blur_v_fs
```

Do not handwrite WGSL. Do not use Slang. Generate the Zig shader module with sokol-shdc.

---

## 14. Text system

Use FontStash + stb_truetype.

```zig
pub const FontManager = struct {
    stash: *FONScontext,
    atlas_image: sg.Image,
    atlas_dirty: bool,

    fonts: std.StringHashMap(FontId),

    pub fn init(renderer: *Renderer) !FontManager;
    pub fn loadFontBytes(self: *FontManager, name: []const u8, bytes: []const u8) !FontId;
    pub fn requestFont(self: *FontManager, path: []const u8, name: []const u8) void;
    pub fn measure(self: *FontManager, text: []const u8, font: FontId, size: f32) Dimensions;
    pub fn draw(self: *FontManager, batcher: *Batcher, text: []const u8, pos: Vec2, style: TextStyle) void;
};
```

Do not pregenerate Unicode ranges. Cache only glyphs actually drawn.

Required test string in native and web:

```text
Příliš žluťoučký kůň úpěl ďábelské ódy.
```

The text system must support:

```text
normal font
monospace font
font size
font color
italic placeholder approximation
underline
strikethrough
selection background
caret
```

If FontStash lacks shaping needed for a current example, add the smallest deterministic shaping support required for that example. Czech text must render correctly with the dynamic atlas.

---

## 15. Images and SVG subset

Implement `background: [url: ...]` for the assets used by current examples.

Required input forms:

```text
data:image/svg+xml;utf8,...
relative asset path
```

Implement `assets/data_uri.zig`:

```text
detect data URI
percent-decode UTF-8 payload
return media type and bytes
```

Implement `render/svg_minimal.zig` for the subset needed by `todo_mvc_physical` generated icons:

```text
<svg width height viewBox>
<circle cx cy r fill stroke stroke-width>
<path fill d="M ... L ... l ... z">
```

For `checkbox_active`, render a stroked circle.

For `checkbox_completed`, render the stroked circle and filled check path.

This minimal SVG renderer is also used by Raybox image backgrounds. Do not add a full SVG dependency.

---

## 16. Input and semantic tree

Every interactive node must be recorded in a semantic tree.

```zig
pub const RenderedNode = struct {
    id: StableId,
    role: Role,
    text: []const u8,
    bounds: Rect,
    visible: bool,
    focused: bool,
    hovered: bool,
    checked: ?bool,
    disabled: bool,
    selected: bool,
    outline_visible: bool,
    input_typeable: bool,
};
```

The expected runner uses this tree, not OCR.

Input events:

```zig
pub const PreviewEvent = union(enum) {
    press: LinkId,
    click: LinkId,
    double_click: LinkId,
    hover: struct { link: LinkId, hovered: bool },
    change_text: struct { link: LinkId, text: []const u8 },
    key_down: struct { link: LinkId, key: Key, text: []const u8 },
    blur: LinkId,
    checkbox_change: struct { link: LinkId, checked: bool },
    select_change: struct { link: LinkId, value: []const u8 },
    slider_change: struct { link: LinkId, value: f64 },
    svg_click: struct { link: LinkId, x: f32, y: f32 },
};
```

The playground shell and preview share the same input system. Focus must be explicit and deterministic.

---

## 17. Persistence and router

### 17.1 Persistence

```zig
pub const PersistStore = struct {
    pub fn read(self: *PersistStore, key: []const u8) ?[]const u8;
    pub fn write(self: *PersistStore, key: []const u8, value: []const u8) void;
    pub fn deletePrefix(self: *PersistStore, prefix: []const u8) void;
};
```

Native backing:

```text
.zig-cache/raybox-zig/state/<project-name>/
```

Browser backing:

```text
localStorage keys prefixed with raybox-zig:<project-name>:
```

Clear State must delete the current example/project state and rerun the preview.

### 17.2 Router

Implement route host API:

```zig
pub const RouteStore = struct {
    pub fn current(self: *RouteStore) []const u8;
    pub fn goTo(self: *RouteStore, route: []const u8) void;
};
```

Native uses in-memory route state.

Browser updates the URL query/hash without reloading.

`pages.bn` must pass.

---

## 18. Expected runner

Parse upstream `.expected` files. Do not invent a new format.

Support these actions:

```text
assert_contains
assert_not_contains
assert_focused
assert_input_typeable
assert_input_empty
assert_input_placeholder
assert_checkbox_count
assert_checkbox_checked
assert_checkbox_unchecked
assert_button_has_outline

click_button
click_text
click_checkbox
click_button_near_text
dblclick_text
hover_text
focus_input
type
key
wait
run
clear_states
```

For `todo_mvc_physical.expected`, the following must pass:

```text
input starts focused
input is typeable
add first todo
add second todo
complete first todo
active filter hides completed todo
return to all todos
clear completed removes checked todo
toggle dark mode button label
switch to Glass theme
selected Glass button has outline
```

Expected runner algorithm:

```text
1. Load example project.
2. Clear state unless the expected file says otherwise.
3. Run project.
4. Wait for stable/quiescent runtime state.
5. Execute each action against semantic tree.
6. After each sequence, assert expected text or semantic property.
7. Capture failures with:
   - current example name
   - action index
   - semantic tree dump
   - frame stats
   - screenshot path
```

---

## 19. Example implementation order

### Phase 1 — Build + Sokol + Clay shell

Deliver:

```text
native window opens
browser canvas opens
screen clears
basic playground shell renders
Clay root layout works
frame stats render
```

Acceptance:

```bash
zig build run-playground
zig build run-playground -Dtarget=wasm32-emscripten
```

### Phase 2 — Corpus import and registry

Deliver:

```text
import upstream examples
generate corpus_manifest.json
generate example registry Zig file
import todo_mvc_physical full tree
```

Acceptance:

```bash
zig build import-upstream
zig build generate-example-registry
zig build verify-corpus
```

### Phase 3 — Renderer primitives

Deliver:

```text
solid rect
rounded rect
independent borders
scissor clipping
premultiplied alpha
basic image quad
custom command plumbing
```

Acceptance:

```bash
zig build test
zig build run-playground
```

### Phase 4 — Text system

Deliver:

```text
load fonts through sokol_fetch
dynamic glyph atlas
Clay text measurement
text rendering
caret
selection
Czech test string
```

Acceptance:

```bash
zig build verify-text-native
zig build verify-text-web
```

### Phase 5 — Boon runtime bridge

Deliver:

```text
boon-zig dependency wired
single-file project compile/run
DocumentSnapshot/SceneSnapshot adapter
diagnostic display
```

Acceptance examples:

```text
minimal
hello_world
counter
```

### Phase 6 — Core UI elements and events

Deliver:

```text
stripe
block/container
text
paragraph
label
button
checkbox
text_input
LINK routing
focus model
keyboard text editing
hover
double-click
```

Acceptance examples:

```text
counter
counter_hold
shopping_list
todo_mvc
checkbox_test
button_hover_test
button_hover_to_click_test
```

### Phase 7 — Build scripts and multi-file modules

Deliver:

```text
Project VFS
BUILD.bn execution
module resolution
Generated/Assets.bn generation
data URI support
minimal SVG support
```

Acceptance:

```bash
zig build verify-examples-native --filter todo_mvc_physical --build-only
zig build verify-examples-web --filter todo_mvc_physical --build-only
```

### Phase 8 — Physical renderer projection

Deliver:

```text
Scene/new support
Theme/lights support
Theme/geometry support
depth/elevation projection
materials
gloss/shine/metal/glow/transparency
raised buttons
recessed text inputs
recessed checkboxes
bevels
drop shadows
inner shadows
glass highlights
theme switching visuals
```

Acceptance:

```bash
zig build verify-physical-native
zig build verify-physical-web
```

### Phase 9 — Remaining example coverage

Deliver remaining controls and features:

```text
select
slider
svg
svg_circle
router
timers
scroll containers
cells editing behavior
```

Acceptance:

```bash
zig build verify-examples-native
zig build verify-examples-web
```

---

## 20. Final definition of done

The implementation is complete only when all commands pass:

```bash
zig build test
zig build verify-corpus
zig build verify-examples-native
zig build verify-examples-web
zig build verify-physical-native
zig build verify-physical-web
zig build run-playground
zig build run-playground -Dtarget=wasm32-emscripten
```

Required final state:

```text
Every upstream playground example is in corpus_manifest.json.
Every example compiles through boon-zig.
Every example renders through Raybox.
Every example works in a native Sokol window.
Every example works in the browser Sokol canvas.
No preview path uses DOM rendering.
No example is rewritten manually as Zig UI.
No example is silently skipped.
todo_mvc_physical runs as an 8-file project.
todo_mvc_physical BUILD.bn generates assets.
todo_mvc_physical expected script passes.
todo_mvc_physical physical visual verifier passes.
Native and browser screenshots are within tolerance.
```

---

## 21. Reference links for implementors

Use these as implementation references:

```text
Boon upstream:
https://github.com/BoonLang/boon

Rust playground main source:
https://github.com/BoonLang/boon/blob/main/playground/frontend/src/main.rs

todo_mvc_physical directory:
https://github.com/BoonLang/boon/tree/main/playground/frontend/src/examples/todo_mvc_physical

todo_mvc_physical RUN.bn:
https://raw.githubusercontent.com/BoonLang/boon/main/playground/frontend/src/examples/todo_mvc_physical/RUN.bn

todo_mvc_physical BUILD.bn:
https://raw.githubusercontent.com/BoonLang/boon/main/playground/frontend/src/examples/todo_mvc_physical/BUILD.bn

todo_mvc_physical expected:
https://raw.githubusercontent.com/BoonLang/boon/main/playground/frontend/src/examples/todo_mvc_physical/todo_mvc_physical.expected

Physical rendering docs:
https://raw.githubusercontent.com/BoonLang/boon/main/playground/frontend/src/examples/todo_mvc_physical/docs/PHYSICALLY_BASED_RENDERING.md

Sokol Zig:
https://github.com/floooh/sokol-zig

Sokol:
https://github.com/floooh/sokol

Clay:
https://github.com/nicbarker/clay

Clay + Sokol rounded rectangles article:
https://www.siltutorials.com/blog/clay-ui-sokol-gfx-and-rounded-rectangles

FontStash:
https://github.com/memononen/fontstash
```
