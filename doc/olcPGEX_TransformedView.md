# In‑Depth Guide to `olcPGEX_TransformedView.h`

> A practical camera/view helper for PGE: pan, zoom, rotate, and draw in **world space** while PGE still renders in **screen space**.

---

## Why TransformedView?

- A typical PGE app draws in screen pixels. As your game grows, you want a **camera** that can pan/zoom/rotate and still let you write code in **world units** (metres, tiles, whatever).
- `olcPGEX_TransformedView` provides a tiny camera: it converts **world ↔ screen** coordinates, and mirrors common PGE drawing calls with `*World(...)` variants so you can draw directly in world space.

Conceptually it manages a transform with:

- **offset** (camera position)
- **scale/zoom** (2D zoom, usually uniform)
- **rotation** (radians)

---

## Quickstart

```cpp
#define OLC_PGE_APPLICATION
#include "olcPixelGameEngine.h"
#include "extensions/olcPGEX_TransformedView.h"

class Demo : public olc::PixelGameEngine {
public:
    olc::TransformedView view; // the camera

    Demo() { sAppName = "TransformedView Quickstart"; }

    bool OnUserCreate() override {
        // Attach to the engine (some PGEX utilities need PGE pointer)
        view.Initialise(this);
        // Choose sensible defaults
        view.SetWorldScale({1.0f, 1.0f});    // 1 world unit = 1 pixel at zoom 1
        view.SetWorldOffset({0.0f, 0.0f});   // camera at world origin
        view.SetRotation(0.0f);
        return true;
    }

    bool OnUserUpdate(float dt) override {
        // --- Basic camera controls ---
        // Zoom at mouse cursor (wheel): >0 zoom in, <0 zoom out
        float wheel = GetMouseWheel();
        if (wheel != 0.0f) view.ZoomAt({(float)GetMouseX(), (float)GetMouseY()}, 1.0f + wheel * 0.1f);

        // Middle-button drag to pan
        static olc::vf2d lastMouse;
        if (GetMouse(2).bPressed) lastMouse = { (float)GetMouseX(), (float)GetMouseY() };
        if (GetMouse(2).bHeld) {
            olc::vf2d m = { (float)GetMouseX(), (float)GetMouseY() };
            view.PanScreen(m - lastMouse); // pan by screen delta
            lastMouse = m;
        }

        // --- Draw world ---
        Clear(olc::VERY_DARK_BLUE);

        // Draw a 100×100 world grid every 10 units
        for (int y = 0; y <= 100; y += 10)
            view.DrawLineWorld({0, (float)y}, {100, (float)y}, olc::DARK_GREY);
        for (int x = 0; x <= 100; x += 10)
            view.DrawLineWorld({(float)x, 0}, {(float)x, 100}, olc::DARK_GREY);

        // Draw a world rectangle at (25,25) size (50,30)
        view.FillRectWorld({25, 25}, {50, 30}, olc::Pixel(64,128,255,180));

        // Convert mouse to world & mark it
        olc::vf2d w = view.ScreenToWorld({(float)GetMouseX(), (float)GetMouseY()});
        view.FillCircleWorld(w, 1.0f, olc::YELLOW);

        // HUD (screen-space): always top-left of the window
        DrawString(4, 4, "Mouse world: (" + std::to_string(w.x) + ", " + std::to_string(w.y) + ")");
        return true;
    }
};

int main(){ Demo app; if (app.Construct(640, 360, 2, 2) != olc::OK) return 1; app.Start(); }
```

> The key pattern is: keep your game logic in **world** units and call the `*World(...)` drawing helpers. The PGEX handles world→screen conversion internally.

---

## Core API (typical)

### Setup

- `view.Initialise(olc::PixelGameEngine* pge)` – must be called before use.
- `view.SetWorldOffset(olc::vf2d offset)` – camera position in world space (what world point maps to screen origin).
- `view.SetWorldScale(olc::vf2d scale)` – zoom; `{1,1}` is 1 world unit → 1 pixel. Use uniform scale for equal‑axis zoom (e.g., `{z,z}`).
- `view.SetRotation(float radians)` – rotate the view about the screen origin (use `RotateAt` to rotate about a point).

### Convenience camera ops

- `view.Zoom(float factor)` – zoom relative to current origin.
- `view.ZoomAt(olc::vf2d screenPx, float factor)` – zoom **about the cursor** (keeps that world point fixed under the mouse).
- `view.PanWorld(olc::vf2d worldDelta)` – move the camera in world units.
- `view.PanScreen(olc::vf2d screenDelta)` – move camera by a screen‑pixel delta (useful while dragging).
- `view.Rotate(float dAngle)` / `view.RotateAt(olc::vf2d screenPx, float dAngle)` – rotate camera; the `*At` version keeps a chosen screen point stable.

### Coordinate conversion

- `olc::vf2d ScreenToWorld(olc::vf2d screenPx) const`
- `olc::vf2d WorldToScreen(olc::vf2d worldPt)  const`

> These are essential for gameplay input: e.g., “select the tile under the mouse” → `tile = floor(ScreenToWorld(mouse) / tileSize)`.

### Drawing in world space

Most common PGE primitives exist as world‑space wrappers. Typical set:

- `DrawWorld(olc::vf2d p, olc::Pixel col)`
- `DrawLineWorld(olc::vf2d a, olc::vf2d b, olc::Pixel col)`
- `DrawRectWorld(olc::vf2d p, olc::vf2d size, olc::Pixel col)` / `FillRectWorld(...)`
- `DrawCircleWorld(olc::vf2d c, float r, olc::Pixel col)` / `FillCircleWorld(...)`
- `DrawTriangleWorld(...)` / `FillTriangleWorld(...)`
- `DrawStringWorld(olc::vf2d p, const std::string& s, olc::Pixel col, olc::vf2d scale)`
- `DrawSpriteWorld(olc::vf2d p, olc::Sprite*, float scale = 1.0f)` / `DrawPartialSpriteWorld(...)`
- `DrawDecalWorld(olc::vf2d p, olc::Decal*, olc::vf2d scale = {1,1}, olc::Pixel tint = olc::WHITE)` / `DrawPartialDecalWorld(...)`

> Internally these convert inputs via `WorldToScreen(...)` and call the usual PGE draw functions.

---

## Camera Recipes

### 1) Smooth pan/zoom (mouse)

```cpp
// Wheel zoom at cursor
if (float wheel = GetMouseWheel()) view.ZoomAt({(float)GetMouseX(), (float)GetMouseY()}, 1.0f + wheel * 0.1f);

// Middle-drag to pan in screen pixels
static olc::vf2d last;
if (GetMouse(2).bPressed) last = {(float)GetMouseX(), (float)GetMouseY()};
if (GetMouse(2).bHeld)  { auto m = olc::vf2d{(float)GetMouseX(), (float)GetMouseY()}; view.PanScreen(m - last); last = m; }
```

### 2) Follow an entity (world‑space camera)

```cpp
// Keep player centred with lerp smoothing
olc::vf2d target = player.pos - 0.5f * view.ScreenSizeInWorld(); // centre player
view.SetWorldOffset( olc::lerp(view.GetWorldOffset(), target, 5.0f * dt) );
```

### 3) Screen‑space HUD over world

```cpp
// World rendering first via view.*World
view.FillRectWorld({10, 10}, {50, 20}, olc::GREEN);
// HUD afterwards in raw PGE space
DrawString(8, 8, "Ammo: 7/30", olc::YELLOW);
```

### 4) Picking: world under mouse

```cpp
olc::vf2d worldMouse = view.ScreenToWorld({(float)GetMouseX(), (float)GetMouseY()});
int tx = (int)std::floor(worldMouse.x / (float)tileW);
int ty = (int)std::floor(worldMouse.y / (float)tileH);
```

### 5) Rotated view (e.g., map tilt or editor)

```cpp
if (GetKey(olc::Key::Q).bHeld) view.Rotate(+1.0f * dt);
if (GetKey(olc::Key::E).bHeld) view.Rotate(-1.0f * dt);
```

---

## Mental Model

Let **S** be screen space (pixels) and **W** be world space (units). The view applies an **affine transform**:

```
S = T_view(W) = Translate(offset) * Rotate(theta) * Scale(scale) ( * W )
```

`WorldToScreen()` applies `T_view`. `ScreenToWorld()` applies the inverse. Using `ZoomAt()` and `RotateAt()` composes the transform so the chosen point stays fixed.

---

## Tips, Pitfalls, and Performance

- Prefer **uniform** scaling for predictable interactions (`{z,z}`); non‑uniform is fine but rotations will skew shapes.
- Beware of integer truncation: use `vf2d` (floats) for transforms & conversions, cast to `vi2d` only at the final pixel draw.
- For large grids, draw only the **visible** region: compute the world AABB of the screen via `ScreenRectToWorld()` (if available) or map the 4 screen corners with `ScreenToWorld()` and clip.
- When zoomed‑out far, anti‑aliasing isn’t automatic—consider thicker lines or decal‑based geometry for readability.
- World‑space text: prefer `DrawStringDecalWorld` (if provided) for scale‑independent crispness; otherwise scale cautiously.

---

## Minimal Reference (cheat‑sheet)

**Camera state**

- `Initialise(pge)` · `SetWorldOffset(v)` · `GetWorldOffset()`
- `SetWorldScale(v)` · `GetWorldScale()`
- `SetRotation(rad)` · `GetRotation()`

**Camera ops**

- `Zoom(f)` · `ZoomAt(screenPx, f)`
- `PanWorld(dW)` · `PanScreen(dS)`
- `Rotate(dAng)` · `RotateAt(screenPx, dAng)`

**Conversion**

- `WorldToScreen(pW)` · `ScreenToWorld(pS)`

**Drawing (world)**

- `DrawWorld(p, col)` · `DrawLineWorld(a,b,col)`
- `DrawRectWorld(p,size,col)` · `FillRectWorld(...)`
- `DrawCircleWorld(c,r,col)` · `FillCircleWorld(...)`
- `DrawTriangleWorld(a,b,c,col)` · `FillTriangleWorld(...)`
- `DrawStringWorld(p, text, col, scale)` · `DrawStringDecalWorld(...)`
- `DrawSpriteWorld(p, spr, scale)` · `DrawPartialSpriteWorld(...)`
- `DrawDecalWorld(p, dec, scale, tint)` · `DrawPartialDecalWorld(...)`

> Exact names may vary slightly by PGE version; the patterns above match the standard extension’s intent.

---

## Testing Your Understanding

Try these exercises:

1. **Zoom to fit** a world rectangle on screen. Given a world AABB, compute `scale` and `offset` so it exactly fills the viewport with margins.
2. Implement **box selection**: click‑drag a **screen** rectangle, convert its corners to world, and list all world points/entities inside.
3. Add **inertial panning**: accumulate velocity while dragging and decay it after release.

---

## Troubleshooting

- **My mouse selection is offset when rotated** – Always convert `mouse` with `ScreenToWorld()` *after* the latest camera changes each frame.
- **Zoom jumps** – Use `ZoomAt(cursor, factor)` rather than `Zoom()` so the point under the cursor stays put.
- **Drawing is clipped** – Remember that PGE’s rendering target is still the screen sprite; extremely large world coordinates may fall far off screen after transform.

---

### Final note

This PGEX is intentionally minimal. Treat it as a utility to keep your game’s **logic** in world coordinates while preserving PGE’s friendly drawing API. Keep world state free of screen‑pixel assumptions and your code will scale from tiny prototypes to large maps with ease.

