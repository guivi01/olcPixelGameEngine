# In‑Depth Guide to `olcPixelGameEngine.h` (PGE 2.x)

> A practical, developer‑oriented reference to the single‑header Pixel Game Engine by OneLoneCoder.

---

## Table of Contents

1. What is PGE? (1‑minute overview)
2. Quickstart: your first window & pixels
3. Building & linking (Windows, Linux)
4. Anatomy of the header (public API tour)
   - Core types: `olc::Pixel`, vectors (`olc::vi2d`, `olc::vf2d`), `olc::rcode`
   - Images in RAM: `olc::Sprite`
   - Images on the GPU: `olc::Decal`
   - The engine class: `olc::PixelGameEngine`
5. Drawing & text (CPU sprites vs. GPU decals)
6. Input: keyboard & mouse (`olc::Key`, `olc::HWButton`)
7. Draw targets & off‑screen rendering (DIY layers)
8. Pixel blending & custom pixel modes
9. Resource packs (`olc::ResourcePack`) in practice
10. Performance patterns & pitfalls
11. Minimal recipes (copy‑paste)
12. Troubleshooting & FAQs

---

## 1) What is PGE? (1‑minute overview)

PGE is a single‑header C++ engine focused on **fast 2D rendering** and **simple input** with minimal setup. You include one file, implement two virtual methods, and you’re drawing immediately. It ships with a built‑in 8×8 font, primitives (lines/rects/circles/triangles), sprites (CPU‑side images), decals (GPU textures), and a light virtual filesystem (resource packs).

**Philosophy:** work at the pixel level when you want to, but scale to modern, high‑DPI output. Keep boilerplate minimal and make prototypes/game‑jams delightful.

---

## 2) Quickstart: your first window & pixels

```cpp
#define OLC_PGE_APPLICATION
#include "olcPixelGameEngine.h"

class Demo : public olc::PixelGameEngine {
public:
    Demo() { sAppName = "PGE Quickstart"; }

    bool OnUserCreate() override {
        // Load assets here
        return true;
    }

    bool OnUserUpdate(float dt) override {
        Clear(olc::DARK_BLUE);
        DrawLine(0, 0, ScreenWidth()-1, ScreenHeight()-1, olc::WHITE);
        DrawCircle(ScreenWidth()/2, ScreenHeight()/2, 30, olc::YELLOW);
        DrawString(4, 4, "Hello, PGE!");
        return true;
    }
};

int main() {
    Demo app;
    if (app.Construct(320, 180, 4, 4) != olc::OK) return 1; // screen: 320×180, pixel: 4×4
    app.Start();
}
```

**Key ideas:**

- `Construct(screenW, screenH, pixelW, pixelH)` picks a logical resolution and a pixel up‑scale.
- Override `OnUserCreate()` once, and `OnUserUpdate(float dt)` every frame.
- Return `true` to keep the app running (returning `false` exits the loop).

---

## 3) Building & linking (Windows, Linux)

### Windows (Visual Studio)

1. Create a Console App (C++).
2. Add the header `olcPixelGameEngine.h` next to your `.cpp`, and include it as shown above.
3. **Important:** `#define OLC_PGE_APPLICATION` must appear in **exactly one** translation unit before including the header, otherwise you’ll get multiple‑definition link errors.
4. Build (no extra libraries are typically needed on Windows/MSVC).

### Windows (other compilers)

- MinGW/MSYS2 and Code::Blocks are supported. Follow the compiler notes in the header comments for exact flags. (The engine contains per‑compiler notes inline.)

### Linux (g++)

- Install dependencies `libpng`, OpenGL/X11 dev headers via your distro.
- Typical command:
  ```bash
  g++ -std=c++17 -o app main.cpp -lX11 -lGL -lpthread -lpng -lstdc++fs
  ```
- If you use some extensions/utilities that require C++20, compile with `-std=c++20`.

> **Tip:** If you see unresolved `std::filesystem` on older GCC, try `-DFORCE_EXPERIMENTAL_FS` or upgrade your toolchain.

---

## 4) Anatomy of the header (public API tour)

### 4.1 Core types

- `` – RGBA (8‑bit each). Predefined colours like `olc::WHITE`, `olc::BLACK`, `olc::BLANK` (transparent), etc. Used everywhere for drawing.
- **Vectors** – Small 2D math types used by many APIs:
  - `olc::vi2d` (int vector), `olc::vf2d` (float vector). Support arithmetic operators; handy for positions, sizes and UVs.
- `` – Simple status code used by a few APIs; treat `olc::OK` as success.

### 4.2 Images in RAM: `olc::Sprite`

Represents a 2D pixel array held on the CPU. Typical usage:

- **Create empty**: `auto spr = std::make_unique<olc::Sprite>(w, h);`
- **Load from file**: `olc::Sprite* s = new olc::Sprite("image.png");`
- **Read/write pixels**: `spr->GetPixel(x,y)`, `spr->SetPixel(x,y, col)`
- **Utility**: common helpers for sampling/size queries; saving PNG is supported.

> Sprites are drawn **by the CPU** with `DrawSprite`/`DrawPartialSprite` and can be modified per‑pixel (terrain editing, procedural art, etc.).

### 4.3 Images on the GPU: `olc::Decal`

A **decal** wraps a `Sprite` as a GPU texture. Create once:

```cpp
std::unique_ptr<olc::Sprite> spr = std::make_unique<olc::Sprite>("ship.png");
std::unique_ptr<olc::Decal>  dec = std::make_unique<olc::Decal>(spr.get());
```

Render with the **decal drawing** functions (fast, supports rotation/scale/tint/warping): `DrawDecal`, `DrawPartialDecal`, `DrawRotatedDecal`, `DrawWarpedDecal`, `DrawStringDecal`, etc. If you alter the sprite’s pixels later, call `dec->Update()` to reupload to the GPU.

> Decals are **not persistent**: redraw them every frame. They’re ideal for particles, spritesheets, and large batches.

### 4.4 The engine class: `olc::PixelGameEngine`

You derive from this class and override the lifecycle:

- `bool OnUserCreate()` – called once after window creation; load assets here.
- `bool OnUserUpdate(float fElapsedTime)` – called each frame; do input, update, draw.
- `bool OnUserDestroy()` – optional cleanup hook.

Core environment & draw‑target functions:

- `int32_t ScreenWidth() / ScreenHeight()` – logical screen size in **engine pixels**.
- `void SetDrawTarget(olc::Sprite* target)` – divert all draw calls to a sprite (nullptr reverts to the screen). Great for off‑screen rendering and compositing.
- `int32_t GetDrawTargetWidth()/Height()` and `olc::Sprite* GetDrawTarget()`.
- `void SetSubPixelOffset(float ox, float oy)` – experimental fractional pixel offset (for smoother motion at high DPI).

Blending & pixel modes:

- `void SetPixelBlend(float a)` – global alpha multiplier for subsequent draws.
- `void SetPixelMode(olc::Pixel::Mode m)` with modes:
  - `NORMAL` (fast, default)
  - `MASK` (skip pixels with alpha ≠ 255)
  - `ALPHA` (full per‑pixel alpha blending)
  - `CUSTOM` (use a lambda to combine source/dest pixels)
- `auto GetPixelMode()` – query current mode.

Primitive & sprite text drawing (CPU):

- `Clear(olc::Pixel p)`
- Pixels & shapes: `Draw`, `DrawLine`, `DrawRect/FillRect`, `DrawCircle/FillCircle`, `DrawTriangle/FillTriangle`
- Sprites: `DrawSprite(x,y,spr, scale)`, `DrawPartialSprite(x,y,spr, ox,oy,w,h, scale)`
- Text: `DrawString(x,y, std::string, col, scale)` – uses built‑in 8×8 font

Decal drawing (GPU):

- `DrawDecal(pos, decal, scale, tint)`
- `DrawPartialDecal(pos, decal, uv_topleft, uv_size, scale, tint)`
- `DrawRotatedDecal(pos, decal, angle, origin, scale, tint)`
- `DrawWarpedDecal(decal, quad[4], tint)`
- `DrawStringDecal(pos, text, tint, scale)` (fast text)

---

## 5) Drawing & text (CPU sprites vs. GPU decals)

- **CPU path (sprites & primitives)**: easy per‑pixel edits; best for small amounts of geometry or dynamic image generation.
- **GPU path (decals)**: high throughput; supports rotation/scale/tint/warping; perfect for thousands of sprites/particles/text.

**Rule of thumb:** draw your world with sprites/primitives if you’re editing pixels, but prefer **decals** for everything else (especially repeated imagery/tiles/particles). Use `DrawStringDecal` for lots of text.

---

## 6) Input: keyboard & mouse

- **Keys:** `GetKey(olc::Key::A)` returns an `olc::HWButton` with:
  - `bPressed` (went down this frame)
  - `bReleased` (went up this frame)
  - `bHeld` (is currently down)
- **Mouse:** `GetMouse(0)` for button state, `GetMouseX()/GetMouseY()` for position.
- **Focus:** `IsFocused()` helps ignore input when your window loses focus.

Common pattern:

```cpp
if (IsFocused()) {
    auto left = GetKey(olc::Key::LEFT);
    if (left.bPressed) {/* tap */}
    if (left.bHeld)    {/* hold */}
}
```

---

## 7) Draw targets & off‑screen rendering (DIY layers)

PGE’s draw calls go to a **current draw target** (the screen by default). You can redirect them to a sprite:

```cpp
olc::Sprite layer0(ScreenWidth(), ScreenHeight());
SetDrawTarget(&layer0);
Clear(olc::BLANK);                  // draw background into layer0
// ... draw terrain, etc.
SetDrawTarget(nullptr);             // back to screen
DrawSprite(0, 0, &layer0);          // composite to screen
```

This pattern gives you **multiple layers** that you can scroll, tint, or scale by drawing the layer sprites as **decals**:

```cpp
olc::Decal decLayer0(&layer0);
DrawDecal({offx, offy}, &decLayer0, {scaleX, scaleY}, tint);
```

> Many projects never need anything more complicated than this off‑screen‑then‑composite flow.

---

## 8) Pixel blending & custom pixel modes

Toggle blending per pass:

```cpp
SetPixelMode(olc::Pixel::MASK);   // ignore partially‑transparent pixels
// draw foliage with cutout alpha …
SetPixelMode(olc::Pixel::NORMAL); // restore
```

Global alpha and fully custom blend:

```cpp
SetPixelBlend(0.5f); // half opacity for all subsequent draws

SetPixelMode([](int x, int y, const olc::Pixel& src, const olc::Pixel& dst){
    // simple additive blend
    auto clamp = [](int v){ return std::min(255, std::max(0, v)); };
    return olc::Pixel(clamp(dst.r + src.r), clamp(dst.g + src.g), clamp(dst.b + src.b), 255);
});
// … draw things with custom blending …
SetPixelMode(olc::Pixel::NORMAL);
```

---

## 9) Resource packs (`olc::ResourcePack`) in practice

Package assets into a single file and load them transparently at runtime.

**Create a pack (one‑off tool or a debug key in your app):**

```cpp
olc::ResourcePack pack;
pack.AddFile("gfx/ship.png");
pack.AddFile("levels/level1.txt");
pack.SavePack("assets.pge");
```

**Load & use in your game:**

```cpp
olc::ResourcePack pack;
pack.LoadPack("assets.pge");

// Many PGE classes accept an optional ResourcePack*
auto spr = std::make_unique<olc::Sprite>("gfx/ship.png", &pack); // loads from pack
```

Why use packs?

- Easier distribution (one data file)
- Lightweight obfuscation of assets
- Uniform loading code (work with files or packs via the same constructor)

---

## 10) Performance patterns & pitfalls

**Do:**

- Prefer **decals** for lots of sprites/particles/text.
- Batch conceptual work per‑frame: clear once, draw opaque first (NORMAL), then alpha stuff.
- Reuse `Sprite`/`Decal` objects; avoid re‑creating them every frame.
- Use off‑screen sprites as **layers** and composite with `DrawDecal` when you need scrolling/tinting.

**Avoid:**

- Thousands of per‑pixel `Draw()` calls every frame—use shapes, sprites, or decals.
- Leaving `ALPHA` pixel mode on for everything; switch back to `NORMAL`/`MASK` when not needed.
- Defining `OLC_PGE_APPLICATION` in more than one `.cpp` (linker errors).

---

## 11) Minimal recipes (copy‑paste)

### A) Load a spritesheet, draw tiles

```cpp
std::unique_ptr<olc::Sprite> tiles = std::make_unique<olc::Sprite>("gfx/tiles.png");

// draw tile (tx,ty) where each tile is 16×16 in the sheet
olc::vi2d tileSize = {16,16};
olc::vi2d tileUV   = {tx * tileSize.x, ty * tileSize.y};
DrawPartialSprite(screenPos, tiles.get(), tileUV, tileSize);
```

### B) Particle using decals

```cpp
struct Particle { olc::vf2d p,v; float a=0; olc::Pixel tint=olc::WHITE; };
std::vector<Particle> ps;

// update
for (auto& e : ps) { e.v.y += 30.0f*dt; e.p += e.v*dt; e.a += 2.0f*dt; e.tint.a = uint8_t(std::max(0.f, e.tint.a - 90*dt)); }

// draw
for (auto& e : ps)
    DrawRotatedDecal(e.p, dec.get(), e.a, {4,4}, {1,1}, e.tint);
```

### C) Fast HUD text

```cpp
DrawStringDecal({8,8}, "Health: 42", olc::YELLOW, {1.0f, 1.0f});
```

### D) Off‑screen blur step (toy example)

```cpp
olc::Sprite ping(ScreenWidth(), ScreenHeight());
SetDrawTarget(&ping);  Clear(olc::BLANK);  /* … draw world … */
SetDrawTarget(nullptr);

olc::Decal decPing(&ping);
// draw slightly scaled to fake a cheap blur/shadow
DrawDecal({1,1}, &decPing, {1.0f,1.0f}, olc::Pixel(0,0,0,120));
DrawDecal({0,0}, &decPing);
```

---

## 12) Troubleshooting & FAQs

- **Black window on Linux** – Missing libs. Rebuild with `-lX11 -lGL -lpthread -lpng` (and `-lstdc++fs` on older GCC).
- **Nothing draws / text invisible** – You might be drawing to an off‑screen target you never blit back, or have `MASK` pixel mode active while your art needs `ALPHA`.
- **Input feels sticky** – Use `bPressed/bReleased` for events, `bHeld` for continuous movement. Don’t poll when the window is unfocused; guard with `IsFocused()`.
- **Sprites are slow** – For many moving images use **decals** instead of CPU sprite drawing.
- **Multiple definition link errors** – Ensure `OLC_PGE_APPLICATION` is defined in exactly one `.cpp`.

---

## Appendix A – Quick reference (selected)

**Engine lifecycle**

- `olc::rcode Construct(uint32_t sw, uint32_t sh, uint32_t pw, uint32_t ph)`
- `void Start()`
- `virtual bool OnUserCreate()` / `virtual bool OnUserUpdate(float dt)` / `virtual bool OnUserDestroy()`

**Environment & target**

- `int32_t ScreenWidth()/ScreenHeight()`
- `void SetDrawTarget(olc::Sprite* s)` / `olc::Sprite* GetDrawTarget()`
- `int32_t GetDrawTargetWidth()/Height()`
- `void SetSubPixelOffset(float ox, float oy)`

**Blending**

- `void SetPixelBlend(float f)`
- `void SetPixelMode(olc::Pixel::Mode m)` / `auto GetPixelMode()`
- `CUSTOM` mode lambda: `(int x, int y, const olc::Pixel& src, const olc::Pixel& dst) -> olc::Pixel`

**Primitives & text**

- `Clear(col)`
- `Draw(x,y,col)`
- `DrawLine(x1,y1,x2,y2,col)`
- `DrawRect/FillRect(x,y,w,h,col)`
- `DrawCircle/FillCircle(x,y,r,col)`
- `DrawTriangle/FillTriangle(x1,y1,x2,y2,x3,y3,col)`
- `DrawString(x,y,text,col,scale)`

**Sprites (CPU)**

- `DrawSprite(x,y,spr, scale)`
- `DrawPartialSprite(x,y,spr, ox,oy,w,h, scale)`

**Decals (GPU)**

- `DrawDecal(pos, decal, scale, tint)`
- `DrawPartialDecal(pos, decal, srcUV, srcSize, scale, tint)`
- `DrawRotatedDecal(pos, decal, angle, origin, scale, tint)`
- `DrawWarpedDecal(decal, quad4, tint)`
- `DrawStringDecal(pos, text, tint, scale)`

**Input**

- `olc::HWButton GetKey(olc::Key k)` → `{ bool bPressed, bReleased, bHeld; }`
- `olc::HWButton GetMouse(uint32_t button)`
- `int32_t GetMouseX()/GetMouseY()`
- `bool IsFocused()`

---

### Final notes

- This guide focuses on the **public surface** of the header and common patterns. The header itself contains further platform/compiler notes and a few advanced hooks.
- PGE has optional extensions (sound, controller input, transforms, etc.) you can bolt on later; keep your core loop clean and data‑driven so adding them is painless.

Happy building! 🎮

